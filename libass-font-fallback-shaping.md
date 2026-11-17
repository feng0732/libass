# libass 字体回退与字形整形分析报告

字幕中的缺字（missing glyph）与混合脚本（mixed script）会触发 libass 的字体回退机制，而回退发生在**码点级别**，整形（shaping）发生在**run 级别**。两者之间的交互决定了最终的字形簇划分、advance 宽度与行布局。本报告基于本仓库源码（分支 `task/37`）分析这一链路，并在末尾区分"当前实现"与"改进建议"。

## 1. 渲染管线中的位置

一次 Dialogue 事件的渲染主流程在 `ass_render_event()`（libass/ass_render.c:2798）中依次经过：

1. `parse_events()`（ass_render.c:2051）：逐码点解析文本与覆盖标签，把每个码点的样式快照存入 `GlyphInfo`（ass_render.h:126）。`\fn`、`\b`、`\i` 等标签通过 `ass_update_font()`（libass/ass_parse.c:83）构造 `ASS_FontDesc`（family/bold/italic/vertical），经 `ass_font_new()` 从字体缓存取出 `ASS_Font`。
2. `split_style_runs()`（ass_render.c:2005）：纯样式层面的 run 切分，设置 `starts_new_run`。切分条件包括字体族、字号、颜色、模糊、旋转、字距（`hspacing`）、粗斜体等约 20 个字段（ass_render.c:2014-2043）。
3. `ass_shaper_find_runs()`（libass/ass_shaper.c:879）：脚本判定 + **逐码点字体回退** + shape run 编号。
4. `ass_shaper_shape()`（ass_shaper.c:971）：FriBidi 求 bidi 嵌入层，然后 FriBidi（简单整形）或 HarfBuzz（复杂整形）按 run 整形。
5. `retrieve_glyphs()` / `preliminary_layout()` / 换行 / 重排 / 对齐（ass_render.c:2180 起）。

关键点：**字体回退（第 3 步）发生在整形（第 4 步）之前**，回退结果直接决定 shape run 的边界。

## 2. 字体选择与回退链

### 2.1 四级回退链

`ass_font_select()`（libass/ass_fontselect.c:884）按以下顺序尝试：

1. 请求的 family（经 provider 的 `get_substitutions` 别名扩展，如 fontconfig 的 `FcConfigSubstitute`，ass_fontconfig.c:314）；
2. `family_default`（`ass_set_fonts` 设置的默认族，ass_fontselect.c:898）；
3. provider 的 `get_fallback`：按**码点**查询系统回退族（fontconfig 实现见 ass_fontconfig.c:263，用 `FcCharSetHasChar` 在全部字体的字符集并集中查找；DirectWrite 见 ass_directwrite.c:509；CoreText 见 ass_coretext.c:254）；
4. `path_default`（固定字体文件，ass_fontselect.c:922）。

全部失败时记录 `fontselect: failed to find any fallback with glyph 0x%X`（ass_fontselect.c:937），该码点最终以 glyph index 0（.notdef，通常显示为豆腐块 □）渲染。

### 2.2 码点级覆盖检查与 face 累积

- 匹配阶段 `find_font()`（ass_fontselect.c:708）对每个候选 face 调用 `check_glyph(font, code)`（ass_fontselect.c:759）。注释（ass_fontselect.c:750-758）明确说明：同一 family 内 regular 覆盖往往最好，bold/italic 变体覆盖较差，因此**宁可选同族中覆盖该码点的较远变体，也不落入异族回退**。这是缺字时"字形风格突变"的第一来源。
- 运行时 `ass_font_get_index()`（libass/ass_font.c:708）对**每个码点**依次扫描 `font->faces[]`；都缺失时调用 `add_face()`（ass_font.c:499）把新 face 追加到同一 `ASS_Font` 上，上限 `ASS_FONT_MAX_FACES = 10`（libass/ass_font.h:37）。超过 10 个回退 face 后 `add_face` 返回 -1，后续缺字直接变 .notdef。
- charmap 归一化由 `ass_charmap_magic()`（ass_font.c:200，优先 MS Unicode cmap）与 `ass_font_index_magic()`（ass_font.c:327）完成，后者处理 MS Symbol 字体偏移（ass_font.c:272）、legacy Arabic PUA 字体（ass_font.c:250，映射表由 legacy_arabic_cmap/ 下的 gen_arabic_charmap.py 生成）以及 SJIS/GB2312/Big5/Wansung/Johab 等 MBCS cmap（ass_font.c:122 起）。这意味着"同一个码点"在不同 face 上可能走完全不同的索引路径，回退 face 的 cmap 类型会影响是否命中。

