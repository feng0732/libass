# 字幕变换动画遇到样式重置时的时间优先级

- **代码基线**：libass 0.17.5（`RELEASEVERSION`），分支 `task/20`，提交 `21b5a54`。
- **方法**：
  1. 纯静态源码阅读；
  2. 对仓库内已提交基准 PNG 的静态解码测量（PNG/zlib 解码、Alpha 掩码质心与旋转拟合，脚本在 `/tmp/libass-report/`，未纳入仓库）；
  3. 把 `ass_parse_tags` 的**参数切分 + `\t` 分派 + 若干吃 `pwr` 的标签**逐行复刻成独立小程序（`/tmp/libass-report/tprobe/`），用它对 §2/§3 的数值与可达性结论做交叉验证。复刻程序**不链接 libass**，只复刻逻辑，用它跑出了 1 331 个结构化用例、30 万条随机用例和若干三重嵌套用例。
- **未执行**：**没有**编译 libass，**没有**启动播放器、`libass_test`、`compare` 或任何渲染进程。本机缺少 freetype2 / harfbuzz / fribidi / libpng 的开发头文件与 `pkg-config` 描述文件（`pkg-config --modversion freetype2` 失败），无法构建。因此下文凡涉及"输出图像"的陈述都来自：(a) 源码路径推导；(b) 仓库内**已提交**的期望输出图（`compare/test/sub1-*.png`）的几何反演；(c) 上述逻辑复刻程序的数值模拟。**没有一项来自对 libass 本体的实际执行**，复刻程序与真实实现的偏差风险已在各条结论中标注。
- 文中 `file:line` 均指本仓库当前内容。所有"建议"均已明确标注，与"当前行为"分开。

---

## 0. 结论速查

1. libass **没有动画列表/动画求解器**。`\t` 不是"注册一个随时间变化的通道"，而是**每帧重新解析整条事件文本**时，按当前帧时间算出混合系数 `k`，把标签目标值与"此位置的当前状态值"做线性插值后立刻写回状态。见 `libass/ass_render.c:2815`（每帧 `parse_events`）、`libass/ass_parse.c:670-728`、时间源 `libass/ass_render.c:3064`。
2. 因此"优先级"的实际含义是**文本位置顺序**，不是标签等级：
   - 样式（含用户选择性覆盖）→ `\r` 重建基准 → 行内静态标签 → `\t` 载荷。后出现者改写先出现者的结果。
   - `\r` 出现在 `\t` 之后会**直接抹掉**该动画；出现在之前则作为动画起点。
   - **无参标签**（`\fscx`、`\frz`、`\bord`、`\blur`、`\c`、`\alpha`、`\be`、`\shad`、`\fsp`、`\fs`、`\fn`、`\fsc`）**在 `\t` 载荷内也忽略 `pwr`**，等价于瞬时回到样式值，动画当场失效。
3. 解析结果按 **glyph 快照**冻结（`libass/ass_render.c:2119-2152`）；渲染阶段不再回看样式表。`\pos`/`\move`/`\org`/`\clip`/`\iclip`/`\an`/`\a`/`\fad`/`\fade` 写的是**事件级** `state` 字段，没有逐 glyph 归属，因此文本中段写 `\clip`/`\an` 会**追溯作用于整条事件**。`\rSTYLE` 还有一个例外：它把 `state->style` 改指向新样式（`:1002`），而事件边距在解析**之后**才读 `state->style->Margin*`（`:2844-2849`），所以 `A{\rOther}B` 的整条事件边距按 `Other` 算。
4. 渲染缓存的变换特异性从低到高：**outline 缓存（完全不含变换）→ bitmap 缓存（含量化后的变换矩阵）→ composite 缓存（含过滤器与量化位置，不含颜色）→ 每帧 `ASS_Image` 的颜色/alpha**。动画几何的可见值 = bitmap 键里**量化后**的矩阵，而不是请求值；位置分辨率 1/8 px（`SUBPIXEL_ORDER=3`，`libass/ass_render.c:47`），误差预算 `POSITION_PRECISION = 8/64 px`（`libass/ass_render.c:45`）。
5. 颜色/透明度动画不使 composite 缓存失效；只有跨过 `alpha==0xFF`/`alpha==0`/`fade==0` 这些**过滤器标志分界**时才会换 composite（`libass/ass_render.c:2466-2487`）。
6. 事件可见区间是 `Start <= now < Start + Duration`（`libass/ass_render.c:3384-3385`）。`\t(t1,0,...)` 的 `t2` 被改写成 `Duration`（`libass/ass_parse.c:695-696`），而 `now == Start+Duration` 时事件不渲染，**所以 `k==1` 的终态永远不会被显示**。仓库基准 `{\t(\frz+180)}A`（`compare/test/sub1.ass:12`）正是这个写法：源码推出 0.5/1.5/2.5 s 应为 22.5°/67.5°/112.5°；基准图的**相对**旋转测量给出 0.5→1.5 s 为 45.5°、1.5→2.5 s 为 45.0°（即 45°/s），2 s 累计约 90°——与"到 2.5 s 时远未到 180°"一致。绝对角度只能由源码给出，测量本身不能独立确定相位。
7. 仓库**没有任何**覆盖 `\r`、动画交叠或事件边界的用例：`compare/test/` 只有 `\t(\frz+180)` 与 `\blur1` 两条（`compare/test/sub1.ass:12`、`compare/test/sub2.ass:12`），`test/test.c` 只是冒烟渲染器，`compare/meson.build:17-19` 的 `art-compare` 依赖外部 `art-samples` 仓库。详见 §3。
8. 嵌套 `\t` 有一个**未文档化的当前行为**：`\t` 的最后一个参数在遇到第一个 `)` 时就被截断（`libass/ass_parse.c:333-339`），因此 `{\t(0,4000,\fscx200\t(0,1000,\fscy300)\frz45)}` 里的 `\frz45` **不属于任何 `\t`**，以 `pwr = 1` 立即生效；真正的多层括号嵌套不成立。反过来，`{\t(0,4000,\fscx200\t(0,1000,\fscy300))}` 里两个动画各按自己的窗口走（1000 ms / 4000 ms），因为内层 `\t` 的 `)` 同时截断了外层载荷。详见 §2.2 与 §3.4。

---

## 1. 各层的优先级

### 1.1 解析管线：每帧一次、顺序、就地写入

```
ass_render_frame                       libass/ass_render.c:3380-3395  (可见性筛选)
  └─ ass_render_event                  libass/ass_render.c:2798-3015
       ├─ free_render_context / init_render_context   :2812-2813, 1116-1153
       ├─ parse_events                 :2815  (逐块解析，写 state，再快照进 glyph)
       ├─ split_style_runs             :2825  (2010-2046)
       ├─ shaper / retrieve_glyphs     :2828-2838
       ├─ preliminary_layout           :2840
       ├─ wrap_lines_smart             :2857
       ├─ ass_process_karaoke_effects  :2860
       ├─ reorder / align_lines        :2862-2864
       ├─ calculate_rotation_params    :2989
       └─ render_and_combine_glyphs    :2991
            └─ get_bitmap_glyph       :1383-1542  (量化 + bitmap/border 缓存)
```

