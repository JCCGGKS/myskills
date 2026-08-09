# linux-dev-env Skill 评估报告

> 评估目标：触达准确性（trigger accuracy）+ 任务完成度（task completion）
> 评估方法：3 个真实用户场景 ×（加载 skill / 不加载 skill）对照，共 6 次运行
> 原始输出：`eval-workspace/iteration-1/eval-{0,1,2}/{with_skill,without_skill}/outputs/response.md`
> 评分明细：各 `eval-*/{with_skill,without_skill}/grading.json`

---

## 一、任务完成度：优秀

两组（with_skill / baseline）都能给出正确、可执行的指导。with_skill 的额外价值在于**一致性**与**不漏手册修正点**：

- 严格遵循手册已修正的坑（SSH `-t ed25519`、nvm `alias default`、Go gvm 引导+版本校验、GOPROXY 切勿设 `off`、Yarn `signed-by`、MySQL `auth_socket`）
- 必有「先展示命令、确认后再执行」的纪律
- 结尾必带「整体验证清单」

具体差异（最有说服力的一处）：**eval-0** 中，with_skill 明确警告「内网不要把 `GOPROXY` 设成 `off`，否则完全无法下载模块」（这正是手册修正过的点），而 baseline 没有提这条。其余坑点两者基本都覆盖。

结论：内容质量高，with_skill 在"稳定不漏手册修正点 + 结构统一"上更可靠。

## 二、触达准确性：当前不达标（关键问题）

这是最该修的地方：

- **目录位置问题**：skill 当前位于 `myskills/linux-dev-env/`，**不在** Claude Code 的技能发现路径
  （项目级 `.claude/skills/` 或全局 `~/.codebuddy/skills/`）。
- **后果**：Claude Code 不会自动加载它 → **实际触发率 ≈ 0**，无论 `description` 写得多么"主动"。
- **本质**：本次 with_skill 之所以"生效"，是因为评估时我手动把 skill 路径注入给了子代理；
  在真实使用里，若 skill 未被发现，用户根本拿不到这份受手册约束的引导，只能靠模型自由发挥。
- **说明**：baseline 输出也不错，说明模型本身能力够，但 skill 的意义是"稳定、不漏修正点、统一风格"——
  而这一目标目前因"无法被触发"而完全落空。

## 三、评分汇总

| 运行 | 范围恰当 | 坑点标注 | 不自动执行 | 验证清单 | 结论 |
|------|:---:|:---:|:---:|:---:|------|
| eval-0 with_skill | ✅ | ✅（含 GOPROXY-off） | ✅ | ✅ | 优 |
| eval-0 baseline | ✅ | ✅（缺 GOPROXY-off） | ✅ | ✅ | 良 |
| eval-1 with_skill | ✅ | ✅（auth_socket） | ✅ | ✅ | 优 |
| eval-1 baseline | ✅ | ✅（auth_socket） | ✅ | ✅ | 良 |
| eval-2 with_skill | ✅ | ✅（nvm alias） | ✅ | ✅ | 优 |
| eval-2 baseline | ✅ | ✅（nvm alias） | ✅ | ✅ | 良 |

> eval-1 的 auth_socket 属于"常识"，两组都能答对，故不是 skill 的差异化点；
> 真正体现 skill 价值的差异化点是 **GOPROXY=off 警告** 与 **统一的"先确认后执行 + 验证清单"纪律**。

## 四、建议（按优先级）

1. **修复触达（最高优先级）**：把 skill 移入 `.claude/skills/linux-dev-env/`（项目级自动触发），
   或 `~/.codebuddy/skills/linux-dev-env/`（全局）。否则它永远无法自动触发。
   - 注意：移入 `.claude/skills/` 后该目录默认不被 git 跟踪；可用 `git mv` + `ln -s` 之类保持可版本化，或接受它作为本地配置。
2. **触发优化**：位置修好后，可跑 `run_loop` 对 `description` 做触发率评估（需 `claude -p`），进一步提升准确性。
3. **description 微调**：现有 description 已较"主动"，可保留；建议补一句"也适用于检查/补齐已有环境"，覆盖 eval-2 这类场景。
4. **保持一致**：`references/handbook.md` 与源手册已同步，后续改动时记得双向同步（或用脚本）。

## 五、结论

- **任务完成度：达标且优秀**——加载 skill 后输出准确、一致、守纪律。
- **触达准确性：当前不合格**——根因是 skill 不在发现路径，导致"写得好但用不上"。
- **下一步**：先把 skill 放到 `.claude/skills/`（或全局），触发问题即可解决；之后再做可选的 description 优化。