### 2.3 度量模拟 GDI

`set_font_metrics()`（ass_font.c:356）用 OS/2 的 `usWinAscent/usWinDescent` 覆盖 FreeType 的 ascender/descender/height，以模拟 VSFilter/GDI 行为。**每个回退 face 独立执行此逻辑**，因此同一行内不同 face 的 ascent/descent 可以差异很大（典型场景：西文主字体 + CJK 回退字体，后者 usWin 度量通常大得多）。

## 3. 脚本分段与 shape run 边界

### 3.1 脚本判定

`ass_shaper_determine_script()`（ass_shaper.c:756）逐码点取 `hb_unicode_script`：

- `Common` 与 `Inherited`（组合标记即属 Inherited）**继承上下文的脚本**：前向扫描继承前一个确定脚本（ass_shaper.c:770-781）；若行首就是 Common/Inherited，再做一次反向扫描取后文脚本（ass_shaper.c:786-798）。
- 效果：组合标记（如 U+0301）在脚本维度上**总是贴着基字符**，不会因脚本不同而被切走。但注意这只是脚本维度——字体维度仍可能把它切走（见 §4）。

脚本还决定 HarfBuzz 的 `props.script` 与语言（`script_to_language()`，ass_shaper.c:532，仿 Pango 的映射；`hb_shaper_get_run_language()`，ass_shaper.c:602），从而影响本地化字形变体（如塞尔维亚西里尔）的选用。

### 3.2 shape run 的切分条件

`ass_shaper_find_runs()`（ass_shaper.c:879-911）在以下任一条件成立时递增 `shape_run_id`：

- `ASS_Font` 指针变化（`\fn`/粗斜体变化）；
- **face_index 变化**——即字体回退发生处（ass_shaper.c:899-900）；
- 脚本变化（ass_shaper.c:902）；
- `starts_new_run`（样式 run 边界，ass_shaper.c:903）；
- 非 whole-text-layout 且 `hspacing` 非零（ass_shaper.c:904）；
- 装饰 flags 变化。

被 `ass_shaper_skip_characters()` 标记为 skip 的 default-ignorable 码点（ZWJ、ZWNJ、软连字符、变体选择符等，清单在 `is_harfbuzz_ignorable()`，ass_shaper.c:846）不参与 face 比较，并继承前一个字符的 face_index（ass_shaper.c:906-907）。

整形时 `shape_harfbuzz()`（ass_shaper.c:673）把**同一 shape_run_id 且同一 bidi level** 的连续码点放进一个 `hb_buffer` 一次性 `hb_shape_plan_execute`（ass_shaper.c:699-739）。因此：**回退边界 == 整形边界**。HarfBuzz 的 GSUB/GPOS（连字、字距、组合标记附着、草书连接）只在 run 内部生效。

### 3.3 bidi 与 whole-text-layout

- 嵌入层按 FriBidi "段落"计算；非 whole-text-layout 时，`starts_new_run` 与 `hspacing` 边界也被当作段落断点（ass_shaper.c:1009-1010），即样式边界会切断 bidi 算法的上下文。
- `ASS_FEATURE_WHOLE_TEXT_LAYOUT` 开启时，`hb_buffer_add_utf32` 传入**整段文本**加 item 边界（ass_shaper.c:707-708），HarfBuzz 能看到 run 外的上下文，跨 run 的连字/附着得以保留。
- 关闭时有一个定向补丁：`is_shaping_control()`（ass_shaper.c:516）识别 ZWJ/ZWNJ，若 run 边界相邻字符是 ZWJ/ZWNJ，则把它作为 lead/trail context 一并送入 buffer（ass_shaper.c:710-720），使跨 run 的草书连接/合字在连接符处仍然成立。**注意此待遇只给 ZWJ/ZWNJ，不给组合标记。**

## 4. 组合字符与回退的交互（核心问题）

组合标记（Mn/Mc/Me，脚本属性 Inherited）在三个维度上的待遇不一致：

| 维度 | 组合标记是否跟随基字符 | 依据 |
|---|---|---|
| 脚本 | 是（Inherited 继承上下文脚本） | ass_shaper.c:770 |
| bidi | 是（FriBidi 按 EN/NSM 规则处理） | ass_shaper.c:984 |
| **字体** | **否（独立做 cmap 查找）** | ass_shaper.c:893 → ass_font.c:708 |

当主字体缺某个组合标记（缺字场景的典型：西文字体缺 U+0301、缺泰文元音符号、缺阿拉伯文 harakat）时：

