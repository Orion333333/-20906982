# AIM 2627 Python Coursework —— 哨兵 Sentry 控制模块

> **全部题目、规范、评分、提交见 [题面.pdf](题面.pdf)。** 本 README 只讲怎么把环境跑起来；没在这里出现的规格细节，一律以题面为准。

## 1. 环境要求

- Python 3.8+，仅标准库（不允许第三方运行时依赖）；
- 开发工具只需 `pytest`（测试）与 `autopep8`（风格，CI 会检查）；
- VS Code 打开仓库会推荐安装 `ms-python.autopep8` 插件（`.vscode/extensions.json`），保存即格式化即可过风格检查。

## 2. 快速开始

```bash
# 1. 用 GitHub 的 Use this template 创建你自己的仓库，然后 clone
git clone https://github.com/<你的用户名>/<你的仓库>.git
cd <你的仓库>   # 直接在 main 分支上开发

# 创建虚拟环境

python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

# 2. 装依赖
python -m pip install pytest autopep8

# 3. 启用 AI 会话归档钩子（课程要求，见下方第 3 节）
python -m pip install 'agent-session-commit[pre-commit]==0.1.3' -i https://pypi.org/simple
agent-session-commit install --pre-commit   # 交互选择你的 AI 助手与会话目录

# 4. 跑测试（刚到手：全部 skip，CI 是绿的）
python -m pytest

# 5. 看演示
python main.py

# 6. 打开 题面.pdf 读题，开始实现 src/main/__init__.py 里的 TODO
```

## 3. AI 会话归档（pre-commit）

