# 窄屏字幕：自动换行、行距、排布与碰撞避让如何共同决定位置

分析对象：本仓库（libass 0.17.5，`RELEASEVERSION`），主要文件 `libass/ass_render.c`、
`libass/ass_font.c`、`libass/ass_parse.c`、`libass/ass_render_api.c`。

**方法说明。** 结论以源码为准；为得到可观察的数值，本文用 `ctypes` 直接调用系统
`libass.so.9`（发行版 0.17.1）测量 `ASS_Image` 列表（`dst_x/dst_y/w/h`），
并解析仓库自带测试字体和 DejaVu 的 `OS/2`/`head`/`hhea` 表。
本仓库与 0.17.1 在本文涉及的布局、换行、碰撞代码路径上一致（0.17.5 的变更集中在
`wrap_lines_measure` 的越界修复、负 `ReadOrder` 位清除、`\fay` 与 cluster、以及
`ass_render_frame` 不再返回全透明图像）；凡涉及 0.17.5 差异处，下文单独标注。
下文所有 `x/y/w/h` 都是**帧像素**，`x+w` 为右边界、`y+h` 为下边界，坐标系原点在帧左上角。

---

## 0. 结论速览

1. **换行宽度只由 `MarginL/MarginR` 决定，与碰撞无关。** 每个事件独立换行
   （`ass_render.c:2851-2857`），碰撞只在每个事件都已经排好版之后、按层分组做整体平移
   （`ass_render.c:3202`）。所以窄屏上"双语字幕 + 同屏事件"的结果差异，来自
   **换行把事件变得更高**，而不是换行本身受碰撞影响。
2. **碰撞是"整块矩形搬移"**，以事件矩形（`event_images->top/height/left/width`）为单位，
   矩形包含 border 而不包含 shadow；上下方向按 `\an` 的垂直对齐决定移动方向
   （`shift_direction`，`ass_render.c:3005`）。
3. **CJK 默认不换行。** 没有 `ASS_FEATURE_WRAP_UNICODE` 时 `ALLOWBREAK` 只认 ASCII 空格
   和 `\N`（`ass_render.c:1724-1730`），因此窄屏下长中文行是**一条超宽行**，
   左右被帧边界裁掉；开启该 feature 后同一文本会变成两行居中。
4. **行距 = 该行最高字形的 `asc+desc` 与最高 `border_y`，再加 `line_spacing`。**
   `asc/desc` 来自字体 OS/2 Win 度量的绝对和，被 FreeType `REAL_DIM` 归一化，
   所以 **`line_spacing = 0` 时基线间距恰好等于 `FontSize`**（实测 fs=25/30/37/50 →
   pitch=25/30/37/50），而不是字形的视觉高度。
5. **空行（`\N\N`）的"字体"是换行符自身的样式**，因此空行高度可以远大于相邻文本行。
6. **超出帧或超出边距的行只被帧边界裁剪，不被边距裁剪**，并且不会被拉回帧内
   （源码注释明确此点是 VSFilter 兼容行为，`ass_render.c:2994-2995`）。

---

## 1. 从文本到位置的处理链

`ass_render_event()`（`ass_render.c:2798`）对每个事件按固定顺序执行：

```
parse_events()            解析标签、生成 GlyphInfo[]        (ass_render.c:2051)
split_style_runs()        按样式差异切 run                    (:2005)
ass_shaper_find_runs/shape 塑形                              (:2831-2834)
retrieve_glyphs()         取字形轮廓、asc/desc、advance       (:2180)
preliminary_layout()      按 advance 累加 pos，仅用于换行测量   (:2210)
--- 计算 max_text_width（仅由 MarginL/MarginR）              (:2851-2854)
wrap_lines_smart()        naive 换行 + rebalance + 去空白 + 量测 (:1902)
ass_process_karaoke_effects()
reorder_text()            bidi 视觉序重排，并按 lines[] 计算行基线 (:2232)
align_lines()             行内水平对齐/两端对齐                (:2297)
compute_string_bbox()                                        (:886)
apply_baseline_shear()
device_x/device_y         由 margin/对齐/\pos 决定             (:2867-2948)
render_and_combine_glyphs() 量化、栅格化、合成                 (:2447)
event_images->top/height/left/width  碰撞矩形                 (:2996-3003)
```

关键点：**换行发生在 `preliminary_layout` 之后、`measure_text` 之前**，而
`measure_text` 的结果（`text_info->height`、`lines[].asc/desc`）才决定事件的纵向尺寸和
后续碰撞矩形。碰撞阶段（`fix_collisions`）在**所有事件渲染完之后**才运行
（`ass_render_frame`，`ass_render.c:3380-3409`），因此碰撞不会反过来改变换行。

---

## 2. 自动换行（wrapping）

### 2.1 可用宽度

```c
// ass_render.c:2844-2854
int MarginL = (event->MarginL) ? event->MarginL : state->style->MarginL;
int MarginR = (event->MarginR) ? event->MarginR : state->style->MarginR;
double max_text_width =
    x2scr_right(state, PlayResX - MarginR) - x2scr_left(state, MarginL);
```

* `x2scr_left/right`（`ass_render.c:248-262`）在 `use_margins=0`（默认）时就是
  `PlayRes → 帧` 的线性映射并加上 API 边距；在 `use_margins=1` 且非 `explicit` 时改用
  `fit_width` 与居中/贴边偏移。
* 注意此处**没有**减去 `Spacing`、`Outline`、`Shadow`，也**没有**考虑字形的实际墨迹外扩
  （见 §5.3 中 `\fsp` 的副作用）。
* `MarginL/R` 不做范围钳制（`ass_parse.c` 无相关代码，`INTVAL(MarginL)` 只是把字符串转
  `uint32_t`）。当 `MarginL+MarginR >= PlayResX` 时宽度变成 0 或负数，实测会退化成
  "每个词一行"（§6.4）。

### 2.2 `WrapStyle`

`state->wrap_style` 初值取自 `track->WrapStyle`（`ass_render.c:1125`），可被 `\q` 覆盖
（`ass_parse.c:906-910`）。`Banner` 效果强制 `wrap_style=2`（`ass_parse.c:970-971`）。

`wrap_lines_naive()`（`ass_render.c:1736`）：

```c
if (FORCEBREAK(cur->symbol, i)) {          // \N，或 \n 且 wrap_style==2
    break_type = 2; break_at = i;
} else if (len >= max_text_width &&
           cur->symbol != ' ' &&
           (state->wrap_style != 2)) {     // 软换行，style 2 关闭
    break_type = 1; break_at = last_breakable;
}
if (ALLOWBREAK(cur->symbol, i))
    last_breakable = i;
```

其中
```c
#define ALLOWBREAK(glyph, index) (unibrks ? unibrks[index] == LINEBREAK_ALLOWBREAK : glyph == ' ')
#define FORCEBREAK(glyph, index) (unibrks ? unibrks[index] == LINEBREAK_MUSTBREAK  : glyph == '\n')
```
（`ass_render.c:1724-1730`）。

因此：

| WrapStyle | 软换行 | 行宽均衡 | `\n` 是否断行 |
|---|---|---|---|
| 0 | 有 | 有（rebalance） | 否（当空格用） |
| 1 | 有 | **无** | 否 |
| 2 | **无** | 无 | **是** |
| 3 | 有 | 有 | 否 |

实测（640×360，PlayRes 640×360，一句长英文）：