1. `ass_font_get_index` 为该标记单独 `add_face` 引入回退字体；
2. 标记的 `face_index` 与基字符不同 → `shape_run_id` 在基字符与标记之间断开（ass_shaper.c:899）；
3. HarfBuzz 把基字符和标记放在**两个 buffer、两个 hb_font** 中分别整形；
4. 结果：标记不再通过 GPOS mark-to-base/mark-to-mark 附着到基字符，而是作为独立簇按其**回退字体自身的 advance/offset** 摆放（`shape_harfbuzz_process_run()`，ass_shaper.c:628，把 pos/advance 直接写入 GlyphInfo）。

反向情况同样存在：基字符缺字落入回退字体而标记在主字体中——簇同样在基字符处断开。

此外，即使不发生字体回退，**样式 run 边界**（如 `\b1` 加粗半个音节、`\fsp` 改变字距，ass_render.c:2014-2043 + ass_shaper.c:903-904）也会把基字符与组合标记切进不同 shape run。ZWJ/ZWNJ 有 §3.3 的上下文补丁，组合标记没有等价机制。

### 可观察结果（组合标记边界）

- 重音符偏离基字符、出现在字间空隙上，或以零宽原点堆在行首；
- 印地/泰文等 abugida 的元音标记丢失重排（reordering）与合字；
- 阿拉伯文 shadda/fatha 等失去 mark 附着，且跨 run 后不再参与草书连接判定；
- 同一组合字符序列在"主字体含该标记"与"不含该标记"的机器上渲染结果不同（环境相关的不稳定渲染）。

## 5. 回退字体度量差异的可观察结果

### 5.1 行高与基线

- 每个 glyph 的 `asc/desc` 来自**其所属 face** 的度量：`ass_outline_construct()` 中 `ass_font_get_asc_desc(k->font, k->face_index, ...)`（ass_render.c:1253，底层 ass_font.c:661，用该 face 的 `face->ascender/descender` 乘 y_scale）。
- 行高取行内所有 glyph 的 max：`measure_text()` 中 `max_asc = FFMAX(max_asc, cur->asc)`（ass_render.c:1639-1640），写入 `lines[].asc/desc`（ass_render.c:1581-1583）。
- **可观察结果**：一行中只要有一个码点落入度量更大的回退字体（如 CJK 字体），整行的 asc/desc 被抬高，事件整体垂直位置下移（`device_y` 依赖 `lines[0].asc`，ass_render.c:2921/2939/2943），多行文本的行距随之增大。缺字修复（安装字体）后行高又会"跳回"——这是回归测试中对齐敏感 diff 的常见来源。

### 5.2 advance 与换行

- HarfBuzz 的 advance 经 `cached_h_advance()`（ass_shaper.c:338）取自各 face 的 `FT_Glyph_Metrics.horiAdvance` 缓存；回退字体的 advance 与主字体无归一化关系。
- **可观察结果**：行总宽变化 → `wrap_lines_smart()` 换行点移动；`align_lines()`（ass_render.c:2297）的居中/右对齐偏移变化；`\fsp` 字距叠加在 `cluster_advance` 上（ass_render.c:2205），与回退无关地继续生效。
- 竖排 `@font` 的 advance 旋转（`get_cached_metrics` 的 rotate 分支，ass_shaper.c:218-236；`DECO_ROTATE`，ass_render.c:2145）同样按 face 分别处理，回退 face 若无 `vmtx` 度量会得到 FreeType 合成的竖排 advance。

### 5.3 装饰线与 faux 粗斜体

- 下划线/删除线的位置与粗细取自**各 face 自己的** `post` 表与 OS/2 表（`ass_get_glyph_outline()`，ass_font.c:844-869），且按 glyph 逐段生成矩形。
- **可观察结果**：下划线在回退边界处高度/粗细突变甚至断开（回退 face `adv <= 0` 的 glyph 不生成线，ass_font.c:844）。
- faux 粗体/斜体按 face 判定：`ass_font_get_glyph()`（ass_font.c:779）只对"该 face 本身不是粗/斜体"的 glyph 做 embolden/italicize。**可观察结果**：请求 `\b1` 时，若回退 face 恰好是 Bold 变体，回退段不加粗而主文本段做了 faux 加粗，同一行内笔画粗细不一致；斜体 shear 角度还因 PostScript 轮廓与否取 tan(10°) 或 tan(18.77°)（ass_font.c:689-701）。
- 斜体悬出修正（ass_render.c:2194-2202）按 `cluster_advance` 与 bbox 比较，跨字体边界时同样逐段生效。

### 5.4 缺字的终态

