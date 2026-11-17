# libass 矢量裁剪与模糊描边的合成顺序分析

> 分析对象：本仓库 libass（当前 `task/30` 分支，HEAD `21b5a54`）。
> 关键文件：`libass/ass_render.c`、`libass/ass_bitmap.c`、`libass/ass_blur.c`、
> `libass/ass_cache.h`、`libass/ass_cache_template.h`、`libass/ass_cache.c`、
> `libass/c/c_blend_bitmaps.c`、`libass/c/rasterizer_template.h`、`libass/ass_parse.c`。

---

## 0. 结论速览（TL;DR）

1. **裁剪发生在模糊之后。** 模糊（`\blur` 高斯、`\be` 盒式）和描边（`\xbord/\ybord`）在
   **合成缓存（composite cache）构造阶段**就已经烘焙进字幕位图；矢量裁剪
   （`\clip(...)`/`\iclip(...)` 绘图形式）是在整行文字的 `ASS_Image` 链表生成**之后**，
   由 `blend_vector_clip()` 对成品位图做一次逐像素**覆盖率相乘**。裁剪遮罩本身只做
   光栅化，**从不被模糊**。
2. 透明度不是"加法叠加"，而是 **alpha 通道的乘法门控**：
   - 正向裁剪 `dst = glyph × clip`（`mul_bitmaps`）；
   - 反向裁剪 `dst = glyph × (255 − clip)`（`imul_bitmaps`）。
   字幕自身的颜色/透明度（`ASS_Image.color` 的 alpha 字节）与位图覆盖率是两个独立量，
   裁剪只改位图覆盖率，不修改 `color`。
3. 缓存分三层，键的设计从结构上保证了"遮罩"和"描边/字形"不会串用：
   - outline cache（轮廓）→ bitmap cache（单个轮廓的光栅化位图）→ composite cache
     （多字形合并 + 模糊 + 描边 + 阴影）。裁剪遮罩走前两层且不携带任何 `FilterDesc`；
     模糊描边走第三层，键里带完整 `FilterDesc`。
4. 必须保留的渲染不变量：**缓存不可变（immutable / COW）**、**裁剪在模糊之后的固定
   相位顺序**、**乘法门控语义（含反向裁剪的补集）**、**几何/量化决定论**、**矩形裁剪
   与矢量裁剪两条互斥路径**。这些不变量正是当前缓存键与 `blend_vector_clip()` 内存分配
   方式所强制保证的。

---

## 1. 裁剪相对模糊的相位、透明度合成、缓存键区分

### 1.1 完整管线与相位顺序

单条事件的渲染入口是 `ass_render_event()` → `render_and_combine_glyphs()`
（`ass_render.c:2447`）→ `render_text()`（`ass_render.c:820`）。相位如下：

| 相位 | 位置 | 做了什么 | 是否涉及模糊/裁剪 |
|---|---|---|---|
| ① 轮廓解析 | `ass_outline_construct()`，`ass_render.c:1234` | 字形/绘图/描边轮廓（`ASS_Outline`） | 描边轮廓在此生成（`OUTLINE_BORDER` 调 `ass_outline_stroke`，`ass_render.c:1281`） |
| ② 单轮廓光栅化 | `ass_bitmap_construct()` → `ass_outline_to_bitmap()`，`ass_render.c:1549`、`ass_bitmap.c:149` | 轮廓 → 8-bit 覆盖率位图（0..255，抗锯齿） | **无模糊、无裁剪** |
| ③ 多字形合并 + 滤镜 | `ass_composite_construct()`，`ass_render.c:2643` | 多个 `BitmapRef` 用 `add_bitmaps` 加法合并到一张大位图，然后 `ass_synth_blur()` 做模糊、`ass_fix_outline()` 修正描边、生成阴影 | **模糊与描边效果在此烘焙完成** |
| ④ 生成图像链表 | `render_text()`，`ass_render.c:820`，逐影子/描边/正文调 `render_glyph()` | 产出有序的 `ASS_Image` 链表（影子→描边→正文） | 矩形 `\clip` 在 `render_glyph()` 内做几何切边 |
| ⑤ 矢量裁剪 | `render_text()` 末尾 `blend_vector_clip(state, head)`，`ass_render.c:881`，实现于 `ass_render.c:700` | 用缓存的裁剪遮罩位图，对链表中**每一张成品图**逐像素相乘 | **矢量裁剪在此，晚于相位③的模糊** |

