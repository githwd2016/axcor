# 发布流程

本文说明 axcor 的自动发布机制与版本策略，对应 `.github/workflows/release.yml`。

## 触发方式

发布由推送到 `main` 或维护分支 `X.Y.x` 触发，不再由手动推送的 `v*` 标签触发：

- 推送命中 `main` 或 `[0-9]+.[0-9]+.x` 分支时运行 `version` job。
- `version` job 比较本次推送前后的根 `pyproject.toml`；只有版本发生变化才推导标签，并在同一次运行内继续 `ci` → `build` → `publish`。
- 手动 `git push` 一个 `v*` 标签不会触发任何发布。

## 版本来源与发布火车

根 `pyproject.toml` 是虚拟 workspace 根（`[tool.uv] package = false`），本身不构建、不发布；它是**仓级发布火车的版本号**，也是发布标签 `vX.Y.Z` 的唯一来源（`uv version --short` 读取根项目版本）。

`uv build --all-packages` 构建的发行包版本来自各自的 `packages/*/pyproject.toml`（例如 kernel `1.0.0`、各插件 `0.1.0`），与发布标签版本互相独立：一个 `vX.Y.Z` 发布标签下的附件版本由各包自己的 `pyproject.toml` 决定，不强制等于 `X.Y.Z`。这是有意的「仓级发布火车」设计——一次仓级发布对齐一组包快照，而不是为每个包单独发版。

因此：

- 触发检测与标签推导只看根 `pyproject.toml`；修改 `packages/**/pyproject.toml` 不会触发发布。
- 需要发布时，先 bump 根 `pyproject.toml` 的 `version`，提交并推送。

## 版本策略与标签约束

- 发布标签固定为 `vX.Y.Z`，其中 `X.Y.Z` 是根 `pyproject.toml` 的版本；该标签必须未被其他提交占用。
- 版本号应单调递增。同一版本重复出现时，`version` job 校验已有标签是否指向本次提交：指向同一提交则复用，指向其他提交则 fail loud（`发布标签 vX.Y.Z 已指向其他提交`）。
- **历史标签占位**：仓库已存在早于本流程的 `v0.1.0`、`v0.2.0` 两个标签，而根版本当前是 `0.1.0`。下次发布必须选择尚未被占用的版本（例如 `>= 0.3.0`），否则会命中标签冲突而中断流水线；选择低于 `v0.2.0` 的版本会发布一个序号更小却可能被标为 latest 的 release，应避免。若确需沿用某个已占用版本，须先移除或移动对应标签。
- `main` 上发布保留 Latest 标记；维护分支 `X.Y.x` 上发布标记 `--latest=false`。

## 新建维护分支的首次推送

`version` job 在分支创建推送（`github.event.created`，此时 `github.event.before` 为全零 SHA）上跳过，因为无法对创建推送做前后 diff。因此新建维护分支必须分两步：

1. 先推送分支本身（例如 `git push -u origin 1.2.x`，不含版本 bump）。
2. 再单独推送包含根 `pyproject.toml` 版本 bump 的提交。

若把「建分支」和「bump 版本」放在同一次 push 里，创建推送会被跳过，而后续推送的 diff 范围从创建后的 head 开始，该次 bump 永远不会被发布（流水线只显示 skipped、不报错）。

## 已存在的 Release 与草稿

`publish` 先查询同名 release：

- 已发布（`isDraft == false`）：校验所有附件齐全；缺失则失败（fail loud），齐全则直接成功（幂等重跑）。
- 草稿（`isDraft == true`）：删除该草稿后重新创建并上传附件，避免 `gh release create` 返回 422 already_exists；删除草稿不会删除已推送的标签。
- 不存在：正常创建。

`dist/` 内若无任何发行包时脚本直接失败，避免创建空 release。
