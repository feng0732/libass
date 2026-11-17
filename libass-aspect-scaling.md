# libass 画幅变化时字幕坐标与描边缩放分析

> 分析对象：本仓库 libass 渲染实现（`libass/ass_render.c`、`libass/ass_render_api.c`、`libass/ass_render.h`、`libass/ass_cache*.h`）。
> 范围：仅基于渲染实现，讨论视频画幅（frame size / storage size / margins / PAR）变化时事件坐标映射、位置缩放与描边缩放的约束，以及缓存失效条件。

## 1. 画幅状态如何进入渲染管线

### 1.1 输入状态与派生量

API 层写入 `ASS_Settings`（`libass/ass_render.h:59-80`）：`frame_width/height`（输出帧）、`storage_width/height`（视频存储尺寸）、四个 `*_margin`、`par`（像素宽高比，0 表示自动）。任何变化都经 `ass_reconfigure()` 折算成帧级派生量（`libass/ass_render_api.c:29-54`）：

- `width/height`：整帧尺寸；
- `frame_content_width/height`：帧尺寸减去 API margins，即视频内容区（`ass_render_api.c:40-43`）；
- `fit_width/fit_height`：内容区按原宽高比"letterbox 适配"进整帧后的尺寸（`ass_render_api.c:44-53`）。当内容区相对帧更宽时 `fit_width = width`、`fit_height = content_h * width / content_w < height`，反之亦然。`use_margins=1` 时字幕相对这个 fit 盒定位，从而可以落进上下/两侧的黑边里。

每帧开始 `ass_start_frame()` 计算 PAR 校正（`libass/ass_render.c:3076-3090`）：

```c
par = settings.par;
if (par == 0. || lr_track) {            // 未显式设置，或脚本带 LayoutRes
    dar = frame_content_width / frame_content_height;
    sar = layout_res.x / layout_res.y;  // LayoutRes 优先，其次 storage size
    par = dar / sar;
}
render_priv->par_scale_x = par;
```

**关键约束：PAR 校正只落在 x 轴**（`par_scale_x`，`ass_render.h:333` 注释 "x scale applied to all glyphs"）。y 方向从不做 PAR 拉伸。

### 1.2 三级缩放系数

每个事件重置渲染上下文时由 `init_font_scale()` 计算（`libass/ass_render.c:1035-1069`）：

| 系数 | 定义 | 用途 |
|---|---|---|
| `screen_scale_x/y` | `font_scr_w / PlayResX`、`font_scr_h / PlayResY` | 字号、字距、矢量剪辑缩放 |
| `border_scale_x/y` | SBAS=yes 时 = `screen_scale_*`；SBAS=no 时 = `blur_scale_*` | 描边宽度、阴影偏移 |
| `blur_scale_x/y` | `font_scr_w / layout_res.x`、`font_scr_h / layout_res.y` | 高斯模糊半径、3D 透视距离 |

其中 `font_scr_w/h` 在非显式定位且 `use_margins=1` 时取 `fit_width/fit_height`，否则取 `frame_content_width/height`（`ass_render.c:1040-1045`）；`layout_res` 由 `ass_layout_res()` 按 LayoutRes > storage size > 由 PAR 反推 > PlayRes 的优先级确定（`ass_render.c:1008-1033`）。`ScaledBorderAndShadow`（SBAS）来自脚本头（`libass/ass.c:577-578`，自定义 format 行时默认 yes，`ass.c:805-820`）。

## 2. 事件坐标映射的约束

### 2.1 坐标映射函数

脚本坐标 → 屏幕坐标（`libass/ass_render.c:241-304`）：

```c
x2scr_pos:   x * frame_content_width / par_scale_x / PlayResX + left_margin   // :243
y2scr_pos:   y * frame_content_height / PlayResY + top_margin                 // :273
x2scr_left/right (use_margins): x * fit_width / par_scale_x / PlayResX
             [+ (width - fit_width)]                                          // :251-261
y2scr*/sub (use_margins):       y * fit_height / PlayResY
             [+ (height - fit_height) * 0.5 或 (height - fit_height)]         // :281-303
```

约束要点：