```
ws=0: 两行，右边界 552 / 526
ws=1: 两行，右边界 608 / 477      ← 不均衡：第一行几乎顶到边距
ws=2: 一行 640 宽，左右被裁
ws=3: 与 ws=0 相同
```
Changelog 也记录了这些行为（`Changelog:551` "Support wrap style 1"、`Changelog:515`
"Fix word-wrapping"）。

### 2.3 均衡（rebalance）

`wrap_lines_rebalance()`（`ass_render.c:1817`）只在 `wrap_style != 1` 时运行，逐行尝试把
末词挪到下一行，若两行长度差 `DIFF(l1,l2)` 减少就接受（`:1870-1875`）。
它**不改变行数**，也**不尝试最优解**（注释 `FIXME: implement style 0 and 3 correctly`）。

这一点对双语布局非常关键：同样一句话，`WrapStyle=0` 与 `1` 产生的行宽分布不同，
因而**事件矩形宽度不同**，于是它与同层其它事件是否水平相交也就不同（§3.2），
最终"是否被推动、推多远"都会变。

### 2.4 `ASS_FEATURE_WRAP_UNICODE`

`wrap_lines_smart()`（`ass_render.c:1902`）在 feature 开启且编译了 libunibreak 时才调用
`set_linebreaks_utf32()`，把每个字符的断行许可写入 `text_info->breaks`，随后
`ALLOWBREAK/FORCEBREAK` 改用这张表。

`Changelog:94` 记录了该 feature（"Add ASS_FEATURE_WRAP_UNICODE to allow automatic
linebreaks in accordance")，`ass.h:274-287` 明确写"disabled by default"、
"incompatible with VSFilter"。

**这是窄屏中英双语差异的最大单一来源。** 320×240、PlayRes 320×240、默认 feature：

| 内容 | 结果 |
|---|---|
| 25 字中文 | 1 个 image，`x=0 w=320`，左右都被帧边界裁（自然宽度 372） |
| 同上 + `WRAP_UNICODE` | 2 个 image，各 186 宽、居中，无裁剪 |
| 英文长句 | 自动换到 3 行 |
| 英文 + 中文同层（默认） | 英文 3 行 + 中文 1 条超宽行，总高 ≈ 4 行 + 边距，中文行左右裁剪 |
| 英文 + 中文同层（`WRAP_UNICODE`） | 英文 3 行 + 中文 2 行，共 5 行，最底行 `y+h=240` 被底边裁 |

值得注意的是，即使打开 `WRAP_UNICODE`，最底行仍然在 `y+h = 240` 处贴边并被裁，
因为**碰撞只做纵向平移，不做"缩到能放下"的处理**，而且平移后的事件矩形仍可能越界。

---

## 3. 行间距（line spacing）

### 3.1 逐行度量：`measure_text`

```c
// ass_render.c:1605-1650（节选）
for (i...) {
    if (glyphs[i].linebreak) {           // 行末
        measure_text_on_eol(...);
        max_asc = max_desc = 0; max_border_y = max_border_x = 0;
        scale = 0.5/64;  cur_line++;
    }
    if (empty_trimmed_line && !cur->is_trimmed_whitespace) {
        max_asc = max_desc = 0; ...      // 行首空白不计度量
    } else if (!empty_trimmed_line && cur->is_trimmed_whitespace) {
        continue;                        // 行尾空白不计度量
    }
    max_asc  = FFMAX(max_asc,  cur->asc);
    max_desc = FFMAX(max_desc, cur->desc);
    max_border_y = FFMAX(max_border_y, cur->border_y);
    max_border_x = FFMAX(max_border_x, cur->border_x);
    if (cur->symbol != '\n') scale = 1.0/64;
}
...
text_info->height += cur_line * render_priv->settings.line_spacing;
```

```c
// ass_render.c:1576-1591
text_info->lines[cur_line].asc  = scale * max_asc;
text_info->lines[cur_line].desc = scale * max_desc;
text_info->height += scale * max_asc + scale * max_desc;
text_info->border_bottom = (int)(state->border_scale_y * max_border_y + 0.5);
if (cur_line == 0)
    text_info->border_top = text_info->border_bottom;
text_info->border_x = FFMAX(text_info->border_x,
        (int)(state->border_scale_x * max_border_x + 0.5));
```

由此得到三条不那么直观的规则：

1. **行高取自该行最高字形的 `asc+desc`**，而不是该行墨迹高度；不同字号混排时，
   行高由最大字号决定（实测 `H{\fs80}H\NH` 的第二行高 64，第一行 32）。
2. **`border_top` 只取第一行，`border_bottom` 只取最后一行**（`:1586-1588`）。
   于是 `\bord` 只出现在中间行时，它既不进 `height` 的上沿也不进下沿；
   而出现在第一行时会同时抬高 `border_top`（`border_top` 从未被后续行更新）。
   实测 `{\bord30}AAA\NBBB` 的碰撞矩形比 `AAA\N{\bord30}BBB` 高出 30px。
   `border_x` 则是对所有行取**最大值**（`:1590`），因此一个行首的大 `\bord` 会让整个
   事件在横向上变宽——`Changelog:181` 修的是"借用前一个同帧事件的最大 xbord"这个 bug。
3. **行首/行尾空白不贡献 asc/desc**；但如果整行只有空白，则不清零（`empty_trimmed_line`
   分支），保证空行仍占位置。

### 3.2 基线推进：`reorder_text`

```c
// ass_render.c:2247-2255
if (text_info->glyphs[i].linebreak) {
    pen.x = 0;
    pen.y += double_to_d6(text_info->lines[lineno-1].desc);
    pen.y += double_to_d6(text_info->lines[lineno].asc);
    pen.y += double_to_d6(render_priv->settings.line_spacing);
    lineno++;
}
```

于是**相邻基线间距 = 上一行 desc + 本行 asc + line_spacing**，
而 `text_info->height = Σ(asc+desc) + n_lines_extra * line_spacing`
（`:1583` 与 `:1649`）。两者对最后一行是否含 desc 的处理不同，
这正是"事件总高"与"末行基线到块底"相差一个 desc 的原因。

### 3.3 `asc` / `desc` 的数值来源（字体度量）

```c
// ass_render.c:1230-1231（GlyphInfo）
info->asc  = ass_lrint(asc  * scale.y);
info->desc = ass_lrint(desc * scale.y);

// ass_font.c:661-668
void ass_font_get_asc_desc(ASS_Font *font, int face_index, int *asc, int *desc)
{
    FT_Face face = font->faces[face_index];
    int y_scale = face->size->metrics.y_scale;
    *asc  = FT_MulFix(face->ascender,  y_scale);
    *desc = FT_MulFix(-face->descender, y_scale);
}
```

`face->ascender/descender` 不是 FreeType 的原始值，而是 `set_font_metrics()`
（`ass_font.c:356-389`）按 GDI 行为改写后的值：

```c
TT_OS2 *os2 = FT_Get_Sfnt_Table(face, FT_SFNT_OS2);
if (os2 && ((short)os2->usWinAscent + (short)os2->usWinDescent != 0)) {
    face->ascender  =  (short)os2->usWinAscent;     // Win 度量优先
    face->descender = -(short)os2->usWinDescent;
    face->height    = face->ascender - face->descender;
}
// 否则退回 sTypoAscender/Descender，再退回 face->bbox
```

字号设定走 `FT_SIZE_REQUEST_TYPE_REAL_DIM`：

```c
// ass_font.c:585-594
rq.type = FT_SIZE_REQUEST_TYPE_REAL_DIM;
rq.height = double_to_d6(size);
FT_Request_Size(face, &rq);
```