因此对一张最终像素而言，数学顺序是：

```
coverage_out(x,y) = Blur( Rasterize(glyph 或 stroke 轮廓) )          // 相位②③
final(x,y)        = coverage_out(x,y) * Mask_clip(x,y) / 255        // 相位⑤
```

**裁剪在模糊之后**——这与 VSFilter 的语义一致：模糊会把字形覆盖率扩散到轮廓之外
（高斯模糊甚至会扩边，见 1.4），而裁剪是对扩散之后的最终覆盖率再做几何门控，所以
模糊产生的"光晕"跨进裁剪边界时会被遮罩硬切，而不是被遮罩一起模糊。

### 1.2 模糊与描边具体在哪里、以什么顺序烘焙

`ass_composite_construct()`（`ass_render.c:2643`）内部：

1. 先按合并包围盒分配 `v->bm`（正文）和 `v->bm_o`（描边），用
   `engine.add_bitmaps()`（饱和加法，`c_blend_bitmaps.c:34`）把各子位图累加进去
   （`ass_render.c:2691`、`2716`）。`\be` 需要的额外边距由 `ass_be_padding()`
   （`ass_render.c:2633`）预留。
2. 然后调用 `ass_synth_blur()`（`ass_render.c:2726-2727`，实现于
   `ass_bitmap.c:68`）：
   - 先 `ass_gaussian_blur()`（`\blur`，`ass_blur.c:173`），再做 `\be` 盒式模糊
     （`engine->be_blur`，多趟）。即 **高斯模糊在盒式模糊之前**；
   - 正文位图在 `FILTER_NONZERO_BORDER` 且非 opaque box 时可跳过，描边位图始终模糊。
3. `ass_fix_outline()`（`ass_bitmap.c:205`）用正文覆盖率削减描边，避免重叠处变厚。
4. 阴影 `v->bm_s` 由正文/描边复制 + `ass_shift_bitmap()` 亚像素移位得到
   （`ass_render.c:2732-2749`）。

模糊半径在进入缓存键前先被 `quantize_blur()`（`ass_render.c:2398`）量化为整数
（`FilterDesc.blur_x/blur_y`），使用时再由 `restore_blur()`（`ass_render.c:2439`）
还原为方差 σ²。量化保证了"视觉上无差别"的半径映射到同一个缓存项。

### 1.3 矢量裁剪的透明度合成：两个乘法原语

裁剪遮罩 `clip_bm` 与字形位图同为 8-bit 覆盖率（0=完全透明，255=完全不透明）。
`blend_vector_clip()`（`ass_render.c:700`）先算出两图的相交矩形
（`left/top/right/bottom`，`ass_render.c:754-763`），然后分两种模式：

**正向 `\clip`（`clip_drawing_mode == 0`，`ass_render.c:783-807`）**

```c
render_priv->engine.mul_bitmaps(nbuffer, ns,
        abuffer + atop * as + aleft, as,    // 字幕位图（已含模糊/描边）
        bbuffer + btop * bs + bleft, bs,    // 裁剪遮罩
        w, h);
```
对应 C 实现（`c_blend_bitmaps.c:71`）：

```c
dst[x] = (src1[x] * src2[x] + 255) >> 8;     // ≈ a·b/255
```
即**覆盖率相乘**：遮罩外（clip=0）结果为 0（全透明），遮罩内（clip=255）保留原覆盖率，
边界抗锯齿带得到平滑的乘积过渡。同时图像被重新裁剪到相交矩形，更新
`dst_x/dst_y/w/h/stride`（`ass_render.c:802-806`）。

**反向 `\iclip`（`clip_drawing_mode == 1`，`ass_render.c:766-782`）**

```c
memcpy(nbuffer, abuffer, ((ah - 1) * as) + aw);   // 先复制整张字幕位图
render_priv->engine.imul_bitmaps(nbuffer + atop*as + aleft, as,
                                 bbuffer + btop*bs + bleft, bs, w, h);
```
对应（`c_blend_bitmaps.c:53`）：

```c
dst[x] = (dst[x] * (255 - src[x]) + 255) >> 8;    // ≈ a·(255-b)/255
```
即乘以**遮罩的补集**：遮罩覆盖区被挖空，遮罩外保留。注意反向模式保留原图全尺寸
（只在相交矩形内改写），因为挖空后图像其余部分仍然有效。