1. **x 映射预先除以 `par_scale_x`**（`ass_render.c:243`），因为在位图合成阶段所有 x 坐标又被乘回来：`device_x = (device_x - left) * par_scale_x + left`（`ass_render.c:2453`）、`info->pos.x = device_x + pos.x * par_scale_x`（`ass_render.c:2543`）。净效果：**`\pos`、边距、滚动等事件坐标在屏幕上占据内容区的相同相对比例，与 PAR 无关**；PAR 只拉伸字形本身和字形间距。乘法以 `left_margin` 为不动点（先减后加），保证 PAR≠1 时左边界对齐不漂移。
2. **y 映射完全不含 PAR**。垂直位置只由 `frame_content_height`（或 `fit_height`）与 `PlayResY` 决定。
3. `use_margins=1` 时定位基准从内容区切换到 fit 盒，并加上 letterbox 偏移 `(width - fit_width)` / `(height - fit_height) * 0.5`（`ass_render.c:259-261, 281-283`）；显式定位事件（`\pos`/`\move`，`state->explicit`）始终使用全内容区映射，不进 fit 盒（`ass_render.c:249-250, 279-280`）。
4. 事件级定位在 `ass_render_event()` 中完成：`\pos` 事件 `device_x = x2scr_pos(pos_x) - base_x`（`ass_render.c:2882-2885`）；普通事件由对齐方式取 `x2scr_left/right(MarginL/R)` 与 `y2scr_top/sub(MarginV)`（`ass_render.c:2900, 2918-2947`）。可折行宽度 `max_text_width = x2scr_right(PlayResX - MarginR) - x2scr_left(MarginL)`（`ass_render.c:2852-2854`）——**画幅变化会改变折行结果**，进而改变行数与整体高度。

### 2.2 字号与字形缩放的不对称

```c
// VSFilter compatibility: font glyphs use PlayResY scaling in both dimensions
info->font_size = fabs(state->font_size * state->screen_scale_y);   // ass_render.c:2129-2131
```

**字体字形在两个方向上都只用垂直缩放系数**（向 VSFilter 兼容）。水平方向的差异完全由 `par_scale_x` 在渲染期补上：

- 字距：`hspacing * screen_scale_x / par_scale_x * scale_x`（`ass_render.c:2158-2160`）；
- 绘图（drawing）轮廓：`scale.x = ... screen_scale_x / par_scale_x`、`scale.y = ... screen_scale_y`（`ass_render.c:1190-1191`）；
- 3D 变换：`scale_x = dist * par_scale_x`，透视距离 `dist = 20000 * blur_scale_y`（`ass_render.c:1362-1369`）；
- 卡拉 OK 计时边界：`effect_timing * par_scale_x`（`ass_render.c:2576-2578`）。

也就是说，**所有水平量都以"PAR=1 的预校正空间"计算，最后在 `ass_render.c:2453/2543` 统一乘 `par_scale_x`**；凡是绕过这两个乘法、独立进入位图管线的水平量（描边、阴影、字距、透视）都必须各自显式除以 `par_scale_x` 来抵消。

## 3. 三种画幅变化情形

设变化前内容区为 `(cw, ch)`，PlayRes 固定。

### 3.1 仅变宽（cw → cw′，ch 不变）

- **事件坐标**：`x2scr*` 的 x 映射系数 `cw′ / par_scale_x / PlayResX` 变化；y 映射不变。事件在水平方向重新分布，垂直位置逐像素不变。
- **位置缩放**：`screen_scale_y` 不变 ⇒ `font_size` 不变 ⇒ **字形像素高度、行高、垂直间距完全不变**；`screen_scale_x` 变化只影响字距与 drawing 的水平缩放。
- **描边缩放**：SBAS=yes 时 `border_scale_x = cw′/PlayResX` 变、`border_scale_y` 不变 ⇒ **水平描边厚度（屏幕像素）随宽度线性变化，垂直描边厚度不变，描边变为各向异性**；SBAS=no 时若 layout_res 不随动（例如显式 LayoutRes 或固定 storage size），`border_scale` 不变，描边厚度保持不变。
- **PAR 联动**：若 `par=0`（自动）且 storage size 固定，`par = dar/sar` 随 cw′ 增大 ⇒ `par_scale_x` 增大，字形被水平拉伸（变形宽银幕效果）；若调用者强制 `par=1`，字形保持原形，仅位置摊开。这是"字幕跟随视频几何"与"字幕保持自身形状"之间的取舍点。

