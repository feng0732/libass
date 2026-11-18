# libass 中矢量裁剪与模糊描边的合成顺序

> 分析对象：本仓库 `task/30` 分支，commit `21b5a54`（`RELEASEVERSION = 0.17.5`）。
> 方法：以源码静态分析为主；`c_blend_bitmaps.c` 的乘法/补集混合公式另用脚本做了逐值验证。
> 环境缺少 freetype/fribidi/harfbuzz 开发头文件，**未构建、未运行**渲染器；下文所有结论均给出可核对的文件:行号。
> 未使用容器。

---

## 0. 结论摘要

1. **裁剪发生在（合成 + 模糊 + 描边裁剪 + 阴影复制）之后**。每个事件的顺序是：
   `字形位图 → 按 run 加性合并 → 生成描边位图 → 高斯/盒式模糊 → 填充与描边互减 → 复制出阴影 → 矩形裁剪 → 矢量裁剪`。
   即 `\blur`/`\be` 先在**未裁剪**的合成位图上生效，`\clip`/`\iclip` 最后在屏幕空间裁掉（或乘掉）已经模糊好的 alpha。
2. **矢量裁剪是一层 alpha 遮罩乘法**：常规 `\clip` 用 `mul_bitmaps`（`a·m/255`），`\iclip` 用 `imul_bitmaps`（`a·(255−m)/255`）。它只改覆盖度（bitmap 字节），**永不改 `ASS_Image.color`**；最终可见度 = 遮罩后的覆盖度 × (255 − 颜色 A 字节)/255（ASS 的 A 是“透明度”）。
3. **缓存键不需要“遮罩/描边”标志位**：语义区分在 outline 层由 `OutlineHashKey.type` 完成（`OUTLINE_DRAWING` vs `OUTLINE_BORDER`），在 bitmap 层由“被引用的 `OutlineHashValue *` + 量化变换”共同完成；裁剪参数不进任何缓存键，因此同一段文字不同 `\clip` 可共享同一 composite，裁剪只在缓存外对 `ASS_Image` 做几何/alpha 后处理。
4. 由此引出三条核心不变量：**缓存值只含与用途无关的覆盖度**、**缓存值不可原地修改（矢量裁剪必须写时复制）**、**裁剪对每个事件恰好应用一次且保持 `ASS_Image` 列表顺序/类型**；第 3 节在此基础上再展开生命周期、几何别名、变化检测、OOM 降级等，并给出 5 条“不可交换”的顺序约束。

---

## 1. 渲染管线与顺序

### 1.1 单个事件的调用链

```
ass_render_frame                      libass/ass_render.c:3370
└─ ass_render_event                   libass/ass_render.c:2798
   ├─ parse_events                    libass/ass_render.c:2051
   │  └─ \clip / \iclip 解析          libass/ass_parse.c:405-423, 729-748
   │     · 4 参数 → clip_x0..y1 + clip_mode(0/1)
   │     · 1-2 参数 → clip_drawing_text / clip_drawing_scale / clip_drawing_mode
   ├─ measure_text / align_lines …（与裁剪无关的排版、碰撞盒计算）
   ├─ render_and_combine_glyphs       libass/ass_render.c:2447
   │  └─ ass_cache_get(composite)     libass/ass_render.c:2587-2601
   │     └─ ass_composite_construct   libass/ass_render.c:2643
   │        · 加性合并 fill / outline libass/ass_render.c:2670-2720
   │        · 模糊                     libass/ass_render.c:2722-2727
   │        · fill/outline 互减         libass/ass_render.c:2729-2730, 2751-2752
   │        · 复制/shift 出阴影         libass/ass_render.c:2732-2749
   ├─ 固定 clip 坐标到屏幕空间         libass/ass_render.c:2949-2979
   ├─ render_text                     libass/ass_render.c:820
   │  ├─ shadow  → render_glyph       libass/ass_render.c:829-835
   │  ├─ outline → render_glyph       libass/ass_render.c:838-850
   │  ├─ fill    → render_glyph       libass/ass_render.c:852-878
   │  │           （矩形裁剪在 render_glyph / render_glyph_i 内）
   │  └─ blend_vector_clip            libass/ass_render.c:881
   └─ return event_images->imgs
之后：fix_collisions → shift_event（再裁上下边界）→ 丢弃 A==0xFF 的图元  libass/ass_render.c:3397-3425
```

关键点：**`render_glyph` 的矩形裁剪和 `blend_vector_clip` 的矢量裁剪都运行在 `ass_composite_construct` 之后**。`ass_composite_construct` 里已经完成了模糊（2725-2727）与阴影复制（2732-2749），而 `render_glyph` 只是按 `state->clip_x0..` 在已模糊的合成位图上“取子矩形”（443-495），`blend_vector_clip` 只是在已成形的 `ASS_Image` 列表上逐张乘遮罩（733-813）。

### 1.2 合成阶段（缓存内，逐 run）

`ass_composite_construct`（2643）对一个“样式 run”的若干字形位图做：

1. **加性合并**：`add_bitmaps` 为 `dst = min(dst + src, 255)`（`libass/c/c_blend_bitmaps.c:34-51`），fill（`v->bm`）与 outline（`v->bm_o`）分别合并（2691-2693、2716-2718）。重叠笔画因此是“并集/饱和相加”，不是 source-over。
2. **模糊**（2722-2727）：
   - outline 位图 `v->bm_o` **总是**调用 `ass_synth_blur`；
   - fill 位图 `v->bm` **仅当**没有描边、或 `BorderStyle=3`（不透明盒）时才模糊。这是代码写死的分支（2725-2726），不是运行时判断：有普通描边时，描边本身已被模糊并向内外扩张，fill 又是画在描边之上的，于是省略 fill 的模糊以省时间；因此**带 `\bord` 的字的 fill 边缘保持锐利**，这是可观察的既有行为。
   - `ass_synth_blur` 先高斯后盒式（`libass/ass_bitmap.c:68-101`）；高斯会把位图向四周扩大并平移 `bm->left/top`（`libass/ass_blur.c:234-235`），所以合成位图的**物理范围大于字形 bbox**（再加上 `ass_be_padding` 的 pad，`libass/ass_render.c:2633-2640, 2670-2678`）。