`RenderContext` 在每帧开头被 `init_render_context` 重置（`libass/ass_render.c:1116-1153`），随后 `init_render_context` 末尾调用 `ass_reset_render_context(state, NULL)` 把状态刷成事件所用样式的值（`:1150`）。**没有任何跨帧持久化的动画状态**：`\t` 的当前值每帧从"解析块位置"重新推导，因此 seek 与逐帧播放完全一致，不会累积漂移。

`\t` 的实现（`libass/ass_parse.c:670-728`）：

```c
t2 = 0 → t2 = state->event->Duration;            // :695-696
delta_t = (uint32_t) t2 - t1;                     // :697
t = render_priv->time - state->event->Start;      // :698
if (t < t1) k = 0.;                               // :699-700
else if (t >= t2) k = 1.;                         // :701-702
else k = pow((t - t1) / delta_t, accel);          // :703-706
```

也就是**阶梯 + 线性（或幂）斜坡**，没有缓动以外的状态。`k` 只用作 `pwr`；是否递归解析载荷取决于参数切分结果（`args[cnt].end < end`，`:716`），与 `k` 无关。

### 1.2 样式层与 `\r`

`\r` 分派在 `libass/ass_parse.c:805-811`：

```c
} else if (tag("r")) {
    if (nargs)  ass_reset_render_context(state,
                     lookup_style_strict(render_priv->track, args->start, len));
    else        ass_reset_render_context(state, NULL);
}
```

- **完全不看 `pwr`**。所以 `{\t(0,4000,\r)}` 与 `{\r}` 等价：没有插值、没有时间窗，整条事件范围内都是"已重置"。（这是"重置 vs 动画"的核心不对称：时间只作用于吃 `pwr` 的标签。）
- 样式名查找是**严格匹配**（含大小写），失败返回 `NULL` 并打 warning（`libass/ass_parse.c:261-273`）；`NULL` 时回落到**事件自身声明的样式**（`libass/ass_render.c:920-922`），而不是"当前 `state->style`"。这条是 Changelog `:463` 记录的 "Make `\r` fall back to line style"。
- 重置前会重新走选择性样式覆盖管线（`ass_reset_render_context` 首行 `libass/ass_render.c:1077` → `handle_selective_style_overrides` `:906-1006`），并**把 `state->style` 指向 `state->override_style_temp_storage`**（`:1002`）。后续如 `libass/ass_render.c:2844-2849` 读 `state->style->MarginL` 时读到的就是这份临时副本。
- 同时重算 `init_font_scale`（`:1079`，定义在 `:1035-1069`），所以 `\r` 之后 `screen_scale_*` / `border_scale_*` / `blur_scale_*` 都会按 `state->explicit` 与 `font_size_coeff` 重新取值。

`ass_reset_render_context`（`libass/ass_render.c:1075-1111`）**重置的字段**（全部取样式值）：

| 字段 | 行 | 备注 |
|---|---|---|
| `c[0..3]` 四种颜色（含 alpha） | 1081-1084 | 透明度也被重置 |
| `flags`（下划线/删除线） | 1085-1087 | |
| `font_size` | 1088 | |
| `family` / `treat_family_as_pattern` / `bold` / `italic` + `ass_update_font` | 1090-1095 | 字体指针可能换 |
| `border_style` / `border_x` / `border_y` | 1097-1099 | |
| `scale_x` / `scale_y` | 1100-1101 | 缩放动画被抹掉 |
| `hspacing` | 1102 | |
| `be` | 1103 | **固定 0**（不是 `style->something`） |
| `blur` | 1104 | `style->Blur`，不是 0 |
| `shadow_x` / `shadow_y` | 1105-1106 | |
| `frx` / `fry` | 1107 | 固定 0 |
| `frz` | 1108 | `style->Angle` |
| `fax` / `fay` | 1109 | 固定 0 |
| `font_encoding` | 1110 | 影响基础书写方向（`:2829`） |

**不重置**的字段（重要，容易被误认为是"整条重置"）：`alignment`、`justify`（只在 `init_render_context:1151-1152` 设置一次）、`pos_x/pos_y`、`org_x/org_y/have_origin`、`clip_*`、`clip_mode`、`fade`、`pbo`、`drawing_scale`、`wrap_style`、`effect_type/timing/skip_timing`、`reset_effect`、`detect_collisions`、`parsed_tags`、`evt_type`、`explicit`、`border_scale_*` 之外的屏幕参数。

> 因此 `\r` **无法**"解除定位/裁切/淡入淡出"。`A{\pos(100,100)}B{\r}C` 中 C 仍然在 (100,100)；`{\fad(500,500)}A{\r}B` 中 B 仍然按整条事件的 fade 衰减（fade 是事件级且"首次生效"，见 §1.4）。

### 1.3 哪些标签吃 `pwr`，哪些无视

对 `ass_parse_tags` 全量扫描（`libass/ass_parse.c:282-921`），`pwr` 只在这几类出现：

**吃 `pwr`（可被 `\t` 插值）**：`\xbord` `\ybord` `\xshad` `\yshad`（355-388）、`\fax` `\fay`（389-404）、`\iclip`（405-424）、`\blur`（425-434）、`\fscx` `\fscy`（436-453）、`\fsp`（457-464）、`\fs`（465-476）、`\bord`（477-488）、`\frx` `\fry` `\frz`/`\fr`（531-555）、`\alpha`（567-582）、`\clip`（729-748）、`\c`/`\1c` `\2c` `\3c` `\4c`（749-776）、`\1a`…`\4a`（777-804）、`\be`（812-824）、`\shad`（869-881）。

**不吃 `pwr`（在 `\t` 载荷内也瞬时取终值）**：

| 标签 | 行 | 无参时的行为 |
|---|---|---|
| `\fsc` | 454-456 | 直接回 `style->ScaleX/Y`，无参分支都不存在 |
| `\fn` | 556-566 | `\fn` 回样式字体 |
| `\b` `\i` | 825-836 | 回样式 |
| `\s` `\u` | 882-897 | 回样式 |
| `\pbo` `\p` `\q` `\fe` | 898-917 | `\pbo`→0、`\p`→0、`\q`→track WrapStyle、`\fe`→样式 Encoding |
| `\pos` `\move` `\org` | 489-530 / 606-621 / 657-669 | 见 §1.4 |
| `\fad` `\fade` | 622-656 | 见 §1.4 |
| `\an` `\a` | 584-605 | 见 §1.4 |
| `\r` | 805-811 | 见 §1.2 |
| `\k` `\kf`/`\K` `\ko` `\kt` | 837-868 | 卡拉 OK 计时，见 §1.5 |