**可观察结果**：窗口仅拉宽、PAR 自动时，同一 `\pos(320,240)` 事件锚点仍在内容区同一相对位置，但文字整体变宽（字形水平拉伸、描边左右变厚）；PAR 强制为 1 时文字形状不变，仅水平位置重排，且因 `max_text_width` 增大，长行可能减少折行。

### 3.2 仅变高（ch → ch′，cw 不变）

- **事件坐标**：y 映射系数 `ch′/PlayResY` 变化；x 映射中 `par_scale_x` 在自动模式下也会变（`par = dar/sar`，dar 变小），从而间接影响 x 的落点。
- **位置缩放**：`screen_scale_y` 变化 ⇒ `font_size` 变化 ⇒ **所有字形度量（轮廓、advance、asc/desc）按新字号重新生成**；这是唯一会改变字形像素尺寸的单轴变化。
- **描边缩放**：SBAS=yes 时 `border_scale_y` 变、`border_scale_x` 不变 ⇒ 垂直描边厚度变化、水平不变，同样各向异性。
- **fit 盒翻转**：`fit_width/fit_height` 的限制维可能交换（`ass_render_api.c:44-53` 中的两个比较式），`use_margins=1` 时字幕定位基准随之从"上下留边"切换到"左右留边"。

**可观察结果**：仅拉高窗口时，字号（像素）不变、文字随内容区垂直下移；若同时 PAR 自动，`par_scale_x` 减小，字形水平变窄——尽管宽度没变，文字形状也会变。这是"PAR 只作用于 x 轴"设计最直接的可见后果。

### 3.3 非均匀调整（cw、ch 同时变且比例不同）

- 上述两种效应叠加：`screen_scale_x ≠ screen_scale_y`，坐标映射在两个轴上系数不同。
- **字号仍只跟 y**：`font_size = FontSize * screen_scale_y`，字形在预校正空间内保持各向同性；屏幕上的最终宽高比 = `par_scale_x`（自动模式下 ≈ dar/sar），即文字变形程度跟随视频显示宽高比，与帧的绝对尺寸解耦。
- **描边各向异性显式化**：`border_scale_x ≠ border_scale_y`，且 x 方向额外除以 `par_scale_x`（见 §4）。SBAS=yes 时屏幕上水平描边 = `Outline * screen_scale_x`、垂直描边 = `Outline * screen_scale_y`，两者之比等于内容区宽高比的变化之比——**描边厚度与视频像素网格绑定，而非与视觉尺寸绑定**（VSFilter 兼容行为）。
- **模糊同样各向异性**：`blur_scale_x/y` 分别进入 `filter->blur_x/blur_y`（`ass_render.c:2514-2517`）。
- **折行与碰撞**：`max_text_width` 变化导致重折行；每事件缓存的碰撞矩形（§5.2）失效前可能短暂沿用旧位置。

**可观察结果**：从 16:9 帧切到 4:3 帧（非均匀）且 PAR 自动时，文字随视频一起被水平压缩、描边左右变薄；若脚本写死 `LayoutRes`，则 PAR 完全由 LayoutRes 与帧内容区推导（`ass_render.c:3078-3086`），API 设置的 `par` 被忽略——字幕几何锚定在脚本的布局坐标系上，而非播放器的假设上。

## 4. 描边（outline）缩放的实现约束

描边在位图管线中以"轮廓偏移 + 描边"实现，关键代码 `get_bitmap_glyph()`（`libass/ass_render.c:1418-1470`）：

```c
// BorderStyle=3（不透明盒子）:
bord = { 64 * border_x * border_scale_x / par_scale_x,
         64 * border_y * border_scale_y };                 // :1425-1429
// 普通描边（OUTLINE_BORDER）:
bord_x = 64 * border_scale_x * info->border_x / tr->scale.x / par_scale_x;  // :1466-1468
bord_y = 64 * border_scale_y * info->border_y / tr->scale.y;                // :1469-1470
```

约束：