3. **填充/描边互减**：`ass_fix_outline` 做 `o = (o > g) ? o − g/2 : 0`（`libass/ass_bitmap.c:205-224`），避免描边与填充在交界处叠加过暗；仅在 `FILL_IN_*` 标志未置位时执行（2729-2730、2751-2752）。
4. **阴影**：阴影**不是单独再做一次模糊**，而是复制已模糊好的 outline（有描边时）或 fill（无描边时）再按量化后的 `filter.shadow` 做整像素 + 亚像素平移（2732-2749，平移/量化见 2518-2524 与 `ass_shift_bitmap`）。因此 `\blur` 对字形与阴影形状的影响自动一致。

> 顺序结论：**模糊发生在阴影克隆之前、裁剪之前**。图像一旦模糊完，后续所有步骤（包括裁剪）都只是读取/裁剪这份覆盖度，不会再重新模糊。

### 1.3 裁剪阶段（缓存外，输出 `ASS_Image`）

**(a) 矩形 `\clip`**：`render_glyph`（424-496）。
- `dst_x += bm->left; dst_y += bm->top; brk -= dst_x`（443-445）把位图坐标换成屏幕坐标；
- 把 `clip_x0..y1` 夹到 `0..render_priv->width/height`（447-451）；
- 求 bitmap 与 clip 矩形的交集 `b_x0..b_y1`（457-468），无交集则**不产生任何 `ASS_Image`**（470-471）；
- 有交集时用 `my_draw_bitmap(bm->buffer + b_y0*stride + b_x0, …)` 直接指向缓存位图的子矩形（476-494）——**零拷贝、只改几何**。

**(b) 矩形 `\iclip`**：`render_glyph_i`（319-410）。把裁剪矩形之外的区域拆成最多 4 个**互不重叠**的矩形（349-370），再裁到屏幕坐标（372-378），对每个有效矩形发射图元（381-407），并可按 `\kf` 在 `brk` 处再左右切分。

**(c) 矢量 `\clip`/`\iclip`**：`blend_vector_clip`（700-814）。
- 以 `OutlineHashKey{type=OUTLINE_DRAWING, text=clip_drawing_text}` 取 outline（707-726），再以 `quantize_transform` 得到的 `BitmapHashKey` 取** bitmap 缓存里的遮罩位图**（728）；
- 遮罩变换只含 `screen_scale_x/y / 2^(clip_drawing_scale−1)` 与左边距/上边距（711-720）；它是**普通栅格化位图，不经过任何模糊**（`ass_bitmap_construct` 只调用 `ass_outline_to_bitmap`，1549-1574）；
- 遍历 `ASS_Image` 列表（733），算出图像与遮罩 bbox 的交集（754-763）；
  - `\iclip`（766-782）：`memcpy` 原图后，仅在交集内做 `imul_bitmaps`（`a·(255−m)/255`）；不相交则 `continue`（即整张保留，符合“遮罩 bbox 之外 m≡0”）；
  - `\clip`（784-807）：不相交则把尺寸清零（787）；相交则分配**新缓冲**、只对交集做 `mul_bitmaps`（`a·m/255`），并把 `dst_x/dst_y/w/h/stride` 改写成交集矩形（792-806）；
  - 最后把 `ASS_Image` 的 buffer 换成新缓冲，并 `ass_cache_dec_ref(priv->source)`（809-812）——从“引用缓存”转为“自持内存”。

**(c′) 矩形裁剪与矢量裁剪可以并存，且顺序固定为“先矩形、后矢量”**

`\clip` 的 4 参数分支无条件写 `clip_x0..y1`（`ass_parse.c:729-744`），而矢量分支只在尚未设置 `clip_drawing_text` 时生效（745-748，先出现的矢量 clip 胜出）。因此同一事件里矩形与矢量裁剪**可以同时激活**，最终可见区域为“矩形裁剪结果”再按矢量模式取交（常规 `\clip`）或取补（`\iclip`）——四种组合都存在，因为 `clip_mode` 与 `clip_drawing_mode` 是互相独立的两个字段（`ass_render.h:243, 258`）。两者都没有幂等性保证（3.6），所以顺序必须固定：先做零拷贝的矩形子矩形选取（`render_glyph`），再做需要分配/乘法的矢量裁剪（`blend_vector_clip`）。另外 `init_render_context` 只重置 `clip_mode`（`ass_render.c:1132-1136`），**不重置 `clip_drawing_mode`**；这一点之所以无害，是因为 `clip_drawing_text` 在 `free_render_context` 中每个事件结束时清零（`ass_render.c:1160-1161`，调用点 3012-3013），而 `blend_vector_clip` 首先检查文本是否为空（702），且解析器保证“设置文本的同时必然设置 mode”（`ass_parse.c:420-423`、`745-747`）。换言之，“文本为空 ⇒ 矢量裁剪整体不生效”是掩盖陈旧 `clip_drawing_mode` 的唯一屏障，属于必须保持的隐式约定。

**(d) 事后裁剪**：`fix_collisions` 之后 `shift_event` 会在屏幕上下边界处减少 `h` 并移动 `bitmap` 指针（3144-3168）。这是纯粹的几何裁剪，同样不重算 alpha。

### 1.4 透明度如何合成

