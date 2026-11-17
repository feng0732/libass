# libass 卡拉OK高亮在双向文字中的错位：静态调查报告

- 仓库：`/srv/fb2/feng0732/0014-libass-1-b3a57225`
- 版本：`RELEASEVERSION` = `0.17.5`，HEAD `21b5a54`
- 方法：**纯静态分析**（通读 `libass/ass_parse.c`、`ass_render.c`、`ass_shaper.c`、`ass_outline.c`、`ass_bitmap.c`，以及 `test/`、`compare/`、`fuzz/` 与 CI 配置）。**未编译、未启动任何播放或渲染程序。**
- 标注约定：**[事实]** 源码直接可验证；**[推导]** 由数据流静态推导出的视觉结果。

---

## 0. 结论摘要

1. **[事实] 时间轴与绘制分段是两套划分，只是共用同一个标志 `starts_new_run`。**
   时间轴由 `ass_process_karaoke_effects()`（`ass_parse.c:1025-1107`）在**逻辑顺序**上按 `starts_new_run` 切词；绘制由 `render_and_combine_glyphs()`（`ass_render.c:2447-2578`）把**同样的** `starts_new_run` 区间合成一张 `CombinedBitmapInfo` 位图，再由 `render_text()`（`ass_render.c:820-878`）用**一条竖线**切开。

2. **[事实] `\k`/`\ko` 是整段布尔开关，`\kf` 是段内一条竖线。**
   `\k`/`\ko` 的 `tm_end` 被强制回退为 `tm_start`（`ass_parse.c:1071-1072`），永不进入插值分支，只有 `x = ±1e8` 两种结果，靠符号选 `c[0]`/`c[1]`（`ass_render.c:857-868`）。只有 `\kf` 才把比例 `dt` 换算成屏幕竖线。

3. **[事实] 错位的核心公式在 `ass_render.c:2576-2578`：**

   ```c
   info->effect_timing = lround(d6_to_double(info->leftmost_x) +
       d6_to_double(info->effect_timing) * render_priv->par_scale_x);
   ```

   竖线 = **该段墨迹最左 x**（`leftmost_x`，`ass_render.c:1404-1405`、`ass_outline.c:473-485`）+ **相对段首字形 `start`、在重排前冻结的逻辑偏移**（`ass_parse.c:1104`）。LTR 时两原点重合；RTL 时 `start`（逻辑首字形）在视觉最**右**，而 `leftmost_x` 属于逻辑**最后**一个字形。**[推导]** 于是竖线沿**视觉左→右**推进，而阅读方向是视觉右→左：填色前沿与阅读方向相反，阅读顺序中**靠后**的字母先亮、靠前**的字母后亮。若以视觉左端为 0 计量，前沿位置是 `dt·W`；按阅读顺序（自右向左）计量的正确前沿应是 `(1-dt)·W`，即实现了 `dt → 1-dt` 的左右镜像。

4. **[事实] 默认配置（`ASS_FEATURE_WHOLE_TEXT_LAYOUT` 关闭）下，每个 `\k` 词还会各自成为一个 bidi “段落”**（`ass_shaper.c:1007-1010`、`1128-1131`），段落基准方向取自 `Encoding`（`ass_shaper.c:1160-1167`，只有 `Encoding=-1` 才得到 `FRIBIDI_PAR_ON`）。`ass.h:256-262` 明言：此时「文本被切成 VSFilter 兼容的段，每段独立处理；其中 bidi 与 shaping 都在每个 run 内单独执行；随后各 run **按从左到右顺序排列，即使其中含 RTL 文本**」。所以默认模式整行词序是逻辑左→右（VSFilter 兼容），词内部才反转；开启 whole-text-layout 后词序才是真正的视觉顺序。

5. **[事实] 分段缺失/重叠不会崩溃，但会静默改动时间链。** 全跳过的段不产生位图（`ass_render.c:2462-2463`、`2571-2573`），但 `timing` 照样前进；`\k0` 词不切段（`ass_render.c:2015`）并并入前一段；负值/溢出时长会污染后续所有词的 `timing` 累加器（`ass_parse.c:1065-1067`）。插值分支被 `tm_start <= t < tm_end` 严格保护（`ass_parse.c:1074-1088`），故**不会除零、不会产生 NaN**。

6. **[事实] 仓库内不存在针对 karaoke 或双向文字的回归样例。** `compare/test/` 只有旋转与模糊两个样例（`compare/test/sub1.ass`、`sub2.ass`，各 1 条事件，无 `\k*`、无 RTL 字符）；`test/test.c` 是单帧导出工具；`fuzz/ass.dict:136-140` 只把 `\kf \K \ko \kt \k` 作为变异字典词条。CI 的 `meson test` 只注册了 `art-compare` 与 `art-fuzz`（`compare/meson.build:19`、`fuzz/meson.build:22`），素材来自外部仓库 `libass/libass-tests`（`.github/workflows/meson.yml:93-97`、`167`），本仓库内不可见。

---

## 1. 事件时间与分段边界如何映射到视觉字形簇

### 1.1 时间基准

- **[事实]** `ass_start_frame()` 每帧写入 `render_priv->time = now`（`ass_render.c:3064`）；只有满足 `event->Start <= now < event->Start + event->Duration` 的事件被渲染（`ass_render.c:3382-3385`，`ass_render.h` 中 `Start`/`Duration` 为毫秒，`ass_types.h:141-142`）。
- **[事实]** 卡拉OK全部时间都是**相对事件起点**：`tm_current = renderer->time - event->Start`（`ass_parse.c:1028`）。
- **[事实]** 标记时长单位是厘秒：`\k/\kf/\ko` 都把参数 `*10` 后经 `dtoi32()` 截断（`ass_parse.c:845-868`，`dtoi32()` 定义在 `ass_parse.c:121-126`）。`\kt` 则把参数 `*10` 存进 `effect_skip_timing`，同时 `effect_timing = 0`、`reset_effect = true`（`ass_parse.c:837-844`）。

### 1.2 解析阶段：标记如何附着到“字符”

- **[事实]** 每个 `{...}` 块在 `parse_events()` 内被 `ass_parse_tags(state, p, q, 1., false)` 就地执行（`ass_render.c:2066-2069`）；随后每读出一个字符就生成一个 `GlyphInfo`，并把当时 `RenderContext` 里的 `effect_type / effect_timing / effect_skip_timing / reset_effect` **快照**进去（`ass_render.c:2125-2128`），随即清零（`ass_render.c:2166-2169`）。
- **[事实]** 因此标签的“作用对象”是**紧随其后的那个码点/绘制对象**，而不是字形簇、更不是视觉字形。绘制对象（`\p`）走另一分支，以 `code = 0xfffc` 的单个 `GlyphInfo` 表示整段矢量图（`ass_render.c:2070-2080`）。
- **[事实]** 一个 `GlyphInfo` 对应一个**原始码点**；字符→字形的膨胀发生在 shaping 之后（见 1.4）。

### 1.3 运行分段（`starts_new_run`）的三次写入

`starts_new_run` 是唯一贯穿“时间轴—bidi 段落—绘制分段”的分段标志。