**与颜色 alpha 的关系。** `ASS_Image.color`（RGBA，`ass.h:71`）携带样式/`\alpha`/
`\fad` 等颜色与透明度，位图 `bitmap` 只存覆盖率。`blend_vector_clip()` 自始至终
不碰 `color`，只改覆盖率缓冲。最终由播放器（API 消费方）按
`dst = color·(coverage/255)` 进行 alpha 混合。因此可观察到的最终不透明度是
`样式alpha × 覆盖率乘积`，裁剪只作用在后者这一因子上。完全透明（颜色 alpha=0xFF）
的图像在 `ass_render_frame()` 汇总时被直接丢弃（`ass_render.c:3416-3419`）。

### 1.4 裁剪遮罩本身：光栅化但不模糊，且独立扩边

裁剪绘图经 `parse_vector_clip()`（`ass_parse.c:214`）保存为
`state->clip_drawing_text`（含 `clip_drawing_scale`）。在 `blend_vector_clip()` 中：

```c
ol_key.type = OUTLINE_DRAWING;                         // ass_render.c:708
ol_key.u.drawing.text = state->clip_drawing_text;
...
key.outline = ass_cache_get(...->outline_cache, &ol_key, ...);   // 703..723
Bitmap *clip_bm = ass_cache_get(...->bitmap_cache, &key, state); // 728
```

它复用字形的 outline→bitmap 两级缓存，但走的是 `ass_bitmap_construct()`
（`ass_render.c:1549`）这条**只光栅化、不经过 composite、不调用 `ass_synth_blur`**
的路径，所以遮罩是一张锐利（仅光栅化抗锯齿）的覆盖率图，没有任何高斯/盒式模糊。

反观被裁剪的字幕位图，`ass_gaussian_blur()` 会**主动扩大位图尺寸**并调整原点
（`ass_blur.c:230-235`，`bm->left -= ...; bm->top -= ...`），模糊核在新边界外按
零（透明）延拓；`\be` 则靠 `ass_be_padding()` 预留边距。这意味着模糊光晕可以延伸到
原始字形包围盒之外，而它与裁剪遮罩的相交判定（`ass_render.c:754-757`）用的是扩边后
的真实位图矩形，光晕能否被裁到完全由两者矩形与覆盖率的重叠决定。

### 1.5 缓存键如何区分"遮罩"与"描边/字形"

三层缓存的键定义在 `ass_cache_template.h` 与 `ass_cache.h`：

**(a) OutlineHashKey（`ass_cache.h:59-71`）——用 `type` 枚举区分轮廓种类**

```c
enum { OUTLINE_GLYPH, OUTLINE_DRAWING, OUTLINE_BORDER, OUTLINE_BOX } type;
union { GlyphHashKey glyph; DrawingHashKey drawing; BorderHashKey border; } u;
```
- 裁剪遮罩：`OUTLINE_DRAWING`，键内容只有绘图文本 `DrawingHashKey{ ASS_StringView text }`
  （`ass_cache_template.h:101-103`）。
- 描边轮廓：`OUTLINE_BORDER`，键为 `BorderHashKey{ outline*, scale_ord_x, scale_ord_y,
  border }`（`ass_cache_template.h:107-114`），与绘图键在 `type` 上就不可能相等；
  `outline_compare()` 首先比较 `ak->type != bk->type`（`ass_cache.c:200`）。

**(b) BitmapHashKey（`ass_cache_template.h:62-69`）——轮廓指针 + 量化变换**

```c
typedef struct {
    OutlineHashValue *outline;   // 指向上一级缓存项，类型已在轮廓键里区分
    ASS_Vector offset, matrix_x, matrix_y, matrix_z;  // quantize_transform 量化后的矩阵
} BitmapHashKey;
```
裁剪遮罩（`ass_render.c:722-728`）和字形/描边位图（`ass_render.c:1407-1412`、
`1531-1536`）都用它，但因为 `outline` 指向的是**不同 `type` 的轮廓缓存项**（指针值
不同、且其键不同），天然落到不同条目。该键里**没有任何模糊参数**——位图缓存只负责
"轮廓在某量化变换下的锐利光栅化结果"，模糊不属于这一层。

**(c) CompositeHashKey（`ass_cache.h:84-88`）——模糊/描边/阴影参数只在这里出现**

