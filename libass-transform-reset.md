# libass 变换动画（`\t`）遇到样式重置（`\r`）时的时间优先级报告

> 静态分析报告。基于本仓库当前源码（分支 `task/20`，HEAD `21b5a54`），未运行播放器。
> 引用格式为 `文件:行号`。文中"当前行为"均有源码或测试证据；"建议"为作者推断，需另行验证。

## 0. 结论摘要

1. **解析层面**：每个事件、每一帧都从零开始重建 `RenderContext`（基础样式 → 行内覆盖 → `\r` 重置 → `\t` 按当前时刻插值），优先级是**纯顺序的"后写覆盖"**，`\t` 只是给其后标签乘上一个时间系数 `k`，并不构成独立的优先级层。
2. **`\r` 不参与插值**：即使 `\r` 写在 `\t(...)` 内部，它也是**瞬时全量重置**，忽略 `\t` 的 `pwr` 系数（`ass_parse.c:805-811` 调用 `ass_reset_render_context` 时完全不看 `pwr`）。
3. **缓存层面**：时间变化的变换（旋转/缩放/剪切）只进入**位图缓存键**（量化后的变换矩阵），轮廓缓存键不含变换；透明度/颜色动画**不进任何缓存键**，在合成时才着色，是"免费"的动画。
4. **事件边界**：`\t`、`\move`、`\fad` 的计时全部相对于本事件的 `Start`，跨事件无任何状态继承；动画在事件结束处被钳制到终值。

---

## 1. 各层样式来源在解析结果与渲染缓存中的优先级

### 1.1 解析结果（RenderContext → GlyphInfo）中的优先级

渲染入口 `ass_render_frame`（`ass_render.c:3370`）对当前时刻激活的每个事件调用 `ass_render_event`（`ass_render.c:2799`），后者**每帧**执行：

```
free_render_context → init_render_context → parse_events → split_style_runs → ...
```

（`ass_render.c:2812-2825`）。即**不存在跨帧的解析状态缓存**，所有优先级都在这一次顺序扫描中决定。优先级从低到高：

| 层级 | 来源 | 代码位置 | 说明 |
|---|---|---|---|
| L0 | 轨道默认/用户选择性覆盖 | `handle_selective_style_overrides`（`ass_render.c:906-945`） | 若用户通过 `ass_set_selective_style_override()` 设置了覆盖样式，则先与脚本样式混合；含 `\pos`/`\move`/`\clip`/`\org`/`\p` 的"显式定位"事件（`ass_event_has_hard_overrides`，`ass_parse.c:1156-1181`）会禁用大部分用户覆盖（`ass_render.c:933-934`） |
| L1 | 事件声明的基础样式 | `init_render_context` → `ass_reset_render_context(state, NULL)`（`ass_render.c:1146-1151`） | `NULL` 时回落到事件自己的样式（`ass_render.c:921-922`） |
| L2 | 行内覆盖标签 | `ass_parse_tags`（`ass_parse.c:282`） | 顺序扫描，绝大多数标签**后写覆盖先写** |
| L3 | `\r` 重置标签 | `ass_parse.c:805-811` → `ass_reset_render_context`（`ass_render.c:1075-1111`） | 把"样式派生子集"瞬时重置为指定样式（或事件样式）；**只重置样式字段**（颜色、字号、字体、边框、ScaleX/Y、Angle、阴影、模糊等），**不重置** `\pos`/`\move`/`\clip`/`\org`/`\fad`/对齐（`alignment` 只在 `init_render_context` 里从样式取，`ass_render.c:1151`） |
| L4 | 随时间变化的 `\t` | `ass_parse.c:670-728` | 计算 `k = ((t-t1)/(t2-t1))^accel`（`t = render_priv->time - event->Start`，`ass_parse.c:698-705`），把 `k` 作为 `pwr` 传给其参数列表中的标签；每个标签自行做 `current*(1-pwr) + target*pwr` 插值（如 `\fscx`、`\frz`、`\alpha`、`\bord` 等） |

**关键细节：**