| 阶段 | 运算 | 位置 |
|---|---|---|
| run 内字形叠加 | `dst = min(dst + src, 255)`（饱和加性） | `c/c_blend_bitmaps.c:34-51`，调用 `ass_render.c:2691-2718` |
| 描边 vs 填充 | `o = max(o − g/2, 0)` | `ass_bitmap.c:205-224` |
| 模糊 | 高斯（σ 由 `\blur` 量化）+ `\be` 盒式 | `ass_bitmap.c:68-101`、`ass_blur.c:173-239` |
| 阴影 | 克隆已模糊位图 + 整/亚像素平移（不做第二次模糊） | `ass_render.c:2732-2749` |
| 矢量 `\clip` | `c' = (c·m + 255) >> 8` ≈ `round(c·m/255)` | `c/c_blend_bitmaps.c:71-90` |
| 矢量 `\iclip` | `c' = (c·(255−m) + 255) >> 8` ≈ `round(c·(255−m)/255)` | `c/c_blend_bitmaps.c:53-69` |
| 矩形 clip | 只切矩形，不改字节 | `ass_render.c:457-494`、`349-378` |
| 与颜色相乘 | 由调用方合成：`color` 的 A 字节是 ASS 约定（`0x00` = 不透明，`0xFF` = 全透明，见 `ass_render.c:3411-3425` 的过滤），最终 alpha ∝ 覆盖度 × (255−A)/255；libass 自身只输出覆盖度 | `ass.h:54-86`、`ass_render.c:3411-3425` |

补充事实（已用脚本逐值验证）：
- `m=255` 时 `mul` 是**逐字节恒等**；`m=0` 时输出 0；`m=0` 时 `imul` 也是逐字节恒等。因此“完全落在遮罩内部/外部”时 alpha 的字节与裁剪前**完全一致**（不含新分配带来的指针差异）。
- 与理想 `round(c·m/255)` 相比，8 位实现的误差 ≤ 1 LSB。具体而言：`mul` 对 `m=255` 恒等、对 `m=254` 仍可能把 `c=128` 降到 127；`imul` 对 `m=0` 恒等、对 `m=1` 可把 `c=128` 降到 127。`mul` 与 `imul` 对同一 `m` 的补集输出之和最多偏离原值 1 LSB；它们不严格互补，但在 `m=0`/`m=255` 处精确。
- 裁剪**只作用于覆盖度**：`blend_vector_clip` 与 `render_glyph*` 都不修改 `cur->color`。所以“A 通道为 0xFF（全透明）即丢弃”的过滤（3411-3425）与裁剪是否把覆盖度乘成 0 无关：被矢量裁剪清零的图元仍然会留在列表里（尺寸为 0，或 alpha 全 0），除非事件颜色本身就是全透明。

### 1.5 缓存键：如何区分遮罩、描边、与裁剪

三层缓存，键的定义在 `libass/ass_cache_template.h`，比较函数由宏生成（`ass_cache.c:45-50`）。

**(A) outline 缓存键 —— 语义/几何来源，靠 `type` 区分**

```c
// libass/ass_cache.h:58-71
typedef struct outline_hash_key {
    enum { OUTLINE_GLYPH, OUTLINE_DRAWING, OUTLINE_BORDER, OUTLINE_BOX } type;
    union { GlyphHashKey glyph; DrawingHashKey drawing; BorderHashKey border; } u;
} OutlineHashKey;
```
- `outline_compare` **先比较 `type`**（`ass_cache.c:196-201`），所以：
  - 矢量裁剪遮罩 → `OUTLINE_DRAWING{text}`（`ass_render.c:707-709`）；
  - 描边 → `OUTLINE_BORDER{outline, scale_ord_x/y, border}`（`ass_render.h` 见 `ass_cache_template.h:105-114`，构造于 `ass_render.c:1462-1518`）；
  - 普通字形 → `OUTLINE_GLYPH{font,size,face,glyph,bold,italic,flags}`；
  - `BorderStyle=3` 的盒子 → `OUTLINE_BOX`（`ass_render.c:1419-1423`）。
  同一段 drawing 文本既可用于 `\p` 正文又可用于 `\clip`，两者**合法的共享**同一个 outline 条目，因为轮廓几何确实相同。
- `OUTLINE_BOX` 刻意不比较载荷：hash 直接返回常量、compare 返回 true（`ass_cache.c:191-192, 209-210`）。所有盒子共享一个“单位方盒”轮廓，实际尺寸完全编码在该 outline 之后的**量化变换**里，因此不会与其他盒子混淆。

**(B) bitmap 缓存键 —— 遮罩/描边/字形的光栅结果，靠“outline 指针 + 量化变换”区分**

```c
// libass/ass_cache_template.h:60-69
START(bitmap, bitmap_hash_key)
    GENERIC(OutlineHashValue *, outline)
    VECTOR(offset)      // 亚像素量化偏移
    VECTOR(matrix_x)    // 量化后的仿射/透视矩阵
    VECTOR(matrix_y)
    VECTOR(matrix_z)
END(BitmapHashKey)
```
- 没有“这是遮罩/这是描边”的字段。`outline` 是按**指针相等**比较的（宏展开为 `a->outline == b->outline`，见 `ass_cache_template.h:22-30` 的 `GENERIC` 与 `:63`），因此区分掩码/描边的是 **outline 条目的身份**，而非任何字符串或标志位：裁剪遮罩与描边位图之所以不会互相命中，是因为它们的 `outline` 指针来自 `type` 不同的 outline 条目；而**描边**的几何差异（不同 `\bord`、不同缩放）已经被 `BorderHashKey` 吸收成不同的 outline 条目（`ass_cache_template.h:105-114`；`ass_render.c:1466-1518` 计算 `scale_ord_x/y` 与 stroke 半径并量化）。
- 唯一“可能重合”的情况：同一 drawing 文本既作文本又作 `\clip`，且量化变换也完全相同。此时两张位图的**光栅字节本就应当相同**，共享是安全且期望的——因为 bitmap 缓存值是纯覆盖度，不含颜色、不含裁剪模式（`Bitmap` 定义见 `ass_bitmap.h:30-35`）。
- 由此得到一条不变量：**`BitmapHashKey` 必须完整描述“如何栅格化一个轮廓”，但不需要描述“栅格化结果将被用于什么”**。裁剪遮罩的变换在 `ass_render.c:711-725` 构造，与字形/描边共用同一 `quantize_transform` 完成量化。