1. **x 方向必须除以 `par_scale_x`**：描边是在字形轮廓（预校正空间）上做的，之后轮廓连同描边一起被 `par_scale_x` 拉伸；预除保证屏幕上的水平描边厚度 = `Outline * border_scale_x`，与 PAR 无关。阴影偏移同理（`ass_render.c:2388-2391`：`shadow_x * border_scale_x / par_scale_x`）。
2. **除以 `tr->scale.x/y`**：描边宽度以 `\fscx/\fscy` 变换前的单位给出，保证 `\fscx200` 只放大字形、不放大描边厚度。
3. **SBAS 开关决定描边跟随哪个分辨率**：`border_scale = screen_scale`（SBAS=yes，跟随 PlayRes→帧映射）或 `border_scale = blur_scale`（SBAS=no，跟随 LayoutRes/storage→帧映射，`ass_render.c:1053-1059`）。后者下描边厚度以存储像素为基准，帧缩放时厚度在"视频像素"意义下恒定。
4. **描边轮廓有独立缓存键**：`BorderHashKey = {outline, scale_ord_x, scale_ord_y, border}`（`libass/ass_cache_template.h:107-114`），`scale_ord_*` 是为把 stroker 误差约束在允许范围而对轮廓预乘的 2 的幂（`ass_render.c:1500-1509` 的误差推导，`STROKER_PRECISION`/`POSITION_PRECISION` 定义于 `ass_render.c:43-45`）。描边宽度变化 → 键变化 → 重新描边；但 `par_scale_x` 变化会改变 `border` 字段的取值（经 `/par_scale_x`），因此 PAR 翻转必然产生新的描边缓存项。
5. **BorderStyle=3 的"双重缩放"**：盒子尺寸与描边都额外乘 `orig_scale = scale_x * scale_fix`（`ass_render.c:1433-1442`），注释明确这是模拟 VSFilter 的怪异行为（"Emulate the WTFish behavior of VSFilter"）——兼容性是比几何纯净性更高的优先级。

## 5. 缓存失效：哪些状态变化必须让什么失效

### 5.1 失效入口

所有画幅相关 setter 在值真正变化时调用 `ass_reconfigure()`（`libass/ass_render_api.c:29-54`）：

```c
priv->render_id++;
ass_cache_empty(priv->cache.composite_cache);
ass_cache_empty(priv->cache.bitmap_cache);
ass_cache_empty(priv->cache.outline_cache);
// 重算 width/height/frame_content_*/fit_*
```

触发者：`ass_set_frame_size`（:56）、`ass_set_storage_size`（:67）、`ass_set_margins`（:88）、`ass_set_pixel_aspect`/`ass_set_aspect_ratio`（:105-117）、`ass_set_font_scale`（:119）、`ass_set_hinting`（:127）、`ass_set_line_position`（:140）、`ass_set_fonts`（:148，额外清空 font/metrics 缓存并重建 fontselect）、两个 selective style override 接口（:169-184）。

**不触发 reconfigure 的 setter**：`ass_set_use_margins`（:100-103）、`ass_set_shaper`（:79-86）、`ass_set_line_spacing`（:135-138）。

### 5.2 各缓存的失效必要性分析

| 缓存 | 键（`libass/ass_cache_template.h`） | 画幅变化后键是否自描述 | 必须失效的原因 |
|---|---|---|---|
| outline 缓存（字形） | `GlyphHashKey{font, size, face_index, glyph_index, bold, italic, flags}`（:88-96） | 是：`size` 已含 `screen_scale_y`（`ass_render.c:2130`） | **hinting 不在键中**：`ass_set_hinting` 改变 FreeType 加载方式却不改变键，不清空会复用旧轮廓——这是 correctness 级失效 |
| outline 缓存（drawing） | `DrawingHashKey{text}`（:101-103） | 键与缩放无关（缩放在位图层施加，`ass_render.c:1190`） | 无需为缩放失效；随 reconfigure 清空属保守做法 |
| bitmap 缓存 | `BitmapHashKey{outline, offset, matrix_x/y/z}`（:62-69） | 是：量化后的变换矩阵嵌入了 screen_scale、par、位置（`quantize_transform`，`ass_render.c:498-…`） | 键自描述，清空主要为内存卫生；量化（`SUBPIXEL_ORDER=3`，1/8 像素）保证微小移动仍能命中 |
| composite 缓存 | `CompositeHashKey{FilterDesc, bitmaps}`（`ass_cache.h:84-88`） | 是：blur/shadow 已含 blur_scale/border_scale（`ass_render.c:2514-2524`） | 同上 |
| 每事件 `RenderPriv` | 仅按 `render_id` 失效（`ass_render.c:3123-3126`） | **否** | 碰撞检测缓存的事件屏幕矩形 `{top,height,left,width}`（`ass_render.h:343-346`）直接依赖画幅；`render_id++` 是唯一的失效机制，故每次 reconfigure 必须递增 |