- `\t` 不是"第 5 层"，而是**修改 L2 标签生效强度的系数**。`{\fscx150\t(0,1000,\fscx200)}` 在 t=500 时等价于 `150*(1-0.5)+200*0.5 = 175`。
- **一次性标签**（不受后写覆盖规则约束）：`\pos`（`ass_parse.c:606-621`，首个生效，重复时仅告警）、`\move`（`ass_parse.c:525-530`，`EVENT_POSITIONED` 置位后忽略后续）、`\org`（`ass_parse.c:664-669`，`have_origin` 守卫）、`\fad`/`\fade`（`ass_parse.c:650-656`，`PARSED_FADE` 守卫）、`\a`/`\an`（`ass_parse.c:584-605`，`PARSED_A` 守卫）。这些标签**忽略 `pwr`**，`\t` 对它们无效。
- **忽略 `pwr` 的其他标签**：`\b`/`\i`（`ass_parse.c:825-836`，直接赋值）、`\fn`（`ass_parse.c:556-566`）、`\r`（见上）、`\k`/`\kf`/`\ko`/`\kt`（`ass_parse.c:837-868`）。
- 每个字符的解析结果在 `parse_events` 中快照进 `GlyphInfo`（`ass_render.c:2107` 起，颜色在 `ass_render.c:2123`，变换在 `ass_render.c:2136-2149`），`split_style_runs`（`ass_render.c:2005-2047`）按这些字段是否变化切分样式 run——**这就是"解析结果优先级"的最终物化**。

### 1.2 渲染缓存中的优先级

三级缓存的键结构定义于 `ass_cache_template.h`：

| 缓存 | 键结构 | 键包含 | 键**不**包含 | 对动画的含义 |
|---|---|---|---|---|
| 轮廓缓存 | `GlyphHashKey`（`ass_cache_template.h:88-96`） | font、size、face、glyph 索引、bold、italic、flags | scale、旋转、剪切、颜色、透明度 | `\t(\fscx…)`/`\t(\frz…)` **复用**同一轮廓；`\t(\b…)`/`\t(\fs…)` 才会产生新轮廓条目 |
| 位图缓存 | `BitmapHashKey`（`ass_cache_template.h:62-69`） | 轮廓指针 + **量化后的变换矩阵** `matrix_x/y/z` 与 `offset` | 颜色、透明度 | 旋转/缩放/剪切动画每跨过一个量化步长就新增一条位图条目 |
| 合成缓存 | `CompositeHashKey`（`ass_cache.h:84-88`） | `FilterDesc`（be、blur、shadow、flags，`ass_cache_template.h:117-123`）+ 位图引用与位置 | 颜色、透明度 | `\be`/`\blur`/`\shad` 动画进入此键；透明度动画不进 |

量化在 `quantize_transform`（`ass_render.c:498-649`）：位置精度 `POSITION_PRECISION = 8.0`（1/64 像素单位，`ass_render.c:45`），即 **1/8 像素**；亚像素阶 `SUBPIXEL_ORDER = 3`（`ass_render.c:47`）。位图缓存上限 `BITMAP_CACHE_MAX_SIZE = 128MB`（`ass_render.h:45`），每帧 `check_cache_limits` 裁剪（`ass_render.c:3021`，调用点 `ass_render.c:3095`）。

**颜色/透明度在哪里生效**：`CombinedBitmapInfo.c[4]` 注释明确写着 "colors (with fade applied)"（`ass_render.h:101`），在 `render_glyph(..., info->c[0], ...)` 调用链中于**绘制时**着色（`ass_render.c:833-876`），`\fad` 的淡出值在 `ass_apply_fade` 中乘入（`ass_render.c:2502` 附近）。因此：

> **缓存优先级结论**：透明度动画（`\t(\alpha…)`、`\fad`）< 颜色动画（`\t(\c…)`）≈ 免费；旋转/缩放/剪切动画 = 每个量化步长一条位图条目；字号/字重动画 = 每个值一条轮廓条目（最贵）。`\r` 本身不进任何缓存键——它通过改变后续 GlyphInfo 字段间接改变键内容。

---

## 2. 动画交叠与中途重置时的位置 / 缩放 / 透明度结果

### 2.1 两个 `\t` 块顺序交叠（同一属性）

例：`{\t(0,1000,\fscx200)\t(500,1500,\fscx100)}`，t=750：