**这就是"基线间距 == FontSize"的原因。** `REAL_DIM` 让
`(ascender - descender) * y_scale == size`，即 `asc+desc` 恒等于字号；
于是相邻两行 `desc+asc` 正好等于 `FontSize`（两行同字号时）。

实测 DejaVu Sans（`upem=2048`，`usWinAscent=1901`，`usWinDescent=483`，
`sTypoAscender=1556`，`sTypoDescender=-492`，`USE_TYPO_METRICS` 未置位）：

```
fs=20 pitch=20   fs=30 pitch=30   fs=45 pitch=45   fs=60 pitch=60
```

而仓库自带的两套测试字体给出了同样的结论，尽管它们的 OS/2 差别很大：

| 文件 | 家族 | upem | WinAsc/Desc | sTypoAsc/Desc | hhea | USE_TYPO_METRICS |
|---|---|---|---|---|---|---|
| `compare/test/font1.ttf` | Pixel Operator Mono | 1600 | 1300 / 300 | 1300 / -300 | 1300/-300 | **yes** |
| `compare/test/font2.otf` | Aileron | 1000 | 970 / 230 | 770 / -230 | 970/-230 | no |

FontSize=100 时两者实测 pitch 都是 **100**，`H` 的墨迹高度都是 **64**（cap/em ≈ 0.64）。
Aileron 的 typo 度量（770+230=1000，即 1.0 em）与 Win 度量（970+230=1200，即 1.2 em）
相差 20%，但 `REAL_DIM` 归一化后不影响行距——只影响**基线在行盒里的相对位置**。

我还把 `font1.ttf` 的 `usWinAscent` 改成 2300 生成变体，pitch 仍是 100，
而 `H` 的墨迹 y 位置从 185 移到 213（基线相对下移），这印证了
"字号→高度归一化、度量只分配 asc/desc 比例"的模型。

> **对双语字幕的含义**：中英混排时若两种字体（或同一字体的中西文）`usWinAscent`
> 与 `usWinDescent` 的比例不同，**行距不变、但基线在行盒中的位置不同**，
> 视觉上表现为"两种文字一高一低"，且这个偏移会被计入 bbox 与碰撞矩形。

### 3.4 `line_spacing` 的量纲

`ass_set_line_spacing()`（`ass_render_api.c:135`）设置的 `settings.line_spacing`
"will not be scaled with frame size"（`ass.h:514`）。它在三处被直接使用：
`measure_text` 的 `height`（`:1649`）、`reorder_text` 的 `pen.y`（`:2253`），
单位是**帧像素**。实测（PlayRes 320×180、帧 640×360，即 2× 缩放）：
`spacing=0/10/20` → pitch `60/70/80`，即**每单位只加 1 像素，不乘缩放**。
同一现场若 `PlayRes = 帧`，则 `spacing=10` 恰好让 pitch 从 30 变 40。

`line_spacing` 还被用来计算事件总高，因此**它会直接改变碰撞矩形的 `height`**，
进而改变堆叠距离。`Changelog:594`（"Fixed ass_set_line_spacing() (was broken before)"）
是这一路径的历史来源。负值不做钳制：实测 `spacing=-40` 会让行互相覆盖，
且块底仍锚在 `MarginV` 位置上，溢出到帧外后被裁剪。

---

## 4. 文字排布方式（alignment / justification / 位置锚点）

### 4.1 行内对齐：`align_lines`

`align_lines()`（`ass_render.c:2297`）先求每行宽度与最大行宽 `max_width`，
再按 `halign`（`alignment & 3`）和 `justify`（`state->justify`，来自 `style->Justify`）
逐行施加水平偏移。Banner 效果把 `justify` 强制设为 `halign` 并把 `halign` 改成左对齐
（`:2308-2311`）。

* `halign` 决定**行盒锚点**（`HALIGN_LEFT/CENTER/RIGHT`，`ass_types.h:80-82`），
* `justify` 决定**行内文本在行盒里靠哪边**（`ASS_JUSTIFY_*`，`ass_types.h:83-86`）。

对窄屏双语的实际意义：`\an1/\an2/\an3` 只改变 `halign`。实测同一对事件
（一小一大）在 `an1/an2/an3` 下，大行分别占据 `x=11..208`、`259..462`、`507..708`，
而小行跟随同样的对齐基准——即**对齐只影响水平锚点，不改变换行后的行数**。

### 4.2 纵向锚点：`device_y`

```c
// ass_render.c:2917-2947
if (valign == VALIGN_TOP) {
    device_y = y2scr_top(state, MarginV) + text_info->lines[0].asc;
} else if (valign == VALIGN_CENTER) {
    device_y = y2scr(state, PlayResY/2.0) - (bbox.y_max + bbox.y_min)/2.0;
} else {   // VALIGN_SUB（默认）
    double line_pos = state->explicit ? 0 : settings.line_position;
    scr_bottom = y2scr_sub(state, PlayResY - MarginV);
    scr_top    = y2scr_top(state, 0);
    device_y = scr_bottom + (scr_top - scr_bottom) * line_pos / 100.0;
    device_y -= text_info->height;
    device_y += text_info->lines[0].asc;
    scr_y0 = scr_top + text_info->lines[0].asc;
    if (device_y < scr_y0 && line_pos > 0) device_y = scr_y0;   // 顶部钳制
}
```

默认（`VALIGN_SUB`）下：

```
块底 = 帧底 - MarginV(缩放后)
块顶 = 块底 - text_info->height
```

**`MarginV` 度量的是"整个文本块的下沿（含 desc 与 border_bottom）到帧底"**，
而不是基线或墨迹底。`desc` 来自字体度量，所以**同一 `MarginV`、不同字体/字号，
墨迹底到帧底的实际空隙不同**。实测（DejaVu Sans，PlayRes=帧，`\bord0`）：

```
MarginV=0 : 'H' 墨迹 y=335 h=25 → 墨迹底 360；'g' y=339 h=21 → 墨迹底 360
MarginV=10: 'H' y=325 h=32 → 357        ; 'g' y=329 h=31 → 360
```

`'H'` 在 `MarginV=10` 时墨迹底是 357，留了 3px 空隙；`'g'` 是 360，贴底。
也就是说**"字幕底边距"对含降部的文本与不含降部的文本视觉上并不一致**，
而这个差异恰好等于 `desc - 字形下缘`。

`line_position`（`ass_set_line_position`）把块底在 `[scr_top, scr_bottom]` 区间内线性插值，
并在 `line_pos > 0` 且顶越过 `scr_top` 时钳制到 `scr_top`（`:2943-2946`）。
实测 `line_position=100/140` 结果相同（都被钳制），`line_position=-40` 则把整块推出帧外，
`ASS_Image` 列表为空。注意 `explicit`（含 `\pos/\move/\clip/\iclip/\org/\pbo/\p` 的事件，
见 `ass_parse.c:1156-1181`）会强制 `line_pos = 0`，即**忽略 `line_position`**。

### 4.3 `use_margins` 与 API 边距

`ass_set_margins()` + `ass_set_use_margins()`（`ass_render_api.c:88-102`）。
`use_margins=0`（默认）时所有坐标映射到 `frame_content_*`（= 帧减去 API 边距），
`use_margins=1` 时改用 `fit_width/fit_height` 并让上下/左右贴到 letterbox 的两侧
（`x2scr_left/right`、`y2scr_top/sub`，`ass_render.c:248-303`）。
实测帧 640×360 + 上下边距 40：