结论：

1. **correctness 级必须失效**的只有两类：(a) `render_id`（碰撞矩形），(b) hinting 变化时的 outline 缓存（键不含 hinting）。字体的 metrics/font 缓存键引用 `ASS_Font*` 指针，`ass_set_fonts` 重建 fontselect 后旧指针可能悬垂，故也必须清空（`ass_render_api.c:160-166`）。
2. 其余缓存在画幅变化后键自然错开，旧项只是死内存；`ass_reconfigure` 一律 `ass_cache_empty` 是用**简单性和有界内存**换**重建开销**——每次窗口缩放都会丢弃全部字形/位图，下一帧全量重光栅化。
3. **潜在不一致点**：`ass_set_use_margins` 改变 `init_font_scale` 的 `font_scr_*` 与全部 `x2scr/y2scr` 映射，却不递增 `render_id`。位图键因矩阵不同仍正确，但每事件 `RenderPriv` 的碰撞矩形沿用旧值：`fix_collisions` 中 `priv->height == imgs[i].height` 即视为"固定事件"并 `shift_event(priv->top - imgs[i].top)` 强制回到旧位置（`ass_render.c:3219-3246`）。高度未变的事件在 `use_margins` 翻转后会停留在旧垂直位置，直到高度变化或下一次 reconfigure。`ass_set_line_spacing`（改变行高）同理。

### 5.3 帧间的常规缓存维护

每帧 `ass_start_frame()` 只做预算裁剪 `check_cache_limits()`（`ass_render.c:3021-3026`，按 `bitmap_max_size`/`composite_max_size`/`glyph_max` LRU 裁剪），不做内容失效——**帧内所有几何都由当帧的 `par_scale_x`、screen scales 重新计算**，跨帧状态只有缓存与 `RenderPriv`，这正是"画幅变化必须走 reconfigure"的架构前提。

## 6. 保持几何一致性的取舍总结

1. **单轴 PAR 校正**：把全部各向异性集中到 `par_scale_x` 一处，y 向度量（行高、垂直定位、字号）永远稳定；代价是水平量必须在 ~8 个位置显式除/乘 `par_scale_x`（`ass_render.c:1190, 1365, 1427, 1468, 2159, 2389, 2453, 2543, 2578`），任何遗漏都会造成 PAR≠1 时的几何错位——这是该设计最脆弱的部分。
2. **字号绑定 PlayResY**（VSFilter 兼容）：窗口变宽不改变文字像素尺寸，行为可预期；代价是非均匀缩放下文字变形完全依赖 PAR 设置正确，而自动 PAR 又依赖调用者正确设置 storage size（`ass.h:418-419` 明确警告 "storage size must be configured ... otherwise libass is forced to make a fallible guess"）。
3. **描边跟随脚本语义而非视觉一致性**：SBAS=yes 时描边厚度绑定 PlayRes 映射、SBAS=no 时绑定 LayoutRes/storage 映射，两种模式下非均匀缩放都产生各向异性描边；这是为了逐像素兼容 VSFilter 输出，放弃了"屏幕上描边视觉厚度恒定"的替代方案。
4. **缓存一键清空**：以缩放/改设置后一帧的全量重渲染为代价，换取无需审计"哪些键真正覆盖了哪些设置"的安全性（hinting 不在键中这一事实说明该审计确实会失败）。
5. **`render_id` 粗粒度失效**：任何 reconfigure 都重置全部事件的碰撞状态，缩放后事件可能跳动一帧；换来的是无需逐事件比对新旧几何。
6. **量化换缓存命中**：`quantize_transform` 以 1/8 像素（`SUBPIXEL_ORDER=3`）和 `POSITION_PRECISION=8/64` 像素的变换误差量化矩阵键，使移动中的字幕仍能命中位图缓存；代价是定位存在亚像素级量化误差，且误差预算随透视变换的 z 放大（`MAX_PERSP_SCALE=16` 钳制，`ass_render.c:46, 550-551`）。
