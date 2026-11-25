# 发布流程

本文说明 axcor 的自动发布机制与版本策略，对应 `.github/workflows/release.yml`。

## 触发方式

发布由推送到 `main` 或维护分支 `X.Y.x` 触发，不再由手动推送的 `v*` 标签触发：

- 推送命中 `main` 或 `[0-9]+.[0-9]+.x` 分支时运行 `version` job。
- `version` job 比较本次推送前后的根 `pyproject.toml`；只有版本发生变化才推导标签，并在同一次运行内继续 `ci` → `build` → `publish`。
- 手动 `git push` 一个 `v*` 标签不会触发任何发布。该结论仅对指向本工作流变更**之后**提交的标签成立：指向变更之前提交的 `v*` 标签仍会命中旧版（`tags` 触发的）release.yml 并可能发布 release。

## 版本来源与发布火车

根 `pyproject.toml` 是虚拟 workspace 根（`[tool.uv] package = false`），本身不构建、不发布；它是**仓级发布火车的版本号**，也是发布标签 `vX.Y.Z` 的唯一来源（`uv version --short` 读取根项目版本）。

`uv build --all-packages` 构建的发行包版本来自各自的 `packages/*/pyproject.toml`（例如 kernel `1.0.0`、各插件 `0.1.0`），与发布标签版本互相独立：一个 `vX.Y.Z` 发布标签下的附件版本由各包自己的 `pyproject.toml` 决定，不强制等于 `X.Y.Z`。这是有意的「仓级发布火车」设计——一次仓级发布对齐一组包快照，而不是为每个包单独发版。

因此：

- 触发检测与标签推导只看根 `pyproject.toml`；修改 `packages/**/pyproject.toml` 不会触发发布。
- 需要发布时，先 bump 根 `pyproject.toml` 的 `version`，提交并推送。
- **bump 必须在同一提交内更新 `uv.lock`**：`uv.lock` 记录了根项目版本（`name = "axcor"` / `source = { virtual = "." }`），ci 的 `uv sync --locked` 会校验两者一致。用 `uv version X.Y.Z` 可同时更新根 `pyproject.toml` 与 `uv.lock`；手工编辑 `pyproject.toml` 则必须在同一次提交内执行 `uv lock`。若只改版本不重锁，ci 会确定性失败，而 `version` job 已先推送了 `vX.Y.Z` 标签，之后补一个只更新 `uv.lock` 的提交不会触发发布（版本未变化），该标签会成为无 release 的死标签。

## 版本策略与标签约束

- 发布标签固定为 `vX.Y.Z`，其中 `X.Y.Z` 是根 `pyproject.toml` 的版本；该标签必须未被其他提交占用。
- 版本号应单调递增，由发版人选定。工作流校验标签格式、发布分支及标签对应的提交。同一版本重复出现时，`version` job 校验已有标签是否指向本次提交：指向同一提交则复用，指向其他提交则 fail loud（`发布标签 vX.Y.Z 已指向其他提交`）。
- **回滚一律通过把版本向前移动完成**（例如发布 `0.3.0` 后要回滚，提交一个 bump 到 `0.3.1` 的修复），而不是 revert 版本 bump 提交：revert 会把根版本改回已发布过的值，`version` job 判定版本变化后会在标签占用校验处 fail loud，main 流水线变红。
- **历史标签占位**：仓库已存在早于本流程的 `v0.1.0`、`v0.2.0` 两个标签，而根版本当前是 `0.1.0`。下次发布必须选择尚未被占用的版本（例如 `>= 0.3.0`），否则会命中标签冲突而中断流水线；选择低于 `v0.2.0` 的版本会发布一个序号更小却可能被标为 latest 的 release，应避免。若确需沿用某个已占用版本，须先移除或移动对应标签。
- `main` 上发布保留 Latest 标记；维护分支 `X.Y.x` 上发布标记 `--latest=false`。同一分支同时最多运行一条发布流水线，避免并行争用 Latest 标记。
- 跨分支同时 bump 到同一版本号时存在已知竞态：不同分支的 `concurrency` 组互不影响、可并行，两个分支在不同提交上 bump 出同一 `vX.Y.Z` 且预检时都未见到标签时，后到者 push 标签会被 git 以 `rejected` 拒绝，报错形态是 git 原生错误而非工作流自己的「已指向其他提交」文案。行为与策略一致（版本冲突 fail loud），属已知特性。

## 同一分支的发布排队

工作流按分支设置 `queue: max`。每个分支最多有一条运行中和 100 条等待中的运行。队列有空位时，后续的普通推送也会排队，已经等待的版本 bump 会继续等待执行。GitHub 按运行进入并发队列的时间排序，推送顺序可能与入队顺序不同，Latest 标记仍可能受实际发布顺序影响。

队列满时，新运行会被取消。每次运行只比较对应推送的前后版本，因此后续普通推送不会补发被取消的 bump。若发现 bump 对应的运行被取消且该版本尚未发布，等队列恢复后在原推送对应的运行上重跑，以保留原来的版本比较范围。参见 [GitHub Actions 并发队列说明](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)。

## 新建维护分支的首次推送

`version` job 在分支创建推送（`github.event.created`，此时 `github.event.before` 为全零 SHA）上跳过，因为无法对创建推送做前后 diff。因此新建维护分支必须分两步：

1. 先推送分支本身（例如 `git push -u origin 1.2.x`，不含版本 bump）。
2. 再单独推送包含根 `pyproject.toml` 版本 bump 的提交。

若把「建分支」和「bump 版本」放在同一次 push 里，创建推送会被跳过，而后续推送的 diff 范围从创建后的 head 开始，该次 bump 永远不会被发布（流水线只显示 skipped、不报错）。

## CI 失败后的恢复方式

标签在 `version` job 中先于 `ci` → `build` → `publish` 推送。若 bump 提交自身带确定性 CI 失败（例如 `uv.lock` 未同步，或 bump 与坏代码在同一次直推中一起落库），重跑原 run 只会重跑同一棵树、同样失败；随后推一个「只修代码、不 bump 版本」的提交不会触发发布（根 `pyproject.toml` 无变化 → 流水线整体 skipped）。恢复方式：

- **继续向前 bump**：在修复提交中把根版本再向前移动（例如 `v0.3.0` 失败后修复并 bump 到 `0.3.1`），新提交会触发完整流水线并发布新版本；失败留下的旧标签 `v0.3.0` 不会自动消失，若确需复用该版本号须带外删除或移动标签。
- 经 PR 合入的 bump 有 PR CI 拦截，正常不会出现该场景；此问题主要影响维护分支「两步直推」这类绕过 PR CI 的流程。

## 已存在的 Release 与草稿

`publish` 先查询同名 release：

- 已发布（`isDraft == false`）：校验附件，缺失的附件用 `gh release upload` 补传（幂等，不破坏已发布的 release）；全部齐全则直接成功（幂等重跑）。
- 草稿（`isDraft == true`）：删除该草稿后重新创建并上传附件，避免 `gh release create` 返回 422 already_exists；删除草稿不会删除已推送的标签。
- 不存在：正常创建。

`dist/` 内若无任何发行包时脚本直接失败，避免创建空 release。