1. **[事实] 样式分段**：`split_style_runs()`（`ass_render.c:2004-2047`）在 shaping 前给每个 `GlyphInfo` 标位。条件包括 `info->effect_timing`（**故意忽略 `effect_skip_timing`**，`ass_render.c:2015`）、`effect_type` 变化、绘制对象、字体/字号/颜色/边框/阴影/旋转/斜切/缩放/间距/粗斜体/装饰位等任一变化。注意 `glyphs[0]` 恒为真（`ass_render.c:2009`）。
2. **[事实] 折行分段**：`wrap_lines_smart()` → `trim_whitespace()`（`ass_render.c:1657-1721`）标记行首/行尾空白为 `skip` 且 `is_trimmed_whitespace`，并在行首首个非空白处、以及每个 `linebreak` 处设置 `starts_new_run = true`（`ass_render.c:1683-1684`、`1717-1718`）。折行点由 `wrap_lines_naive()` 写入 `glyphs[lead].linebreak = break_type`（`ass_render.c:1786-1791`）。
3. **[事实]** 初始 `calloc` 为全零，故其余字符的 `starts_new_run` 为假（`ass_render.c:57-61`）。

### 1.4 shaping：码点 → 字形簇

- **[事实]** `shape_harfbuzz_process_run()` 用 `glyph_info[j].cluster + offset` 把 HarfBuzz 字形回写到**根** `GlyphInfo`（`ass_shaper.c:637-639`）。若一个簇产出多个字形，就 malloc 一个 `GlyphInfo` 并挂到 `info->next` 链上（`ass_shaper.c:641-652`）。每个字形的 `advance`/`offset` 记录在链上，而其总和累加到**根**的 `cluster_advance`（`ass_shaper.c:655-664`）。
- **[事实]** 库未设置 `HB_BUFFER_CLUSTER_LEVEL`（全仓库 grep 无命中），故沿用 HarfBuzz 默认簇级；多码点连字（如阿拉伯文 lam-alef）会被归入**较低的 cluster 索引**，即文章中的“逻辑首个码点”在字形产出里可能代表多个码点。
- **[事实]** `retrieve_glyphs()` 会为“斜体→非斜体”处追加 `cluster_advance`（`ass_render.c:2193-2202`），并统一追加 `hspacing_scaled`（`ass_render.c:2204-2205`）。
- **结论 [事实]** “分段边界”永远落在**根 GlyphInfo 的索引**上；一个簇的所有附加字形**不能**被拆到别的段里，因为遍历与切分都只按根索引（`ass_render.c:2459-2465` 只沿 `next` 向内展开，不再改变段边界）。

### 1.5 时间轴：逻辑顺序上的切词与累加

`ass_process_karaoke_effects()` 的主体（`ass_parse.c:1034-1106`）：

- **[事实]** 循环 `i = 0..length`（含尾哨兵），凡 `glyphs[i]` **不是**段首就 `continue`，只在该处把 `skip_timing` 累积起来（`ass_parse.c:1035-1049`）。因此“词边界”= `starts_new_run` 为真的索引（外加末尾哨兵）。
- **[事实]** 每遇到一个边界，就把 `[last_boundary, i)` 视作一个词。`start` 为空时（第一个边界之前）直接跳过（`ass_parse.c:1054-1055`）。
- **[事实]** 时间推进：

  ```c
  long long tm_start = timing + start->effect_skip_timing;
  long long tm_end   = tm_start + start->effect_timing;
  timing = !has_reset * tm_end + skip_timing;
  ```

  （`ass_parse.c:1065-1067`）`timing` 是**词起点的时间累积器**，逐词串行相加。`has_reset` 由段内出现 `reset_effect`（即 `\kt`）置位（`ass_parse.c:1038-1041`）：它把 `skip_timing` 清零，并在结算时令 `!has_reset` 为 0，从而**丢弃该段的终点**，使下一词起点回落到 `skip_timing`（含 `\kt` 自己的值）。详见 §2.3。
- **[事实]** `\k`/`\ko` 强制 `tm_end = tm_start`（`ass_parse.c:1071-1072`），故其 `x` 只可能是 `-1e8` 或 `+1e8`；`\kf` 才保留区间做插值。
- **[事实]** 视觉坐标只取该段**首尾可见根字形**：`first_visible` 从 `start` 向后跳过 `skip`，`last_visible` 从 `end-1` 向前跳过 `skip`（`ass_parse.c:1080-1084`）；

  ```c
  int x_start = first_visible->pos.x;
  int x_end   = last_visible->pos.x + last_visible->advance.x;
  double dt   = (double)(tm_current - tm_start) / (tm_end - tm_start);
  ```

  （`ass_parse.c:1086-1088`）注意 `x_end` 用的是 **`advance`**（单字形），不是 `cluster_advance`；对多字形簇这是**少算**了附加字形的宽度。

- **[事实]** `frz` 在 `(90,270)` 开区间时做“从右往左”填充，并把**整段**的 `c[0]`/`c[1]` 互换（`ass_parse.c:1089-1098`）；然后

  ```c
  x = x_start + ass_lrint((x_end - x_start) * dt);
  ```

  （`ass_parse.c:1099`）最后把绝对值换算成**相对每个字形原点**的偏移：

  ```c
  for (GlyphInfo *info = start; info < end; info++) {
      info->effect_type = effect_type;
      info->effect_timing = x - info->pos.x;
  }
  ```

  （`ass_parse.c:1102-1105`）

- **[事实] 关键前提**：此刻仍是**重排之前**的布局，`pos.x` 沿**逻辑顺序**单调递增（`preliminary_layout()` 从 0 起累加，`ass_render.c:2209-2228`；折行后 `wrap_lines_naive()` 也可能写入新的行首，但 `reorder_text()` 尚未执行——见 `ass_render.c:2857-2862` 的顺序，且 `ass_render.c:2859` 的注释明确说明“依赖 run 内 x 坐标单调，所以必须在重排前完成”）。

### 1.6 重排：逻辑索引 → 视觉位置

- **[事实]** `reorder_text()` 取 `ass_shaper_reorder()` 返回的 `cmap`，然后**按视觉顺序**重新走笔：

  ```c
  for (int i = 0; i < text_info->length; i++) {
      GlyphInfo *info = text_info->glyphs + cmap[i];
      if (text_info->glyphs[i].linebreak) { ... 换行 ... }
      if (info->skip) continue;
      ASS_Vector cluster_pen = pen;
      pen.x += info->cluster_advance.x;
      ...
      while (info) { info->pos.x = info->offset.x + cluster_pen.x; ... }
  }
  ```

  （`ass_render.c:2247-2268`）即 `pos.x` 被**覆盖**为视觉坐标，而 `effect_timing` 里冻结的“相对偏移”**不随之更新**。
- **[事实]** `cmap` 由 `fribidi_reorder_line()` 逐“段”生成（`ass_shaper.c:1111-1145`）。reorder 的段界是 `glyphs[i+1].linebreak`、`FRIBIDI_TYPE_BS`、以及**仅在 `whole_text_layout` 关闭时**的 `glyphs[i+1].starts_new_run`/`glyphs[i].hspacing`（`ass_shaper.c:1128-1131`）。
- **[事实] 嵌入级别（`emblevels`）的段落划分与 reorder **不完全相同**：`ass_shaper.c:1007-1010` 的段界只含 `FRIBIDI_TYPE_BS` 与（非 whole-text 时的）`starts_new_run`/`hspacing`，**不含 `linebreak`**。因此一段被软折行切开的文本，其 bidi 级别仍按整行（或整个 run）计算，而 `cmap` 的重排是分行的。**[推导]** 这在正常情况下列方向一致、结果无差别（换行点两侧方向通常不互相影响），但它意味着“折行”与“bidi 段落”并非同一个概念，不能把二者当作等价的段落边界。
- **[事实]** `shape_harfbuzz()` 只接受 `shape_run_id` 与**嵌入级别都相同**的连续字形（`ass_shaper.c:695-701`），并据此把 `HB_DIRECTION` 设为 RTL/LTR（`ass_shaper.c:723-724`）。