```c
typedef struct {
    FilterDesc filter;                       // flags, be, blur_x, blur_y, shadow
    size_t bitmap_count;
    BitmapRef *bitmaps;                      // {bm, bm_o, pos, pos_o} 数组
} CompositeHashKey;
```
`FilterDesc`（`ass_cache_template.h:117-123`）含 `flags / be / blur_x / blur_y /
shadow`，`BitmapRef`（同文件 127-132）引用 (b) 层产出的正文与描边位图及其定位。
**只有 composite 缓存携带模糊与描边参数**；裁剪遮罩从不进入这一层。于是：

- 同一段文字、不同 `\blur/\be/\bord/\shad` → `FilterDesc` 不同 → composite 键不同；
- 同一张锐利字形位图被多个 composite 项共享（经 `ass_cache_inc_ref/dec_ref` 引用计数）；
- 同一份裁剪绘图在多帧/多事件间复用同一个遮罩位图（Changelog 0.9.10 "Cache vector
  clip masks"），与文字内容完全解耦。

### 1.6 矩形 `\clip` 与矢量 `\clip` 是两条互斥路径

- 矩形四参数形式只设置 `clip_x0..clip_y1` 与 `clip_mode`（`ass_parse.c:729-748`、
  `405-423`），在 `render_glyph()`（正向，`ass_render.c:447-471`）或
  `render_glyph_i()`（反向，`ass_render.c:319-410`，把图像拆成裁剪框外最多 4 个矩形）
  内做纯几何切边，不产生遮罩位图、不做像素乘法。
- 矢量形式设置 `clip_drawing_text` 与 `clip_drawing_mode`；解析时若矢量裁剪已存在则
  忽略后续矢量标签（`ass_parse.c:745`、`421` 的 `!state->clip_drawing_text.str` 条件）。
- `render_glyph()` 入口用 `if (state->clip_mode) return render_glyph_i(...)`
  （`ass_render.c:430`）分流；`blend_vector_clip()` 在无矢量绘图时立即返回
  （`ass_render.c:702-703`）。两条路径不会同时对同一像素生效。

---

## 2. 完全相交、部分相交、空裁剪区域的可观察结果

记字幕位图覆盖率为 `a ∈ [0,255]`（已含模糊/描边），遮罩覆盖率为 `b ∈ [0,255]`。
正向结果 `f = a·b/255`，反向结果 `g = a·(255−b)/255`。

### 2.1 完全相交（字幕位图矩形 ⊆ 遮罩矩形，且遮罩内部 b≈255）

- **正向 `\clip`**：相交矩形等于字幕位图整框，`mul_bitmaps` 后 `f ≈ a·255/255 = a`，
  视觉上与无裁剪**完全一致**，模糊光晕、描边、阴影全部保留。实现上仍会新分配一块
  缓冲并逐像素乘一遍（`ass_render.c:792-801`），但数值不变；`dst_x/dst_y/w/h` 不变。
- **反向 `\iclip`**：遮罩覆盖区 `b≈255` ⇒ `g ≈ 0`，整张图被挖空成全透明。由于走
  `imul_bitmaps` 分支且矩形覆盖全图，输出缓冲仍在，但内容为 0；若该 `ASS_Image`
  连带颜色 alpha 也满透明，会在帧汇总时被释放（`ass_render.c:3416`）。
- 遮罩若为带抗锯齿的填充形状，其**内部**是 255、只有轮廓边缘一圈是中间值，所以"完全
  相交且落在内部"严格等价于不裁剪，这是乘法恒等元 `b=255` 的直接结果。

### 2.2 部分相交（两矩形只重叠一部分，或相交落在遮罩抗锯齿边上）

正向 `\clip`：

- **矩形不重叠判定**：`ax+aw<bx || ay+ah<by || ax>bx+bw || ay>by+bh || !h || !w`
  （`ass_render.c:785-786`）。只要有相交区域，就分配**仅覆盖相交矩形**的新缓冲
  （`ns = align(w)`，`ass_render.c:792-793`），把 `cur->dst_x/y` 平移、`w/h` 收缩到
  相交区（`802-806`）。可观察结果：
  - 图像被硬切到相交矩形，矩形之外的部分（含模糊光晕）**整体消失**；
  - 裁剪边界若是遮罩抗锯齿边（`0<b<255`），出现一条宽度约 1 像素的**柔和衰减带**，
    其值为 `a·b/255`——模糊的亮边被遮罩的渐变覆盖率调制，而不是出现第二条模糊边；
  - 若字幕本身经过强模糊，跨越裁剪边界的光晕在边界内侧按遮罩斜率衰减到 0，边界外侧
     abruptly 截断（因为新位图只到相交矩形为止）。