并且 **无参写法会"复位"而不是"保持不变"**，这是"遇到重置"的第二类来源（隐式重置）：

```c
} else if (tag("fscx")) { ... val = state->scale_x*(1-pwr)+val*pwr; ... else val = state->style->ScaleX;  // 443
} else if (tag("frz") || tag("fr")) { ...  else state->frz = state->style->Angle;                       // 554-555
} else if (tag("bord"))  { ... else xval = yval = state->style->Outline;                                 // 486
} else if (tag("blur"))  { ... else state->blur = 0.0;                                                   // 434
} else if (tag("shad"))  { ... else xval = yval = state->style->Shadow;                                  // 879
} else if (tag("alpha")) { ... else 四个 c[i] 的 alpha 回样式值;                                          // 573-582
} else if (tag("be"))    { ... else state->be = 0;                                                       // 824
```

→ `{\fscx100\t(0,4000,\fscx)}A` 与 `{\fscx100\fscx}A` 在**所有时刻**等价：A 一直是样式缩放，动画不存在。

### 1.4 逐 glyph 快照 vs 事件级字段

`parse_events` 的循环体（`libass/ass_render.c:2059-2170`）在每取到一个字符时，把当时的 `state` 拷进 `GlyphInfo`（`:2119-2152`），随后立刻清掉"一次性"字段（`:2164-2169`）：

- 逐 glyph 快照：`c[4]`、`font_size`、`be`、`blur`、`shadow_x/y`、`scale_x/y`、`border_style`、`border_x/y`、`hspacing`、`bold`、`italic`、`flags`、`frx/fry/frz`、`fax/fay`、`fade`、`drawing_text/drawing_scale/drawing_pbo`（`:2113-2152`）。
- 一次性清零（"只作用于紧随其后的第一个 glyph"）：`effect_type`、`effect_timing`、`effect_skip_timing`、`reset_effect`（`:2166-2169`）。
- 注意 `\p`/`\pbo` **属于逐 glyph 类**：它们通过 `drawing_*`（`:2113-2117`）落到 glyph 上，只影响其后的矢量图文本；`\p0` 切回文字后不再有影响。它们不是事件级字段。

**事件级 / 先到先得 / 全事件生效**的字段（与 glyph 顺序无关或只认第一次）：

| 标签 | 语义 | 源码 |
|---|---|---|
| `\pos` | 第一次生效；`\move`/`\pos` 之后再来 `\pos` 会被忽略并打日志 | `:606-621`（`EVENT_POSITIONED` 门控），`:613-615` |
| `\move` | **只认第一次**（同样被 `EVENT_POSITIONED` 门控），第二次被静默丢弃 | `:489-530`，`:525-530` |
| `\org` | 第一次生效（`have_origin`） | `:657-669` |
| `\fad` / `\fade` | 第一次生效（`PARSED_FADE`），**不可动画** | `:622-656`，`:650-656` |
| `\an` / `\a` | 第一次生效（`PARSED_A`） | `:584-605` |
| `\clip` / `\iclip` | **最后一次生效**（无门控，直接覆盖 `clip_*`/`clip_mode`） | `:405-424`，`:729-748` |
| `\t` | 副作用：无条件 `state->detect_collisions = 0` | `:694` |
| `\r` | 见 §1.2，位置无关但**作用于其后所有 glyph** | `:805-811` |

两个直接后果，回答"位置优先级"：

1. **`\t` 不能动画位置**。`\pos`/`\move` 不吃 `pwr`；唯一随时间变化的位置是 `\move` 自己的线性插值（`:511-524`，且 `t1>t2` 会被**交换**，`:503-507`）。`\t` 里的 `\pos`/`\move`/`\org` 会在 `k` 到达任意值时立即生效/被忽略，与 `k` 无关。
2. **`\clip` 的中段写法会追溯整条事件**：`A{\clip(...)}B` 里 A 也被裁掉，因为 `clip_*` 只在渲染时读一次（`:2950-2979`）。同理 `A{\an8}B` 会让整行按上对齐（`state->alignment` 在 `:2842`、`align_lines:2304`、`get_base_point:1941-1969` 使用）。
3. **`\rSTYLE` 会连带改写整条事件的外边距**。`state->style` 在解析期被 `\r` 改指向新样式（`libass/ass_render.c:1002`），而 `MarginL/R/V` 是在 `parse_events` **之后**才从 `state->style` 读取（`:2844-2849`）。因此 `A{\rOther}B` 中即使 A 在 `\r` 之前，整条事件的边距也按 `Other` 计算（仅当事件自身 `Margin*` 非零时例外，`:2844-2849` 的三元判断）。这与"`\r` 只影响其后 glyph"的直觉相悖，是逐 glyph 快照模型的一个例外。

### 1.5 卡拉 OK 的 `reset_effect` 与 `\r` 是两个不同机制

`\kt` 把 `state->reset_effect = true`（`libass/ass_parse.c:837-844`），快照进 glyph（`libass/ass_render.c:2128`），随后在 `ass_process_karaoke_effects` 里被**消费**（`libass/ass_parse.c:1025-1107`）：

```c
if (glyph[i].reset_effect) { has_reset = true; skip_timing = 0; }   // :1038-1041
...
if (start->reset_effect) timing = 0;                                 // :1062-1063
timing = !has_reset * tm_end + skip_timing;                          // :1067
```

它只影响**卡拉 OK 字词计时**（`EF_KARAOKE*`），不碰样式状态。而 `\r` 只碰样式状态，不碰卡拉 OK 计时。若把两者混为一谈会得出"`\r` 会重新计时"的错误结论。另一处需要注意：`\k` 系列会写 `state->effect_timing`，而 `effect_timing != 0` 本身就是 `starts_new_run` 的条件（`libass/ass_render.c:2013-2015`，注释明确"ignore `effect_skip_timing`"）。

### 1.6 渲染缓存：变换在哪一层进入键

| 缓存 | 键内容 | 是否含变换 | 源码 |
|---|---|---|---|
| outline（字形轮廓 / 描边 / 矢量图 / 盒子） | `font,size,face_index,glyph_index,bold,italic,flags`；描边版再加 `scale_ord_x/y,border`；盒子只用一个占位键 | **完全不含变换** | `libass/ass_cache_template.h:88-96`、`:107-114`；`libass/ass_cache.h:59-71` |
| bitmap | `outline` 指针 + **量化后的** `offset`、`matrix_x`、`matrix_y`、`matrix_z` | **含，且已量化** | `libass/ass_cache_template.h:62-69` |
| composite | `filter{flags,be,blur_x,blur_y,shadow}` + `bitmap_count` + `BitmapRef*`（含每个 `bm`/`bm_o`/`pos`/`pos_o`） | 含量化位置，**不含颜色** | `libass/ass_cache.h:84-88`；`libass/ass_cache_template.h:117-132`；构造 `libass/ass_render.c:2587-2602` |
| 每帧 `ASS_Image` | 颜色 + `dst_x/dst_y` + 位图指针 | 颜色在这一层 | `libass/ass_render.c:820-884`，`:210-236` |