**(C) composite 缓存键 —— 模糊/阴影参数 + 字形位图集合**

```c
// libass/ass_cache.h:84-88
typedef struct { FilterDesc filter; size_t bitmap_count; BitmapRef *bitmaps; } CompositeHashKey;
// FilterDesc: flags(描边样式/填充策略), be, blur_x, blur_y, shadow   —— ass_cache_template.h:117-123
// BitmapRef: bm, bm_o, pos, pos_o                                    —— ass_cache_template.h:127-132
```
- 模糊参数在**这一层**，不在 bitmap 层：同一字形位图可被“模糊的 run”与“不模糊的 run”共享，模糊结果单独缓存。
- `FilterDesc.flags` 编码了 `BORDER_STYLE_3 / NONZERO_BORDER / NONZERO_SHADOW / FILL_IN_*`（`ass_cache.h:73-79`，`ass_render.c:2465-2487`），这决定了 1.2 中“是否模糊 fill、是否互减”的分支。
- `shadow` 偏移按模糊半径派生出的掩码量化（`ass_render.c:2512-2522`），所以偏移在量化粒度内相同的 run 共享 composite。

**(D) 裁剪参数为什么完全不在键里**

`clip_x0..y1`、`clip_drawing_text`、`clip_drawing_mode` 全部是 `RenderContext` 的字段（`ass_render.h:241-258`），在 `render_glyph*`/`blend_vector_clip` 中**只读使用**，从不参与任何 `ass_cache_get` 的键（对照 `ass_render.c:1407-1414, 1531-1536, 2587-2591, 721-728`）。后果：

- 相同文本、不同 `\clip` 的两个事件共享同一 composite 位图；分辨它们的成本只有“取子矩形/乘遮罩”。这也是矩形裁剪能逐帧动画而不产生缓存项的原因（`Changelog:43` “Improve quality of animated rectangular clips” 只在坐标取整层面优化）。
- 矢量裁剪遮罩本身**是**缓存项，键是 `OUTLINE_DRAWING(text)` + 变换；同一 `\clip(path,scale)` 在不同帧上不会重复栅格化。

---

## 2. 完全相交、部分相交、空裁剪区域的可观察结果

以下以“**保留区域 R**”定义三种情形，注意 R 是裁剪后仍然可见的那部分，而不是裁剪矩形本身：

- **完全相交**：R 覆盖整张图元（裁剪把所有像素都留下）；
- **部分相交**：R 与图元有非空交集但不覆盖整张图元；
- **空裁剪区域**：R 与图元不相交（裁剪把所有像素都去掉）。

所有 alpha 都是 `ASS_Image` 的覆盖度字节（未乘颜色的 A），矩形裁剪不会引入任何羽化。

### 2.1 矩形 `\clip(x0,y0,x1,y1)`（`render_glyph`，424-496）

| 情形 | 可观察结果 |
|---|---|
| **完全相交**（字形/阴影完全在裁剪矩形内） | `b_x0=0,b_y0=0,b_x1=bm->w,b_y1=bm->h`，`ASS_Image` 的 `w/h/stride/dst_*` 与裁剪前**相同**，`bitmap` 指针仍指向 composite 内同一位置；**alpha 字节逐字节不变**（无新缓冲、无乘法）。 |
| **部分相交** | 图像被裁成子矩形：`w=b_x1−b_x0`、`h=b_y1−b_y0`，`dst_x/dst_y` 平移到子矩形左上（476-494）；`bitmap` 指向 composite 内部偏移。**无重采样、无补齐**：边界处 alpha 就是模糊后的原值突然截断——不会为了“截断后能量守恒”而提亮邻近像元，也不会在裁剪边产生模糊过渡。这正是“先模糊后裁剪”的可观察特征：模糊核在边界外侧已经“看到”了将被裁掉的内容，裁掉后不会重算。 |
| **空裁剪区域**（与位图无交集） | `b_y0>=b_y1 || b_x0>=b_x1` → `return tail`，**不生成任何 `ASS_Image`**（470-471）。事件其余图元照常输出；若整事件都被裁掉，`render_text` 返回 `NULL`（880 的 `*tail = 0`）。注意 `event_images->width/height/top/left` 仍按排版度量计算（2996-3003），**碰撞占位不受裁剪影响**。 |

补充：
- `\clip` 与 `\kf` 叠加时，一个裁剪矩形会在 `brk` 处再切左右两半，两半像素互不重叠（388-405）。
- 裁剪坐标先被夹到 `0..render_priv->width/height`（448-451），再与位图求交，因此完全在屏幕外的裁剪得到与“空裁剪”相同的结果。

### 2.2 矩形 `\iclip(x0,y0,x1,y1)`（`render_glyph_i`，319-410）

| 情形 | 可观察结果 |
|---|---|
| **完全相交**（保留区域 = 整个位图，即裁剪矩形不与位图相交） | 补集矩形会把整张位图完整铺满且互不重叠（相邻块共享边、不共享像素）：裁剪矩形在左/右侧之外时恰好 1 块整图；在上下方之外但横向覆盖位图内部时，按左/中/右切成 2–3 个竖直条带。所有块内 alpha 与裁剪前**逐字节相同**。 |
| **部分相交** | 位图沿裁剪矩形的边被切成最多 4 块（上/下/左/右），每块 `dst_*` 与尺寸独立计算，块之间**互不重叠**；块内 alpha 保持模糊后的原值，裁剪边是硬边。 |
| **空裁剪区域**（位图完全处于裁剪矩形内） | 四个矩形全部无效 → 不生成任何图元（355-370 的 `i` 不增加；384-385 再次过滤）。事件无可见内容，但排版与碰撞盒不变。 |