- **反向 `\iclip`**：保留整图尺寸（先 `memcpy` 全图，`ass_render.c:779`），仅在相交
  矩形内乘 `(255−b)`。可观察结果：遮罩覆盖处被挖空、遮罩外的字幕（含原本就伸到外面
  的模糊光晕）原样保留；挖空边缘同样是抗锯齿软边。图像的 `dst_x/y/w/h` 不变。

### 2.3 空裁剪区域（无重叠，或裁剪绘图为空/退化）

分两类"空"：

**(a) 几何上无重叠**——`blend_vector_clip()` 的矩形判定命中：

- 正向：`cur->w = cur->h = cur->stride = 0; continue;`（`ass_render.c:787-788`）。
  该图像变成零尺寸，`ass.h:58` 规定 w/h 为 0 的位图不应被渲染，可观察上**该层
  （影子/描边/正文中的一张）彻底消失**，但链表节点保留。
- 反向：直接 `continue;`（`ass_render.c:768-771`），**原图原样保留**——因为遮罩与
  图像没有任何公共像素，补集门控对它没有作用。这是正向"全裁掉"与反向"全保留"的
  对称语义。

**(b) 裁剪绘图本身为空/退化轮廓**——在遮罩获取阶段短路，`blend_vector_clip()`
提前 `return`：

- `ass_drawing_parse()`（`ass_drawing.c:314`）对无法产生任何轮廓的文本返回失败，或
  解析出的轮廓包围盒退化；`ass_outline_construct()` 把退化包围盒归零并仍标记
  `valid`（`ass_render.c:1317-1319`），随后 `quantize_transform()` 因退化/非法变换
  返回 false，或 `ass_outline_to_bitmap()` 因空 bbox 失败
  （`ass_bitmap.c:162-163`）。
- 于是 `blend_vector_clip()` 中的
  `if (!key.outline || !key.outline->valid || !quantize_transform(...)) return;`
  （`ass_render.c:724-726`）或 `if (!clip_bm) return;`（`729-730`）命中，**整段裁剪
  被跳过**，链表中的字幕位图保持未相乘的原样。
- Changelog 记录的 VSFilter 兼容约定是 **"Treat empty clip outlines as if everything
  was outside the clip"**（空裁剪轮廓视为一切都在裁剪区之外）。需要注意本仓库这版
  代码在"取不到有效遮罩"时的实际动作是**跳过门控**（等价于不裁剪），该约定的落实
  依赖上游对空轮廓的判定与短路路径，而不是构造一张全 0（正向=全透明）或全 255
  （恒等）的遮罩；阅读或改动这段短路逻辑时，必须同时对照该兼容目标核对最终可见性，
  不能把"跳过"误当作已满足"全部在外"。

### 2.4 三种情况对照表

| 情形 | 正向 `\clip`（mul，`a·b/255`） | 反向 `\iclip`（imul，`a·(255−b)/255`） |
|---|---|---|
| 完全相交、遮罩内 b=255 | 不变（`f=a`），模糊/描边全保留 | 挖空（`g≈0`），图像透明 |
| 部分相交（矩形重叠） | 收缩到相交矩形，外侧硬切消失；遮罩 AA 边呈软衰减 | 全尺寸保留，相交区内按补集挖空，边缘软 |
| 几何无重叠 | 零尺寸图像，该层不渲染（`w=h=0`） | `continue`，原图完全保留 |
| 空/退化裁剪轮廓 | 遮罩获取失败→整段裁剪短路返回，位图保持原样（兼容目标见 §2.3(b)） | 同一短路路径，同样跳过门控 |
| 边界像素 0<b<255 | 乘积产生 1px 级柔和过渡，不产生额外模糊 | 补集乘积产生对称柔和过渡 |

---

## 3. 合成顺序与缓存复用冲突时必须保留的不变量及实现依据

"缓存复用"倾向于把尽可能多的计算结果共享、且把缓存项当作不可变数据；"固定合成顺序"
则要求每个像素严格按 光栅化→（合并）→模糊/描边→裁剪 相位产出。两者的张力在于：
裁剪是**每事件、每帧几何相关**的，而模糊位图是**跨帧共享**的。libass 用以下不变量
化解，任何重构（例如想把裁剪提前、或想就地修改缓存位图以省一次分配）都不得破坏它们。