- 回退链全部失败 → `glyph_index = 0`（.notdef），日志 `Glyph 0x%X not found in font`（ass_font.c:759-764）。.notdef 的 advance 计入行宽，豆腐块参与换行与对齐。
- 超过 `ASS_FONT_MAX_FACES = 10` 后新缺字不再尝试回退（ass_font.c:508-509），同一事件内第 11 种脚本的字符直接 .notdef。

## 6. 已有测试材料

- `compare/`：图像回归框架（README.md 说明用法）。`compare/test/` 现有两个用例：`sub1.ass`（Aileron + `\t(\frz+180)` 旋转动画，3 个时间点 PNG）与 `sub2.ass`（Pixel Operator Mono + `\blur1`）。框架要求所有字体放在输入目录内（`ASS_FONTPROVIDER_NONE`），**当前没有用例覆盖字体回退、混合脚本或组合标记**——新增用例时只需放入主字体 + 一个覆盖互补的回退字体及对应 PNG。
- `test/test.c`：逐帧渲染 ass 文件并写 PNG 的冒烟工具（`ass_render_frame`，test.c:253），用 fontconfig（`sans-serif` 默认族，test.c:125），适合手工观察回退行为，但结果依赖系统字体，不可作为稳定回归。
- `fuzz/fuzz.c` + `fuzz/ass.dict`：libFuzzer 入口，间接覆盖缺字/异常 cmap 路径的健壮性（不校验渲染正确性）。
- `legacy_arabic_cmap/`：legacy Arabic PUA 字体的映射表与生成脚本，是"特殊 cmap 影响回退命中"（§2.2）的既有测试素材来源。
- `checkasm/`、`profile/profile.c` 与本主题无关（汇编校验与性能剖析）。

## 7. 当前实现 vs 建议

### 当前实现（事实陈述）

1. 字体回退是**逐码点**的：每个码点独立查 cmap、独立 `add_face`，不回看基字符/簇（ass_shaper.c:889-895，ass_font.c:726-766）。
2. 回退边界即整形边界：face_index 变化直接产生新 shape run（ass_shaper.c:899-900），HarfBuzz 按 run 独立整形（ass_shaper.c:699-739）。
3. 组合标记仅在脚本与 bidi 维度跟随基字符，在字体维度不跟随（§4 表格）。
4. 跨 run 上下文仅对 ZWJ/ZWNJ 有效，且仅在 whole-text-layout 关闭时需要（ass_shaper.c:516-518, 710-720）；组合标记无等价机制。
5. 行高、装饰线、faux 粗斜体、竖排 advance 全部按 face 独立计算，不做跨 face 归一（§5）。
6. 回退 face 上限 10 个/字体描述；缺字终态为 .notdef（ass_font.h:37，ass_font.c:759）。
7. 回归测试材料未覆盖回退/混合脚本/组合标记场景（§6）。

### 建议（改进方向，非现状）

1. **簇感知的回退**：在 `ass_shaper_find_runs` 中先做 Unicode 字形簇切分（至少把 gc=Mn/Mc/Me 及 ZWJ 序列归并到基字符），对整个簇统一选择"覆盖簇内全部码点"的 face；只有当没有任何单一 face 覆盖整簇时才在簇内回退。可显著减少 §4 的标记漂移。
2. **组合标记的跨 run 上下文**：仿照 `is_shaping_control` 的 lead/trail context 机制（ass_shaper.c:710-720），把 run 边界处的组合标记（或整簇）作为上下文送入相邻 run 的 hb_buffer，使 GPOS 附着与草书连接在回退边界处仍然成立；或优先推动默认开启 whole-text-layout 路径。
3. **行高稳定化选项**：提供渲染选项，使 `lines[].asc/desc` 优先取主 face（face_index 0）的度量而非全行 max，避免单个回退字符抬行高导致的整事件跳动（ass_render.c:1639）。
4. **装饰线归一**：下划线/删除线统一采用主 face 的 `post`/OS/2 值（ass_font.c:844-869），消除回退边界处的线宽/位置突变。
5. **回退预算与日志**：`ASS_FONT_MAX_FACES` 触顶时目前静默降级为 .notdef，建议增加一次性 MSGL_WARN；同时把 §5 的"fontselect: Using default/fallback"日志与最终 face_index 关联，便于定位缺字。
6. **测试补齐**：在 `compare/test/` 增加三类用例（框架已支持目录内自带字体）：(a) 主字体缺字 + 回退字体度量差异大的拉丁/CJK 混排行高用例；(b) 基字符 + 组合标记分别落在两个字体的附着用例；(c) ZWJ 序列跨样式 run 的连字用例。这些用例的 PNG 基线即对应 §4/§5 列出的可观察结果。