**映射链总结 [事实]：**

```
event.Start ──(tm_current)──▶ ass_process_karaoke_effects
                                   │ 按 starts_new_run 切词（逻辑索引）
                                   ▼
                        x（逻辑坐标下的分界点）
                                   │ effect_timing = x - pos.x   （逐字形）
                                   ▼
                              ... 折行 / 重排 ...
                                   │ pos.x 被视觉化覆盖
                                   ▼
render_and_combine_glyphs：按 starts_new_run 合并位图，leftmost_x = 视觉最左墨迹
                                   │ effect_timing = leftmost_x + (x - pos.x)*par
                                   ▼
render_text → render_glyph(brk)：一条屏幕竖线把**整段合成位图**左右染色
```

结论：**时间只在逻辑域推进，颜色只在视觉域切割；两者的“原点”并不一致**，一致性只在 LTR + 无旋转 + `par_scale_x = 1` 时成立。

### 1.7 位图合成与切分

- **[事实]** `render_and_combine_glyphs()` 从逻辑索引顺序遍历，`starts_new_run` 置位即开新段（`ass_render.c:2461`），段内所有字形（含 `next` 链）合成到 `current_info->bitmaps`，并累加 `leftmost_x`（`ass_render.c:2506`、`2545`）。
- **[事实]** 段首的 `effect_type/effect_timing` 被**整段复制**一次（`ass_render.c:2504-2505`），段内后续字形自己的 `effect_timing` 被忽略。这意味着**段内所有字形共享同一个切分位置**。
- **[事实]** `\kf` 的最终切分位置在此计算（`ass_render.c:2576-2578`）：`leftmost_x`（26.6 设备亚像素）加上 `effect_timing`（26.6 逻辑亚像素，补乘 `par_scale_x`），结果直接当作**设备像素整数**使用，随后在 `render_text()` 中作为 `brk` 传入 `render_glyph()`（`ass_render.c:869-873`），并在函数内减去 `bm->left` 变成位图内坐标（`ass_render.c:443-445`），最后按 `brk > b_x0` / `brk < b_x1` 画左右两块（`ass_render.c:473-494`）。反裁剪路径 `render_glyph_i()` 同样逻辑（`ass_render.c:382-407`），最多拆出 6 张位图（`ass_render.c:316-317`）。
- **[事实]** `\k`/`\ko` 不走 `brk`：整段只染一种颜色（`ass_render.c:857-868`）；`\ko` 还在 `effect_timing <= 0` 时**连轮廓都不画**（`ass_render.c:842-844`）。

---

## 2. 分段重叠或缺失字形时应出现的结果

下面逐条给出静态可推出的行为（均标注证据）。

### 2.1 段内出现“被跳过”的字形（缺失字形）

- **[事实]** 两类 `skip`：
  - 剪掉的空白：`trim_whitespace()` 置 `skip + is_trimmed_whitespace`（`ass_render.c:1669-1714`）；
  - 零宽/默认可忽略字符：`is_harfbuzz_ignorable()` → `ass_shaper_skip_characters()`（`ass_shaper.c:846-874`）。
- **[事实]** 时间轴对空白**不做跳过**：`ass_process_karaoke_effects()` 只用 `skip` 去找首尾**可见**字形（`ass_parse.c:1080-1084`）。若整段全被 `skip`，两个 `while` 都因 `first_visible < last_visible` 不成立而停在 `end - 1`，于是 `first_visible == last_visible == end - 1`，`x_start = pos(end-1)`、`x_end = pos(end-1) + advance(end-1)`，仍会算出一个值。
- **[推导]** 若词的首尾都是空白（例如 `{\k20} A B{\k20}` 因折行/裁剪导致行首尾空白被剪），可见范围收缩成 `end - 1` 一个字形，高亮几何退化为**该段最后一个字形**处一条约 1 个 advance 宽的窄条；同时该段在绘制时可能一张位图都没有（见 2.3）。
- **[事实]** `\kf` 的 `leftmost_x` 由 `ass_outline_update_min_transformed_x()` 对**每个实际生成位图的字形**取最小（`ass_render.c:1404-1405`、`ass_outline.c:473-485`）；被 `skip` 的字形在 `get_bitmap_glyph()` 入口即返回（`ass_render.c:1391`），因此不参与 `leftmost_x`。
- **[推导]** 于是“时间域取首尾（含可能的 skip 退化）+ 视觉域取真实墨迹最左”这两个原点，在**含被剪空白的词**里也会分开。

### 2.2 段重叠（`\k0` 或跨标签的连续标记）

- **[事实]** `split_style_runs()` 用 `info->effect_timing` 判断切段，且**注释明确忽略 `effect_skip_timing`**（`ass_render.c:2015`）。因此 `{\k12345\k0}` 之后**没有** `starts_new_run`，第二个标记不会开新段。
- **[事实]** 这种情形的时间处理被 `ass_process_karaoke_effects()` 的兼容分支专门覆盖：

  ```c
  // VSFilter compatibility: if we have \k12345\k0 without a run
  // break, subsequent text is still part of the same karaoke word,
  // the current word's starting and ending time stay unchanged,
  // but the starting time of the next karaoke word is advanced.
  skip_timing += (uint32_t) text_info->glyphs[i].effect_skip_timing;
  ```

  （`ass_parse.c:1043-1047`）即：`\k0` 词的时长**不延长本段**，而是累加到 `skip_timing`，在下一个边界处并入下一段的起点（`ass_parse.c:1067`）。
- **[事实]** 解析侧在遇到新的 `\k*` 时已经把上一标记的时长搬进 `effect_skip_timing`（`ass_parse.c:850-851`、`858-859`、`866-867`），保证“一个预算只被消费一次”。
- **[推导]** 视觉上：两个标记只对应**一条**分段与**一条竖线**，因为 `\k0` 不开段、段内共享 `effect_timing`（`ass_render.c:2504-2505`）。注意 `\k0` **本身**的时长是 0，所以“总时长”等于前一词的时长；被 `\k0` 从**前一标记**搬运过来的时长（解析侧 `ass_parse.c:850-851`）则作为 `effect_skip_timing` 记在 `\k0` 所在字符上，经 `ass_parse.c:1047` 累加起来，**不属于**本段亮度进度，而是加到下一段的起点。`ass_render.c:2015` 的 `effect_timing ||` 只在 `effect_timing != 0` 时断段，因此 `\k0` 不产生任何颜色切换点，与注释所述 VSFilter 行为一致。
- **[推导] 但样式变化会破坏“一词一段”**：若词中夹着样式变化（如 `{\k50}foo{\c&HFF&}bar{\k0}`），`bar` 会因 `c[0]` 变化而满足 `split_style_runs()` 的断段条件（`ass_render.c:2022`），原词被切成两段。此时 `\k0` 承载的 `effect_skip_timing` 落在第二段段首，第一段的 `x_start/x_end` 只覆盖 `foo`，观众看到的是“同一词上两条独立的扫光（其中一条长度为 0）”。

### 2.3 段内完全没有可见字形（缺段）