关键点：

1. **缩放/字号动画的两条路径代价差别很大，而且默认配置下都被"固定字号"优化掉了**：
   - `\fscx/\fscy`：只改 `info->scale_x/y`（`libass/ass_render.c:2136-2137`），outline 缓存必然命中，bitmap 缓存换键。
   - `\fs`：改 `state->font_size`（`libass/ass_parse.c:465-476`），随后 `info->font_size = |state->font_size · screen_scale_y|`（`libass/ass_render.c:2129-2131`）；**但紧接着 `fix_glyph_scaling`（`:1980-2002`）会在 `hinting == ASS_HINTING_NONE` 时把 `font_size` 改写成固定的 `256.0`**，把原始字号折算进 `scale_x/scale_y` 与 `scale_fix`。于是 outline 键里的 `size` 恒为 256（键定义见 `libass/ass_cache_template.h:88-96`），**默认配置下 `\fs` 动画同样命中 outline 缓存**，代价与 `\fscx` 相当；只有开启了 hinting（`hinting != ASS_HINTING_NONE`）才会变成 `ft_size = scale_y · font_size` 而每次换键。默认 hinting 是 `NONE`（`libass/ass_render.h:74` 的 `ASS_Hinting hinting` 字段由 `ass_render.c:122` 的 `calloc` 置 0，枚举 `NONE = 0` 见 `libass/ass.h:103`）。这也解释了 `fix_glyph_scaling` 的注释（`:1971-1976`）：目的正是让"按字号动画"不被 FreeType 的 grid fitting 破坏。
2. **变换量化发生在建键时**（`quantize_transform`，`libass/ass_render.c:498-649`）：中心位置先除以 `64 >> SUBPIXEL_ORDER = 8` 再 `ass_lrint`（`:541-545`），小数余数进 `key->offset`（`:643-644`），矩阵系数按误差预算量化（`:599-648`）。`restore_transform`（`:651-687`）只保证在 `POSITION_PRECISION` 量级内复原。→ **动画的可见值 = 量化值**；慢速动画会出现离散"台阶"，但不会累积漂移（每帧从绝对时间重算）。
3. **同一 run 内共用 `offset`**：`get_bitmap_glyph(..., !current_info->bitmap_count, flags)`（`:2545-2546`）让后续字形复用首个字形的亚像素余量，避免 run 内互相抖动（`:635-638`、`:1409`）。
4. **描边宽度对缩放做了反向补偿**：`bord_x = 64 * border_scale_x * info->border_x / tr->scale.x / par_scale_x`（`:1466-1470`）。所以 `\fscx` 动画放大字形时描边**视觉上不变粗**，而 `\bord` 动画才改描边；这两者可叠加，命中不同缓存（`BorderHashKey` 含 `border`，`libass/ass_cache_template.h:107-114`）。
5. **颜色/透明度不进 composite 键**：`render_text` 在每帧把 `info->c[]` 直接塞进 `ASS_Image::color`（`:852-878`）。所以 `\t(\alpha...)` / `\t(\c...)` 不产生新 composite，只产生新 `ASS_Image`。但 `alpha` 参与**过滤器标志**：`_a(info->c[0]) != 0xFF` 会加 `FILTER_FILL_IN_SHADOW`，`== 0` 会加 `FILTER_FILL_IN_BORDER`（`:2466-2487`），跨过这两个边界会换 composite 键（因此视觉上可能出现边界处的一次性重算）。
6. **fade 是 run 级、由 run 首个字形决定**：`memcpy(&current_info->c, &info->c, ...)` 后统一 `ass_apply_fade(&current_info->c[i], info->fade)`（`:2500-2502`）。由于 `\fad`/`\fade` 只生效一次（`:650-656`），`fade` 在整条事件内恒定，故不构成"同 run 内取值不一致"的问题——但也意味着 **fade 无法被 `\t` 动画**，`\t(\fade...)` 的载荷即使写了也不会随时间变化。

---

## 2. 交叠与中途重置时的预期结果

以下每条都给出可验证的表达式。记某字段的样式值为 `v₀`、标签目标值为 `a`、当前系数为 `k`；线性插值统一写作 `lerp(v,a,k) = v·(1-k) + a·k`，与源码写法一一对应（`calc_anim`，`libass/ass_parse.c:128-131`，各标签内联同式）。

### 2.1 `k` 的时间优先级（事件的三个边界）

| 时刻 | `k` | 含义 |
|---|---|---|
| `now < Start + t1` | 0 | 完全保持"此处之前的当前值"，**不是**"样式值"；若前面有静态标签，保持的是那个静态值 |
| `t1 <= now < Start + t2` | `pow((t-t1)/(t2-t1), accel)` | 线性/幂次斜坡 |
| `now >= Start + t2` | 1 | 终值 |
| `now == Start + Duration` | **事件不渲染**（`libass/ass_render.c:3384-3385` 右侧开区间） | 若 `t2` 为 0→`Duration`，`k==1` 永不可见 |
| `now == Start` | 渲染（左闭） | `t1=0` 时 `k=0` |

边界细节（均为**当前行为**）：

- `t2 == 0` 被无条件改写为 `Duration`（`libass/ass_parse.c:695-696`），**即使用户显式写了 `\t(0,0,...)`**。
- `t1 < 0`：`delta_t` 变大，`k(0) = -t1/(t2-t1) > 0`，动画在事件开头**已经推进了一部分**。
- `t1 > t2`（显式给出非零 `t2`）：`t < t1` 命中 `k=0`；`t >= t1` 时必有 `t >= t2`，命中 `k=1`。**中间分支不可达**（`:717` 之前的 `:703-706` 与 `:704` 的断言都不会执行），因此 `delta_t` 的无符号回绕（`:697`，`t1=2000,t2=1000` 时得到 `-1000`）**不影响结果**；实际效果是**在 `t1` 处阶跃**。复刻程序已确认：`{\fscx100\t(2000,1000,\fscx300)}A` 在 1500 ms 为 100%、2000 ms 为 300%。注意 `\move` 会交换 `t1/t2`（`:503-507`），`\t` **不会**，两者行为不一致。
- `t1 == t2`：`t<t1` 与 `t>=t2` 已覆盖全部时刻，`assert(delta_t != 0.)`（`:704`）不可达。
- `\t` 的 `t` 用 `render_priv->time - event->Start`（`:698`），`time` 在 `ass_start_frame` 中设为 `now`（`libass/ass_render.c:3064`）→ 绝对时间，不依赖前一帧。