```
use_margins=0: 文本墨迹在 y=292..320（在 40..320 的内容区内）
use_margins=1: 文本墨迹在 y=332..360（落入下方黑边）
```

`Changelog:208-211` 记录了 0.15.0 起 `ass_set_use_margins(true)` 语义简化为
"放置在整屏"，不再尝试与 pan&scan 对齐。

---

## 5. 碰撞避让（collision avoidance）

### 5.1 事件矩形

```c
// ass_render.c:2993-3005
// VSFilter does *not* shift lines with a border > margin to be within the
// frame, so negative values for top and left may occur
event_images->top    = device_y - text_info->lines[0].asc - text_info->border_top;
event_images->height = text_info->height + text_info->border_bottom + text_info->border_top;
event_images->left   = (device_x + bbox.x_min) * par_scale_x - text_info->border_x + 0.5;
event_images->width  = (bbox.x_max - bbox.x_min) * par_scale_x + 2 * text_info->border_x + 0.5;
event_images->detect_collisions = state->detect_collisions;
event_images->shift_direction = (valign == VALIGN_SUB) ? -1 : 1;
```

要点：

* **矩形是"文本块 + border"，不含 shadow。** 实测 `Shadow=40` 不改变任何 `dst_y`，
  因为 `shadow` 从未进入 `top/height/left/width`。
* `left/width` 用的是 `bbox.x_min/x_max`（`compute_string_bbox`，`:886-904`，
  按 `cluster_advance` 累加，**不含 border 也不含 shadow**），再加 `±border_x`。
* `bbox.y_min` 固定为 `-lines[0].asc`、`bbox.y_max = y_min + height`（`:891-892`），
  即**纵向 bbox 是行盒而不是墨迹**。
* `top` 可以为负（注释明确），碰撞阶段不会把越界的事件拉回帧内。

### 5.2 分组与顺序

```c
// ass_render.c:3397-3409
qsort(priv->eimg, cnt, sizeof(EventImages), cmp_event_layer);   // Layer, 再 ReadOrder
for (i = 1; i < cnt; i++)
    if (last->event->Layer != priv->eimg[i].event->Layer) {
        fix_collisions(priv, last, priv->eimg + i - last);       // 每层一组
        last = priv->eimg + i;
    }
fix_collisions(priv, last, priv->eimg + cnt - last);
```

`cmp_event_layer`（`:3100-3113`）先比 `Layer`，再比 `ReadOrder`
（外部 ASS/SSA 无 `ReadOrder` 时按文件顺序赋值，`ass.c:1582-1584`）。

**层不同 = 完全不做避让**：实测同层 `English(2 行) + Chinese(1 行)` 会把中文顶到
`y=261`，而把中文放 `Layer=1` 后两者都在底端重叠（`y=324/325`）。
`Changelog:553` 记录"Use the event bounding box (instead of vertical position and height)
for collision detection"是这一结构的来源。

### 5.3 `fix_collisions` 与 `fit_rect`

```c
// ass_render.c:3202-3278（结构）
// 第一遍：把所有"已固定"（上一次渲染留下的 render_priv）事件放入 used[]
// 第二遍：对未固定事件调用 fit_rect()，成功后写回 render_priv
```

```c
// ass_render.c:3172-3199
static int fit_rect(Rect *s, Rect *fixed, int *cnt, int dir)
{
    int shift = 0;
    if (dir == 1)               // 向下
        for (i = 0; i < *cnt; ++i) {
            if (s->y1 + shift <= fixed[i].y0 || s->y0 + shift >= fixed[i].y1 ||
                s->x1 <= fixed[i].x0 || s->x0 >= fixed[i].x1) continue;
            shift = fixed[i].y1 - s->y0;
        }
    else                        // dir == -1，向上
        for (i = *cnt - 1; i >= 0; --i) {
            if (s->y1 + shift <= fixed[i].y0 || s->y0 + shift >= fixed[i].y1 ||
                s->x1 <= fixed[i].x0 || s->x0 >= fixed[i].x1) continue;
            shift = fixed[i].y0 - s->y1;
        }
    fixed[*cnt] = ...; (*cnt)++;
    qsort(fixed, *cnt, sizeof(*rect), cmp_rect_y0);
    return shift;
}
```

算法本质是**贪心的一维（纵向）打包**：

1. 只处理"矩形相交面积非零"的事件（`overlap`，`:3131-3137` 用严格不等号；
   注释 `:3214-3215` 说明零面积事件视同固定）。
2. 按 `shift_direction` 从底向上（`VALIGN_SUB`）或从顶向下（其余）逐个推开；
   每接受一个事件就把它的新矩形插入 `used[]` 并按 `y0` 重排。
3. **所有事件最终都被"固定"**（`:3268-3272` 写回 `render_priv`），
   下一次同 `render_id` 的渲染会把这些位置当作已占用。

由此产生的可观察特性：

* **推动是纯纵向的，从不横向重排**，也从不缩小字号或压缩行距。
  窄屏里"两条独立字幕"若在水平方向相交，就只会上下叠起来。
* **`shift` 只累加到"刚好不重叠"**，所以 `used[]` 里可能出现 `y0<0` 的矩形，
  后续事件继续向上排就会整块离开帧（实测 320×240 上堆 6 个额外事件后，
  顶部 5 个 image 的 `h=0` 或只剩 1px，见 §7.4）。
* **`shift_event` 会裁剪**（`:3145-3168`）：`dst_y<0` 时抬升 `bitmap` 指针并减小 `h`，
  超过帧底时减小 `h`，`h<=0` 时把 `h` 置 0、`dst_y` 置 0。这正是"长文本在边缘出现
  被切掉一部分但 figure 仍在列表里"的原因。
* **历史依赖**：`render_priv` 按 `render_id` 复用（`get_render_priv`，`:3115-3129`；
  `render_id` 在 `ass_reconfigure` 里递增，`ass_render_api.c:33`）。如果同一事件在相邻帧
  高度变化，第一遍会打印 `"Event height has changed"` 并清空固定状态（`:3225-3232`）；
  但 `\t` 动画的事件在**最初**就已经 `detect_collisions=0`（`ass_parse.c:694` 的 `tag("t")`），
  所以它根本不参与碰撞，也不会报这条警告。实测 `\t(...\fscy...)` 的事件稳定地不推开邻居。

### 5.4 `detect_collisions=0` 的触发条件

`detect_collisions` 在 `init_render_context` 里置 1（`ass_render.c:1137`），
以下标签会清零（`ass_parse.c`）：

| 位置 | 标签 | 备注 |
|---|---|---|
| `:528` | `\move` | 同时置 `EVENT_POSITIONED` |
| `:618` | `\pos` | 同上；重复 `\pos` 只警告 |
| `:668` | `\org` | 只清 collision，不置 positioned |
| `:694` | `\t` | **只要出现 `\t` 就退出碰撞**，即使动画是恒等变换 |
| `:970` | `Banner` effect | 同时 `wrap_style=2` |
| `:1009` | `Scroll up/down` | |

实测（两个同层事件，第一事件带不同标签，观察第二事件是否被顶起）：

```
plain   : SECOND y=290   ← 被推开
\clip   : SECOND y=290   ← 仍参与
\fad    : SECOND y=290   ← 仍参与
\pbo    : SECOND y=290   ← 仍参与（ass_event_has_hard_overrides 认 pbo，
                             但只有 \org 会清 collision）
\t      : SECOND y=324   ← 不参与，重叠
\pos    : 位置互换        ← 不参与
\org    : SECOND y=324   ← 不参与
\iclip  : 只剩一个 image  ← 不是碰撞问题，是 inverse clip 把另一事件裁没了
```

