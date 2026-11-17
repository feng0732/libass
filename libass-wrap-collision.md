# libass 窄屏字幕布局与同屏碰撞避让实现分析

> 代码基线：本仓库 `RELEASEVERSION` 标注 0.17.5，核心实现位于 `libass/ass_render.c`（3467 行）、`libass/ass_parse.c`、`libass/ass_font.c`、`libass/ass_render_api.c`。
> 所有引用均为 `文件:行号`，可直接跳转核对。
> 注意一个基线矛盾：`Changelog:2` 提到修复了 `wrap_lines_measure` 中的越界（CVE-2026-61627），但本快照源码中并不存在该函数，换行实现仍叫 `wrap_lines_naive` / `wrap_lines_rebalance`（全树 grep 仅命中 Changelog 一处）。即 Changelog 比源码新，报告以实际源码为准，并在第 8 节列出此差异。

---

## 1. 问题场景与总体结论

窄屏（物理渲染宽度小、或脚本分辨率与画面宽高比不一致导致 letterbox）下，两个因素叠加会让同屏多个事件的最终位置明显不同：

1. **换行宽度按设备像素（缩放后的屏幕坐标）计算**，而不是脚本坐标。屏幕越窄，`max_text_width` 越小，软换行产生的行数越多；
2. 每个事件的**包围盒高度随行数、行内最大字体度量、行间距线性增长**，而碰撞避让 `fix_collisions` 只做矩形垂直方向的整体平移，行数多一行就可能触发完全不同的避让路径。

双语字幕在 ASS 里通常表现为同一事件内的两行（`\N` 硬换行或软换行），或两个时间重叠的独立事件。前者受行度量与对齐控制，后者走碰撞矩形流程；两条路径的决定因素不同，这正是“布局结果明显不同”的根源。

整体管线（`ass_render_event`，`ass_render.c:2798-3016`）：

```
parse_events → split_style_runs → shaping(HarfBuzz/FriBidi) → retrieve_glyphs
→ preliminary_layout → wrap_lines_smart（换行）
→ karaoke → reorder_text（bidi 视觉序 + 行推进）
→ align_lines（水平对齐/两端对齐）→ compute_string_bbox
→ 垂直定位（top/mid/sub）→ 裁剪矩形修正 → 位图合成
→ 生成 EventImages 矩形 → 帧末 fix_collisions（同层事件垂直避让）
```

---

## 2. 自动换行：宽度预算、四种 WrapStyle 与 Unicode 断行

### 2.1 换行宽度预算在屏幕坐标中计算

`ass_render_event` 在换行前计算单行最大宽度（`ass_render.c:2851-2857`）：

```c
double max_text_width =
    x2scr_right(state, render_priv->track->PlayResX - MarginR) -
    x2scr_left(state, MarginL);
wrap_lines_smart(state, max_text_width);
```

`x2scr_left/right`（`ass_render.c:246-262`）把脚本坐标乘以缩放系数映射到设备像素：

- 事件**无硬覆盖且 `use_margins` 为真**时走 `fit_width/fit_height` 路径（letterbox 适配尺寸，`ass_reconfigure` 计算，`ass_render_api.c:44-53`）；
- 否则（含 `\pos`/`\move`/`\clip` 等硬覆盖事件，判定见 `ass_event_has_hard_overrides`，`ass_parse.c:1156`）走完整 `frame_content_width`。

事件级 `MarginL/MarginR` 非零时覆盖样式边距（`ass_render.c:2844-2849`）。**关键推论**：同样的 ASS 脚本，渲染到 320px 宽与 1920px 宽的画面时，换行点完全不同——换行判断用的是字形在设备像素下的墨迹宽度（`wrap_lines_naive` 里的 `len`，`ass_render.c:1748-1749`），窄屏必然折出更多行。

### 2.2 两段式算法：naive + rebalance

`wrap_lines_smart`（`ass_render.c:1901-1933`）依次调用：