- **[事实]** `render_and_combine_glyphs()` 遇到 `info->skip` 直接 `continue`（`ass_render.c:2462-2463`），故段头字形若全被 skip，`current_info` 甚至不会建立（`new_run` 在段首才被消费，`ass_render.c:2489-2539`）。
- **[事实]** 若段建立了却没有任何位图（例如 `\k` 落在零宽字符上，或 `\fscy0` 导致 `scale_y=0`），则 `bitmap_count == 0`，该段被 `free(info->bitmaps); continue;` 丢弃（`ass_render.c:2570-2573`）。
- **[推导]** 时间轴并不会因此回退：`timing` 已在 `ass_process_karaoke_effects()` 里累加过了（`ass_parse.c:1067`），所以后续词的开始时间**照常**向后推进。屏幕上表现为“某个词标记的时间被吞掉，后续高亮按原时间表继续”，不会错位到上一词，也不会出现“卡住不动”。
- **[事实] `\kt` 的语义是“把时间基准重置为一个绝对值”，不是叠加延迟。** `\kt` 只写 `effect_skip_timing`，并置 `effect_timing = 0`、`reset_effect = true`（`ass_parse.c:837-844`）。随后它有两种被消费的方式，取决于下一个字形是否开新段：
  - **下一个字形开新段**（`{\k50}A{\kt20}{\k50}B`，B 的 `effect_timing = 500` 非零）⇒ 在 B 的边界处，`start` 就是 B，于是 `if (start->reset_effect) timing = 0;`（`ass_parse.c:1062-1063`）先把累积时间清零，再 `tm_start = timing + start->effect_skip_timing = 200`（`ass_parse.c:1065`）。**B 在事件起点后 200ms 点亮，A 原来的 500ms 被丢弃。**
  - **下一个字形不开新段**（`{\k50}A{\kt20}B{\k50}C`，B 的 `effect_timing = 0` 且 `effect_type` 已清空 ⇒ `ass_render.c:2015` 不置 `starts_new_run`）⇒ B 并入 A 的段；时间轴在 B 处走非边界分支，看见 `reset_effect` 便置 `has_reset = true` 并**清零** `skip_timing`（`ass_parse.c:1038-1041`），然后累加 B 的 200。段结算时 `timing = !has_reset * tm_end + skip_timing = 0 + 200`（`ass_parse.c:1067`）——**A 的 500ms 终点被丢弃，C 在 200ms 点亮**。副作用是 B 与 A 同段，因此 B 在 `t=0` 就随 A 一起点亮。

  两条路径的净效果一致：**`\kt` 把“下一词起点”重设为 `\kt` 参数（相对事件起点），并抹掉此前累积的时长**。`ass_render.h:152` 的 “delay after the end of last karaoke word” 注释描述的是它在 `tm_start = timing + effect_skip_timing` 中的**加数**角色，但在 `has_reset` / `start->reset_effect` 生效时，这个“加数”会连同前面累积的 `timing` 一起被替换，因此在含 `\kt` 的事件里不能简单把它理解为“后一词延后 N ms”。这是 `\kt` 最容易被误读的静态行为。

### 2.4 段边界的精度与单位

- **[事实]** `effect_timing` 在不同阶段有两种单位，**这是读代码最容易出错的地方**：
  - **时间轴阶段：毫秒。** `\k*` 参数是厘秒，×10 后写入（`ass_parse.c:845-868`）；`\kt` 的 `effect_skip_timing` 同样是厘秒 ×10（`ass_parse.c:839-842`）。`tm_start/tm_end` 与 `state->renderer->time - event->Start`（`ass_parse.c:1028`，毫秒）直接相加，单位一致。
  - **绘制阶段：26.6 亚像素偏移。** `ass_process_karaoke_effects()` 末尾把它覆写为 `x - info->pos.x`（`ass_parse.c:1104`），此时两边都是逻辑布局的 26.6 坐标。
  - `ass_render.h:149-151` 的注释（“time duration” / “distance in subpixels”）描述的就是这两次含义切换。
- **[事实]** `leftmost_x` 是 26.6 **设备**亚像素，clamp 到 `±OUTLINE_MAX`（`ass_outline.c:482`，`OUTLINE_MAX = (1<<28)-1`，`ass_outline.h:87`）；`get_bitmap_glyph()` 只在 `effect_type == EF_KARAOKE_KF` 时更新它（`ass_render.c:1404-1405`），因此 `\k`/`\ko` 段的 `leftmost_x` 保持初值 `OUTLINE_MAX` 且**不参与**计算（`ass_render.c:2576` 的 `if` 排除）。
- **[事实]** `effect_timing` 作为“相对字形原点的偏移”在**逻辑布局空间**中产生（`x - pos.x`，`ass_parse.c:1104`），而 `leftmost_x` 在**设备像素空间**中测量（`ass_render.c:2545` 传入的 `pos` 已由 `ass_render.c:2543` 乘过 `par_scale_x`）。`ass_render.c:2577-2578` 对偏移部分补乘 `par_scale_x`，两者量纲因此一致——**PAR 本身不引入错位**，只是把逻辑偏移换算到与 `leftmost_x` 同一空间。
- **[推导]** 亚像素误差来源有三：`dt` 的量化（标记只有 10ms 粒度）、`effect_timing` 的整型化、以及 `x_end` 用 `advance` 而非 `cluster_advance` 造成的多字形簇**系统性偏窄**。对 LTR 表现为“扫光略早于字尾到达”；对 RTL 这些误差会叠加在镜像错误上。

### 2.5 极端/畸形取值

- **[事实]** 负时长：`\k-1` → `dtoi32(-10) = -10`，`tm_end = tm_start - 10`；`tm_current >= tm_end` 立即成立 → 立刻判为“已完成”（`ass_parse.c:1077-1078`）。而 `timing = tm_end + skip_timing` 会让下一词的起点**倒退 10 毫秒**（`ass_parse.c:1067`）。
- **[事实]** 超大时长：`dtoi32()` 对 `>= INT32_MAX+1` 的输入返回 `INT32_MIN`（`ass_parse.c:123-124`），即一个**巨大的负值**，触发的也是上面的“倒退”路径。`effect_skip_timing` 在解析时以 `(uint32_t)` 转换累加（`ass_parse.c:850-867`），在时间轴里再以 `(uint32_t)` 转换累加（`ass_parse.c:1047`），回绕语义是静默的。
- **[事实]** `tm_start`/`tm_end` 是 `long long`（`ass_parse.c:1065-1066`），不会溢出，但可以远超事件时长，使词永远“未开始/已完成”。
- **[事实]** `\kf` 若整段没有任何 `starts_new_run` 之前的内容（即以 `\k` 开头但有前导空白被剪），则首边界是 `i=0`，`start == NULL` → `continue`（`ass_parse.c:1054-1055`），这段的 `effect_type` 仍是 `EF_NONE`；不过 `split_style_runs()` 已把 `glyphs[0]` 置为段首（`ass_render.c:2009`），所以实际上 `start` 只在 `i` 为**第一个真边界之前**才为空，即仅第一轮，随后轮次总能用上一轮边界。
- **[事实]** `\k` 与 `\kf` **并列**时：后写的标记覆盖 `effect_type` 并走同一个 `effect_skip_timing` 搬运（`ass_parse.c:845-868`），因此无“双效果叠加”，最终类型是**最后一个**写到该 `GlyphInfo` 的标签类型。`\k` 之后的 `\kf` 不会在同一段里出现第二条竖线。

### 2.6 旋转/镜像