### 2.2 同一字段被多个 `\t` 覆盖：**嵌套 lerp**，不是后者覆盖前者

因为 `\t` 是"就地改写 `state`"，第二个 `\t` 的基准 `v` 是第一个 `\t` 写回后的值：

```ass
{\fscx100\t(0,4000,\fscx200)\t(0,4000,\fscx300)}A
```

`t = 2000`（`k = 0.5`，`accel = 1`）时的实际值（已用 `/tmp/libass-report/tprobe/numsim.c` 复刻验证）：

```
v₁ = lerp(100, 200, 0.5) = 150
v₂ = lerp(150, 300, 0.5) = 225      ← 不是 200（后者覆盖）也不是 250（并集插值）
t = 1000: v₁ = 125, v₂ = 168.75
t ≥ 4000: v₁ = 200, v₂ = 300
t = 0:    v₁ = 100, v₂ = 100
```

两个**窗口不重叠**时，各用各的 `k`：`{\fscx100\t(0,1000,\fscx200)\t(2000,3000,\fscx400)}A` 的复刻结果是 `t=1000 → 200%`、`t=2000 → 200%`、`t=3000 → 400%`（第二个动画从第一个的终值 200% 起步）。若两个窗口顺序颠倒，则第二个（时间上更早的）动画先把值推到终值，第一个随后覆盖——**最终可见值由文本中最后一个 `\t` 的目标值决定，中间态由书写顺序决定**。

**嵌套 `\t` 的真实语义（复刻已确认）**：`{\t(0,4000,\fscx200\t(0,1000,\fscy300))}A` 的结果是"直觉的"——`fscx` 按外层的 4000 ms 走，`fscy` 按内层的 1000 ms 走：

| 时刻 | `fscx`（外层 k，4000 ms） | `fscy`（内层 k，1000 ms） |
|---|---|---|
| 1000 ms | 125% | **300%**（内层已到终值） |
| 2000 ms | 150% | 300% |
| 4000 ms | 200% | 300% |

原因不是"内层 k 泄漏"，而是**载荷截断**：`\t` 的最后一个参数在遇到第一个 `)` 时就结束（`libass/ass_parse.c:333-339`），所以内层 `\t` 的 `)` 同时也关闭了外层的载荷；递归解析的区间恰好止于那个 `)`，内层 `k` 只作用于自己的载荷。而 `memchr(r, ')', end - r)` 不包含 `end` 本身，因此内层 `\t` 的载荷总能延伸到区间末端、总是走末尾分支（`:719-727`），这就是 `assert(!nested)` 不可达的原因（§3.4）。

**真正反直觉的是截断的副作用**：内层 `)` 之后的标签会**逃出动画**，在外层以 `pwr = 1` 立即生效：

```ass
{\t(0,4000,\fscx200\t(0,1000,\fscy300)\frz45)}A
```

`\frz45` 不属于任何 `\t`，解析时 `pwr = 1`，因此**从第一帧起就是 45°**，与两个动画的进度都无关（复刻跟踪显示它由 `d0 n0 pwr=1.00` 处理）。

### 2.3 `\t` 与静态标签的顺序

| 写法 | `t < t1` | 窗口内 | `t >= t2` |
|---|---|---|---|
| `{\fscx150\t(0,1000,\fscx200)}` | 150 | `lerp(150,200,k)` | 200 |
| `{\t(0,1000,\fscx200)\fscx150}` | 150 | 150 | 150（动画被覆盖） |
| `{\t(0,1000,\fscx150)\r}` | `v₀` | `v₀` | `v₀`（动画被抹掉，见 §2.4） |
| `{\r\t(0,1000,\fscx150)}` | `v₀` | `lerp(v₀,150,k)` | 150 |

即：**静态标签在 `\t` 之后 ⇒ 动画对该字段无效；在 `\t` 之前 ⇒ 成为动画起点。**

### 2.4 中途重置：**位置性的阶跃，没有时间语义**

`{\t(0,4000,\fscx200)}ABC{\r}DEF`：

- `ABC`：`scale_x = lerp(v₀, 2, t/4000)`。
- `DEF`：`scale_x = v₀`（样式值），**恒定**，且在 `{\r}` 处出现**一次不连续跳变**。
- 两侧共享同一条布局：`align_lines` 按 `cluster_advance` 求和居中（`libass/ass_render.c:2313-2365`），`compute_string_bbox` 也按 advance 取包围盒（`:886-903`），`device_x` 由 base point 反推（`:2881-2885`）。所以 `ABC` 变宽会把整行（含 `DEF`）的居中位置一起推走——**"被重置的部分"也会因为别人的动画而移动**。
- 若 `\r` 带来不同字体/字号，`split_style_runs` 会因为 `font`/`font_size` 不等而断 run（`:2014-2043`），`composite` 也随位置变化重建；若 `\r` 前后字段完全相同，则**不断 run、composite 键不变**（纯粹的位置/矩阵层面无变化）。

其它"重置"路径的结果：

| 写法 | 位置 | 缩放 | 透明度 |
|---|---|---|---|
| `{\fscx200}X{\r}Y` | 行宽按 X 的 2× advance 计算，Y 正常宽度，整行重新居中 | X 200%，Y 100% | 不变（`\r` 会把颜色 alpha 也复位到样式值，若之前 `\alpha` 动过则此处跳回） |
| `{\alpha&H00&\t(0,4000,\alpha&HFF&)}A{\r}B` | 不变 | 不变 | A 逐渐变透明；B **从一开始就**是样式 alpha（解析顺序决定，与时间无关） |
| `{\bord2\t(0,4000,\bord8)}A{\r}B` | 描边宽度按 `border_x` 换算（`:1466-1470`） | — | — |
| `{\t(0,4000,\be5)}A{\r}B` | — | — | `\be` 被固定回 **0**（`:1103`），不是样式值 |
| `{\t(0,4000,\blur5)}A{\r}B` | — | — | `\blur` 回到 `style->Blur`（`:1104`） |
| `{\move(0,0,100,0,0,4000)}A{\r}B` | 位置**不变**：`\move` 是事件级、只认一次（`:525-530`），`\r` 不碰 `pos_*` | — | — |
| `{\pos(10,10)}A{\r}B` | 同上，仍在 (10,10) | — | — |
| `{\fad(0,1000)}A{\r}B` | — | — | fade 仍是事件级、只认一次（`:650-656`），B 同样在 1 s 内淡出 |
| `{\kt100}A{\k50}B` | — | — | 无样式影响；只把卡拉 OK 计时清零（`:1062-1063`、`:1067`） |