### 不变量 1：缓存项不可变 + 写时复制（COW），裁剪绝不能就地改写共享位图

- 字形/描边位图（bitmap cache）与 composite 位图（`CompositeHashValue.bm/bm_o/bm_s`）
  是引用计数共享的：`my_draw_bitmap()` 对 `source` 调 `ass_cache_inc_ref`
  （`ass_render.c:230-231`），同一 composite 可被多个 `ASS_Image`、多帧引用。
- `blend_vector_clip()` 因此**绝不就地修改** `cur->bitmap`：两个分支都
  `ass_aligned_alloc` 一块**新缓冲** `nbuffer`（`ass_render.c:774`、`793`），正向用
  `mul_bitmaps` 写入新缓冲，反向先 `memcpy` 再 `imul`；随后
  `priv->buffer = cur->bitmap = nbuffer; ass_cache_dec_ref(priv->source);
  priv->source = NULL;`（`809-812`）——断开与共享 composite 的引用，把新缓冲挂到
  图像的自有 `buffer` 上，随 `ass_free_image()` 释放（`ass_render.c:3350-3358`）。
- **依据**：若为省内存而就地相乘，同帧另一张引用该 composite 的图像（如同事件的描边
  与正文，或缓存命中的下一帧）会被意外裁剪，违反"一次缓存命中必须等价于重新计算"。
  COW 是缓存正确性的前提。

### 不变量 2：相位顺序固定——模糊/描边只在 composite 构造期发生，裁剪只在其后发生

- `ass_synth_blur()` 的全部调用点都在 `ass_composite_construct()` 内
  （`ass_render.c:2726-2727`）；`blend_vector_clip()` 内对遮罩只做光栅化获取，不含
  任何 blur 调用。代码层面没有任何路径能让"先裁剪后模糊"发生。
- **为什么不能反过来**：乘法门控与线性模糊核**不可交换**。`Blur(a·b) ≠ Blur(a)·b`：
  先乘遮罩再模糊，会把遮罩的硬边界也糊开，产生穿透裁剪边界的泄漏光晕，且模糊在零
  延拓下还会改变边界内侧的归一化能量。固定"先模糊后相乘"才能匹配 VSFilter 观感
  （硬裁剪边 + 遮罩自身 AA 软边，见 §2.2）。
- **依据**：composite 键（`CompositeHashKey`）的设计本身就把 `FilterDesc` 封死在
  缓存值里，裁剪参数根本不进入该键，结构上排除了"带裁剪的模糊缓存项"，从而无法用
  缓存复用诱导出错误相位。

### 不变量 3：乘法门控语义与正/反向补集，独立于几何相交判定

- 正向 `mul_bitmaps`（`a·b/255`）、反向 `imul_bitmaps`（`a·(255−b)/255`）是两条
  不同的引擎函数指针（`ass_bitmap_engine.h:76-77`），并有 C/x86/aarch64 多套实现，
  语义由 `c_blend_bitmaps.c:53-90` 固定。`clip_drawing_mode`
  （`ass_render.h:258`）在运行期选择，**不参与任何缓存键**——同一张遮罩位图可同时
  服务于正向与反向用法，方向语义只在相乘那一刻注入。
- 几何包围盒相交（`ass_render.c:754-757`）只是快速路径：它决定要不要分配新缓冲、
  以及新缓冲的尺寸，**不改变相交区内的逐像素公式**。无重叠时正向置零尺寸、反向
  `continue` 的不对称（§2.3）也必须保留，因为它正是补集语义在空集上的正确外延。
- **依据**：若缓存复用试图把"已乘过遮罩"的结果回填进通用缓存，正/反向两种用法会
  互相污染；保持遮罩缓存中性（只存 `b`）、把方向留给门控函数，才能让一份遮罩安全
  复用。

### 不变量 4：量化决定论——变换矩阵、模糊半径、描边宽度的量化是缓存等价性的唯一裁判

- `quantize_transform()`（`ass_render.c:498`）把连续变换量化进
  `BitmapHashKey` 的 `offset/matrix_*`；`quantize_blur()`（`ass_render.c:2398`）把
  模糊半径量化进 `FilterDesc.blur_x/y`，描边宽度量化进 `BorderHashKey.border`
  （`ass_render.c:1517-1518`）。哈希/比较函数由 `ass_cache_template.h:14-45` 统一
  生成，逐字段 `==` 比较。