本课程允许使用 AI，提交的 commit 需要携带 AI 会话归档作为透明化记录：每次 `git commit` 后，钩子会把新增会话自动 amend 进同一个提交（`.agent-sessions/bundles/`），不产生额外的归档提交。支持 Claude Code、OpenAI Codex CLI、GitHub Copilot CLI、Qoder、ZCode、Trae、Tencent CodeBuddy 等（完整名单见 [AgentLedger](https://github.com/Gentle-Lijie/AgentLedger)）。

- 配置是仓库本地的：每个 clone 运行一次 `agent-session-commit install --pre-commit`，方向键选择 agent、确认其会话目录即可；
- 不想用 TUI 可手动配置：`git config --local agent-session.agent claude`、`git config --local agent-session.source "<会话目录>"`，然后 `python -m pip install 'pre-commit>=3.2.0' && pre-commit install`；
- 归档是普通 Git 内容且会推送到公开仓库——不要在 AI 会话里粘贴令牌等敏感信息；
- 换了 agent 或目录就重跑一次安装命令；卸载：从 `.pre-commit-config.yaml` 移除该条目后重跑 `pre-commit install`。

## 4. 本地开发循环

- **写代码**：全部作业在 `src/main/__init__.py`，按题面各题规范补全每个标有 TODO 的函数；注释里标注了对应的题面主题，推荐顺序 Q1 → Q6。
- **跑测试**：`python -m pytest` —— 可见测试是规格书的一部分，未实现的函数自动 skip，实现一个、对应测试亮一个。本地全绿 ≠ 满分（见题面）。
- **看演示**：`python main.py`（等价于 `PYTHONPATH=src python -m main`），随实现进度逐段点亮，不进测试。
- **Q6 自测**：`python tools/run_seeds.py --q6`（200 张固定地图统计），单 seed 渲染 `python tools/run_seeds.py --q6 --seed <N> --render`，Bonus 模式 `python tools/run_seeds.py --bonus`。

## 5. 仓库结构（哪些能改）

| 路径 | 说明 | 能否修改 |
|---|---|---|
| `src/main/__init__.py` | 你的全部作业（TODO 所在） | ✅ |
| `README.md` | 仅末尾两个"你来写"小节 | ✅ |
| `题面.pdf` | 题面（唯一规格说明） | ❌ 勿改 |
| `src/main/legacy_patrol.py` | Q7 模块（与主体同步发布，修复其缺陷） | Q7 时 ✅ |
| `.pre-commit-config.yaml` | AI 会话归档钩子配置 | ❌ 勿改 |
| `src/tests/`、`tools/`、`.github/`、`conftest.py`、`pytest.ini`、`main.py` | 测试与基础设施 | ❌ 勿改 |

CI 只允许修改 `src/main/**`、`README.md` 与 `.agent-sessions/**`（AI 会话归档）——其余文件改了直接红；autopep8 `--diff` 非空即败。提交方式（push、问卷、commit 粒度）见题面"提交与验收"一节。

Q1使用分支结构与定义函数，自动确定电量与血量状态
Q2  1.初始化。
    2.遍历，除去脏行，去除字间空格，用json和传感器产出数据的特征，分流数据至json和传感器。
    3.使用try移除异常数据且不报错，程序持续运行并减少日志阅读量。
    4.程序先遍历日志，用“，”为标志将大段文字分段。随后用partition将日志数据切分，防止爆出三行数据列。最后去除各类脏数据并将伤害转为整数后传向 统计处
    5.统计各分支传来的日志。

    AI会话归档

## 设计说明（Q1–Q5）

模块按"自检 → 日志统计 → 网格运动 → 贪心寻敌 → 规则决策"逐层搭建，
每层的职责与边界如下（Q6 见下节）。

### Q1 自检与报告
- `hp_ratio`：按 `hp / max_hp` 计算 0–100 的整数百分比（四舍五入后钳制；
  `max_hp <= 0` 返回 0）；
- `status_report`：用 `if/else` 按电量读数分出 OK / WARNING / LOW 三档
  （分界以题面与可见测试为准），并按固定宽度拼出逐字符格式的报告行。

### Q2 战斗日志分析
- 支持两种合法行：传感器行（如 `F:32,L:5,R:12`）与 JSON 行
  （`{"armor": ..., "damage": ...}`）；其余一律视为脏行、跳过而不抛异常；
- 传感器行先校验全部字段再计入统计；任一段非法时整行丢弃。
  数字形态检查通过但整数转换失败的输入也会跳过，不保留同一脏行的部分伤害；
- armor 必须是 front/left/right 之一，damage 必须是正整数；
  带 id 的 JSON 行"先校验、后去重"，同一 id 只计第一次；
- 输出固定契约：`total`、`by_armor`、`most_hit`（并列时按 front→left→right）、
  `avg`（保留两位四舍五入；空日志为 0.0）。

### Q3 `SentryGrid` 方法
- `current_pos` setter：类型与长度校验（非法抛 `TypeError`），
  存入前统一转 int 并钳制回地图范围；
- `move_forward`：无电原地返回；目标被阻挡（含越界）时碰撞计数 +1，
  位置与朝向不变、不耗电；可通行时移动并扣 1 电，永远返回执行后的位置；
- `turn_left` / `turn_right`：四方向显式映射旋转，原地、不耗电。

### Q4 贪心寻敌
- 候选 = 四邻域中"非障碍且曼哈顿距离严格减小"的方向；
- 两轴都有候选时取坐标差较大的轴；平局（`|dx| == |dy|`）取 x 轴；
- 无候选（含已在终点）时返回 `current_facing`；不感知地图边界，
  越界交给 `move_forward` 处理。

### Q5 规则决策
- 先做契约校验：缺失字段 / frames 为空或超过 6 / state 非法 → `ValueError`；
- 再做防御式规范化：取末帧真值、距离归一化、机型限定、血量百分比 0–100；
- 无效血量导致类型、取值或溢出异常时，按既有安全规则归为 0% 血量；
- R1–R7 按固定顺序首条命中即返回；题面未钉死的取值采用保守选择并写进注释：
  R2 恢复线 50；R5"持续丢失"按连续两帧判定；距离未知按"远"处理（不开火）；
  非法机型按 INFANTRY；`heat` 参数不参与 R1–R7 判定。

## Q6 设计说明与自测

巡逻优先使用 Q4 的贪心方向；没有能严格缩短曼哈顿距离的可通行邻格时，
切换到沿墙模式。左手沿墙超过地图宽度与高度之和后改用右手；右手尝试
超过两倍该上限后重新评估贪心方向。具备贪心候选且当前距离不大于脱困
入口距离时恢复贪心。沿墙仍可能重复走过同一路线，因此任务还受步数和
当前电量限制，失败地图会正常终止。

`steps` 记录每轮对齐朝向后的前进尝试；同轮转向不额外计步。
`visited_count` 包含起点，按不同坐标去重。成功以到达敌方坐标为准，
`found_enemy` 与 `success` 相同。JSON 使用排序键和紧凑分隔符，
不依赖字典的插入顺序，也不修改输入统计字典。

200 张固定地图自测：成功 188 张（94.0%），平均碰撞 0.00，
成功案例的平均步数/BFS 比为 1.2094，三条阈值均通过。
抽查失败 seed 23、31、133，均存在重复访问，在 500 步处退出，
剩余电量为 500，碰撞为 0。另验证了已在终点、零步数、零电量、
低电量、短步数上限、恰好抵达和起点被围住七种边界，
以及 120 种键插入顺序下的 JSON 确定性。

## Q7 缺陷修复说明

定位方法：以各函数 docstring 为契约逐条对照，用可见测试与最小复现输入验证；
两组遮蔽关系通过"仅修一处再运行"的内存实验确认。

### BUG-01 路线单位换算缺失（`total_route_meters`）
- 失败输入：`total_route_meters([(0, 0), (3, 0), (3, 4)])` 返回 `700`，应为 `7`。
- 根因：`segment_length_cm` 返回厘米，汇总变量却按米命名并原样返回。
- 修复：汇总改名为 `distance_cm`，返回前 `/ 100`（保留浮点，不截断）。

### BUG-02 校准遗漏无正样本分支（`calibrate`）
- 失败输入：`calibrate([-1, -2])` 抛 `TypeError`（对 `None` 做减法）。
- 根因：`first_positive` 无正样本时返回 `None`，`calibrate` 未处理该分支；
  空列表碰巧绕过累加循环，单独检查空列表无法暴露问题；负样本测试能复现。
- 修复：`baseline is None` 时直接返回 0；`first_positive` 的 `None` 契约保持不变。

### BUG-03 事件上界不包含等号（`summarize_events`）
- 失败输入：两条事件（id 1、2）在 `max_id=2` 时只统计到 1 条，应为 2 条。
- 根因：筛选条件写成 `< max_id`，把"不超过"丢掉了等号。
- 修复：`<` 改为 `<=`。

### BUG-04 默认参数跨调用共享（`log`）
- 失败输入：连续两次 `log("a")`、`log("b")` 返回同一个列表（`a is b` 为真）。
- 根因：可变默认参数 `history=[]` 只在函数定义时创建一次、跨调用复用。
- 修复：默认值改为 `None`，仅当 `history is None` 时新建列表。

### BUG-05 模拟终止条件写反（`run_legacy_sim`）
- 失败输入：`run_legacy_sim(10, 100)` 第 1 轮后即返回（体力仍为 92）。
- 根因：退出比较写反成 `stamina > 20`——正常体力刚扣完一轮就退出，
  而低体力时永远不退出（与 BUG-06 叠加成死循环）。
- 修复：改为 `stamina <= 20` 时退出，判断位置保留在本轮扣除与 trace 记录之后。

### BUG-06 轮号不递增（`run_legacy_sim`）
- 失败输入：`run_legacy_sim(2, 28)` 无限循环、trace 无限增长且轮号恒为 0。
- 根因：循环体从未递增 `round_`，轮数上限与"第 4 轮起额外 -5"都无法生效。
- 修复：改用 `for round_ in range(rounds)`，轮号依次取 0..rounds-1。

### 两组互相遮蔽关系
1. BUG-03 遮住 BUG-02：上界事件被排除后，其全负样本没有机会让 `calibrate`
   对 `None` 做减法；必须同时满足筛选与校准契约。
2. BUG-05 遮住 BUG-06：正常体力下第一轮就提前返回，看不出轮号不增长；
   先修退出比较才会暴露轮数上限失效。

### 额外发现：`parse_event` 的整数转换（不属于六处缺陷）
- 失败输入：`parse_event("MOVE,²")`——上标数字 `isdigit()` 为真但 `int()`
  抛 `ValueError`；超长数字串（超过 `sys.get_int_max_str_digits()`）同理。
- 修复：把 `int()` 转换包进小范围 `try/except ValueError`，失败返回 `None`，
  保持"脏行不抛异常"的契约。

### 修复后验证（实际运行结果）
- Q7 可见测试 7/7 全过（含两条模拟测试，均有限步内终止）；
- 补充边界全部符合契约：`[]`/`[-1, -2]`/`[0, 0]` 返回 0，`[2, 3, 5]` 返回 4，
  `[0, -1, 2, 3]` 返回 -4；`run_legacy_sim` 五组预期值逐一相等；
  上标数字与超长数字均返回 `None`；
- `autopep8 --diff` 为空；`git diff --check` 无空白问题；
- 全仓库 pytest 另有既有 Q1 档位用例失败；完整 `src/` 的 autopep8 检查
  另有受保护测试文件的格式差异，与本次 Q7 修复无关。

## 提交前复核（2026-10-08）

- 可见测试：33 通过、1 失败、1 跳过。失败项为 Q1 电量档位；
  按当前要求保留 Q1 代码，待处理。跳过项为尚未实现的可选 Bonus BFS。
- Q6 固定地图：94.0% 成功率、0.00 平均碰撞、1.21 平均步数/BFS 比，全部达标。
- 60 项补充检查通过，覆盖 Q2 脏行与去重、Q5 异常血量、Q7 边界及 Q1 未改动核对。
- `src/main/` 格式检查通过，累计修改文件符合白名单；完整 `src/` 格式检查
  仍因初始模板 `src/tests/test_main.py` 的三处格式差异失败，该文件保持原样。