注意 `\clip`/`\pbo` 会置 `state->explicit`（`ass_parse.c:1156-1181` 的
`ass_event_has_hard_overrides`，列表为 `pos, move, clip, iclip, org, pbo, p`），
从而改变 `x2scr_left/right` 与 `line_position` 的行为，**但不**退出碰撞；
`\fad` 既不清 collision 也不置 `explicit`。这是一个容易被误判为"碰撞差异"的耦合。

---

## 6. 长文本与边缘字幕的可观察裁剪结果

### 6.1 超出右边距的长词（换行并不保证不溢出）

`wrap_lines_naive` 只在**遇到可断点**时换行；一个不可断的长词会一直向右延伸。
`ALIGN`/`justify` 之后它仍然超出 `max_text_width`，而 `render_glyph`
（`ass_render.c:420-494`）只按 `clip_x0..clip_x1`（且最终钳到帧 `0..width`）裁剪：

```c
clip_x0 = FFMINMAX(state->clip_x0, 0, render_priv->width);   // :448
...
if ((b_y0 >= b_y1) || (b_x0 >= b_x1)) return tail;           // :473
```

实测（帧 320×240、PlayRes 320×240、`MarginL=MarginR=40`、一个 70 字符 URL）：

```
image: x=0 y=204 w=320 h=36
```

即**墨迹覆盖整个帧宽，越过了 40px 的左右边距，只被帧边界裁掉**。
`margin` 只影响换行宽度，**不是裁剪矩形**。

### 6.2 纵向：超长文本块从顶部被裁

默认 `VALIGN_SUB` 把块底锚在 `MarginV` 上，块越高顶越靠上。
13 行以上（640×360、FontSize 30）时块顶就变成负值。实测 12 行 + 额外事件：

```
12 lines + EXTRA @640×360:
  image[0]  y=0  h=27   ← 首行只剩 27px（原 32）
  image[1]  y=0  h=33
  ...
  image[11] y=324 h=33
```

以及 320×240（PlayRes 320×180）上 12 行 + EXTRA：

```
  y=0 h=0  ×4（完全裁掉，仍在列表中）
  y=0 h=25
  y=30 h=32 ... y=193 h=32
```

**被完全裁掉的行仍以 `h=0` 的 `ASS_Image` 出现在返回列表中**，
`dst_y` 被 `shift_event` 重置为 0。0.17.5 的 `Changelog:9`
（"ass_render_frame no longer returns fully transparent images"）针对的是
`_a(color)==0xFF` 的**全透明**图像（`ass_render.c:3416`），`h==0` 的图像不在此列——
调用方仍需自行跳过 `w==0 || h==0`。

### 6.3 边缘的"半个字"

`shift_event` 的裁剪是在**位图层面**做的：

```c
// ass_render.c:3150-3164
if (cur->dst_y < 0) {
    int clip = -cur->dst_y;
    cur->h -= clip;
    cur->bitmap += clip * cur->stride;
    cur->dst_y = 0;
}
if (cur->dst_y + cur->h >= render_priv->height) {
    int clip = cur->dst_y + cur->h - render_priv->height;
    cur->h -= clip;
}
if (cur->h <= 0) { cur->h = 0; cur->dst_y = 0; }
```

于是**边缘的字会被水平切掉一段**（不是整体消失），这与你看到的"边缘字幕被裁"一致。
另外注意这段代码用的是 `render_priv->height`（整帧），而不是 `frame_content_height`，
所以 API 边距不构成裁剪边界。

### 6.4 边距导致宽度退化

`max_text_width` 不做钳制（§2.1）。实测（帧 640×360、PlayRes 640×360、
`one two three`、事件级 `MarginL=MarginR`）：

| MarginL=R | max_text_width | 结果 |
|---|---|---|
| 10 | 620 | 1 行，宽 180 |
| 100 | 440 | 1 行，宽 180 |
| 300 | 40 | **3 行**（每词一行），宽 48/48/69 |
| 320 | 0 | 同上 |
| 400 / 1000 / 5000 | 负 | **与 320 完全一致** |

即 `max_text_width <= 0` 时退化为"逐词断行"，且结果与更大的边距**完全相同**——
因为 `len >= max_text_width` 在 `len>=0` 时恒真。这是一个容易在窄屏/极端边距下
观察到的非单调行为。

### 6.5 空行与首尾换行

* **空行贡献高度**：实测（PlayRes=帧、DejaVu Sans、`\bord0`）
  ```
  fs=30: 'H\NH' pitch=30   'H\N\NH' pitch=45 (+15)   'H\N\N\NH' pitch=60 (+30)
  fs=90: pitch 90 / 135 / 180        → 每多一个空行 = +FontSize/2
  ```
  空行高度 = `FontSize/2`，因为空白行的 `scale` 被留成 `0.5/64`（`ass_render.c:1613,1624`），
  而正常行是 `1.0/64`（`:1644`）。
* **空行的高度取决于换行符自身的样式**：`{\fs80}H\N\N{\fs30}H` 的 pitch 是 112，
  而 `{\fs80}H{\fs30}\N\N{\fs30}H` 是 87。因为 `\N` 本身是一个 `symbol=='\n'` 的
  `GlyphInfo`，它的 `asc/desc/font_size` 在 `parse_events` 时由当时的 state 决定
  （`:2121-2131`），而 `measure_text` 中 `if (cur->symbol != '\n') scale = 1.0/64;`
  让换行符那一格保持 `0.5/64`，于是**空行的高度来自该换行符所属样式的一半字号**。
* **行首 `\N` 被忽略**：实测 `\None`、`\N\None`、`\N\N\None` 三者结果完全相同
  （与 `one` 一样）。`Changelog:217`（0.15.0）"Fix positioning of events with leading
  line breaks" 是这一行为的历史来源。
* **行尾 `\N`**：`one\N` 与 `one` 相同；`one\N\N` 则多出半个字号的高度
  （因为多出来的那一行是"空行"而非"行尾换行"）。

---

## 7. 字体度量与样式覆盖的作用

### 7.1 字体度量如何进入几何

| 度量 | 进入位置 | 影响 |
|---|---|---|
| `face->ascender/descender`（Win / typo / bbox） | `ass_font.c:661` → `info->asc/desc`（`ass_render.c:1230`） | 行高、块高、`MarginV` 到墨迹的实际空隙、空行高度 |
| `REAL_DIM` 归一化 | `ass_font.c:585-594` | 使 `asc+desc == FontSize`，因此 `line_spacing=0` 时 pitch==FontSize |
| 字形 `advance` | `ass_render.c:1227` | 换行宽度判定、`bbox` 宽度、碰撞矩形宽度 |
| `border_x/border_y` | `measure_text_on_eol`（`:1586-1591`） | 碰撞矩形四边外扩；**`border_top` 只来自第一行、`border_bottom` 只来自最后一行** |
| `shadow` | 只在 `render_and_combine_glyphs`（`:2471-2524`） | **不进碰撞矩形** |
| `ScaleX/ScaleY`、`\fscx/\fscy` | `info->scale_x/y`（`:1221-1231`） | 直接缩放 advance 与 asc/desc，从而改变换行点与行高 |
| `Spacing`、`\fsp` | `hspacing_scaled`（`:2154-2158`）、`cluster_advance`（`:2205`） | 见 §7.2 |
| `Kerning` | shaper 层 | 改变 cluster 宽度，从而改变换行点 |