1. **`wrap_lines_naive`**（`ass_render.c:1735-1794`）：从行首累加字形，记录最近一个可断点 `last_breakable`。
   - 强制断点：`FORCEBREAK`——`\n` 字形（来自 `\N`；wrap style 2 下 `\n` 也映射为换行，`ass_parse.c:1126-1130`），`break_type=2`（`ass_render.c:1751-1755`）；
   - 软断点：当 `len >= max_text_width`、当前字符不是空格、且 `wrap_style != 2` 时，在最近的可断点断行，`break_type=1`（`ass_render.c:1756-1764`）；
   - 没有任何可断点时 `break_at = -1`，**该行保持超宽（overfull），不会截断单个词**；
   - 行数翻倍扩容，realloc 失败时丢弃上一个断点避免 UB（`ass_render.c:1771-1784`）。
2. **`wrap_lines_rebalance`**（`ass_render.c:1816-1885`）：不改变行数，只把软断点在相邻行间前后移动，当移动后两行长度差 `DIFF(l1_new,l2_new) < DIFF(l1,l2)` 时接受（`ass_render.c:1870-1875`），迭代到收敛。循环条件 `wrap_style != 1`（`ass_render.c:1823`）。
3. **`trim_whitespace`**（`ass_render.c:1657-1722`）：标记行首/行尾/冗余空格为 `skip`，断点处的空白及其后续连续空白一并修剪；随后 `measure_text` 重新度量。

可断点定义（`ass_render.c:1724-1730`）：

```c
#ifdef CONFIG_UNIBREAK
  #define ALLOWBREAK(g,i) (unibrks ? unibrks[i]==LINEBREAK_ALLOWBREAK : g==' ')
  #define FORCEBREAK(g,i) (unibrks ? unibrks[i]==LINEBREAK_MUSTBREAK  : g=='\n')
#else
  #define ALLOWBREAK(g,i) (g == ' ')
  #define FORCEBREAK(g,i) (g == '\n')
#endif
```

### 2.3 四种 WrapStyle 与 `\q`、`\N`、`\h`

| WrapStyle | naive 软换行 | rebalance 均长 | 实际行为 |
|---|---|---|---|
| 0（智能折行） | 是 | 是 | 断词 + 行长均衡 |
| 1（行尾折行） | 是 | **否**（`:1823` 排除） | 先溢出再断，不均长 |
| 2（不折行） | **否**（`:1758` 排除） | 是，但无软断点可移动 | 只有 `\N` 断行；`\n` 被当空格（`ass_parse.c:1131-1134`） |
| 3（智能折行 v2） | 是 | 是 | 当前与 0 走同一路径 |

源码两处 FIXME 明确承认 0/3 未按 VSFilter 语义正确实现：`ass_render.c:1814`（`FIXME: implement style 0 and 3 correctly`）与 `ass_render.c:1899`。样式级默认值来自 `track->WrapStyle`（`ass_render.c:1125`，解析于 `ass.c:575,897-899`），事件内 `\q` 可覆盖，非法值回退到轨道值（`ass_parse.c:905-909`）。Banner 效果强制 `wrap_style = 2`（`ass_parse.c:971`），滚动字幕永不自动折行。

其他文本细节：tab 转空格（`ass_parse.c:1120-1124`）；`\h` 生成 NBSP，不可断（`ass_parse.c:1135-1138`）；`\r` 不重置 wrap style（历史 bug 修复记录 `Changelog:445`）。

### 2.4 Unicode 断行特性（默认关闭）

`ASS_FEATURE_WRAP_UNICODE`（`ass.h:276-287`）启用后用 libunibreak 的 `set_linebreaks_utf32` 按 Unicode 行断算法取断点，并结合 `track->Language` 做语言特化（`ass_render.c:1909-1925`）。**默认关闭时，中日韩文本因不含 ASCII 空格而几乎没有 ALLOWBREAK 机会**——长 CJK 串会成为一整条 overfull 行。头文件也明确警告“无断点的长词仍可能产生超宽行”（`ass.h:280-282`）。这是窄屏中文/日文双语字幕最常见的横向裁剪来源。

---

## 3. 行间距与行高度量

### 3.1 line_spacing 的单位与注入点

API `ass_set_line_spacing`（`ass_render_api.c:135-138`）直接写入 `settings.line_spacing`，文档注明单位是**帧像素**（`ass.h:515-517`），不随 PlayRes 缩放，可为负。它在两个地方生效：