**`\r` 与 `\t` 嵌套**：`\r` 在 `\t` 载荷内仍然无视 `pwr`，因此 `{\t(0,4000,\r)}A ≡ {\r}A`。这与 §1.3 的无参标签（`{\fscx100\t(0,4000,\fscx)}A` 恒为 `100%`，已由复刻程序确认）属于同一类"动画被完全忽略"的写法：二者的共同点是**标签本身不读 `pwr`**，而不是"重置特别强"。

### 2.5 缓存量化对"结果"的影响

- `\frz` 慢速动画：每帧矩阵被量化（`:599-648`），可见角度以 1/8 px 位置精度为限跳变；`POSITION_PRECISION = 8/64 px` 是误差上限（`:45`），不是步长。
- `\fscx` 从 0 附近增长：`\t(\fscx0)` 的目标值不会被钳到最小值（只有负值被钳为 0，`libass/ass_parse.c:441`）。`scale.x = 0` 使 `info->advance.x = ass_lrint(val->advance * scale.x) = 0`（`libass/ass_render.c:1227`），该字形在布局中占 0 宽；随后 `apply_baseline_shear` 因 `!info->scale_x` 把它标为 `skip`（`:2285-2286`），但它**已经**参与了前面的 `preliminary_layout` / `wrap_lines` / `align_lines`（顺序见 `:2840-2870`）。因此动画从 0 开始时会表现为"整行按零宽排好、字逐渐撑开"，而不是"字形从不存在变为存在"。
- 纯透明帧被丢弃：`_a(cur->color) == 0xFF` 的 `ASS_Image` 在拼接阶段释放（`:3415-3424`），所以 `\alpha&HFF&` 的段落**不产生图像**，`detect_change` 会报告内容变化。
- `BorderStyle=4` 的方块背景在 `add_background` 里用 `state->shadow_x/y` 与 `state->c[3]`（`:2759-2790`），所以 `\t(\shad...)` / `\t(\4c...)` 会让方块**按最后解析到的值**（事件级 `state`，即文本末尾的值）运动/变色，而不是逐 glyph——这是 `\t` 作用于事件级字段时的特例，值得在文档里点名。

### 2.6 静态测量：仓库基准图与源码预测一致

`compare/test/sub1.ass:12`：

```ass
Dialogue: 0,0:00:00.00,0:00:04.00,Default,,0,0,0,,{\t(\frz+180)}A
```

- 解析：`t1 = 0`，`t2 = 0 → Duration = 4000`，`accel = 1`。`\frz` 是**绝对值**标签（不做 `+`/`-` 相对解释，相对解释只有 `\fs` 有，见 `libass/ass_parse.c:465-476`），`ass_strtod` 接受前导 `+`（`libass/ass_strtod.c:126-136`），故 `val = 180`；`state->frz = val·k + frz·(1-k)`（`:547-555`），样式 `Angle = 0`。
- 预测：`k = t/4000`，角度 `= 180·t/4000` → 0.5 s: 22.5°，1.5 s: 67.5°，2.5 s: 112.5°，2 s 跨度 90°。
- 对已提交 PNG（`sub1-0500.png` / `sub1-1500.png` / `sub1-2500.png`，320×240 RGBA8）解码 Alpha 掩码后做旋转拟合。基准图是透明背景上的白字，`compare/compare.c:40-60` 的 `blend_image` 对透明像素执行 `dst[3] -= ((dst[3]-0)*k+offs)>>23`，所以 Alpha 通道正好是字形覆盖率；旋转中心固定在画布中心 (160,120)，在 ±80° 范围内以 0.25° 细搜 IoU：

| 对比 | 最佳拟合旋转 | IoU |
|---|---|---|
| 0.5 s → 1.5 s | −45.5° | 0.9715 |
| 1.5 s → 2.5 s | −45.0° | 0.9642 |
| 0.5 s → 2.5 s | −90.75°（−90.0° 时 0.9433） | 0.9509 |

与源码预测的 45°/s、2 s 累计 90°吻合；2.5 s 处**不是** 180°（`+45°` 假设的 IoU 仅 0.42）。这同时验证了 §2.1 的"终值不可见"：事件在 4 s 结束，`k==1` 的那一帧永远不会被渲染。旋转拟合的最优中心是 (160, 120)，即 PlayRes 320×240 的画布中心；`compare/compare.c:352-353` 用目标图尺寸同时设置 storage 与 frame 尺寸（320×240），无 PAR 修正，Alignment=5（subtitle 居中，见 `libass/ass_utils.h:154-170` 的 `numpad2align`）时 `get_base_point` 取 bbox 中心（`libass/ass_render.c:1941-1969`），与测量一致。

> 该测量只验证**幅度与中心**。屏幕上的旋转方向由 `calc_transform_matrix` 的符号约定决定（`libass/ass_render.c:1331-1337` 中 `sz = -sin(frz)`），静态反演无法独立确认方向，故本文不对方向下结论。

---

## 3. 测试证据、当前行为与建议

### 3.1 现有测试覆盖

| 载体 | 内容 | 是否覆盖 `\r` / 交叠 / 边界 |
|---|---|---|
| `compare/test/sub1.ass:12` | `{\t(\frz+180)}A` + 三张期望 PNG | 仅覆盖 `t2==0 → Duration` 的斜坡；**不覆盖** reset、交叠、`t2` 边界 |
| `compare/test/sub2.ass:12` | `{\blur1}Test_`（153 s） | 静态 `\blur`，与动画无关 |
| `test/test.c` | 把一帧渲染并写出 PNG（`:220-266`） | 冒烟，无断言值；CI 用 `meson.yml:235` 调它，仅检查"能跑通" |
| `compare/compare.c` + `compare/meson.build:17-19` | 模糊图像比对，`art-compare` 需要 `-Dart-samples=<外部仓库>` | 仓库内 `compare/test` **没有**接入 `meson test`；只有设置了 `art-samples` 才跑外部样本（`meson.yml:167,181`） |
| `compare/README.md:23-26,44-45` | `<sub>-NNNN.png` 按毫秒渲染 | 机制存在，但没有配套的 `\t`/`\r` 用例 |
| `fuzz/ass.dict:133,163` | 字典含 `\\r`、`\\t` 等关键字 | 只做崩溃/健壮性覆盖，无行为断言 |
| `Changelog:216,230,231,445,463,480,481` | 历史修复记录（嵌套 `\t`、`\t(T,T)` 边界、`\r` 不重置 wrap style、`\r` 回落行样式） | 仅有文字描述，**无对应测试** |

→ **当前没有任何自动化证据能区分"`\r` 在 `\t` 前/后"、"两个 `\t` 交叠"、"`k==1` 不可见"、"嵌套 `\t` 的 `pwr` 传播"这四类行为。** 这是本次报告最需要强调的缺口。§2 与 §3.4 的数值表全部来自逻辑复刻程序（`/tmp/libass-report/tprobe/`），不是来自 libass 本体运行，落地前应以真实构建产生的 compare 基线替换。