### 2.3 矢量 `\clip(path)`（常规遮罩乘法，`blend_vector_clip` 784-807）

| 情形 | 可观察结果 |
|---|---|
| **完全相交**（保留区域覆盖整张图元：图像完全落在遮罩**已绘制**的区域内，`m≡255`） | 输出图像**几何重写为遮罩 bbox 交集**（792-806）：`dst_x/dst_y` 可能平移，`w/h` 变为交集尺寸，`stride` 变为新对齐值。alpha 在 `m=255` 处**逐字节恒等**（数值已验证）；`m=254` 的 AA 内边最多低 1 LSB。`bitmap` 指针必然换成**新缓冲**（793），与原 composite 不再是同一地址。 |
| **部分相交** | 输出图像收缩到“图像 ∩ 遮罩 bbox”，交集内逐像素 `a·m/255`，交集外**直接不存在**（新图像尺寸就是交集，不保留透明边）。遮罩 bbox 内但未绘制的区域 `m=0` ⇒ alpha 归 0（可能输出一张全零 alpha 的矩形）。裁剪**边界本身**由遮罩位图的栅格化抗锯齿决定（约 1 像素），**不随 `\blur` 变宽**——遮罩位图从未被模糊（728 只查 bitmap 缓存，`ass_bitmap_construct` 只做光栅化，`ass_render.c:1549-1574`）；像素是否仍在边界内侧变软，只取决于图像自身的模糊衰减是否还留在交集内。 |
| **空裁剪区域**（保留区域与图像不相交：图像落在遮罩 bbox 之外） | `!h || !w` → `cur->w = cur->h = cur->stride = 0`（787），**图元仍留在列表里**但尺寸为 0；按 `ass.h:58` “w/h can be zero, in this case the bitmap should not be rendered”，消费端跳过。`bitmap` 指针与 `color` 不变，`priv->source` 的缓存引用也**未释放**（787 的 `continue` 在 809 之前）。注意“图像在 bbox 内、但遮罩路径完全没画到该处”走的是**部分相交**分支且结果同为不可见（alpha 全 0），几何上却保留交集尺寸。 |

### 2.4 矢量 `\iclip(path)`（补集乘法，`blend_vector_clip` 766-782）

| 情形 | 可观察结果 |
|---|---|
| **完全相交**（保留区域 = 补集覆盖整张图元：图像落在遮罩 bbox 之外，或 bbox 内但 `m≡0`） | 交集为空时 `continue`（768-771）→ 图像**完全不变**；bbox 内但 `m≡0` 时 `imul` 逐字节恒等（数值已验证）→ 也**完全不变**。语义正确：补集处处为 1。 |
| **部分相交** | 图像尺寸**不变**，仅交集内按 `a·(255−m)/255` 压暗；交集外通过 `memcpy` 原样保留（779-782）；`m=0` 处 `imul` 逐字节恒等，因此“遮罩 bbox 内但未绘制”的像元**完全不变**。裁剪边为遮罩栅格化的 1 像素 AA。 |
| **空裁剪区域**（保留区域与图像不相交：图像完全落在遮罩覆盖区域内，`m≡255`） | 交集内 `a·0/255 = 0`：图像**不可见**；但仍是原 `dst_*`/`w/h` 的图元，alpha 全 0（不会被裁剪阶段删除，除非 `color` 的 A 字节本身就是 0xFF，3411-3425）。 |

### 2.5 空 / 非法矢量裁剪路径

| 情形 | 可观察结果 |
|---|---|
| **可解析但为空的 drawing**（如只有 `m` 的路径 `\clip(m 0 0)`） | `ass_drawing_parse` 返回 true 但 `n_points=0`（`ass_drawing.c:314-397`）；outline 条目的 `cbox` 被归一为 `(0,0)-(0,0)`、`valid=true`（`ass_render.c:1314-1319`）；随后 `ass_outline_to_bitmap` 因 bbox 非法返回 false → `memset` 出空 `Bitmap`（`ass_render.c:1567-1568`，`ass_bitmap.c:162-163`），`buffer=NULL`、`w=h=0`（缓存仍返回这个零尺寸值，`ass_cache_get` 不返回 NULL）。遮罩 `w=h=0`：**常规 `\clip` 把每张图清零（等价“全部在裁剪区外”）**，**`\iclip` 全部保留**。这与 `Changelog:159`（0.16.0）“Treat empty clip outlines as if everything was outside the clip”一致。 |
| **解析失败 / 轮廓无效**（`!key.outline->valid`）或量化变换失败 | `blend_vector_clip` 在 724-726 **整体提前返回**：常规与反向模式都**完全不裁剪**（等价于没有该 `\clip`）。 |
| **tag 参数不足** | 解析器不设置 `clip_drawing_text`（`ass_parse.c:216-227`；如 `\clip()` 使 `nargs==0`，既不满足 4 参数分支也无法进入 `parse_vector_clip`），`blend_vector_clip` 在 702 直接返回 → **完全不裁剪**（不是“裁成空”）。 |

### 2.6 汇总