- **[事实] 颜色互换只对 `\kf` 生效，对 `\k`/`\ko` 不可达。** `frz` 的“右→左”判断与 `c[0]`/`c[1]` 整段互换位于 `ass_process_karaoke_effects()` 的**插值分支内部**（`ass_parse.c:1089-1098`，即 `tm_start <= tm_current < tm_end` 的 `else` 块）。而 `\k`/`\ko` 在进入分支前已把 `tm_end` 设回 `tm_start`（`ass_parse.c:1071-1072`），该条件恒假，故**永远走不到互换代码**；`\k`/`\ko` 的颜色只由 `effect_timing` 的符号决定（`ass_render.c:857-868`）。因此旋转下 `\k`/`\ko` 不会出现“已唱/未唱颜色对调”，而对 `\kf` 会。
- **[事实]** `leftmost_x` 是**变换后**的 x（`ass_outline.c:477-479` 使用完整矩阵 `m`），所以对 `frz=180` 或负 `scale_x` 这类镜像变换，它取到的是屏幕左侧的墨迹。**[推导]** 而 `effect_timing` 是在**重排前的未镜像逻辑空间**中算出的偏移 `x_logical − S`（`ass_parse.c:1104`），镜像后不会自动跟随；`frz∈(90,270)` 时 `dt → 1-dt` 只翻转了比例，并未把 `x_start/x_end` 投影到屏幕空间。于是镜像变换下 `L`（屏幕空间）与 `x_logical − S`（逻辑空间）虽同量纲，方向约定却不一致，构成另一处潜在错位（**静态观察，未实测**）。

---

## 3. 场景覆盖：正常文本、双向文字、畸形分段

以下每个场景给出**可直接手写**的事件文本与**应出现的边界**（推导线），并附源码/测试证据。所有示例假设：默认样式、`Encoding` 按各例说明、无 `\frz`、`par_scale_x = 1`、字号与字体均可显示 ASCII 与希伯来/阿拉伯字母。

### 3.1 正常文本（LTR，`Encoding=1` 即 `FRIBIDI_PAR_LTR`）

**事件**：`{\k30}A{\k50}B{\k40}C`

- 解析（`ass_render.c:2125-2129`）：`A/B/C` 三个 `GlyphInfo`，各自 `effect_type=EF_KARAOKE`、`effect_timing = 300/500/400`（厘秒→毫秒由 `*10` 完成）。
- 分段：`split_style_runs()` 因 `effect_timing` 非零把每个字形标为段首（`ass_render.c:2015`）→ 三个段。
- 时间（`ass_parse.c:1065-1067`）：`A:[0,300)`、`B:[300,800)`、`C:[800,1200)`。
- 绘制：`\k` 走“整段布尔”，`x = ±1e8`（`ass_parse.c:1071-1072`，`1075-1078`）→ 每段只有 `c[0]` 或 `c[1]`（`ass_render.c:857-868`）。
- **高亮边界 [推导]**：**注意 `\k` 在 `tm_start` 到达时立即点亮**，因为 `tm_end` 被置回 `tm_start`（`ass_parse.c:1071-1072`），故 `tm_current >= tm_end` 在词的**起点**即成立 → `x = +1e8` → `effect_timing > 0` → `c[0]`（`ass_render.c:859-863`）。
  - `t = 0..299`：A 已 `c[0]`（首词在事件起点就亮），B/C 仍 `c[1]`；
  - `t = 300..799`：A、B 为 `c[0]`，C 为 `c[1]`；
  - `t = 800..1199`：A、B、C 全为 `c[0]`。
  - 字幕**结束前**不会出现 `c[1]` 残留，除非某一词从未到达 `tm_start`（例如被后续 `\kt` 重置掉）。
- **测试证据 [事实]**：仓库内**没有**这个场景的样例；`compare/test/sub1.ass:12` 与 `sub2.ass:12` 是 `{\t(\frz+180)}A` 与 `{\blur1}Test_`，均不含 `\k*`。

### 3.2 正常文本 + `\kf`（LTR）

**事件**：`{\kf60}AB{\kf40}CD`，`Encoding=1`

- 时间（`ass_parse.c:1065-1067`）：AB:[0,600)、CD:[600,1000)。
- `\kf` 走插值：`tm_start <= t < tm_end` 时（`ass_parse.c:1074-1099`）
  `x = x_start + lrint((x_end-x_start) * (t-tm_start)/(tm_end-tm_start))`。
- `x_start` 取 `AB` 中首个非 skip 根字形的 `pos.x`，`x_end` 取最后一个根字形的 `pos.x + advance.x`（`ass_parse.c:1086-1087`）。
- 绘制期：`leftmost_x` 是该段所有生成位图字形**变换后**控制点的最小 x（`ass_render.c:1404-1405`）；LTR 无镜像时 `leftmost_x ≈ x_start`（受 hinting/量化影响，`POSITION_PRECISION=8`，`ass_render.c:45`）。
- **高亮边界 [推导]**：竖线从 AB 的**视觉左端**匀速走到右端，`t=300` 时约在 AB 中点；CD 在 `t=600` 起从自己的左端开始扫。**边界绝对位置**为 `lround(leftmost_x + (x - pos.x))`（`ass_render.c:2577`），LTR 下与 `x` 基本重合。
- **注意 `\kf` 的“越过区间”行为 [事实]**：`t >= tm_end` 时 `x = 100000000`（`ass_parse.c:1077-1078`），`render_glyph()` 的 `brk` 远大于位图右界，于是走 `brk > b_x0` 分支把**整个**位图染 `c[0]`，且 `brk < b_x1` 不成立 → 只产出一张图（`ass_render.c:473-494`）。因此 `\kf` 完成后与 `\k` 的终态一致（整段 `c[0]`），不会残留半亮。
- **测试证据 [事实]**：`fuzz/ass.dict:136-137` 收录 `\kf`、`\K`，说明这两个标签会进入模糊测试语料，但**该 harness 的 oracle 只是“不崩溃/无越界”**：`fuzz/fuzz.c:139-143` 断言 `ASS_Image` 的尺寸与 `dst` 在 frame 内、`stride >= w`；`144-147` 行（`ASSFUZZ_HASH_WHOLEBITMAP` 默认 0，`fuzz/fuzz.c:54-55`）只读最后一个像素。**它不校验颜色分区是否正确。**

### 3.3 双向文字（同段 LTR+RTL，默认模式）

**事件**：`{\k50}abc {\k50}אבג`（`Encoding=1`）

- 解析：7 个码点（含空格）→ 7 个 `GlyphInfo`；`effect_timing` 分别落在 `a`（索引 0，500）与 `א`（索引 4，500）上。
- 样式分段：`a`（`i=0`，恒段首，`ass_render.c:2009`）与 `א`（`effect_timing != 0`，`ass_render.c:2015`）各为段首，`b`/`c`/空格因 `effect_timing == 0` 且样式未变而不断段 → 两段 `[0,4)`=`abc␠` 与 `[4,7)`=`אבג`。
- **[事实] 该空格不会被裁剪**：`trim_whitespace()` 只裁**整段文本**的首尾空白与**紧邻 `linebreak`** 的空白（`ass_render.c:1666-1719`），行内空格不在其列，`IS_WHITESPACE` 也不含 `linebreak` 字形（`ass_render.c:1655-1656`）。因此空格保留 `skip == false`，会参与 `ass_process_karaoke_effects()` 的 `last_visible` 计算：第一段的 `x_end = pos(␠).x + advance(␠).x`（`ass_parse.c:1086-1087`），扫光右界**包含空格宽度**。
- **默认模式下 bidi 按“每段一个段落”**：`ass_shaper.c:1007-1010` 在 `!whole_text_layout` 时把 `starts_new_run` 视为段落界；`ass_shaper.c:1128-1131` 同理切分 `fribidi_reorder_line()`。`ass.h:256-262` 明确“每个 run 独立处理，随后**按从左到右**顺序排列”。所以**视觉上 `abc␠` 在左、`אבג` 在右**（VSFilter 兼容），且左段内部的字序不被右段影响。
- 段内 `אבג` 的嵌入级别为 RTL，该段自己的 `cmap` 把字序反转为视觉 `גבא`；`shape_harfbuzz()` 也以 `HB_DIRECTION_RTL` 塑形（`ass_shaper.c:723-724`）。
- **高亮边界 [推导]**：`t=0..499`：两段皆 `c[1]`；`t=500..999`：段 1 变 `c[0]`、段 2 保持 `c[1]`。RTL 段内字形簇为视觉 `גבא`，但它们**整段**同色，所以看起来是“整块高亮”，没有中间错位。**这个例子不暴露镜像问题**，因为两段都是 `\k`（整段布尔）。
- **测试证据 [事实]**：无语料；`Changelog:63-66`（0.17.2 条目）只说 “All BiDi paragraph separators now delimit BiDi runs”，并未涉及 karaoke。