1. **总高度累加**：`measure_text` 末尾 `text_info->height += cur_line * line_spacing`（`ass_render.c:1649`）。注意乘数是 `cur_line`（= 行数−1），即每个行间隙加一次，首行不加；
2. **换行时笔位推进**：`reorder_text` 遇到行断点时
   ```c
   pen.y += lines[lineno-1].desc + lines[lineno].asc + line_spacing;
   ```
   （`ass_render.c:2251-2254`）。

### 3.2 行高 = 行内最大 asc + 最大 desc

`measure_text_on_eol`（`ass_render.c:1576-1592`）在每个行尾结算：`lines[k].asc/desc` 取该行所有字形的最大值（`ass_render.c:1639-1640`），`height` 累加 `scale*(max_asc+max_desc)`。这意味着：

- 双语两行若字号/字体不同，**每行高度由该行最高的字形决定**，混排一个大字就抬高整行；
- 被修剪的行首/行尾空白字形的度量被忽略，但整行修剪为空时保留空白度量以维持行存在（`ass_render.c:1628-1638`）——纯空白行仍占高度；
- 描边宽度参与行结算：`border_bottom` 用 VSFilter 兼容的偏置四舍五入（`ass_render.c:1584-1586`），`border_x` 取**整个事件所有行的最大值**，即使该描边字形远离边缘（`ass_render.c:1589-1591`，注释直接引用 xy-VSFilter RTS.cpp）。

---

## 4. 文字排布：shaping、bidi、对齐与定位

### 4.1 水平排布

- `preliminary_layout`（`ass_render.c:2209-2229`）按 cluster advance 做一次初步摆位，供换行测量使用；
- `reorder_text`（`ass_render.c:2231-2269`）通过 FriBidi 重排映射把逻辑序转视觉序，同时完成行内笔位与行间隙推进；
- `align_lines`（`ass_render.c:2297-2366`）先求所有行的最大宽度 `max_width`，再按 `halign`（左/中/右，numpad 对齐经 `\an`/`\a` 解析，`ass_parse.c:584-605`）和样式 `Justify`（`ass_types.h:132`）给每行计算 `shift` 平移。水平滚动事件强制左对齐（`ass_render.c:2308-2311`）；
- `compute_string_bbox`（`ass_render.c:886-904`）用未跳过字形的笔位与 cluster advance 求 x 范围，y 范围直接由 `lines[0].asc` 与总 `height` 决定。

### 4.2 垂直定位（`ass_render.c:2872-2948`）

- `\pos`/`\move` 定位事件：以对齐基点 `get_base_point`（`:1941-1969`）反推 `device_x/y`（`:2878-2886`），忽略样式边距；
- 顶部字幕：`device_y = y2scr_top(MarginV) + lines[0].asc`（`:2918-2921`）；
- 中部字幕：画面中心减去 bbox 中心（`:2922-2925`）；
- 底部字幕：从 `y2scr_sub(PlayResY - MarginV)` 起，按 `line_position`（API `ass_set_line_position`，0–100）插值，再减去总高度、加回首行 asc（`:2926-2939`）。仅当 `line_pos > 0` 时把越顶结果钳制到顶部（`:2943-2946`），默认路径**不做顶部钳制**——超高字幕的顶部可以越出画面上沿，等待渲染期裁剪；
- 滚动事件沿 y 轴运动并把 clip 矩形收窄到滚动区间（`:2904-2916, 2981-2987`）。

### 4.3 字体度量的来源