| 模式 | 完全相交（R 覆盖整张图元） | 部分相交 | 空裁剪（R 与图元不相交） |
|---|---|---|---|
| 矩形 `\clip` | 几何与字节均不变 | 子矩形，字节不变、硬边 | 不产生图元 |
| 矩形 `\iclip` | 全图保留（1–3 块，字节不变） | 切成 2–4 块，硬边，字节不变 | 不产生图元 |
| 矢量 `\clip` | 收缩到交集；`m=255` 处字节恒等（但 `bitmap` 换新缓冲） | 收缩到交集并乘 mask；交集外消失 | `w=h=stride=0` 的图元保留在列表（若只是 bbox 内未绘制，则走“部分相交”且 alpha 全 0） |
| 矢量 `\iclip` | 图像**完全不变**（不相交，或 bbox 内 `m≡0`） | 尺寸不变，交集内乘补集，外侧字节不变 | alpha 全 0（图元尺寸不变） |

---

## 3. 合成顺序与缓存复用冲突时必须保留的不变量

这一节把“顺序”与“复用”之间的张力逐条落到代码依据上。每条的格式为：**不变量 → 为什么冲突 → 实现依据**。

### 3.1 缓存值只含“与用途无关”的覆盖度

- **不变量**：任何进入缓存的位图（outline/bitmap/composite）不得编码颜色、裁剪模式、alpha 混合结果等“使用方式”信息；否则同一几何在不同用途下无法共享，1.5(D) 的“裁剪不进键”也会失效。
- **冲突**：裁剪是逐事件、逐帧变化的，若把裁剪结果放进 composite，键就必须包含 `clip_*`，动画裁剪会击穿缓存。
- **依据**：`Bitmap` 只有 `left/top/w/h/stride/buffer`（`ass_bitmap.h:30-35`）；`CompositeHashKey` 只有 filter + bitmap refs（`ass_cache.h:84-88`）；颜色在 `my_draw_bitmap` 时才由参数 `color` 带入（`ass_render.c:210-236`，`uint32_t color` 原样存入 `img->result.color`）。

### 3.2 已发布到缓存的位图不可原地修改

- **不变量**：裁剪（尤其矢量裁剪的乘法）必须**写时复制**；绝不能就地改写 `composite_cache` 里的 `v->bm/v->bm_o/v->bm_s`。
- **冲突**：`render_glyph` 会把同一 composite 的**多个子矩形**以多个 `ASS_Image` 暴露（388-405、476-494），且多个事件可共享同一 composite（键相同即同一条目）。若矢量裁剪原地乘遮罩，会污染同帧其它事件，并永久破坏缓存值。
- **依据**：`blend_vector_clip` 在两种模式下都 `ass_aligned_alloc` 新缓冲（774、793），然后 `priv->buffer = cur->bitmap = nbuffer` 并 `ass_cache_dec_ref(priv->source)`（809-812）。`render_glyph*` 只调整 `bitmap/dst_*/w/h/stride`，不写字节。允许原地写的只有 `ass_composite_construct` 内**尚未发布**（`ass_cache_get` 先 `construct` 再挂入哈希桶，`ass_cache.c:411-431`）的 `v->bm*`——这就是模糊、`ass_fix_outline`、阴影平移可以原地做的前提。

### 3.3 生命周期：引用缓存或自持内存，二选一

- **不变量**：`ASS_Image` 的 `bitmap` 若指向缓存内存，必须持有 `CompositeHashValue` 引用；一旦裁剪产生新缓冲，必须立刻放弃缓存引用并由 `ASS_ImagePriv` 自己释放。
- **冲突**：`ass_cache_get` 的返回值“仅保证到下一次 `ass_cache_cut`”（`ass_cache.c:381-384`），而 `ass_cache_cut` 在每帧开始时执行（`check_cache_limits`，`ass_render.c:3021-3026`，调用点 3095）。事件渲染期间的帧内分割/裁剪若不拷贝，就会用到可能已被驱逐的内存。
- **依据**：`my_draw_bitmap` 保存 `source` 并 `ass_cache_inc_ref`（230-232）；`ass_free_image` 只释放 `priv->buffer` 与 `source` 引用（3350-3358）；**未裁剪**图像的 `priv->buffer = source ? NULL : bitmap`（232）——即“图中指向缓存就不释放”；**裁剪后**新缓冲由 `priv->buffer` 接管（810），`source` 被置 NULL（812）。`ass_cache_dec_ref` 在计数归零时真正析构（`ass_cache.c:467-484`）。

### 3.4 几何别名：`w/h/stride/dst_*` 可缩小，但字节必须来自同一缓冲

- **不变量**：`render_glyph` 的“子矩形”是**指针运算 + 尺寸缩小**，不是重排内存；因此裁剪不得改变行的语义（stride 不变、行起点偏移为 `Δy·stride + Δx`），且最后一行未必铺满 stride（`ass.h:59-60` 的保证是 `stride*(h−1)+w`）。
- **冲突**：若为了让裁剪后的图像“整齐”而重新分配/重排，会破坏与 composite 的别名关系，使 3.3 的引用计数失去意义，也会让 `ass_image_compare` 的指针比较得出错误结论。
- **依据**：`render_glyph` 以 `bm->buffer + b_y0*bm->stride + b_x0` 构造（476、487），`stride` 原样传入 `my_draw_bitmap`（476-478）；`render_glyph_i` 同样（389-391、399-401）。矢量常规裁剪则相反：**分配新缓冲并重写 stride**（792-806），因为此时已经必须拷贝。

### 3.5 `ASS_Image` 列表顺序与 `type` 必须保持

- **不变量**：shadow → outline → fill 的输出顺序，以及每张图的 `type`（`IMAGE_TYPE_SHADOW/OUTLINE/CHARACTER`），是消费端正确 source-over 的前提；裁剪（矩形拆分、矢量乘法）不得重排、合并或改变类型。
- **冲突**：`render_glyph_i` 为了表示补集会复制同一个 `type` 到 2–4 个图元；矢量裁剪会替换 buffer/尺寸；这些都很容易顺手改变顺序或复用一个 `ASS_Image`。
- **依据**：`ass.h:54-58` “These images have to be rendered in-order for the correct screen composition”；`render_text` 三段循环固定顺序（829-835、838-850、852-878）并逐个设置 `img->type = type`（393、403、480、491）；`blend_vector_clip` 只原地修改当前节点（733 的 `for (cur = head; cur; cur = cur->next)`），不插删节点。