**一个具体的窄屏陷阱**：`asc/desc` 用的是字体度量的**绝对和**，与字形实际墨迹无关。
DejaVu Sans 的 `usWinAscent+usWinDescent = 2384/2048 ≈ 1.164 em`，
而 `H` 的墨迹只有 0.73 em。所以在 320×240、FontSize 30 时，
每行的"盒子"是 30px，但"字"只有约 22px；三行英文占 90px 盒子高度，
而墨迹只占 66px，剩下的空隙会被误当成"行距过大"。

### 7.2 `\fsp` / `Spacing` 的横向副作用

```c
// ass_render.c:2205
info->cluster_advance.x += info->hspacing_scaled;
```

`hspacing_scaled` 在 `parse_events` 里按 `screen_scale_x / par_scale_x * scale_x` 缩放
（`:2154-2158`）。它在**每个字形之后**都加一次，包括行尾最后一个字形，因此：

* `bbox.x_max` 比实际墨迹右缘**多出一个 `Spacing`**；
* 居中对齐按这个"多出来的宽度"居中，于是文本整体**左移 `Spacing/2`**。

实测（`\fsp0/5/20/60`，两个字母，帧 640）：

```
sp= 0: x=302 w=36  → 墨迹中心 320.0
sp=60: x=242 w=96  → 墨迹中心 290.0   ← 左偏 30 = fsp/2
```

同时 `Spacing` 也进入 `max_text_width` 的判定（因为 `cluster_advance` 参与了
`wrap_lines_naive` 的 `len`），所以它会**改变换行点**。`Changelog` 中
"Match VSFilter's \fsp effects on bidi and shaping" 与
"Do not apply \fsp to vector drawings" 说明这一路径被反复调整过。

### 7.3 样式覆盖（selective style override）

`handle_selective_style_overrides()`（`ass_render.c:906-1006`）：

```c
state->apply_font_scale =
    !explicit || !(requested & ASS_OVERRIDE_BIT_SELECTIVE_FONT_SCALE);   // :929-930
if (explicit)
    requested = 0;                                                       // :933-934
```

默认 `settings.selective_style_overrides = ASS_OVERRIDE_BIT_SELECTIVE_FONT_SCALE`
（`ass_render.c:159`）。因此：

* **普通（非 `explicit`）事件受 `ass_set_font_scale()` 影响**；
* **`\pos/\move/\clip/\iclip/\org/\pbo/\p` 事件不受影响**（`ass_parse.c:1156-1181`）。

实测 `font_scale=2.0`：

```
"HELLO"                    → 宽 95 → 175，高 33 → 49
"\pos(320,300)HELLO"       → 宽 95 → 95   高 33 → 33   （不变）
"\an1HELLO"                → 宽 95 → 174             （受影响）
"\move(0,0,640,360)HELLO"  → 不变
```

`font_scale` 缩放的是 `screen_scale_x/y`（`init_font_scale`，`:1035-1069`），
**但 `MarginL/R/V` 不跟着缩放**（它们走 `x2scr_*`，用的是 `frame_content_*`）。
结果：`font_scale` 越大，**换行越早、行数越多，而左右边距不变**。
实测同一句长英文：

```
font_scale=1.0 → 2 行，高 37/36
font_scale=1.4 → 3 行，高 39/39/39
font_scale=2.0 → 3 行，高 58/58/58
font_scale=3.0 → 4 行，高 80/69/80/64
```

`ASS_OVERRIDE_BIT_MARGINS` 的用户样式则**整体替换** `MarginL/R/V`（`:993-997`），
不做 PlayRes 缩放（只有 `FontSize/Spacing/Outline/Shadow/Blur` 才乘
`PlayResY/288`，`:950-985`）。所以一个为 288 高脚本设计的用户边距，在 `PlayResY=360`
的脚本上会显得偏小。

### 7.4 与 `PlayRes` 的耦合

`init_font_scale`（`:1035-1053`）里：

```c
state->screen_scale_x = font_scr_w / PlayResX;
state->screen_scale_y = font_scr_h / PlayResY;
```

`font_scr_*` 在非 `explicit` 且 `use_margins` 时为 `fit_*`（letterbox 后的内容尺寸）。
因此**同一 `FontSize` 在 `PlayResY` 不同的脚本里字高不同**，
由此行高、行数、碰撞矩形全部不同。实测同一 `FontSize=30`、帧 640×360：

| PlayRes | 墨迹高度 | 墨迹底 | 行数（长英文） |
|---|---|---|---|
| 320×180 | 49 | 338 | 2 |
| 640×360 | 33 | 357 | 2 |
| 640×480 | 27 | 360 | 2 |
| 1280×720 | 16 | 358 | 2 |

`FontSize` 的语义因此是"相对于脚本坐标系"，而**边距也是脚本坐标系**，
两者的比值才是视觉比例。窄屏调参时若只改帧尺寸而不改 PlayRes（或反之），
会出现"字变小了但行数没变/变了"的跳跃。

---

## 8. 引用材料汇总

### 8.1 源码

| 主题 | 位置 |
|---|---|
| 事件渲染主流程 | `libass/ass_render.c:2798-3016` |
| 换行宽度（margin 决定） | `libass/ass_render.c:2844-2857` |
| `wrap_lines_smart` / `naive` / `rebalance` | `libass/ass_render.c:1902` / `1736` / `1817` |
| `ALLOWBREAK` / `FORCEBREAK` 宏 | `libass/ass_render.c:1723-1731` |
| `trim_whitespace` | `libass/ass_render.c:1655-1722` |
| `measure_text` / `measure_text_on_eol` | `libass/ass_render.c:1605` / `1576` |
| `reorder_text`（基线推进） | `libass/ass_render.c:2232-2269` |
| `align_lines` | `libass/ass_render.c:2297-2366` |
| `compute_string_bbox` | `libass/ass_render.c:886-904` |
| 纵向锚点 / `line_position` | `libass/ass_render.c:2917-2947` |
| `event_images` 矩形 | `libass/ass_render.c:2993-3006` |
| `init_font_scale` | `libass/ass_render.c:1035-1069` |
| 选择性样式覆盖 | `libass/ass_render.c:906-1006` |
| `render_glyph`（帧裁剪） | `libass/ass_render.c:425-494` |
| `shift_event`（位移+裁剪） | `libass/ass_render.c:3145-3170` |
| `fit_rect` / `fix_collisions` | `libass/ass_render.c:3172` / `3202` |
| 按层分组调用碰撞 | `libass/ass_render.c:3397-3409` |
| `cmp_event_layer` / `get_render_priv` | `libass/ass_render.c:3100` / `3115` |
| `detect_collisions` 清零点 | `libass/ass_parse.c:528, 618, 668, 694, 970, 1009` |
| `ass_event_has_hard_overrides` | `libass/ass_parse.c:1156-1181` |
| `\q` / `\an` / `\a` / `\fsp` / `\fs` | `libass/ass_parse.c:906` / `588` / `596` / `461` / `467` |
| `set_font_metrics`（Win/typo 度量） | `libass/ass_font.c:356-389` |
| `ass_font_get_asc_desc` | `libass/ass_font.c:661-668` |
| `ass_face_set_size`（`REAL_DIM`） | `libass/ass_font.c:585-594` |
| `ass_set_line_spacing/position/margins/use_margins/font_scale` | `libass/ass_render_api.c:135/140/88/100/119` |
| `ASS_Feature` / `ASS_OVERRIDE_BIT_*` 文档 | `libass/ass.h:240-289` / `126-203` |
| API 文档（边距、行距、行位置） | `libass/ass.h:430-525` |

