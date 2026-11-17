# libass 字体回退与 shaping 分析：缺字、混合脚本如何改变字形簇与布局

> 分析对象：本仓库 libass（分支 `task/37`，基线提交 `21b5a54`）
> 范围：字体选择（font selection / fallback）、Unicode 脚本分段、组合字符（combining marks）与 HarfBuzz 字形簇（cluster）、回退字体度量差异、样式范围（style run）边界，以及它们在渲染输出上可观察到的结果。
> 约定：行号均指向本仓库当前源码；技术名词保留英文（shape run、cluster、face、advance、ascender/descender 等），以便与代码对照。

---

## 目录

1. [渲染管线总览：一次事件文本经过哪些阶段](#1-渲染管线总览)
2. [字体选择：逻辑字体、多 face 与逐码位回退](#2-字体选择逻辑字体多-face-与逐码位回退)
3. [Unicode 脚本分段：Common/Inherited 的上下文继承](#3-unicode-脚本分段commoninherited-的上下文继承)
4. [shape run 的切分规则](#4-shape-run-的切分规则)
5. [HarfBuzz shaping 与字形簇：组合字符、连字、多字形 cluster](#5-harfbuzz-shaping-与字形簇)
6. [回退字体度量不同带来的布局后果](#6-回退字体度量不同带来的布局后果)
7. [组合标记边界与跨 run 的 shaping 上下文](#7-组合标记边界与跨-run-的-shaping-上下文)
8. [可观察结果汇总（症状 → 触发条件 → 代码根因）](#8-可观察结果汇总)
9. [现有测试与验证材料](#9-现有测试与验证材料)
10. [当前实现 vs. 建议（明确区分）](#10-当前实现-vs-建议明确区分)

---

## 1. 渲染管线总览

单个 ASS 事件在主渲染函数 `ass_render_event()`（`libass/ass_render.c:2797` 起）中的处理顺序是理解所有现象的基础：

| 顺序 | 阶段 | 入口 | 关键产物 |
|---|---|---|---|
| 1 | 解析 override tags，逐码位建 `GlyphInfo` | `parse_events()` `ass_render.c:2051` | 每个 Unicode 码位一个 `GlyphInfo`（`ass_render.h:126-181`），记录逻辑字体 `info->font`、样式参数 |
| 2 | 按样式切 style runs | `split_style_runs()` `ass_render.c:2005-2047` | `info->starts_new_run` 标记 |
| 3 | 逐码位选 face/字形、定脚本、切 shape runs | `ass_shaper_find_runs()` `ass_shaper.c:879-911` | `info->face_index`、`info->glyph_index`、`info->script`、`info->shape_run_id` |
| 4 | FriBidi bidi + HarfBuzz shaping | `ass_shaper_shape()` `ass_shaper.c:971-1039` | 字形索引、offset/advance、`GlyphInfo.next` 多字形链 |
| 5 | 取轮廓、加 hspacing | `retrieve_glyphs()` `ass_render.c:2180-2207` | `info->outline`、`cluster_advance` |
| 6 | 预布局、换行 | `preliminary_layout()` `:2210`、`wrap_lines_smart()` `:1902` 起 | `linebreak` 标记 |
| 7 | 视觉重排、对齐、剪切 | `reorder_text()` `:2232`、`align_lines()` `:2297`、`apply_baseline_shear()` `:2271` | 最终 `pos` |
| 8 | 位图生成与组合 | `render_and_combine_glyphs()` `:2447` | 输出图像 |

关键点：**字体回退发生在第 3 步（shaping 之前），按码位逐个决策**；HarfBuzz 在第 4 步看到的已经是"每段一个具体 face"的输入。这与浏览器/Pango 那种"shaping 引擎内部驱动 font fallback"的模型有本质区别，后文所有现象都源于此。

---

## 2. 字体选择：逻辑字体、多 face 与逐码位回退

### 2.1 两级结构：一个逻辑 `ASS_Font` 挂最多 10 个物理 face

样式中的 family（含粗/斜/竖排描述符 `ASS_FontDesc`）对应一个缓存的逻辑字体对象 `struct ass_font`（`libass/ass_font.h:42-50`）：

```c
#define ASS_FONT_MAX_FACES 10          // ass_font.h:37
struct ass_font {
    ASS_FontDesc desc;
    int faces_uid[ASS_FONT_MAX_FACES];
    FT_Face faces[ASS_FONT_MAX_FACES];
    struct hb_font_t *hb_fonts[ASS_FONT_MAX_FACES];
    int n_faces;
};
```

逻辑字体构造时只加载主 face（`ass_font_construct()` → `add_face(..., ch=0)`，`ass_font.c:565-583`、`499-550`）。**后续每遇到一个主 face 不覆盖的码位，就向同一个逻辑字体追加一个回退 face**，直到 10 个上限。

### 2.2 逐码位查找与回退触发链

`ass_font_get_index()`（`ass_font.c:708-773`）对每个非控制码位执行：

1. 依次扫描已挂载的 face，用 `ass_font_index_magic()` 处理 symbol/SJIS/GB2312/Big5/Wansung/Johab 等非 Unicode cmap 的码点换算（`ass_font.c:327-354`，含旧阿拉伯 PUA 字体映射 `convert_unicode_to_legacy_symbol()` `:250-270`，数据表在 `legacy_arabic_cmap/`），再 `FT_Get_Char_Index()` 查字形（`:726-733`）。
2. 全部 miss 时打日志 `"Glyph 0x%X not found, selecting one more font..."`（`ass_font.c:737-740`），调用 `add_face()` → `ass_font_select()`（`ass_fontselect.c:884-941`）。
3. `ass_font_select()` 的四级回退顺序（`ass_fontselect.c:894-929`）：
   - 按原 family 在字体库中匹配（`select_font()` → `find_font()`，`:812-871`），其中 `find_font()` 对每个候选 face 调 `check_glyph()` 验证**该码位确实存在**（`:759`），且同 family 内粗/斜变体覆盖率不同也能跨变体命中（注释 `:751-758`）；
   - 配置的默认 family（`family_default`，`:898-906`）；
   - 系统 provider 的 `get_fallback()` 回调（`:908-920`），返回的 family 以 `match_extended_family=true` 再匹配（即允许命中 `extended_family` 字段，`ass_fontselect.h:197-203`）；无 family 时以 `"Arial"` 作为模式（`:910-911`）；
   - 最后的默认路径 `path_default`（`:922-929`）。
4. 全部失败则记录 `MSGL_ERR`：`"Glyph 0x%X not found in font..."`（`ass_font.c:760-763`）与 `fontselect: failed to find any fallback with glyph 0x%X...`（`ass_fontselect.c:936-938`），`glyph_index` 保持 0。
5. 同一 provider UID 的 face 不重复挂载（`ass_font.c:517-523`）。

三个平台 provider 的 `get_fallback` 实现差异：

- **fontconfig**（Linux，`ass_fontconfig.c:263-312`）：对 `sans-serif` 模式做一次 `FcFontSort()` 并缓存（`cache_fallbacks()` `:223-261`，特意删除 `FC_LANG` 避免 locale 干扰排序，`:243-245`），然后按 `FC_CHARSET` 线性找第一个含该码位的 family；并集 charset 里都没有则直接返回 NULL（`:285-289`）。
- **DirectWrite**（Windows，`ass_directwrite.c:509-565`）：用单码位建 `IDWriteTextLayout`，靠 dummy renderer 截获实际选用的字体，再用 `IDWriteFont_HasCharacter()` 二次确认（`:557-564`）。
- **CoreText**（macOS，`ass_coretext.c:254-292`）：`CTFontCreateForString()` 对单字符串做系统级 cascade，取 family 名。

嵌入式字体（`ass_add_font` / 字幕内挂字体）由独立的 embedded provider 承载（`ass_fontselect.c:1016-1034`），其 face 与系统 provider 的 face 进入同一个 `font_infos` 数据库，因此回退可以在挂字体与系统字体之间互相串联。

### 2.3 10-face 上限的可观察后果

`add_face()` 在 `n_faces == ASS_FONT_MAX_FACES` 时直接返回 -1（`ass_font.c:508-509`）。一段含 10 种以上不同缺字脚本的文本（例如拉丁正文中混排 CJK、阿拉伯、天城文、泰文、表情符号等），第 11 种及以后的缺字无法再获得 face，`glyph_index=0` 进入 HarfBuzz，最终渲染为 `.notdef`（空心框/空白），即使系统里明明装了覆盖该码位的字体。这是一个硬上限，不随内存或缓存调整。

---

## 3. Unicode 脚本分段：Common/Inherited 的上下文继承

`ass_shaper_determine_script()`（`ass_shaper.c:756-799`）为每个码位调 HarfBuzz 的 `hb_unicode_script()` 取 `hb_script_t`，然后做两遍扫描：

1. **前向扫描**（`:765-782`）：`HB_SCRIPT_COMMON`（空格、数字、标点等）和 `HB_SCRIPT_INHERITED`（绝大多数 combining mark、variation selector 等）继承**前一个**已定脚本的码位的 script；若前面还没有有效 script（UNKNOWN 不算，`:772-778`），标记需要反向扫描。
2. **反向扫描**（`:785-798`）：行首的 Common/Inherited 码位从**后一个**已定 script 的码位继承。

### 3.1 对混合脚本文本的实际影响

- 拉丁句子 `"Hello, 世界"`：逗号、空格是 Common，跟随 Latin；CJK 二字各自得到 `HB_SCRIPT_HAN`。脚本变化点在 `ass_shaper_find_runs()` 中成为 shape run 边界（见第 4 节）。
- 组合标记（如 U+0301 COMBINING ACUTE ACCENT）的 script 是 INHERITED：基字符是 Latin 它就是 Latin，基字符是 Cyrillic 它就是 Cyrillic。**正常顺序下基字符与标记不会因 script 不同而被切开**；但标记落在 run 起点（例如标记前正好有样式 override 边界）时，反向继承会让它取后面的 script。
- 中性标点的归属决定它进哪个 HarfBuzz shape plan：`props.script` 会影响 OpenType 特性的选择（如 `locl`、阿拉伯/Arabic 呈现形态、各 Indic 脚本的 reordering 规则），同一标点在不同 script 段中可能产出不同字形或不同定位。
- 数字在拉丁与阿拉伯文本之间的 script 归属（跟随上下文）也决定其 bidi 行为与 shaping 语言。

### 3.2 script → language 映射

每个 shape run 的 HarfBuzz language 由 `hb_shaper_get_run_language()`（`ass_shaper.c:602-619`）决定：用户通过 `ass_shaper_set_language()`（`:931-941`，对应 track 的 `Language` 头）显式设置时全局覆盖；否则用 `script_to_language()`（`:532-593`）的静态表（Latin→`en`、Arabic→`ar`、Hiragana/Katakana→`ja` 等，注释说明仿照 Pango），表中没有的 script 回退到 `hb_language_get_default()`（`:615-616`）。language 直接影响语言相关字形（如塞尔维亚 Cyrillic 变体，函数注释 `:927-929`）。

---

## 4. shape run 的切分规则

`ass_shaper_find_runs()`（`ass_shaper.c:879-911`）在脚本判定和 ignorable 标记之后，逐码位递增 `shape_run_id`。相邻两码位 **i-1 与 i** 满足以下任一条件即断开（`:896-908`）：

```c
if ((last->font != info->font ||
        (!info->skip && last->face_index != info->face_index) ||  // 回退 face 变化
        last->script != info->script ||                            // 脚本变化
        info->starts_new_run ||                                    // 样式 override 边界
        (!shaper->whole_text_layout && info->hspacing) ||          // 非零 \fsp
        last->flags != info->flags))                               // DECO_* 位变化
    shape_run++;
else if (info->skip)
    info->face_index = last->face_index;                           // 隐形字符借脸
```

逐条含义：

1. **逻辑字体不同**（`\fn` 换 family）。
2. **face_index 不同**：这正是字体回退改变 run 结构的位置——同一行里主字体与回退字体的码位被切成不同 shape run，分别用不同 `hb_font_t` shaping（`shape_harfbuzz()` 中每个 run 调 `get_hb_font()`，`ass_shaper.c:692`、`466-509`）。
3. **script 不同**：见第 3 节。
4. **`starts_new_run`**：`split_style_runs()`（`ass_render.c:2005-2047`）依据颜色、字号、粗斜体、`\fscx/\fscy`、描边、阴影、旋转、剪切、`\fsp`、family、竖排等**全部**样式字段差异设置；另外 `trim_whitespace()` 也会在修剪后的行首设置该标记（`ass_render.c:1684`、`:1718`）。
5. **非零 hspacing**：仅在未启用 whole text layout 时断开（VSFilter 兼容，见 Changelog 0.17.0 "Match VSFilter's `\fsp` effects on bidi and shaping"，`Changelog:154-155`）。同时 `set_run_features()` 会在 hspacing 非零时关闭 `liga`/`clig` 连字特性（`ass_shaper.c:184-188`）。
6. **flags 不同**：`DECO_UNDERLINE/STRIKETHROUGH/ROTATE`（`ass_font.h:38-40`），即下划线/删除线开关与竖排旋转边界也切 run。

另外 `shape_harfbuzz()` 真正送入 HarfBuzz 前还要求 bidi embedding level 相同（`ass_shaper.c:699-701`），所以 bidi 层级变化同样分段。

**被 skip 的码位（HarfBuzz default-ignorable 集合，`is_harfbuzz_ignorable()` `ass_shaper.c:846-862`，含 ZWJ/ZWNJ、ZWSP、bidi 控制符、variation selector FE00–FE0F、软连字符等）不参与 face 比较，并继承前一码位的 `face_index`（`:906-907`）**——它们不触发回退，也不主动断 run。

---

## 5. HarfBuzz shaping 与字形簇

### 5.1 每 run 独立 shaping，ot 后端固定

`shape_harfbuzz()`（`ass_shaper.c:673-749`）：

- 固定使用 `{"ot", NULL}` shaper 列表以兼容 GDI 系渲染器（`:675-676`）；
- 每个 shape run 建/复用 `hb_shape_plan_t`，缓存键是 segment properties（direction、script、language）+ 特性集（`:730-738`）；
- run 的 direction 由 FriBidi level 决定 RTL/LTR（`:723-724`）；
- 启用的特性仅 5 个：`vert`、`vkna`（竖排）、`kern`、`liga`、`clig`（`init_features()` `:158-171`）。没有启用 mark 定位相关的额外开关——`mark`/`mkmk` 等 GPOS 定位按 HarfBuzz 默认行为执行。

libass 没有调用 `hb_buffer_set_cluster_level()`，即使用 HarfBuzz 默认 cluster level（现代 HarfBuzz 为 monotone-graphemes）。

### 5.2 cluster 到 GlyphInfo 的映射：多字形挂链

`shape_harfbuzz_process_run()`（`ass_shaper.c:628-666`）按 HarfBuzz 输出的 `glyph_info[j].cluster` 找回根 `GlyphInfo`：

- 一个输入码位产出多个字形（连字分解的反向情形、一个基字符 + 多个 mark 字形、 Indic 重组等）时，通过 `info->next` 链表挂接额外 `GlyphInfo`（`:643-652`），注释明确写着 "if we have more than one glyph per cluster"；
- 每个字形记录自己的 `glyph_index`、HarfBuzz 给出的 `offset` 与 `advance`（`:655-660`）；
- **整个 cluster 的 advance 累加到根节点 `root->cluster_advance`（`:663-664`）**，布局与换行只消费根节点的 `cluster_advance`（`ass_render.c:2226`、`:2259`、`:2320` 等）。

反过来，HarfBuzz 也可能把多个输入码位合并成一个字形（连字）：此时多个根 `GlyphInfo` 指向同一 cluster，HarfBuzz 的 cluster 合并（monotone 级别下）保证它们映射一致；而 `ass_shaper_skip_characters()` 已先把 default-ignorable 码位标 skip（`:867-874`）， shaping 开始时所有 glyph 先统一置 `skip=true`，被 HarfBuzz 输出覆盖到的 cluster 才取消 skip（`:682-683`、`:655`）。

### 5.3 自定义 hb_font_funcs：metric 来源是 FreeType 缓存

libass 不用 HarfBuzz 自带 FreeType font funcs，而是注册整套回调（`ass_shaper.c:1058-1072`）：nominal glyph 走 `get_glyph_nominal()`（`:302-318`，内部仍过 `ass_font_index_magic()`，保证 symbol cmap 与旧阿拉伯字体一致），advance/extents 走 `ass_glyph_metrics_construct()` 的缓存（`:252-275`，加载标志含 `FT_LOAD_IGNORE_GLOBAL_ADVANCE_WIDTH`）。缓存键含 font 指针、face_index、size、glyph_index（`:224-229`），**不同回退 face 的同码位字形 metric 各自独立缓存，绝不混用**。

### 5.4 SIMPLE 模式的差异

`ASS_SHAPING_SIMPLE`（`ass_shaper.c:1031-1034` → `shape_fribidi()` `:806-828`）只做 FriBidi 阿拉伯 joining 与镜像，逐码位 `FT_Get_Char_Index` 直接取字形，无 OpenType GPOS mark 定位、无连字；并且 `get_outline_glyph()` 在 SIMPLE 模式下用轮廓自身 advance 覆盖 shaping advance（`ass_render.c:1226-1229`）。默认是 COMPLEX（`ass_render.c:162`），本报告其余结论针对默认模式。

---

## 6. 回退字体度量不同带来的布局后果

### 6.1 每个 face 各自的 em-square 与垂直度量

所有 face 都以**同一请求字号** `info->font_size` 通过 `ass_face_set_size()`（`ass_font.c:585-594`，`FT_SIZE_REQUEST_TYPE_REAL_DIM`）设置，但物理度量完全由各 face 自身表决定：

- HarfBuzz 缩放按该 face 的 `units_per_EM` 与 `FT_Size_Metrics` 换算（`update_hb_size()` `ass_shaper.c:196-202`）；
- 垂直度量模仿 GDI：优先 OS/2 `usWinAscent/usWinDescent` 覆盖 face 的 ascender/descender（`set_font_metrics()` `ass_font.c:356-389`），缺失时退回 typo metrics 或 bbox；
- 每个字形的 `info->asc/info->desc` 在取轮廓时按**该字形所属 face** 计算（`ass_outline_construct()` `ass_render.c:1253-1254` → `ass_font_get_asc_desc()` `ass_font.c:661-668`，缩放后写入 `ass_render.c:1230-1231`）。

### 6.2 行高取整行最大值：一个回退字形抬高整行

`measure_text()`（`ass_render.c:1605-1650`）对每行维护 `max_asc/max_desc`，对所有非修剪空白码位取 FFMAX（`:1639-1640`），结果存入 `text_info->lines[].asc/desc`（`:1581-1583`）。因此：

- 一行中只要有一个字形来自 ascender 明显更大的回退字体（典型：CJK 字体回退到拉丁行里），**整行行高、换行后的行距（`reorder_text()` 中 `pen.y += 上一行 desc + 下一行 asc`，`ass_render.c:2251-2253`）、事件总高度 `text_info->height` 与碰撞检测**全部随之变化；
- 边框样式 3（不透明底框）的盒子高度直接用 `info->asc + info->desc`（`ass_render.c:1431`），同一 style run 内不同 face 的字形盒高可能不同。

### 6.3 水平 advance 与视觉重量不一致

- 回退字形的 advance 来自其自身 face 的 `horiAdvance`（经 HarfBuzz 缓存回调），同一码位在不同字体里宽度不同；混排行的总宽、`preliminary_layout()`/`wrap_lines_naive()` 的断行点（`ass_render.c:1748-1749` 用 `bbox + pos` 判宽）随之改变——**换一个系统字体配置就可能换一行**。
- 粗/斜合成按 face 独立判定：`ass_font_get_glyph()`（`ass_font.c:779-813`）只在该 face 自身没有 italic/bold 风格位时才做 faux italic / faux embolden。回退 face 若自带粗体，主字体做合成粗体而它用真粗体，笔画重量不连续。
- 下划线位置/粗细取自**当前 face** 的 `post` 表（`ass_get_glyph_outline()` `ass_font.c:845-856`），删除线取自 OS/2 `yStrikeoutPosition/yStrikeoutSize`（`:857-869`）：跨回退边界时一条下划线的厚度与基线距离会跳变。

### 6.4 轮廓/位图缓存按 face 隔离

`GlyphHashKey` 含 `font + face_index + glyph_index + size + bold + italic + flags`（`ass_render.c:1199-1205`），保证回退字形不会与主字体字形错误共享缓存；副作用是混排文本的缓存工作集更大。

---

## 7. 组合标记边界与跨 run 的 shaping 上下文

### 7.1 默认模式（未开 whole text layout）：run 间 shaping 严格隔离

`shape_harfbuzz()` 的 run 循环只在同 `shape_run_id` 且同 bidi level 内累积（`ass_shaper.c:699-701`）。默认模式下送入 HarfBuzz 的 buffer 默认只含 run 自身，唯一例外是相邻的 ZWNJ/ZWJ：

- `is_shaping_control()`（`ass_shaper.c:516-518`）只认 U+200C/U+200D；
- 仅当该控制字符**不**位于 `starts_new_run` 边界时，才把它作为 lead/trail context 加进 buffer（`:710-720`）。

可观察后果：

1. **基字符与 combining mark 落在不同 shape run 时（典型：基字符缺字触发回退 face，或标记前后有 `{...}` override），HarfBuzz 在两个独立 buffer 中分别 shaping**。标记拿不到基字符的 GPOS mark attachment 锚点，只能按它自己 face 的默认定位（通常是零字宽定位规则）摆放，出现叠印位置偏移、标记漂移到下一个字形上方、或重复/漏叠。
2. 跨样式边界的 ZWJ 连字（例如 `a{\fnX}&zwj;b`）在默认模式下连接意图被切断——这是刻意的 VSFilter 兼容行为（Changelog:154 "Run bidi and shaping on each glyph run separately by default"）。
3. bidi 段落本身也按 style run / hspacing 分段计算（`ass_shaper_shape()` `:1005-1029`），分段处 bidi 算法看不到全局，中性字符方向解析可能与整段处理不同。

### 7.2 whole text layout 模式：全事件上下文，但 face 分段仍在

启用 `ASS_FEATURE_WHOLE_TEXT_LAYOUT`（API 文档 `ass.h:256-274`；`ass_shaper_set_whole_text_layout()` `ass_shaper.c:958-964`）后：

- 每个 run 调 `hb_buffer_add_utf32()` 时传入**整段事件文本**、只圈定 run 的起止下标（`ass_shaper.c:706-708`），HarfBuzz 因此能看到 run 外的上下文做 joining/标记定位，`process_run` 的 offset 也传 0（`:742`）；
- bidi 与 `\fay` 剪切也在整事件范围计算（`apply_baseline_shear()` `ass_render.c:2277-2284`）；
- `Encoding=-1`（自动方向）隐式进入该模式（`ass_shaper_set_base_direction()` `ass_shaper.c:917-924`，`ass.h:271-272`）。

但注意：**shape run 的 face/script 切分本身没有消失**。HarfBuzz 拿到的是整段文本，却用 run 对应的单个 `hb_font_t` 执行 plan（`:739`）；跨 face 的 mark attachment 是否改善取决于 HarfBuzz 对 nominal glyph 回调在"上下文码位不属于本 face"时的处理——libass 的 `get_glyph_nominal()` 对查不到的码位返回 false，HarfBuzz 会以 `.notdef`/零宽处理上下文。换句话说，whole text layout 修复的是**上下文可见性**（joining 类型、bidi、中性字符方向），并不修复"基字符与标记物理上来自不同 face、锚点规格不同"的问题。

### 7.3 隐形/控制字符的边界行为

- variation selector（U+FE00–FE0F）在 `is_harfbuzz_ignorable()` 中被 skip（`ass_shaper.c:856-857`），不单独触发回退；它们对字形呈现的影响只能在同一 run 内被 HarfBuzz 的 `rvrn`/`cjkv` 等特性消费——若 emoji 基字符本身已因缺字进入回退 face，selector 跟随借脸（`:906-907`），但回退 face 是否支持该呈现变体没有任何保证。
- ZWJ/ZWNJ 同样 skip，但通过 7.1 的 context 机制在 run 内生效；emoji ZWJ sequence 中任一码位缺字导致 face 切换时，序列被切成多段，连字 emoji 退化为独立字形甚至 `.notdef`。

### 7.4 缺字最终形态

所有回退都失败时 `glyph_index=0`（`ass_font.c:769-770` 的 HACK 仅保证 face_index 非负）。HarfBuzz nominal 回调对 0 码位返回 false（`get_glyph_nominal()` `:312-313`），该 cluster 输出 `.notdef`，画面上是空心方框（tofu）或空白，同时日志保留 `MSGL_ERR` 记录（`ass_font.c:760-763`），可作为外部诊断信号。

---

## 8. 可观察结果汇总

| # | 可观察症状 | 触发条件 | 代码根因 |
|---|---|---|---|
| 1 | 同一行字形风格/字重/字面大小突变 | 主字体缺字，系统回退 face 命中 | `ass_font_get_index()` 追加 face（`ass_font.c:735-766`），face_index 变化切 shape run（`ass_shaper.c:898-900`），各 face 独立 upem/度量（`ass_shaper.c:196-202`） |
| 2 | 整行行高/行距被个别"外来"字形撑大 | 回退 face 的 Win asc/desc 更大 | 行度量取全行 max（`ass_render.c:1639-1640`、`:1581-1583`），face 度量来自 `set_font_metrics()`（`ass_font.c:356-389`） |
| 3 | 混排行换行位置随系统字体配置变化 | 回退字体 advance 与主字体不同 | 断行按 `bbox+pos` 实测宽度（`ass_render.c:1748-1749`），advance 来自 face 自身 horiAdvance 缓存（`ass_shaper.c:338-348`） |
| 4 | combining mark 错位、悬空、叠到相邻字形 | 基字符与标记分属不同 face/shape run，且未开 whole text layout | run 级独立 buffer（`ass_shaper.c:699-721`），GPOS 锚点跨 run 不可见 |
| 5 | 行首标点/数字按"错误的"脚本规则 shaping | Common/Inherited 反向继承（`ass_shaper.c:785-798`），script 决定 shape plan（`:725-727`） | 脚本继承规则与 `script_to_language()` 静态映射（`:532-593`） |
| 6 | ZWJ 连字/emoji ZWJ 序列在样式边界处断开 | override tag 使 `starts_new_run=true`，默认模式 context 仅限 ZWNJ/ZWJ 且不跨样式边界（`ass_shaper.c:710-715`） | VSFilter 兼容的逐 run shaping（Changelog:154-155） |
| 7 | 非零 `\fsp` 时连字消失 | hspacing ≠ 0 | `liga/clig` 被关（`ass_shaper.c:184-188`）且 run 被切分（`:903`） |
| 8 | 下划线/删除线厚度与位置在混排处跳变 | 装饰线参数逐 face 取自 post/OS-2 表 | `ass_font.c:845-869` |
| 9 | 粗体文本中回退字形笔画重量不一致 | 回退 face 自带 bold 而主 face 靠合成 | faux 样式按 face 风格位独立判定（`ass_font.c:806-812`） |
| 10 | 第 11 种缺字脚本起出现 tofu，即使系统有字体 | 单逻辑字体 face 达 10 个上限 | `ASS_FONT_MAX_FACES` 硬上限（`ass_font.h:37`、`ass_font.c:508-509`） |
| 11 | 完全无回退时输出 `.notdef` 空心框 | 所有 provider 均无覆盖 | `glyph_index=0` + ERR 日志（`ass_font.c:759-764`、`ass_fontselect.c:936-938`） |
| 12 | 同一样式段内位图仍被合成为同一图层，但内部字形来源不同 | 位图组合只认 `starts_new_run`，不认 `shape_run_id` | `render_and_combine_glyphs()` `ass_render.c:2461`、`:2489-2538` |

---

## 9. 现有测试与验证材料

仓库内与本主题相关的测试材料有限，且**没有**专门覆盖字体回退/混合脚本的回归用例：

1. **像素回归框架**：`compare/`（`compare/compare.c` + `compare/README.md`）。README 说明该工具以 `ASS_FONTPROVIDER_NONE` 运行，所有字体必须由输入目录提供，按 `<sub_name>-NNNN.png` 做图像比对。现有唯一用例 `compare/test/`：
   - `font1.ttf`、`font2.otf` 两个自附字体；
   - `sub1.ass`：单字体 Aileron 的旋转测试（`{\t(\frz+180)}A`）；
   - `sub2.ass`：Pixel Operator Mono 的 `\blur1` 测试。
   两者均为单字体、单脚本，不触发 fallback 路径。
2. **冒烟测试**：`test/test.c`（266 行）只做库初始化、provider 枚举与基本渲染，不断言 shaping/回退行为。
3. **fuzz  harness**：`fuzz/fuzz.c` 用 `ass_set_fonts(..., "sans-serif", ASS_FONTPROVIDER_AUTODETECT, ...)`（`fuzz/fuzz.c:107-108`），依赖宿主系统字体，能间接走到 fontconfig/系统回退，但语料目录不在本仓库，且只做崩溃检测不做语义断言。
4. **旧阿拉伯 cmap 数据**：`legacy_arabic_cmap/ArabicPUASimplified.txt`、`ArabicPUATraditional.txt` 与生成脚本 `gen_arabic_charmap.py`，对应 `ass_font.c:250-270` 的 legacy Arabic 字体符号换算，是唯一与"特殊 cmap 下字形选择"直接相关的数据材料。
5. **行为变迁记录**：`Changelog` 中 0.17.0 的 whole text layout 与逐 run shaping 条目（`Changelog:144-156`）、0.15.x 的 Core Text/DirectWrite 回退修复（`:165-179`）、早期 "Properly handle script and language in the HarfBuzz shaper"（`:467`）。

---

## 10. 当前实现 vs. 建议（明确区分）

### 10.1 当前实现（事实，代码可证）

1. **回退决策模型**：逐码位、shaping 前决策；一个逻辑 `ASS_Font` 聚合最多 10 个物理 face（`ass_font.h:37`，`ass_font.c:499-550`、`:708-773`）；fontconfig/DWrite/CoreText 三平台各自实现 `get_fallback`，fontconfig 侧按 charset 线性筛选并缓存一次 `FcFontSort` 结果。
2. **run 切分**：face、script、样式边界、hspacing、DECO flags、bidi level 任一变化即切 shape run（`ass_shaper.c:896-908`、`:699-701`）；默认逐 run 隔离 shaping，仅 ZWNJ/ZWJ 在非样式边界处提供一码位上下文（`:705-721`）。
3. **脚本处理**：Common/Inherited 双向扫描继承（`:756-799`）；script→language 静态映射，可被 track `Language` 覆盖（`:532-619`）。
4. **cluster 处理**：HarfBuzz 默认 cluster level；多字形经 `GlyphInfo.next` 挂链、advance 汇总到根 cluster（`:628-666`）；default-ignorable 码位预先 skip 并借邻接 face（`:846-874`、`:906-907`）。
5. **度量**：全部 face 同请求字号但各自 em-square/OS-2 度量；行高取整行 max asc/desc；装饰线、faux 粗斜均按 face 独立计算（第 6 节）。
6. **whole text layout**：可选、默认关、与 VSFilter 不兼容；开启后提供全事件 bidi/shaping/`\fay` 上下文，但不改变 face 切分，也不统一跨 face 度量（`ass.h:256-274`，`ass_shaper.c:706-708`）。
7. **失败语义**：无覆盖时 `.notdef` + 明确的 `MSGL_ERR/MSGL_WARN` 日志；10-face 上限后静默停止追加 face。

### 10.2 建议（非现状，供讨论；按预期收益排序）

1. **补齐回退/混排回归测试矩阵**（成本最低、收益最直接）：在 `compare/test/` 增加自附双字体用例——一个主字体刻意缺 CJK/阿拉伯/mark 字形，第二个字体覆盖它们，断言（a）回退命中日志、（b）行高与断行像素、（c）mark 叠印位置。现有框架已支持多字体目录，只需新增 `*.ttf + *.ass + *.png`。
2. **mark 跨 run 定位的显式处理**：对"基字符与 combining mark 被 face 切换拆开"的情形，可在 `ass_shaper_find_runs()` 中让 INHERITED 码位强制继承基字符的 face 选择（即便该 face 不含 mark 时再二级回退），或在 whole text layout 模式下对跨 face cluster 做二次 mark 定位修正；现状下该场景只能依赖 HarfBuzz 默认行为，输出不稳定（第 7.1 节症状 4）。
3. **度量协调选项**：提供可选的"行度量固定为主字体"模式（用主 face asc/desc 统一行高，回退字形按比例缩放或容忍溢出），消除单个回退字形撑高整行的问题（症状 2）。需作为非默认、非 VSFilter 兼容特性提供，与 `ASS_FEATURE_*` 同层管理。
4. **放宽或可配置 `ASS_FONT_MAX_FACES`**：现代多脚本字幕（尤其含 emoji）容易触顶；建议至少把上限提高并在触顶时输出一次性 WARN（目前 `add_face()` 返回 -1 后只有下游 ERR，定位困难）。
5. **whole text layout 下的 face 上下文协商**：当 run 外上下文码位在本 face 无 nominal glyph 时，可让 HarfBuzz 回调临时返回相邻 run face 的字形信息（需要重构 `get_hb_font()` 与回调的私有数据结构），使跨样式边界的 ZWJ 序列与 mark attachment 获得正确锚点。
6. **回退可观测性 API**：增加调试接口或结构化日志，输出每码位选中的 family/face_index/script/shape_run_id，便于上层（播放器、字幕工具）复现与诊断混排问题；目前信息散落在 `fontselect:` 自由文本日志里。
7. **VS 选择器与 ZWJ 序列的覆盖校验**：在 `ass_font_get_index()` 回退判定中把 variation selector/ZWJ 序列作为整体检查覆盖率（而非逐码位独立回退），减少 emoji 序列被 face 边界拆断（症状 6 的变体）。

> 兼容性提醒：建议 2、3、5、7 若改变默认输出都会偏离 VSFilter 像素行为，应像 `ASS_FEATURE_WHOLE_TEXT_LAYOUT` 一样作为显式 opt-in 特性；建议 1、4、6 不改变既有渲染结果，可独立先行。

---

### 附：核心源码索引

| 主题 | 文件:行 |
|---|---|
| 逻辑字体与 10-face 上限 | `libass/ass_font.h:37-50` |
| 逐码位 face 查找与回退触发 | `libass/ass_font.c:708-773` |
| 四级字体选择链 | `libass/ass_fontselect.c:884-941` |
| 候选字形覆盖校验 | `libass/ass_fontselect.c:700-706`、`:751-764` |
| fontconfig/DWrite/CoreText 回退 | `libass/ass_fontconfig.c:223-312`；`libass/ass_directwrite.c:509-565`；`libass/ass_coretext.c:254-292` |
| 脚本判定与继承 | `libass/ass_shaper.c:756-799` |
| shape run 切分 | `libass/ass_shaper.c:879-911` |
| HarfBuzz run shaping 与上下文 | `libass/ass_shaper.c:673-749` |
| cluster → GlyphInfo 映射 | `libass/ass_shaper.c:628-666` |
| ignorable 码位表 | `libass/ass_shaper.c:846-874` |
| 自定义 metric 回调与缓存 | `libass/ass_shaper.c:214-418`、`:1044-1083` |
| style run 切分 | `libass/ass_render.c:2005-2047` |
| 行度量 max asc/desc | `libass/ass_render.c:1576-1650` |
| 布局/重排/换行 | `libass/ass_render.c:2180-2269`、`:1736-1794` |
| face 级装饰线度量 | `libass/ass_font.c:833-909` |
| whole text layout 语义 | `libass/ass.h:256-274`；`libass/ass_shaper.c:917-924`、`:958-964` |