### 3.6 裁剪对每个事件恰好应用一次（幂等性的前提）

- **不变量**：矩形裁剪与矢量裁剪各只执行一次；同一遮罩不得对同一覆盖度重复相乘。
- **冲突**：`mul` 不是幂等运算（连乘两次等于 `a·m²/255²`）；若把矩形裁剪结果再交给矢量裁剪“顺带”处理，或事件合并后重复调用 `blend_vector_clip`，遮罩边缘会被平方压暗。
- **依据**：`render_text` 在输出完三类图元后**只调用一次** `blend_vector_clip(state, head)`（880-881）；`render_glyph` 的矩形裁剪在每次 `my_draw_bitmap` 之前一次性算好 `b_x0..b_y1`（457-468），不会对同一图元二次裁剪。

### 3.7 布局、碰撞与事件度量不以裁剪结果为输入

- **不变量**：裁剪只影响可见图元，不影响排版、行高、`event_images->width/height/top/left` 与碰撞占位。
- **冲突**：裁剪后图像可能被清成 0 尺寸（787）或不生成（470），若用图像尺寸反向推导事件尺寸，会导致字幕在裁剪边缘“跳位”或与其它事件重叠。
- **依据**：`event_images->top/height/left/width` 由 `text_info` 排版度量与 `border_top/bottom/x` 计算（2996-3003），发生在 `render_text` 之前；`fix_collisions` 只使用这些度量和 `detect_collisions`（3216、3255），`shift_event` 的上下裁剪发生在其后（3144-3168）。`Changelog:410`（0.12.0）“Reset clipping mode on every rectangle clip” 与 `Changelog:159` 也说明这些语义被当作兼容性约定维护。

### 3.8 变化检测只能按“身份”比较，矢量裁剪必然每帧“内容变化”

- **不变量**（现状）：`detect_change` 的判定只能依赖 `ASS_Image` 的字段与 `bitmap` 指针，不比较像素内容。
- **冲突**：矢量裁剪每帧都分配新缓冲（774、793），因此 `i1->bitmap != i2->bitmap` 恒成立；即使画面完全静止，含矢量 `\clip` 的事件也会报 `detect_change = 2`。这是“裁剪在缓存外逐帧物化”的直接代价，无法在不比较像素的前提下消除。
- **依据**：`ass_image_compare` 对 `w/h/stride/color/bitmap` 做比较，`bitmap` 不等即返回 2（`ass_render.c:3286-3303`）；`ass_detect_change` 逐节点调用它（3310-3340）。对矩形裁剪则没有这个问题：图元仍指向缓存位图，指针稳定（除非该缓存项在帧间被 `ass_cache_cut` 驱逐后重新构造）。
- 这是**保守方向的误报**（宁可报“内容变了”也不漏报）；`Changelog:418`（0.11.2）“Fix change detection with vector clips” 说明矢量裁剪下的变化检测历史上正是 bug 高发点。新增裁剪/合成优化时，应把 3.8 当作**已知行为**维护：不要为“省一次分配”而让矢量裁剪复用上一帧的缓冲，也不要改写缓存内的共享缓冲。

### 3.9 OOM 时允许“部分裁剪”，但不得破坏所有权

- **不变量**：任何分配失败都必须以“该图元保持原样/停止后续处理”的方式降级，且不得出现悬空 `source` 或重复释放。
- **现状**：`blend_vector_clip` 在 774-776/793-795 分配失败时 `break`，**列表中剩余图元不再裁剪**——同一帧可能出现“前半被裁、后半未裁”的中间状态；`render_glyph` 里 `my_draw_bitmap` 失败则是 `break`/`return tail`（392、402、479、490），同样只保证已输出的部分正确。
- **依据**：失败分支都没有触碰 `priv->source`（只在成功路径 811-812 释放引用），因此不会产生 use-after-free；但这是可用性层面的 best-effort，不应被当作“裁剪是原子操作”的保证。

### 3.10 与顺序直接相关的“不可交换”关系

以下顺序若交换，会同时破坏 3.1/3.2/3.6：

1. **模糊必须在裁剪之前**：否则 `\blur` 的核会以裁剪边界为边界条件（零填充/镜像），在裁剪边产生额外的亮度衰减或羽化；而当前实现中边界内 alpha 与“无裁剪时”完全一致（2.1/2.3 的“字节不变”），这正是缓存的 composite 可以跨裁剪复用、且只依赖 filter+glyph 集合的原因（1.5(C)）。反过来说，**必须先模糊再裁剪也意味着裁剪边不守恒“总能量”**：模糊溢到裁剪区外的部分被直接丢弃，而不是补回边界内。
2. **描边/填充互减必须在模糊之后**：`ass_fix_outline`（`ass_bitmap.c:205-224`）读取的是模糊后的 `bm`/`bm_o`（2729-2730）；若先互减再模糊，`g/2` 的减法会被模糊抹开，描边会变淡，且该结果无法由 `FilterDesc` 唯一决定（互减还依赖 `FILL_IN_*` 标志，后者已在 filter flags 中，2479-2487）。
3. **阴影必须在模糊之后克隆**：2732-2749 直接复制已模糊位图；否则需要对阴影再做一次同参数模糊，缓存键将不得不包含“是否阴影”这一用途信息，违背 3.1。
4. **屏幕空间裁剪必须在排版之后**：`clip_x0..y1` 在 2949-2979 用 `x2scr_pos_scaled`/`y2scr_pos` 换算到设备像素；矢量遮罩的变换（711-720）也用 `screen_scale`/margin 构造。若在 PlayRes 空间裁剪，则 `\clip` 的取整、margin 与 PAR 行为都与 VSFilter 不一致（相关兼容性记录见 `Changelog:410,544`），且遮罩缓存键必须携带整套 `settings`（目前它只携带由 `screen_scale_x/y` 与 `clip_drawing_scale` 量化的矩阵）。
5. **设备参数变化必须清空三层缓存**：`ass_reconfigure` 在分辨率、边距、PAR、存储尺寸等变化时 `ass_cache_empty` 掉 composite/bitmap/outline（`ass_render_api.c:29-37`，触发点 63/75/96/115/123/131/144/158/173/183）。矢量遮罩位图的键只含量化后的变换（`ass_render.c:711-725`），若不清空，帧尺寸一变旧遮罩就会以新尺寸被复用——这条不变量与 `Changelog` 0.17.2 节“Updating selective overrides now forces a cache clear to avoid issues with outdated caches”同源。