### 3.4 双向文字 + `\kf`（默认模式）——**核心错位场景**

**事件**：`{\kf100}אבג`（`Encoding=1`，单段 RTL）

- 时间：`[0,1000)`；`\kf` 保留区间。
- 逻辑空间（重排前，`ass_render.c:2209-2228`）：`pos.x` 沿**逻辑**顺序 `א,ב,ג` 递增；`x_start = pos(א).x`，`x_end = pos(ג).x + advance(ג).x`（`ass_parse.c:1086-1087`）。
- `\kf` 的比例：`dt = (t - tm_start)/(tm_end - tm_start)`，`x = x_start + lrint((x_end-x_start)*dt)`（`ass_parse.c:1099`）。
- 重排（`ass_render.c:2247-2268`）：`pos.x` 被**视觉化覆盖**；视觉顺序是 `ג,ב,א`，所以 `pos(א).x` 现在位于**最右**。
- 关键事实：`effect_timing` 是在**重排前**算出的 `x - pos(א).x`（`ass_parse.c:1102-1105`），重排**不会**重算它。因此进入合成时，这个值是 `x_logical - x_start = (x_end - x_start) * dt = 段宽 × dt`，而 `leftmost_x` 是**视觉最左**（此处即 `ג` 的左缘，`ass_outline.c:477-479`）。
- 合成（`ass_render.c:2576-2578`）：

  ```
  brk = leftmost_x + (x_logical - x_start) * par_scale_x
      ≈ 视觉左端 + 段宽 * dt
  ```

  即 `dt=0` 时 `brk` 落在**视觉左端**，`dt=1` 时落在**视觉右端**。**注意：`pos(א).x` 在重排后虽已变成视觉最右，但它并未出现在最终公式里**——被加进 `leftmost_x` 的是重排前冻结的量。
- **高亮边界 [推导]**：竖线沿**视觉左→右**匀速推进（这与 LTR 的实现路径完全相同，代码中除 `frz` 外没有任何方向判断）。但在 RTL 段中，视觉左端是**阅读顺序的最后一个字母** `ג`，视觉右端才是第一个字母 `א`。于是填色从词的**尾部**开始、把阅读顺序**倒着**点亮：`ג` 先变 `c[0]`，`א` 最后。这就是题述的高亮错位；`\kf` 的“已唱量”在任意时刻与正确值相同（都是 `dt·W`），但填在了**错误的一端**，即实际填色区域是正确区域的镜像。
- 与 `\k` 对比：`\k` 在 RTL 段内是**整段**变色的，因而不受影响；这解释了“`\k` 正常、`\kf` 错位”的现象。
- 若段内夹着行内空格，例如 `{\kf100}אב ג`：该空格**不会**被 `trim_whitespace()` 剪掉（只剪整体首尾与 `linebreak` 邻近者，`ass_render.c:1666-1719`），因此它会成为 `last_visible`，使 `x_end` 额外包含空格宽度（`ass_parse.c:1087`），扫光在词尾之后还会继续走完一个空格；错位的主体（`x_logical − S` 是逻辑量、`leftmost_x` 是视觉量）**依旧**。
- **测试证据 [事实]**：无任何样例覆盖；`compare/test/` 的两个 `.ass` 无 RTL 字符（已逐一读取）。

### 3.5 双向文字 + `\kf` + 开启 `ASS_FEATURE_WHOLE_TEXT_LAYOUT`

**前提**：`ass_track_set_feature(track, ASS_FEATURE_WHOLE_TEXT_LAYOUT, 1)`，或样式 `Encoding = -1`（`ass_shaper.c:1160-1167` 返回 `FRIBIDI_PAR_ON`，`ass_shaper.c:921-923` 将其升级为 `WHOLE_TEXT_LAYOUT_IMPLICIT`）。

**事件**：`{\kf50}abc {\kf50}אבג`

- **[事实]** 此时 `ass_shaper.c:1007-1010` 与 `1128-1131` 的 `!whole_text_layout` 分支**失效**，段落界只剩 `BS` 与 `linebreak`；整行一次性跑 `fribidi_get_par_embedding_levels` / `fribidi_reorder_line`。
- **[推导]** `cmap` 现在把 `אבג` 放在 `abc` 的**左侧**：整行首个强方向字符是 `a`（L 类），故段落基准为 LTR；按 UBA，LTR 段落中的 RTL run 排在它前面的逻辑内容右侧，`abc` 与 `אבג` 之间的空格为中性、按段落方向解析为 L。于是视觉顺序为 `אבג` + 空格 + `abc`，即 **RTL run 在左、LTR run 在右**。
- **[推导]** 两个 `\kf` 段仍然各自按逻辑坐标算 `x`，再各自加自己的 `leftmost_x`——**镜像错误在每段内部完全照旧**（3.4）。额外多了一个错位来源：段的**水平位置**变了，但 `effect_timing` 记录的是相对逻辑 `pos.x` 的偏移，重排后没有重算（不过这属于“段整体平移”，段内比例不受影响）。
- **[事实]** `shape_harfbuzz()` 在 whole-text 模式下把整段文本喂给 HarfBuzz（`ass_shaper.c:706-708`），`cluster` 变为**全事件索引**，`shape_harfbuzz_process_run()` 的 `offset` 也改成 0（`ass_shaper.c:742`）；这会影响多码点簇的归属，但**不改变** `starts_new_run` 边界的判定（仍是 `GlyphInfo` 根索引）。
- **高亮边界 [推导]**：视觉上 `אבג` 在左、`abc` 在右。`t=0..499` 时扫的是 `abc` 段：该段内 `a` 是逻辑首、视觉最左，所以是**唯一不出现镜像错误的 RTL 混排情形**（LTR run 内部正常）。`t=500..999` 时扫 `אבג` 段：该段内部 RTL，前沿沿视觉左→右推进，即仍然先点亮 `ג`、后点亮 `א`（3.4）。因此**整行看起来正确，只有 RTL run 内部反向**；若把事件写成 `{\kf50}אבג {\kf50}abc`，则首个 run 就是 RTL，错位立即出现。
- **测试证据 [事实]**：`ass.h:256-274` 的文档注释即为此行为的规格说明；`Changelog:144-146` 记载该特性为 0.16.0 新增，且“非可移植的 `Encoding=-1` 模式总是表现得如同该特性已启用”。无 karaoke 用例。

### 3.6 畸形分段（缺失、重叠、空段）汇总