- 第一个 `\t`：`k=0.75`，scale_x = `100*0.25 + 200*0.75 = 175`；
- 第二个 `\t`：`k=0.5`，从**当前状态 175** 插值：`175*0.5 + 100*0.5 = 137.5`。

**当前行为**：顺序复合，后一个 `\t` 的权重 `k→1` 时完全主导。这不是"两条曲线叠加"，而是"第二条曲线以第一条的瞬时值为起点"。

### 2.2 嵌套 `\t`（内层系数**替换**而非相乘）

`ass_parse.c:707-708`：

```c
if (nested)
    pwr = k;
```

内层 `\t` 算出自己的 `k` 后**直接覆盖**外层传入的 `pwr`（不是 `pwr *= k`）。`{\t(0,1000,\fscx200\t(0,500,\fscy300))}` 中，`\fscy` 之后的标签将使用内层 `k` 而丢失外层插值系数。尾调用优化分支（`ass_parse.c:719-728`）同理。**建议**：脚本作者避免嵌套 `\t`；若需复合，拆成顺序块。库侧若要修正为相乘语义，需注意 VSFilter 兼容性。

### 2.3 中途重置（`\r` 在动画进行中出现）

例：`{\t(0,1000,\fscx200)A\r B}`，t=500：

- `A`：scale_x = 150（插值中）；
- `B`：`\r` 瞬时重置为样式 ScaleX=100 —— **硬切，无过渡**。

例：`{\t(0,1000,\r\fscx200)}`（`\r` 在 `\t` 内部）：

- `\r` 忽略 `k`，每帧都先把样式字段重置到样式值（`ass_parse.c:805-811` 无 `pwr` 分支）；
- 随后 `\fscx200` 以 `k` 从**刚重置的样式值**插值。

**当前行为**：`\t(\r…)` 实际等价于"从样式值出发的确定性动画"，这反而是可利用的惯用法；但 `\t(\fscx200\r)`（`\r` 在列表末尾）会把前面的插值结果全部抹掉，动画完全失效。**建议**：文档化"`\r` 在 `\t` 内不插值"这一事实（与 VSFilter 行为一致，改动有兼容风险）；脚本作者应把 `\r` 放在 `\t` 参数列表的**开头**。

### 2.4 位置结果（`\move` / `\pos` 与 `\t` 不交叠）

- `\move` 自行用自身 `t1/t2` 计算线性 `k`（`ass_parse.c:515-524`），**完全忽略 `pwr`**；`\t(\move…)` 不会加速/减速/延迟移动。
- `\pos` 直接赋值且首个生效（`ass_parse.c:606-621`）。
- 因此**位置动画不存在交叠问题**：同一事件里只有第一个 `\move`/`\pos` 生效；`\r` 也**不能**重置位置（`ass_reset_render_context` 不触碰 `pos_x/pos_y`，`ass_render.c:1075-1111`）。若脚本需要"动画中途改位置"，当前唯一手段是拆成多个事件。

### 2.5 透明度结果

- `\t(\alpha&HXX&)` 经 `change_alpha`（`ass_parse.c:155`）按 `pwr` 插值，四通道同时（`\alpha`）或单通道（`\1a`–`\4a`）。
- `\fad`/`\fade` 只解析一次（`PARSED_FADE`，`ass_parse.c:650-656`），其结果 `state->fade` 在合成前由 `ass_apply_fade` 乘入颜色（`ass_render.c:2502` 附近），与 `\t(\alpha)` **乘法叠加**在同一 alpha 通道上。
- `\r` 会把 `state->c[0..3]` 重置为样式颜色（`ass_render.c:1081-1084`），**包括 alpha 分量**——即中途 `\r` 会硬切透明度；但 `\r` **不影响** `state->fade`（`fade` 在 `init_render_context` 置 0，`ass_render.c:1138`，`\r` 不动它），所以 `\fad` 淡入淡出不受中途 `\r` 干扰。
- 全透明位图在拼合时被丢弃（`ass_render.c:3416-3419`）。

### 2.6 事件边界