---

## 4. 建议的验证用例（可用于后续回归/像素比对）

> 当前环境无法构建（缺 freetype/fribidi/harfbuzz 开发头文件），以下用例供有构建环境时执行。

1. **顺序判定：模糊外溢被裁掉而非反弹**
   `{\blur20\clip(400,300,800,600)}A`：观察裁剪边外侧无 alpha；裁剪边内侧像素与去掉 `\clip` 时的同一像素**逐字节相同**（可用 `ASS_Image.bitmap` 直接对比）。
2. **矩形 vs 矢量边界**
   同一几何分别用 `\clip(...)` 与 `\clip(m ... l ...)` 裁同一段带 `\blur` 的字：矩形边应无过渡（硬边），矢量边应只有遮罩栅格化带来的约 1 像素 AA 过渡，且**该过渡的宽度不随 `\blur` 增大**（变宽的只有字形自身的模糊外溢）。
3. **反向裁剪的补集完备性**
   `\iclip(...)` 使裁剪矩形横跨字形：输出应为 ≤4 个互不重叠的 `ASS_Image`，其 `dst_x/dst_y` 覆盖的区域并集等于“字形区域减裁剪矩形”。
4. **空路径的两极性**
   `{\clip(m 0 0)}A`（可解析、空轮廓）：常规模式应**无任何可见图元**，`\iclip` 版本应**完全可见**。对照 `{\clip()}A`：参数不足，解析器根本不设置裁剪，应**与不加 tag 完全一致**（既不是“全裁掉”，也不是“裁成空”）。
5. **完全包含的字节恒等**
   把矢量 `\clip` 画成覆盖整字的实心大矩形：输出图元 alpha 应与无裁剪时逐字节相同（仅 `bitmap` 指针不同、`dst/w/h` 可能因 bbox 收缩而变）。反向用 `\iclip` 画一个完全避开字形的图形：输出应与无裁剪时逐字节相同，且 `bitmap` 指针也保持不变。
6. **缓存共享**
   两个事件文本/样式相同、仅 `\clip` 不同：在一次 `ass_render_frame` 之后检查 `CompositeHashValue` 只有一个条目（可通过调试日志或给 `ass_cache_get` 打计数），证明裁剪未进键。
7. **变化检测（记录现状）**
   仅含矢量 `\clip` 的静止事件连续两帧：预期 `detect_change == 2`（3.8），而仅含矩形 `\clip` 的静止事件应为 0。

---

## 附录：关键位置索引

| 主题 | 位置 |
|---|---|
| 事件主流程 / 裁剪坐标落到屏幕 | `libass/ass_render.c:2798-3016`，`2949-2979` |
| 矩形 `\clip` 交点与发射 | `libass/ass_render.c:424-496` |
| 矩形 `\iclip` 四矩形补集 | `libass/ass_render.c:319-410` |
| 矢量裁剪主体 | `libass/ass_render.c:700-814`，调用点 `880-881` |
| 矢量遮罩键构造（drawing + 变换） | `libass/ass_render.c:707-728` |
| composite 内：合并/模糊/互减/阴影 | `libass/ass_render.c:2643-2757` |
| 模糊参数量化与恢复 | `libass/ass_render.c:2398-2444`，`2512-2524` |
| 字形/描边 bitmap 键 | `libass/ass_render.c:1407-1414`，`1418-1541` |
| `FilterDesc` / flags | `libass/ass_render.c:2465-2487`，`libass/ass_cache.h:73-79` |
| 缓存键模板 | `libass/ass_cache_template.h:60-69, 105-123, 127-132` |
| outline 键的 type 分派与比较 | `libass/ass_cache.c:181-212`，`libass/ass_cache.h:58-71` |
| 缓存生命周期/驱逐 | `libass/ass_cache.c:381-431, 458-516`，`libass/ass_render.c:3021-3026, 3095` |
| 混合公式（加/乘/补乘） | `libass/c/c_blend_bitmaps.c:34-51, 53-69, 71-90` |
| `ass_fix_outline` | `libass/ass_bitmap.c:205-224` |
| 模糊实现（含 bbox 扩张） | `libass/ass_blur.c:173-239`，`libass/ass_bitmap.c:68-101` |
| `ASS_Image` 生命周期/顺序/尺寸语义 | `libass/ass.h:52-86`，`libass/ass_render.c:210-236, 3350-3358` |
| 变化检测 | `libass/ass_render.c:3286-3340` |
| 全透明图元过滤 | `libass/ass_render.c:3411-3425` |
| 碰撞平移后的上下裁剪 | `libass/ass_render.c:3144-3168` |
| 兼容性历史 | `Changelog:43`（0.17.2），`Changelog:135`（0.17.0），`Changelog:159`（0.16.0），`Changelog:410, 418, 544` |