### 8.2 已有测试材料

* `test/test.c`：单帧渲染工具（`<image> <subtitle> <time> [<w> <h>]`），
  CI 用它做冒烟测试（`.github/workflows/meson.yml:235`：
  `build/test/libass_test /tmp/out.png compare/test/sub1.ass 0.5`）。
* `compare/`（`README.md` 说明用法）：按 `sub.ass` + `sub-NNNNN.png` 做逐像素回归，
  支持 `-s` 超采样与 `-p` 容差等级。CI 以 `--enable-compare`/`-Dcompare=enabled` 构建
  （`.github/workflows/ghci.yml:232`、`méson.yml:167`、`vm.yml:182`、`qemu-arch.yml:262`）。
* `compare/test/` 现有用例：
  * `sub1.ass`：`PlayRes 320×240`、`FontSize 200`、`Alignment 5`、`Outline 8`、
    单字 `A` 带 `\t(\frz+180)`。**这是最适合扩展为"窄屏+旋转+大边框"碰撞/裁剪用例的样本**。
  * `sub2.ass`：`PlayRes 320×180`、`Pixel Operator Mono`、`FontSize 100`、`\blur1`。
  * `font1.ttf`（Pixel Operator Mono，`upem=1600`，Win 1300/300，有 USE_TYPO_METRICS）、
    `font2.otf`（Aileron，`upem=1000`，Win 970/230，typo 770/-230）。
    **这两套字体恰好覆盖"Win 与 typo 度量不一致"的情况，很适合做行距用例。**
  * 目标 PNG：`sub1-*.png` 为 320×240/8bpc，`sub2-153000.png` 为 320×180/16bpc
    （compare 支持 8/16 bpc RGBA）。
* `Changelog` 中与本文直接相关的行为记录：
  * `:9` 0.17.5 `ass_render_frame` 不再返回全透明图像
  * `:79` 0.17.1 soft-wrap + `WHOLE_TEXT_LAYOUT` 的越界读修复
  * `:94` 0.17.0 新增 `ASS_FEATURE_WRAP_UNICODE`
  * `:181` 0.15.1 修 "事件借用前一事件最大 xbord"
  * `:198` 0.15.1 零面积事件不参与碰撞
  * `:208-211` 0.15.0 `ass_set_use_margins(true)` 语义变更
  * `:217` 0.15.0 行首换行的定位修复
  * `:551` 0.9.8 wrap style 1
  * `:553` 0.9.8 用事件 bounding box 做碰撞
  * `:557` 0.9.8 换行后去空白
  * `:594` 0.9.5 `ass_set_line_spacing()` 修复

### 8.3 本文实测环境

系统 `libass.so.9`（0.17.1）、FriBidi 1.0.13、HarfBuzz 8.3.0、fontconfig；
字体 DejaVu Sans（`upem=2048`、Win 1901/483、typo 1556/-492）与
仓库 `compare/test/font{1,2}`。测量脚本位于 `/tmp/assprobe/`（未纳入仓库）。
所有数值均为 `ASS_Image` 的 `dst_x/dst_y/w/h`（帧像素）。

---

## 9. 当前行为（观察到的，可能出乎预期）

1. **换行宽度与碰撞完全解耦。** 双语字幕的"两条事件是否左右相交"取决于各自的
   `MarginL/R`、`WrapStyle`、`font_scale`，而非同屏是否有别的字幕。
   同一个事件在有/无邻居时换行结果相同。
2. **默认 CJK 不软换行。** 只有 ASCII 空格与 `\N` 可断。窄屏中文是超宽单行，
   左右被帧边界裁；必须显示调用 `ass_track_set_feature(track, ASS_FEATURE_WRAP_UNICODE, 1)`
   才能按 Unicode 断行（需要编译期 libunibreak）。
3. **`line_spacing=0` 时基线间距 == `FontSize`，与字体无关**，因为
   `REAL_DIM` 把 `asc+desc` 归一化成字号。视觉上"行距看起来偏大"的原因是
   `asc+desc` 远大于墨迹高度（DejaVu 为 1.164 em）。
4. **空行高度 = `FontSize/2`（按换行符所在样式），且与相邻行无关**。
   `\N\N` 会让块高增加半个字号。
5. **`border_top` 只取第一行、`border_bottom` 只取最后一行，`border_x` 取所有行最大值。**
   同一 `\bord` 放在首行 vs 中间行，碰撞矩形高度可以差一个 `border_y`。
6. **shadow 不进碰撞矩形**；shadow 很大时会视觉溢出，但不推开邻居。
7. **任何 `\t` 标签都让事件退出碰撞**，即使动画为恒等变换；`\clip/\fad/\pbo/\p`
   反而仍参与碰撞。其中 `\clip/\pbo/\p` 会置 `explicit`（`\fad` 不会），
   从而额外改变 `use_margins`/`line_position`/`font_scale` 的处理。
8. **碰撞几何是"整块矩形"**：只要水平投影相交 1px，就会整块纵向推开；
   即使两行文字在视觉上左右分离，只要矩形相交就会被推。实测把两个事件分别放在
   `MarginL=10` 与 `140`（墨迹间隙 ~40px）时，`bord=30` 会把它们推到两行，
   而 `bord=10` 不会——因为 `border_x` 把矩形撑到相交。
9. **碰撞只向上/下推，从不横向重排、不缩字号、不压行距**；堆到帧外后由
   `shift_event` 逐位图裁剪，顶部可出现 `h=0` 的条目。
10. **`max_text_width <= 0` 时逐词断行，且结果对更大的负宽度不再变化**（饱和）。
11. **`\fsp`/`Spacing` 让 bbox 比墨迹宽出一个 spacing**，导致居中文本左偏 `spacing/2`，
    并使换行点提前。
12. **`font_scale` 缩放字体但不缩放 `MarginL/R/V`**，所以放大字号会同时"减少每行字数"，
    行数可能突增；而对 `explicit` 事件 `font_scale` 完全不生效。
13. **碰撞的 `render_priv` 状态按事件复用**（`render_id`）。同一事件在相邻帧高度变化时，
    会在第一遍打印 `"Event height has changed"` 并重置；`\t` 事件因为不参与碰撞而不会触发该警告。
14. **未被裁剪的悬空 `h=0` 条目仍在返回值里**（0.17.5 只过滤全透明 color）。

---

## 10. 建议

按"是否会影响现有 VSFilter 兼容性"分层给出。

### 10.1 用户可立即采用的规避（不改代码）

1. **窄屏强制开 Unicode 断行**：`ass_track_set_feature(track, ASS_FEATURE_WRAP_UNICODE, 1)`；
   若目标设备未必编译 libunibreak，则**在字幕文本里主动插入 `\N`**
   （中文按语义断句），这是当前唯一可靠的跨环境方案。
2. **双语用两条独立事件并显式设置 `Layer`**。同层会被碰撞推动（结果依赖 ReadOrder 与
   矩形相交），不同层则完全不动——后者在窄屏上更可预测。若坚持同层，按
   「上层 vs 下层」的期望顺序排列 `Layer`/`ReadOrder`，并为两者设置一致的
   `MarginL/R`，让它们的换行宽度一致、矩形稳定相交。