### 3.2 与 `\r` 相关的两处"当前行为"值得单独记录（未在文档中说明）

1. **`\r` 不重置 `\pos`/`\org`/`\clip`/`\fade`/`\an`/`\p`/`\pbo`/`\wrap_style`/`alignment`**（对照 §1.2 的字段表）。直觉上"样式重置"容易被理解为"回到初始状态"，实际是"回到样式提供的字段子集"。
2. **同一字段的 `\t` 是嵌套复合而非覆盖**（§2.2）；而**嵌套 `\t` 的载荷在第一个 `)` 处被截断**，内层 `)` 之后的标签会逃出所有动画、以 `pwr = 1` 立即生效（§3.4）。多个 `\t` 连续作用于同一字段时，结果依赖书写顺序与每个窗口自己的 `k`，不是"最后一个 `\t` 说了算"。

### 3.3 静态阅读得到、建议先用测试固化的边界（均为**建议**，不是现状）

| 场景 | 预期（按源码推导） | 依据 |
|---|---|---|
| `{\t(0,4000,\fscx300)}A{\r}B` | A 在 4 s 内 100→300；B 恒为样式缩放 | `ass_parse.c:805-811`、`ass_render.c:2136-2137` |
| `{\r\t(0,4000,\fscx300)}A` | 从样式值开始动画（与上一条 A 相同） | `ass_parse.c:436-444` |
| `{\t(0,4000,\fscx200)\t(0,4000,\fscx300)}A` @2 s | 225%（嵌套 lerp） | `ass_parse.c:670-728`（复刻已确认） |
| `{\t(0,4000,\fscx)}A` | 全程样式缩放（无参忽略 `pwr`） | `ass_parse.c:436-444` |
| `{\t(\frz180)}A` @ `Duration-1` / `Duration` | ≈179.95° / **无图像** | `ass_parse.c:695-696`、`ass_render.c:3384-3385` |
| `{\t(1000,2000,\fscx200)}A` @ 999 / 1000 / 2000 | `k=0` / `k≈0` / `k=1` | `ass_parse.c:699-702` |
| `{\t(2000,1000,\fscx200)}A` @1500 | `k=0`（阶跃在 2000，不回绕成错误插值；与 `\move` 的交换行为不同） | `ass_parse.c:697` vs `:503-507` |
| `{\k50}A{\kt100}B` | 只重置卡拉 OK 计时，不改样式 | `ass_parse.c:837-844`、`:1025-1107` |
| `A{\clip(0,0,10,10)}B` | A 也被裁（事件级） | `ass_parse.c:729-748`、`ass_render.c:2950-2979` |
| `{\kt100}A{\r}B` | `\r` 不影响卡拉 OK 计时，`\kt` 不影响样式 | §1.5 |

### 3.4 被我初稿误判、经复刻实验推翻的一点（记录过程，避免误信）

初稿曾推断 `libass/ass_parse.c:717` 的 `assert(!nested)` 可被 `{\t(0,4000,\fscx200\t(0,4000,\fscy300)\frz45)}` 触发。**这个推断是错的**，我保留记录以便读者不要重复这个错误：

- 按 `ass_parse_tags` 的切分规则（`:312-348`），`\t` 的**最后一个参数**（载荷）在遇到第一个 `)` 时就被整体吞下（`:333-339` 的 `memchr(r, ')', end - r)`），因此内层 `\t` 的 `args[cnt].end` 恒等于本层 `end`，走的是 `:719-727` 的**末尾分支**，不是 `:717` 的递归分支。
- 反过来，递归分支（`:716-718`）只在外层 `\t` 的载荷**确实在 `end` 之前结束**时进入，而那种情况下递归层一开始 `nested = true`，其内部所有 `\t` 仍然会命中"末尾分支"。
- 复刻程序验证：1 331 个结构化用例 + 300 000 条随机用例，`t_calls = 310 332`，**`assert` 违例 0**，`max_depth` 恒为 1。
- 另外 `\t` 的载荷**不支持真正的多层括号嵌套**：`{\t(0,4000,\fscx200\t(0,1000,\fscy300)\frz45)}` 中，内层的 `)` 会提前关闭外层载荷，`\frz45` 落到**外层块**继续解析（复刻程序的跟踪显示它由 `d0 n0 pwr=1.00` 处理）。这与 `Changelog:230` 的"Treat invalid nested `\t` tags like VSFilter"是同一类"畸形嵌套"处理，不是崩溃点。

**真正需要记录的当前行为**（复刻已确认，仍建议用真实构建复核）：

| 写法 | 复刻结果 | 说明 |
|---|---|---|
| `{\t(0,4000,\fscx200\t(0,1000,\fscy300))}` | `fscy` 按 **1000 ms** 走完，`fscx` 按 4000 ms | 各自使用本层 `k`；内层 `)` 同时截断外层载荷（`:333-339`） |
| `{\t(0,4000,\fscx200\t(0,1000,\fscy300)\frz45)}` | `\frz45` **从第一帧起就是 45°**（不属于任何 `\t`） | 内层 `)` 截断外层载荷，`\frz45` 留给外层以 `pwr = 1` 处理 |
| `{\t(0,4000,\fscx200\t(0,1000,\fscy300\t(0,500,\frz90)\fax5))}` | `frz` 按 500 ms 走完；`fax5` **从第一帧起立即生效** | 最内层的 `)` 同时截断所有外层载荷，`\fax5` 逃到最外层 |
| `{\fscx100\t(0,4000,\fscx300\r)}` | 全程 100% | `\r` 无视 `pwr`，抹掉动画（见 §2.4） |
| `{\fscx100\t(0,4000,\fscx)}` | 全程 100% | 无参 `\fscx` 无视 `pwr`（见 §1.3） |
| `{\fscx100\t(2000,1000,\fscx300)}` | `t<2000` 恒 100%，`t>=2000` 跳到 300% | 反向窗口退化为在 `t1` 处阶跃（中间分支不可达，见 §2.1） |

**建议**（不是现状判定）：为上述前三行各加一个 compare 用例，因为它们的"当前行为"在任何文档、Changelog 或测试里都没有记录。第 2、3 行尤其值得固化：一个"看起来在动画里的标签"实际完全不动画，这是作者最容易误判的写法。若确认应改为"支持真正的嵌套"或是"把逃逸标签也并入外层动画"，都属于语义变更而非修 bug，应当先有用例再动手。

### 3.5 建议补齐的测试清单（**建议**）

按"能区分行为"的原则，最小集：