| 场景 | 事件示例 | 静态结果 | 证据 |
|---|---|---|---|
| `\k0` 并入前词 | `{\k30}A{\k0}B{\k40}C` | `\k0` 不开段（`effect_timing==0`），故 B 与 A 同段；`\k0` 不会延长该段，C 仍在 300ms 起 | `ass_render.c:2015`；`ass_parse.c:1043-1047` |
| 连续 `\k` 无字符间隔 | `{\k30}{\k40}A` | 解析时第二个标签把第一个的时长搬进 `effect_skip_timing`（`ass_parse.c:850-851`），随后清零；A 只承载 400，另 300 成为相对 A 起点的延迟（`ass_parse.c:1065`） | `ass_parse.c:850-851`、`1065` |
| 段内仅空白 | `{\k30} {\k30}A` | 空格段在 `trim_whitespace()` 变 `skip`；若它是段首则整段无位图被丢弃；但 `timing` 已推进 300 | `ass_render.c:1669-1673`、`2462-2463`、`2571-2573`；`ass_parse.c:1067` |
| 零宽字符承载标记 | `{\k30}​{\k50}A` | U+200B 被 `is_harfbuzz_ignorable()` 置 skip（`ass_shaper.c:853`）；段内首尾可见字形退化为 `A`；30 厘秒仍被消费 | `ass_shaper.c:846-874`；`ass_parse.c:1080-1084` |
| `\kt` 重置基准 | `{\k50}A{\kt20}B{\k50}C` | `\kt` 自身不切段；B 也不切段（`effect_timing==0`）。`has_reset` 使本段 500ms 终点被**丢弃**，C 的起点被重设为 **200ms**（而非 500ms） | `ass_parse.c:837-844`、`1038-1041`、`1062-1063`、`1067` |
| `\kt` + 紧随 `\k` | `{\k50}A{\kt20}{\k50}B` | 无字形间隔，`\kt` 的 200 留在 B 的 `effect_skip_timing` 上；B 切段 ⇒ `start->reset_effect ⇒ timing=0`，B 在 **200ms** 点亮，A 的 500ms 被丢弃 | `ass_parse.c:842-851`、`1062-1065` |
| 负/溢出时长 | `{\k-1}A{\k50}B` | `tm_end < tm_start` → 立刻“完成”；`timing` 倒退，后续词提前 | `ass_parse.c:121-126`、`1065-1067`、`1077-1078` |
| 绘制段夹在两个标记之间 | `{\k30}A{\p1}m 0 0 l 10 0{\k30}B` | 绘制对象因 `drawing_text.str` 强制开段（`ass_render.c:2017-2018`）；绘制字形有 `advance`（`ass_render.c:1225-1227` 走 `ASS_SHAPING_SIMPLE`/drawing 分支），因此可参与 `x_start/x_end`；其 `leftmost_x` 由包围盒变换得 | `ass_render.c:2017-2018`、`1225-1227`、`1404-1405` |

- **[事实] 关于“重叠”的一个特别点**：`ass_process_karaoke_effects()` 的 `timing` 更新是 `timing = !has_reset * tm_end + skip_timing`（`ass_parse.c:1067`）。当段内出现 `\kt`（`has_reset`）时，该段终点被**丢弃**，`timing` 被替换为 `skip_timing`（清空后重新累加的 `\kt` 值）。因此 `\kt` 之后的下一个 `\k` 从**`\kt` 参数**而不是“本词终点”开始——这可能让后一词与前一词的时间区间**重叠**（后词起点早于前词终点）。这是 VSFilter 兼容语义，但在静态推演时间表时必须逐段跟踪 `has_reset`，不能只看标记数值。
- **[事实] 关于“缺失”的另一个特别点**：段被丢弃（2.3）时，`combined_bitmaps` 的槽位被 `free` 后 `continue`（`ass_render.c:2571-2573`），但 `nb_bitmaps` 已自增（`ass_render.c:2537`）；`render_text()` 逐槽遍历，空槽因 `info->bm` 为 `NULL` 被三次循环各自的 `if (!info->bm…) continue` 跳过（`ass_render.c:827-830`、`837-840`、`852-855`）。**[推导]** 因此不会留下错误颜色的“幽灵矩形”，但 `text_info->n_bitmaps` 仍包含空槽，`render_text()` 的遍历成本按槽数计。

---

## 4. 高亮边界工具箱：把公式写清楚

设某词（段）在**重排前**的逻辑布局中：

- `S = start->pos.x`（**段首字形**的 x——即 `last_boundary` 指向的那个 `GlyphInfo`，可能是 `skip` 的）
- `p_first = first_visible->pos.x`（段内**首个非 skip** 根字形的 x）
- `p_last = last_visible->pos.x`（段内**末个非 skip** 根字形的 x）
- `w_last = last_visible->advance.x`（**单字形** advance，非 `cluster_advance`）

则时间轴里 `x_start = p_first`、`x_end = p_last + w_last`（`ass_parse.c:1086-1087`），而写进**每个**字形的偏移以 `S` 为基准：

```c
info->effect_timing = x - info->pos.x;   // ass_parse.c:1104
```

段合成时又只取**段首**这一个值（`ass_render.c:2504-2505`），故进入竖线计算的量是 `x - S`。

设该段实际生成位图后 `L = leftmost_x`（变换后控制点最小 x，`ass_outline.c:477-479`）。

`\k` / `\ko`（`ass_render.c:857-868`）：

```
color(段, t) = c[0]   if (x - S) > 0
               c[1]   otherwise
其中 tm_end := tm_start ⇒ x ∈ {-1e8, +1e8}（ass_parse.c:1071-1078）
```

`\kf`（`ass_render.c:2576-2578` 与 `render_glyph()`）：

```
for tm_start <= t < tm_end:
    dt = (t - tm_start) / (tm_end - tm_start)
    x_logical = x_start + round((x_end - x_start) * dt)
    brk = round(L + (x_logical - S) * par_scale_x)
    像素 x < brk → c[0]（已唱）；x >= brk → c[1]（未唱）
else:
    t < tm_start → 全部 c[1]
    t >= tm_end  → 全部 c[0]
```

- **[事实] 出现两个不同的“起点”**：扫光的**比例基准**是 `p_first`（`x_start`），而换算到屏幕的**偏移基准**是 `S`（段首字形，`ass_parse.c:1104`）。二者只在“段首字形恰好就是首个可见字形且无水平偏移”时相同。若段首是被 `trim_whitespace()` 剪掉的空白、或 `\hspacing`/`\fsp` 让 `pos.x` 与墨迹错开，`S ≠ p_first`，于是 `brk` 会额外平移 `(p_first - S) * par_scale_x`。这是一处独立于 bidi 的静态偏移来源。
- **LTR，无镜像**：`L ≈ S`（受量化影响，`POSITION_PRECISION=8`，`ass_render.c:45`；`pos` 经 `qr >> SUBPIXEL_ORDER` 取整，`ass_render.c:639-642`）⇒ `brk ≈ x_logical`，边界正确。
- **RTL 段**：`S` 是**逻辑**首字形的 x，在重排后被搬到**视觉最右**；而 `L` 是**视觉最左**。`(x_logical - S)` 仍是“逻辑上已走过的宽度”，与视觉位置无关 ⇒ `brk = 视觉左端 + 已走过宽度`。

  ```
  brk ≈ L + (x_logical - S)
  ```

  当 `dt=0` 时 `x_logical = x_start ≈ S` ⇒ `brk ≈ L`（视觉**左**端）；当 `dt=1` 时 `x_logical = x_end` ⇒ `brk ≈ L + (x_end − S)` ≈ 视觉**右**端。
  **[推导]** 所以 `brk` 随 `dt` 从 `L` 走到 `L+段宽`，即前沿沿**视觉 x 正方向**（左→右）推进。但 RTL 段的阅读顺序是视觉 x **负方向**（右→左）：阅读顺序的第一个字母 `א` 位于视觉最右，最后一个字母 `ג` 位于视觉最左。于是前沿每推进一格，点亮的都是**阅读顺序上更靠后**的字母——填色从词尾向词头倒退。若以阅读进度衡量，正确的前沿应为 `(1-dt)·W`（自视觉右端起算），当前实现给的是 `dt·W`（自视觉左端起算），二者关于段中心互为镜像。