- 含义：两个"量化后相等"的键必须产出**像素级一致**的位图，裁剪阶段才能无差别套用。
  这要求模糊的零延拓扩边（`ass_blur.c:234-235` 的 `left/top` 调整）、`\be` 边距
  （`ass_be_padding`）、亚像素移位（`ass_shift_bitmap`）等所有几何副作用都只依赖
  键内参数，不依赖帧内临时状态。
- **依据**：裁剪在模糊之后，意味着裁剪复用的模糊位图可能来自另一事件/另一帧的缓存
  命中；任何游离于键之外的状态都会让"命中缓存"与"现场重算"产生不同的模糊边缘，
  进而在同一裁剪边界上出现帧间闪烁。

### 不变量 5：三条图像通道（影子/描边/正文）接受同一遮罩的一致门控，且顺序不变

- `render_text()` 按 影子（`IMAGE_TYPE_SHADOW`）→描边（`IMAGE_TYPE_OUTLINE`）→正文
  （`IMAGE_TYPE_CHARACTER`）的固定顺序建链（`ass_render.c:827-878`），
  `blend_vector_clip(state, head)` 对整条链表**无差别遍历**（`for (cur=head; cur;
  cur=cur->next)`，`733`），三种 `type` 用同一 `clip_bm`、同一模式处理。
- 播放器必须按链表顺序叠加（`ass.h:54-57` "rendered in-order for the correct
  screen composition"）。裁剪不得重排链表、不得只裁某一通道——否则描边与正文在
  裁剪边界处会错位，破坏描边"包裹"字形的视觉关系。
- **依据**：遮罩只取一次、其生命周期覆盖整个遍历；若为复用而把裁剪结果按 `type`
  分别缓存，就必须额外保证三者几何严格对齐，而当前实现通过"单遮罩 + 单次线性遍历"
  天然规避了这一对齐风险。

### 不变量 6：矩形裁剪与矢量裁剪路径互斥，缓存不得跨越两者共享结果

- 矩形裁剪是 `render_glyph()` 内的指针/尺寸调整（共享同一底层缓冲，仅改起点和
  w/h），矢量裁剪是分配新缓冲的像素级乘法（§1.6）。两者由 `clip_mode` 与
  `clip_drawing_text.str` 互斥分流，解析层还会拒绝重复矢量标签
  （`ass_parse.c:745`）。
- **依据**：矩形路径不产生覆盖率乘积（硬切边），矢量路径产生 AA 软边，二者像素
  结果不同；若缓存层把矩形切边后的描述与矢量门控后的位图混用，会在动画化矩形裁剪
  （Changelog "Improve quality of animated rectangular clips"）与矢量裁剪切换时
  出现边缘跳变。保持键空间与代码路径双重隔离，复用才不会越过语义边界。

---

## 附：关键代码索引

| 主题 | 位置 |
|---|---|
| 矢量裁剪主逻辑（相交、mul/imul、COW） | `libass/ass_render.c:700-814` |
| 裁剪在图像链表生成后调用 | `libass/ass_render.c:881` |
| 矩形正向/反向裁剪 | `libass/ass_render.c:319-471` |
| composite 构造（合并→模糊→描边修正→阴影） | `libass/ass_render.c:2643-2757` |
| 模糊实现（高斯 + 盒式、扩边） | `libass/ass_bitmap.c:68-101`、`libass/ass_blur.c:173-240` |
| 位图乘法原语语义 | `libass/c/c_blend_bitmaps.c:53-90` |
| 三层缓存键定义 | `libass/ass_cache.h:59-88`、`libass/ass_cache_template.h:60-132` |
| 轮廓类型分流与空轮廓处理 | `libass/ass_cache.c:180-251`、`libass/ass_render.c:1234-1321` |
| 模糊/变换/描边量化 | `libass/ass_render.c:498-560`、`2398-2444`、`1517-1518` |
| 裁剪标签解析（矩形/矢量、正/反向互斥） | `libass/ass_parse.c:214-228`、`405-423`、`729-748` |
| 图像生命周期与全透明丢弃 | `libass/ass_render.c:210-236`、`3350-3419` |
| 覆盖率取值范围（0..255 抗锯齿） | `libass/c/rasterizer_template.h:141-224` |