- 字形级 asc/desc 在轮廓缓存构造时由 `ass_font_get_asc_desc`（`ass_font.c:659-668`）经 `FT_MulFix(face->ascender/descender, y_scale)` 取得，再在 `get_outline_glyph` 中乘 `scale_y` 写入 `GlyphInfo.asc/desc`（`ass_render.c:1230-1231`）；
- `set_font_metrics`（`ass_font.c:356-389`）刻意模仿 GDI：优先 OS/2 表的 `usWinAscent/usWinDescent`，缺失时退回 typo 度量，再退回 `face->bbox`。注释说明没有 Win 度量的字体本就不被 VSFilter 支持，此时只保证“可读”；
- 因此**字体替换（fontconfig/CoreText/DirectWrite 选到不同字体）会改变 win 度量 → 改变每行 asc/desc → 改变换行后总高 → 改变碰撞位移**，同一份脚本在不同系统上的布局可能系统性不同；
- `fix_glyph_scaling`（`ass_render.c:1980-2002`）处理 FreeType grid fitting 对字号动画的破坏：无 hinting 时固定 256px 轮廓再缩放，有 hinting 时归一化 `scale_y`。

---

## 5. 同屏事件碰撞避让

### 5.1 哪些事件参与碰撞

`detect_collisions` 每个事件初始为 1（`ass_render.c:1137`），以下标签/效果把它清零：

| 来源 | 位置 |
|---|---|
| `\move` | `ass_parse.c:528` |
| `\pos` | `ass_parse.c:618` |
| `\org`（旋转原点） | `ass_parse.c:668` |
| `\t`（动画变换） | `ass_parse.c:694` |
| Banner / Scroll up/down 效果 | `ass_parse.c:970, 1009` |

即凡显式接管位置的事件都不参与自动避让，VSFilter 语义如此。零宽/零高的事件矩形也被跳过（`ass_render.c:3216, 3255`），注释说明 VSFilter 按“非零面积相交”判定碰撞，零面积事件因此被当作固定事件处理（对应 `Changelog:198` 的历史修复）。

### 5.2 分层、排序与两遍算法

帧渲染时先按 `Layer` 再按 `ReadOrder` 排序（`cmp_event_layer`，`ass_render.c:3100-3113`，调用点 `:3399`），然后**每个 Layer 分组各自独立调用 `fix_collisions`**（`ass_render.c:3401-3409`）——不同层之间永不避让，只按层覆盖绘制。

`fix_collisions`（`ass_render.c:3202-3278`）分两遍：

1. **固定事件收集**：每个事件通过 `get_render_priv`（`:3115-3129`）持有跨帧持久的矩形（`ASS_RenderPriv`，按 `render_id` 失效）。上一帧存过位置且高度未变、与其他固定矩形不重叠的事件，先平移回历史位置并加入 `used[]`；高度变化或互相重叠的事件被取消固定资格（`:3219-3247`）。这一机制让字幕在连续帧间保持稳定，避免逐帧跳动；
2. **自由事件找空位**：非固定事件构造当前矩形，调 `fit_rect` 求位移并 `shift_event` 落位，随后把新位置写回 priv 成为新的固定矩形（`:3253-3273`）。

### 5.3 fit_rect 的几何语义

`overlap`（`:3131-3137`）是严格的 x、y 双向非零面积相交。`fit_rect`（`:3172-3199`）：

- 方向 `dir` 来自 `shift_direction`：底部字幕为 −1（向上挪），顶部字幕为 +1（向下挪），在 `ass_render.c:3005` 由 valign 决定；
- 向上时从最高（y0 最大）的固定矩形倒序遍历，向下时正序；只要 x 投影不相交或 y 已分离就跳过，否则把位移累加为“贴着固定矩形边缘”的量（`shift = fixed.y0 - s.y1` 或 `fixed.y1 - s.y0`）；
- 每处理一个事件就重新插入并按 y0 排序（`:3191-3196`），后续事件看到更新后的占用集合。

**重要局限**：`fit_rect` 完全不感知屏幕边界，也不尝试水平方向避让。避让只在垂直轴上做整体平移；当事件被推到画面外时，由 `shift_event`（`:3144-3168`）在平移后对每张位图做上下边缘裁剪（`dst_y<0` 裁顶、`dst_y+h>=height` 裁底，`:3151-3164`）。也就是说避让失败的可观察结果不是重叠，而是**字幕行被静默裁掉**。

### 5.4 碰撞矩形的构成

事件矩形在 `ass_render.c:2996-3005` 生成：