- 每个事件的渲染上下文独立重建（`ass_render.c:2812-2813`），`\t`/`\move`/`\fad` 的 `t` 均为 `render_priv->time - event->Start`（`ass_parse.c:698`、`ass_parse.c:516`、`ass_parse.c:652`）；`\t` 缺省 `t2=0` 时取本事件 `Duration`（`ass_parse.c:695-696`）。
- `t >= t2` 时 `k=1` 钳制到终值（`ass_parse.c:701-702`）；事件结束后不再渲染（`ass_render.c:3384-3385` 的激活窗口判断）。
- 卡拉 OK 计时同样在事件内累计，`\kt` 置 `reset_effect` 把累计计时清零（`ass_parse.c:837-844`、`ass_parse.c:1038-1069`）——这是事件内唯一的"计时重置"机制，且与 `\r` 无关。
- **当前行为**：跨事件的"连续动画"必须在每个事件里重写完整 `\t` 并手动对齐时间轴；两个时间上重叠的事件各自独立插值，互不影响。**建议**：无（此语义与 VSFilter 一致，属格式固有约束）。

---

## 3. 测试证据

1. **`compare/test/sub1.ass`**：事件 `Dialogue: 0,0:00:00.00,0:00:04.00,...,{\t(\frz+180)}A`，配套 `sub1-0500.png`、`sub1-1500.png`、`sub1-2500.png` 三张基准图。这是 `\t` 时间插值的回归测试：同一事件在 500/1500/2500 ms 三处采样，验证 `k` 随 `render_priv->time - event->Start` 单调推进且旋转角度量化稳定（对应 §1.2 的位图缓存量化）。测试框架见 `compare/README.md`：以 `ASS_FONTPROVIDER_NONE` 隔离环境，按 `<sub>-NNNN.png` 的毫秒时刻渲染比对，pass-level 0 为逐位相同。
2. **`compare/test/sub2.ass`**：`{\blur1}Test_` 在 153000 ms 采样，覆盖 `FilterDesc` 进入合成缓存键的路径（§1.2 第三行）。
3. **`test/test.c`**：崩溃回归驱动器，`main`（`test.c:220`）在 `test.c:253` 以 `ass_render_frame(ass_renderer, track, tm*1000, NULL)` 逐时间点渲染，验证解析-缓存流水线在任意时刻采样下的健壮性。
4. **现有覆盖缺口（建议补充）**：仓库内**没有**针对 `\r` 与 `\t` 组合的回归用例。建议在 `compare/test/` 增加：
   - `{\t(0,2000,\fscx200)A\r B}` 在 t=1000 采样 → 断言 `B` 为样式缩放（硬切）；
   - `{\t(0,2000,\r\fscx200)}A` 在 t=1000 采样 → 断言 scale=150；
   - 嵌套 `\t` 用例 → 固化 §2.2 的"替换"语义，防止无意变更。

---

## 4. 当前行为 vs 建议 一览

| 场景 | 当前行为（有源码证据） | 建议 |
|---|---|---|
| `\r` 在 `\t` 内 | 瞬时重置，忽略 `pwr`（`ass_parse.c:805-811`） | 保持行为，文档化；脚本侧把 `\r` 放 `\t` 列表开头 |
| 顺序交叠 `\t` | 后者以前者瞬时值为起点插值（`ass_parse.c:282` 顺序扫描 + 各标签 `current*(1-pwr)+target*pwr`） | 保持；文档化"非叠加"语义 |
| 嵌套 `\t` | 内层 `k` **替换**外层 `pwr`（`ass_parse.c:707-708`） | 文档化；如改为相乘需评估 VSFilter 兼容性 |
| `\move`/`\pos` 与 `\t` | 忽略 `pwr`，首个生效（`ass_parse.c:515-530`、`606-621`） | 保持；位置动画不可经 `\t` 调制 |
| 透明度动画 | 不进缓存键，绘制时着色（`ass_render.c:833-876`） | 性能敏感脚本优先用 `\alpha`/`\fad` 而非 `\fscx`/`\frz` 动画 |
| 旋转/缩放动画 | 每 1/8px 量化步长一条位图缓存条目（`ass_render.c:498-649`） | 长动画注意 128MB 位图缓存上限（`ass_render.h:45`） |
| 事件边界 | 上下文每事件重建，计时相对本事件 `Start`（`ass_render.c:2812-2813`、`ass_parse.c:698`） | 跨事件连续动画需逐事件重写 `\t` 并对齐时间 |