1. **重置在动画之后**：`{\fscx100\t(0,4000,\fscx300)}A{\r}B`，取 1000/2000/3999 ms。
2. **重置在动画之前**：`{\r\t(0,4000,\fscx300)}A`，同一组时间点。
3. **`\r` 在 `\t` 载荷内**：`{\t(0,4000,\fscx300\r)}A`，应恒为样式缩放（复刻已确认 100%）。
4. **双 `\t` 同字段交叠**：`{\t(0,4000,\fscx200)\t(0,4000,\fscx300)}A` @2000 ms，期望 225%。
5. **窗口错位交叠**：`{\t(0,1000,\fscx200)\t(2000,3000,\fscx400)}A` @1000/2000/3000 ms，复刻期望 200%/200%/400%。
6. **无参标签在 `\t` 内**：`{\fscx300\t(0,4000,\fscx)}A`，期望恒为样式缩放。
7. **事件边界四联**：`{\fscx100\t(1000,2000,\fscx300)}A` @ 999/1000/1999/2000 ms（compare 目录支持任意 `NNNN.png`，见 `compare/README.md:23-26`）。复刻期望 100%/100%/约 299.8%/300%。
8. **`t2` 缺省值的终值不可见**：`{\t(\frz180)}A` @ `Duration-1` 与 `Duration`（后者应为空帧，可顺带验证 `ass_render_frame` 不再返回全透明图，`Changelog:9`）。
9. **反向窗口**：`{\t(2000,1000,\fscx200)}A` @1500/2500 ms，与 `\move` 的 t1/t2 交换行为（`libass/ass_parse.c:503-507`）对照。
10. **`\r` 不触碰事件级字段**：`{\pos(10,10)\fscx200}A{\r}B`，验证 B 仍在 (10,10) 且缩放复位。
11. **`\kt` 与 `\r` 分离**：`{\k50}A{\kt100}B{\r}C`，取多个时间点验证卡拉 OK 填充进度。
12. **事件级 `\clip` 的追溯性**：`A{\clip(0,0,10,10)}B`。
13. **嵌套 `\t` 的载荷截断**：`{\t(0,4000,\fscx200\t(0,1000,\fscy300))}A` @1000/2000/4000 ms（各按本层 `k`），以及 `{\t(0,4000,\fscx200\t(0,1000,\fscy300)\frz45)}A` @0/1000/4000 ms（`\frz` 应恒为 45°）。把 §3.4 表格里的前三行固化成基线。

每条都应同时给"期望值"的数值（如上表），否则 compare 只能保证"没变"，不能保证"正确"。

---

## 附录：引用索引（便于核对）

**时间与优先级**
- `libass/ass_parse.c:670-728` — `\t` 解析、`k`、`pwr` 传播、载荷末端的 `)` 截断
- `libass/ass_parse.c:707-708` / `:719-727` — 嵌套时的 `pwr = k`；末尾分支（`nested` 在此置真并终止本层循环）
- `libass/ass_parse.c:333-339` — 载荷在第一个 `)` 处截断（嵌套 `\t` 的实际行为来源）
- `libass/ass_parse.c:695-706` — `t2==0 → Duration`、`k` 三段式
- `libass/ass_parse.c:312-348` — 参数切分；`:333-339` 为最后一个参数的 `)` 截断
- `libass/ass_parse.c:351-352` — `tag()` / `complex_tag()` 宏（无括号参数也入栈）
- `libass/ass_parse.c:128-158` — `calc_anim` / `change_color` / `change_alpha`
- `libass/ass_parse.c:805-811` — `\r`
- `libass/ass_parse.c:261-273` — `lookup_style_strict`
- `libass/ass_render.c:1075-1111` — `ass_reset_render_context`（重置字段全集）
- `libass/ass_render.c:1116-1153` — `init_render_context`（每帧起点、`explicit`）
- `libass/ass_render.c:3384-3385` — 可见区间（右开）
- `libass/ass_render.c:3064` — `render_priv->time = now`

**快照与事件级字段**
- `libass/ass_render.c:2059-2170` — `parse_events` 主循环、glyph 快照、一次性字段清零
- `libass/ass_render.c:2010-2046` — `split_style_runs`（哪些字段差异会断 run）
- `libass/ass_parse.c:489-530` / `606-621` / `657-669` / `622-656` / `584-605` / `729-748` — `\move`、`\pos`、`\org`、`\fad`、`\an`、`\clip` 的门控与"先到先得 vs 最后生效"
- `libass/ass_parse.c:1156-1181` — `ass_event_has_hard_overrides`（`\pos\move\clip\iclip\org\pbo\p`，**不含** `\t`/`\r`）

**缓存与量化**
- `libass/ass_cache_template.h:62-69` / `:88-96` / `:107-114` / `:117-132` — bitmap / glyph / border / filter+ref 键
- `libass/ass_cache.h:59-71` / `:84-88` — `OutlineHashKey`、`CompositeHashKey`
- `libass/ass_render.c:498-649` — `quantize_transform`（含误差预算推导）
- `libass/ass_render.c:651-687` — `restore_transform`
- `libass/ass_render.c:1383-1542` — `get_bitmap_glyph`（矩阵合成、描边反补偿）
- `libass/ass_render.c:2447-2606` — `render_and_combine_glyphs`（flags、fade、ref 位置）
- `libass/ass_render.c:820-884` — 颜色在 `ASS_Image` 层注入
- `libass/ass_render.c:3415-3424` — 全透明图像丢弃
- `libass/ass_render.h:43-47` — 缓存上限与 `SUBPIXEL_ORDER`/`POSITION_PRECISION`

**卡拉 OK 与重置的区别**
- `libass/ass_parse.c:837-868` — `\kt`/`\k`/`\kf`/`\ko`
- `libass/ass_parse.c:1025-1107` — `ass_process_karaoke_effects`（`reset_effect` 消费）

**布局/位置**
- `libass/ass_shaper.c:657-663` — advance 乘 `scale_x/y`
- `libass/ass_render.c:2210-2228` / `:2297-2366` / `:886-903` — 预布局、居中、包围盒
- `libass/ass_render.c:2368-2394` / `:1326-1373` — 旋转中心与变换矩阵
- `libass/ass_render.c:1466-1470` — 描边随缩放的补偿
- `libass/ass_render.c:1980-2002` — `fix_glyph_scaling`（`\fs` 动画与 grid fitting）

**测试与 CI**
- `compare/test/sub1.ass:12`、`compare/test/sub1-{0500,1500,2500}.png`
- `compare/test/sub2.ass:12`、`compare/test/sub2-153000.png`
- `compare/README.md:23-26,44-45` — `NNNN` = 毫秒
- `compare/meson.build:17-19` — `art-compare` 依赖外部 `art-samples`
- `test/test.c:220-266`、`.github/workflows/meson.yml:167,181,235`
- `Changelog:216,230,231,237,445,463,480,481`