```c
top    = device_y - lines[0].asc - border_top;
height = text_info.height + border_bottom + border_top;
left   = (device_x + bbox.x_min)*par_scale_x - border_x + 0.5;
width  = (bbox.x_max - bbox.x_min)*par_scale_x + 2*border_x + 0.5;
```

描边（含离边很远的 `\bord` 最大值，见 3.2）被算进矩形，PAR 只作用于 x。注释 `:2994-2995` 明确：描边大于边距时 VSFilter 不把事件挪回画面，`top/left` 允许为负。

---

## 6. 长文本与边缘字幕的可观察裁剪结果

把上述机制串起来，窄屏上的可观察现象有确定的代码出处：

1. **无断点长词/CJK 长串（默认无 libunibreak 或 wrap style 2）**：naive 找不到 `last_breakable`，行宽超过 `max_text_width` 也不折。非定位事件的 x 起点固定为 `x2scr_left(MarginL)`（`:2899-2901`），超宽部分向右延伸，最终在 `render_glyph` 的裁剪计算（`:447-471`）中按 `clip_x0/x1` 逐位图裁掉；完全在裁剪区外的字形直接不入图像链表（`:470-471`）。
2. **折行后总高超过画面**：底部字幕 `device_y = scr_bottom - height + asc`（`:2937-2939`），行数越多顶沿越靠上，可越过 y=0。默认 `line_position=0` 时没有顶部钳制（`:2944` 的条件不满足），越过部分在 `render_glyph` 按 `clip_y0` 裁去——表现为**最上面的双语行只剩半截或消失**。
3. **碰撞上推**：多个底部事件同屏时，`fit_rect` 给出的 shift 可把事件顶到画面外，`shift_event` 再做硬裁剪（`:3151-3164`）；被裁事件仍在 `used[]` 中占位，后续事件继续依据它避让，可能产生连锁空移。
4. **裁剪矩形的三种口径**（`:2950-2987`）：无 `\clip` 且启用 margins 时裁整个 frame；有显式 clip 时把脚本坐标换算后再与屏幕边界求交（`:2961-2972`）；垂直滚动再与滚动区间求交。`\iclip` 反裁剪走 `render_glyph_i`（`:319` 起），同样在最后与屏幕坐标求交（`:334, 372-378`）。
5. **BorderStyle=4 不透明底框**：`add_background`（`:2759-2790`）把底框四边用 `FFMINMAX` 钳进画面，框内文字仍按各自规则裁剪，因此可能出现“底框完整、文字被切”的组合。
6. **边缘负坐标**：`top/left` 为负是预期行为（`:2994-2995`），描边和阴影可以从画面边缘外开始绘制，可视部分从物理边缘起算。

---

## 7. 样式覆盖如何改变上述所有结果

### 7.1 三层覆盖

1. **样式表基线**：`ass_reset_render_context`（`:1075-1111`）从 `ASS_Style` 灌入 FontSize、family、粗斜体、`Outline→border_x/y`、ScaleX/Y、Spacing、Shadow、Angle 等；
2. **行内标签**：`ass_parse.c` 在每个 `{...}` 块内增量改写同一 state（`\fs/\fscx/\fscy/\fn/\bord/\xbord/\ybord/\q/\an/\fsp` 等），`\r` 调 `ass_reset_render_context` 回退（`:1075`）。注意 `\r` 不回退 wrap style；
3. **应用级选择性覆盖**：`handle_selective_style_overrides`（`:906-1006`）按 `ASS_OVERRIDE_BIT_*` 位掩码（`ass.h:133-199`）把宿主通过 `ass_set_selective_style_override` 提供的样式混入。用户样式按 PlayResY=288 的基准重新缩放（`:947-954`，描边与模糊在 `:978-985` 同样缩放）；**一旦事件含硬覆盖（explicit），所有选择性覆盖被禁用**（`:932-934`），`ASS_OVERRIDE_BIT_SELECTIVE_FONT_SCALE` 还能豁免字号系数（`:929-930`）。

### 7.2 字号缩放与窄屏的特殊交互

`init_font_scale`（`:1035-1069`）：