3. **需要精确控制位置时用 `\pos` + `\an`**，并显式给 `\an` 指定垂直锚点；
   这会自动退出碰撞，但需要注意它同时会关闭 `line_position`/`use_margins` 的映射
   与 `font_scale`。
4. **调行距优先改 `Spacing`/行距 API 之前先确认字号语义**：`line_spacing=0` 已经是
   `pitch == FontSize`；想让视觉行距变小，应减小 `FontSize` 或用负的 `line_spacing`
   （负值会被接受，但会造成相邻行覆盖，慎用）。
5. **不要把边框放在多行事件的首行或末行之外的位置来"微调"**：它不会影响
   `height` 的上下沿，却会影响 `border_x` 与首末行外扩，容易造成
   "看得见的重叠/看不见的推开"。

### 10.2 应用侧可做的健壮性处理（即使 libass 不变）

1. **遍历 `ASS_Image` 时跳过 `w<=0 || h<=0`**，不要假设每个条目都可见。
2. **对 `\t`/`\pos`/`\move`/`\org`/Banner/Scroll 事件不要依赖碰撞结果**；
   如果应用自己实现了字幕位置策略，可考虑对两类事件分别处理。
3. **需要"永不越界"的输出（如硬字幕烧录、直播）时，在渲染后自行检测越界**：
   `dst_x<0 || dst_y<0 || dst_x+w>frame_w || dst_y+h>frame_h`，并做降级
   （缩小字号 / 减少行数 / 手动换行）。
4. **`ass_set_font_scale()` 与 `MarginL/R` 一起调**：只调前者会改变换行点。
5. 需要固定"到帧底的视觉距离"时，**用 `\pos`/`\an` 或按字体 `desc` 补偿 `MarginV`**，
   因为 `MarginV` 锚定的是包含 desc 的行盒底，而非墨迹底。

### 10.3 libass 侧可考虑的改进（需评估兼容性）

1. **文档化 `line_spacing` 的量纲与"pitch == FontSize"的结论**（`ass.h:512-517` 目前只说
   "in pixels, not scaled with frame size"），并说明 `asc+desc` 由 `REAL_DIM` 归一化。
2. **在 `ass.h` 的 `ASS_Image` 段落里明确 `h==0` 条目可能出现**（现在只说明
   stride/padding 与 color 变换），或考虑在 0.17.5 "不再返回全透明图像"的同一处
   也过滤 `h==0`——但这会改变 `ASS_Image` 列表长度，需要 ABI/行为评审。
3. **考虑在 `wrap_lines_naive` 里对"整行无断点"的情况给出可观察的降级**（如日志警告），
   便于上游发现"字幕超出帧"的成因；目前只能靠调用方事后检测。
4. **`max_text_width <= 0` 的饱和行为**（逐词一行、且对更大边距不敏感）可以考虑
   在解析/渲染时钳制为 ≥1，并记录一次 `MSGL_WARN`，避免"边距越大结果不变"的困惑。
5. **`border_top/border_bottom` 的取值规则（首/末行）值得讨论**：它与 `border_x` 的
   "取最大"不对称。若要统一会改变大量既有定位结果，需与 VSFilter 行为比对后再决定；
   短期内至少应在 `measure_text_on_eol` 处加注释说明这是 VSFilter 兼容行为
   （现在只有 `border_x` 那一行有注释）。
6. **`\t` 无条件退出碰撞**（`ass_parse.c:694`）可以改成"仅当动画实际改变几何时退出"，
   但这会改变与 VSFilter 的差异，需谨慎；当前至少可以在文档中写明。
7. **为窄屏/双语/碰撞补充 `compare/` 用例**：
   * 新增 `sub3.ass`：`PlayRes 320×240`，同层英文（自动换行）+ 中文（无 Unicode 断行），
     配 `WRAP_UNICODE` 开/关两张目标图，锁定 §2.4 的行为；
   * 新增 `sub4.ass`：两事件 `MarginL` 使矩形在 `border_x` 边界上刚好相交/不相交，
     锁定 §5.3 的 `fit_rect` 行为；
   * 新增 `sub5.ass`：`\N\N` 与 `{\fs}` 穿插的空行，锁定 §6.5 的 `FontSize/2` 规则；
   * 新增 `sub6.ass`：12 行以上长块 + 额外事件，交付含 `h=0` 条目的目标图，
     锁定 §6.2 的裁剪行为。
   用仓库已有的 `font1.ttf`/`font2.otf` 就能覆盖"Win/typo 度量不一致"的情形
   （`compare/README.md` 的目录约定与 PNG 命名规则直接适用）。
8. **测试工具**：`test/test.c` 目前只输出 PNG，不暴露 `ASS_Image` 列表；
   若希望把本文这类"位置/裁剪"断言纳入 CI，可考虑给 `test` 增加一个
   `--dump-images` 模式（输出 `type,dst_x,dst_y,w,h`），这样可以不依赖像素比对
   就能回归 §5 的碰撞几何。

---

## 附录 A：可复现的最小实验（伪码）

以下实验都用同一套参数：`PlayRes=帧`、`FontSize=30`、`MarginL/R/V=10`、
`ScaledBorderAndShadow=yes`、`Outline=2`、`Shadow=0`、`Alignment=2`。

```text
A. 换行与碰撞解耦
   事件1: "This is a rather long English subtitle line for testing wrap behaviour"
   事件2: "Second"
   结果: 事件1 换 2 行，事件2 被推到 y=290；把事件2 的 Layer 改成 1 则两者都贴底。

B. CJK 不换行
   单事件: 25 字中文，320×240
   默认      → 1 image, x=0 w=320（左右裁剪）
   WRAP_UNICODE → 2 images, 各 186 宽、居中

C. 行距 == 字号
   "H\NH"，FontSize = 20/30/45/60 → pitch = 20/30/45/60
   "\fsp60" 两字母 → 墨迹中心左偏 30px

D. 空行
   "H\N\NH" → pitch = FontSize + FontSize/2
   "{\fs80}H\N\N{\fs30}H" → 空行按 \N 当时样式的半字号计

E. 碰撞矩形含 border 不含 shadow
   "FIRST"/"SECOND" 同层：Shadow=40 时 dst_y 不变；
   Border 放到首个事件且 MarginL 间隙 40 → bord=30 时被推开，bord=10 时不动

F. 边缘裁剪
   12 行块 + "EXTRA"：顶部若干 image 的 h 减小或为 0；
   320×240 上额外堆 5 个事件后，顶部 5 个 image 全为 h=0

G. 超宽不可断词
   "MarginL=MarginR=40" + 70 字符 URL → 单个 image x=0 w=320（越过边距，只被帧裁）

H. `\t` 退出碰撞
   事件1 带 "\t(0,9000,\fscy100)" 时，事件2 不再被顶起（y=324 而非 290）
```

## 附录 B：关键代码片段索引（便于 review）

* 换行判定条件（是否软换行）：`ass_render.c:1756-1758`
* 断点记录：`ass_render.c:1763-1765`
* 行数上限与 realloc 失败回退：`ass_render.c:1771-1784`
* 行高与 border 的逐行累积：`ass_render.c:1617-1645`
* 块高公式：`ass_render.c:1583` + `:1649`
* 基线推进：`ass_render.c:2249-2254`
* 事件矩形：`ass_render.c:2996-3003`
* 层分组：`ass_render.c:3401-3409`
* 推开方向：`ass_render.c:3005`
* 位图级裁剪：`ass_render.c:3150-3164`
* 度量归一化：`ass_font.c:585-594`
* Win/typo 度量选择：`ass_font.c:356-389`