- **`frz ∈ (90,270)`**：`dt` 被 `1-dt` 取代且颜色互换（`ass_parse.c:1089-1098`），于是 `brk` 反向，视觉扫光方向随之翻转一次。对 `\kf` 这是“修正旋转”而非“修正 bidi”。
- **`par_scale_x ≠ 1`**：`pos.x` 在设备化时已乘过 `par_scale_x`（`ass_render.c:2543`），`L` 也在该空间测量，故 `brk` 与段宽同步缩放，比例关系保持一致；此参数**不引入**新的错位，只是把量纲统一到设备像素。

---

## 5. 测试与证据小结

| 证据 | 位置 | 说明 |
|---|---|---|
| 标签解析与时长单位 | `ass_parse.c:837-868` | `\kt/\kf/\ko/\k`，`*10` 与 `dtoi32` 截断 |
| 时间轴切词与累加 | `ass_parse.c:1025-1107` | 逻辑顺序；`x` 只由首尾可见字形决定 |
| 段首判定（含 `\k0` 不断段） | `ass_render.c:2004-2047` | `info->effect_timing ||` 且注释明确忽略 `effect_skip_timing` |
| 折行/修剪空白设段首 | `ass_render.c:1657-1721` | 行首、`linebreak` 处 |
| 合成段与 `leftmost_x` | `ass_render.c:2447-2578` | 段内共享 `effect_timing`；`\kf` 在此换算竖线 |
| 竖线切分 | `ass_render.c:424-495`、`382-407` | `brk` 减 `bm->left`；左块 `c[0]`、右块 `c[1]` |
| `\k`/`\ko` 布尔染色 | `ass_render.c:857-868`、`842-844` | `\ko` 在 `effect_timing<=0` 连轮廓不画 |
| bidi 段落与段落基准方向 | `ass_shaper.c:1007-1030`、`1120-1143`、`1160-1167` | 默认按 run 切段；`Encoding=-1` 才 LTR 自动 |
| whole-text-layout 语义 | `ass.h:256-274`、`Changelog:144-146` | 开启后整事件统一 bidi/shaping |
| 镜像点检测 | `ass_outline.c:473-485` | 变换后控制点最小 x，clamp `OUTLINE_MAX` |
| 测试样例 | `compare/test/sub1.ass:12`、`sub2.ass:12` | 仅旋转/模糊，无 karaoke、无 RTL |
| 测试注册 | `compare/meson.build:16-19`、`fuzz/meson.build:22` | `art-compare`、`art-fuzz`；素材外置（`meson.yml:93-97`、`167`） |
| fuzz 只查越界/尺寸 | `fuzz/fuzz.c:136-156` | 不校验颜色分区；整图哈希默认关闭（`fuzz.c:54-55`） |
| karaoke 词条仅变异字典 | `fuzz/ass.dict:130-140` | `\kf \K \ko \kt \k`；非语义测试 |
| 单帧导出工具 | `test/test.c:220-250` | 按给定毫秒渲染一帧，无断言，CI 只做冒烟（`meson.yml:235`） |

### 缺失的证据（必须说明）

- **仓库内没有**任何 `\k`/`\kf`/`\ko`/`\kt` 的回归图像或 `.ass` 断言。`compare/test/` 只有 8 个文件（2 个 `.ass` + 3 个 `.png` + 2 个字体），已逐一确认。
- `git log` 只有 1 个提交（`21b5a54`，浅克隆），因此**无法**通过历史回溯定位引入镜像问题的具体改动；`Changelog:224` 与 `Changelog:246`（均在 0.15.0 条目内，该条目起于 `Changelog:204`）仅以“Multiple fixes for karaoke override tags”概括，不足以判定具体行为。
- 外部 `libass-tests` 仓库可能含相关用例，但本仓库内**不可见**，且按环境要求不做网络抓取。

---

## 6. 建议（基于静态分析的判定，不含实现）

1. **把“逻辑已走宽度”重新投影到视觉空间**，而不是直接加到视觉最左墨迹上。根因不是“两个字段取了不同字形”，而是 `ass_render.c:2577` 把**逻辑空间**的量 `(x_logical − S)` 与**视觉空间**的锚点 `leftmost_x` 直接相加。可行方向有两种：(a) 在重排之后按 `cmap` 重新计算 `x_logical` 的视觉位置；(b) 把 `leftmost_x` 换成 `S` 所对应字形在重排后的 `pos.x`（RTL 时即视觉最右），再配合方向判断决定哪一侧为“已唱”。无论哪种，都必须让**锚点与偏移属于同一空间、同一字形**。
2. **`x_end` 应用 `cluster_advance` 而非 `advance`**（`ass_parse.c:1087`），否则多字形簇（连字/阿拉伯文 lam-alef）扫光会系统性偏窄；`cluster_advance` 已在 `retrieve_glyphs()` 里补齐 hspacing 与斜体追加（`ass_render.c:2193-2205`）。同时应注意 `effect_timing` 的偏移基准是**段首字形 `S`**而非 `x_start`，若段首是被裁掉的空白，二者会差一个空隙（见 §4）。
3. **`\k`/`\ko` 的整段布尔判断在 whole-text-layout 下应重新审视**：默认模式下每个词自成一个 bidi 段落（`ass_shaper.c:1009-1010`），`c[0]/c[1]` 的“整段”语义与段落边界一致；但开启该特性后 `cmap` 会跨词重排（`ass_shaper.c:1130-1131` 的 `!whole_text_layout` 条件失效），而 `effect_timing` 的符号仍按逻辑索引算出，**词序与时间序可能不一致**（同一逻辑词在视觉上可能落在相邻词的左侧或右侧）。需要专门验证。
4. **补一条回归用例**：`{\kf100}אבג` 与 `{\k100}אבג` 在 `t = 250/500/750` 的三张期望图，外加 `{\k30}A{\k0}B{\k40}C` 的时间表断言。素材应放进本仓库可复现的位置（如 `compare/test/`），否则 `art-compare` 对这两类问题的覆盖仍然是零。
5. **明确 `\kf` 在 `frz` 镜像与 `scale_x < 0` 下的投影规则**：当前只有 `frz∈(90,270)` 触发 `dt → 1-dt` 与颜色互换（`ass_parse.c:1089-1098`），没有任何对 `x_start/x_end` 的镜像投影；而 `leftmost_x` 已是变换后空间（`ass_outline.c:478-479`），所以两者在镜像变换下必然不一致。

---

## 7. 一句话回答题述

**事件时间按“逻辑顺序上的段”串行累加（`ass_parse.c:1065-1067`），而高亮边界是“合成位图上的一条视觉竖线”（`ass_render.c:2576-2578`）。`\kf` 的竖线以视觉最左墨迹 `leftmost_x` 为锚点、加上重排前冻结的“逻辑已走宽度”；在 RTL 段里这两者分处段落两端，于是前沿沿视觉左→右推进，与阅读方向（右→左）相反——填色从词尾开始倒退，等价于把进度镜像成 `1-dt`。`\k`/`\ko` 因为是“`tm_start` 一到即整段变色”的布尔开关（`ass_parse.c:1071-1072`、`ass_render.c:857-868`），不受此影响。分段缺失/重叠不会崩溃，但会通过 `timing` 累加器静默改变后续词的时间（`\kt` 更会把基准整体重置）。**