- 普通事件 + `use_margins`：用 `fit_width/fit_height`（letterbox 内的适配尺寸）作缩放基准；
- 硬覆盖事件：用完整 `frame_content_width/height`；
- `ass_set_font_scale` 的 `font_size_coeff` 乘进所有 screen/border/blur 缩放（`:1061-1068`）。

另有 VSFilter 兼容怪癖：字形字号在两个维度都按 PlayResY 缩放（`:2129-2131` 注释 “font glyphs use PlayResY scaling in both dimensions”）。窄屏脚本若 PlayResX/PlayResY 比例与画面不一致，水平方向的字距、`\fsp`（hspacing，`:2154-2159, 2205`）与换行宽度的比例关系会与直觉不同——字体本身不横向压扁，但换行预算随真实画面宽度收缩，这是窄屏折行偏多的另一个推力。

### 7.3 度量→布局的传导链

`ScaleY`、替换字体、win 度量差异改变 `asc/desc` → 改变每行高度与 `text_info.height`（第 3 节）→ 改变底部定位 `device_y`（第 4.2 节）与碰撞矩形高（第 5.4 节）→ 改变 `fit_rect` 的位移量。`ScaleX`、`Spacing`、字体替换改变 advance → 改变 `len` 与换行点（第 2 节）。`Outline/\bord` 改变碰撞矩形外扩。任何一个覆盖项变化都可能同时移动换行点和避让结果，双语字幕因此对样式覆盖格外敏感。

---

## 8. 源码与已有测试材料引用索引

### 8.1 关键源码位置

| 主题 | 位置 |
|---|---|
| 换行总入口 / 宽度预算 | `ass_render.c:1901-1933, 2851-2857` |
| naive 折行 / 超宽不拆词 | `ass_render.c:1735-1794` |
| 行长均衡 | `ass_render.c:1816-1885` |
| 空白修剪 | `ass_render.c:1657-1722` |
| 行度量 / line_spacing 注入 | `ass_render.c:1576-1650`（间距在 `:1649`，行推进在 `:2251-2254`） |
| bidi 重排 / 对齐 / bbox | `ass_render.c:2231-2269, 2297-2366, 886-904` |
| 垂直定位 | `ass_render.c:2872-2948` |
| 裁剪矩形修正 / 字形裁剪 | `ass_render.c:2950-2987, 319, 424-471` |
| 碰撞主流程 / fit_rect / shift_event | `ass_render.c:3202-3278, 3172-3199, 3144-3168` |
| 分层排序与逐层避让 | `ass_render.c:3100-3113, 3397-3409` |
| 碰撞开关（标签侧） | `ass_parse.c:528, 618, 668, 694, 970, 1009`；初值 `ass_render.c:1137` |
| GDI 兼容字体度量 | `ass_font.c:356-389, 659-668`；写入字形 `ass_render.c:1230-1231, 1253-1254` |
| 样式重置 / 选择性覆盖 / 字号缩放 | `ass_render.c:1075-1111, 906-1006, 1035-1069` |
| WrapStyle 解析与 `\q` | `ass.c:575, 897-899`；`ass_parse.c:905-909, 971, 1126-1134` |
| 渲染器尺寸/边距/行距 API | `ass_render_api.c:29-146` |

### 8.2 现有测试材料

- **回归图像对比框架** `compare/`（用法见 `compare/README.md`）：`--enable-compare` 构建，按 `<sub>-NNNN.png` 在 NNNN 毫秒渲染并做模糊像素比对，SAME/GOOD/BAD/FAIL 四级。现有用例 `compare/test/sub1.ass`（320×240，旋转动画）、`sub2.ass`（320×180，blur）只覆盖基础渲染，**没有任何专门针对换行或碰撞的基线用例**；
- **PNG 渲染样例程序** `test/test.c`：`ass_set_storage_size` + `ass_set_frame_size` 等尺寸初始化在 `:123-125`，帧渲染在 `:253`，可作为快速复现脚手架；
- **fuzz 桩** `fuzz/fuzz.c`：固定 854×480 渲染器（`:95-110`），以随机 ASS 输入驱动完整管线，历史上换行相关越界（见下）主要靠这类输入暴露；
- **Changelog 相关条目**：Unicode 断行特性 `:94-97`；零面积事件碰撞 `:198`；wrap 0/3 均衡修复 `:296`；折行修复 `:515`；wrap style 1、碰撞、修剪空格 `:551-557`；`ass_set_line_spacing` 修复 `:594`；`\r` 不重置 wrap `:445`。
- **基线矛盾**：`Changelog:2` 的 CVE-2026-61627 指向 `wrap_lines_measure`，本快照无此函数（实际为 `wrap_lines_naive`），说明安全修复尚未合入此树，审阅或回填补丁时需先确认函数改名/重构关系。

---

## 9. 当前行为清单与建议

### 9.1 当前行为（事实）

1. 换行预算按设备像素、含事件边距计算；窄屏天然多折行，折行点随输出尺寸变化。
2. 默认断词只认 ASCII 空格（无 libunibreak 时）；CJK 长串与 wrap style 2 文本可形成 overfull 行并在右缘硬裁剪。
3. WrapStyle 0 与 3 实现路径相同，源码自认未兼容 VSFilter（两处 FIXME）。
4. `line_spacing` 是帧像素单位、按行间隙数注入，可负；行高取行内最大 asc/desc，空白行保留高度。
5. 碰撞避让仅同 Layer 内生效、仅垂直平移、不感知屏幕边界；推不出画面时由位图裁剪兜底，视觉上表现为丢行而非重叠。
6. 固定事件矩形跨帧持久（按 render_id 失效），高度变化或互撞即失固定资格，保证逐帧稳定优先。
7. `\pos/\move/\org/\t` 与滚动/Banner 效果关闭碰撞；显式定位事件同时禁用应用级选择性样式覆盖。
8. 碰撞矩形包含整事件最大描边宽度，允许负的 top/left，不做入画修正。
9. 字体度量模仿 GDI（OS/2 win 度量优先），字体替换会级联影响换行、行高与避让。
10. Changelog 记录的 `wrap_lines_measure` 安全修复不在本快照中。

### 9.2 建议

**面向字幕作者 / 集成方（不改代码即可见效）**

- 窄屏 CJK/双语场景启用 `ASS_FEATURE_WRAP_UNICODE`（需编译 libunibreak）并设置 `track->Language`，获得词内断行机会；对必须 VSFilter 兼容的发行版脚本，则改用显式 `\N` 控制双语行边界，不要依赖自动折行；
- 对同屏多事件（如对话双语双事件）优先用同一事件内 `\N` 而非两个事件：行间由度量保证贴合，不会进入碰撞矩形的垂直堆叠路径；
- 避免在窄屏脚本里使用大 `\bord`：它按整事件最大值外扩碰撞矩形且不入画修正，会放大避让位移；
- 用 `ass_set_line_spacing` 微调时按目标帧像素取值，并验证最窄目标宽度下的总行高，防止底部字幕顶沿越界被裁；
- 宿主侧对关键字幕轨道可通过 `ass_set_frame_size` 前的布局预估（先以目标尺寸渲染一次读取 `ASS_Image` 坐标）检测越界，再调整 MarginV/字号。

**面向库演进（源码级）**

- 补齐 WrapStyle 0/3 的 VSFilter 语义（消化 `ass_render.c:1814, 1899` 两个 FIXME），并为 rebalance 增加宽度约束感知，避免均衡后重新溢出；
- `fit_rect` 可增加屏幕边界约束选项（在保持 VSFilter 默认行为的前提下作为扩展特性）：当垂直空间不足时回退到缩小行距或允许水平错位，而不是静默裁剪；
- 把换行/碰撞纳入 `compare/` 回归基线：新增小 PlayResX（如 128–320 宽）多事件同层、不同 WrapStyle、不同 `line_spacing` 的 `.ass` + PNG 用例，目前该领域零覆盖；
- 核对并回填 CVE-2026-61627 对应的换行越界修复，先厘清上游 `wrap_lines_measure` 与本树 `wrap_lines_naive` 的重构对应关系；
- 碰撞矩形可考虑区分“墨迹 bbox”与“描边外扩 bbox”，给宿主提供更紧的避让选项（当前 `border_x` 取全事件最大值的策略在混合描边样式下过于保守）。
