# Paperagent T2：手动云端构建

2026-10-12 已在专用仓库 `chendesong/paperagent-t2` 完成真实云端构建、官方完整 Linux 样例检查、GHCR 发布与匿名拉取。当前作品是用于接入比赛的官方数值基线，不是经过性能优化的策略。

- [构建与官方检查成功](https://github.com/chendesong/paperagent-t2/actions/runs/38153519069)：样例输出 `admissible=True`。
- [匿名拉取与离线 CLI 验证成功](https://github.com/chendesong/paperagent-t2/actions/runs/38153864508)。
- 固定镜像：`ghcr.io/chendesong/paperagent-t2@sha256:bb148b1f85a5ad3c4742c2c9a51b4d728fc699ea112a8fc9cb40c30bc1b54b02`。
- 镜像页面实际显示 Public；没有额外修改访问权限。

工作流仅支持手动触发，使用标准 `ubuntu-24.04` runner；权限只有 `contents: read`、`packages: write`。官方 Actions 全部固定到提交 SHA。登录仅使用运行时自动生成的 `GITHUB_TOKEN`，无需提供 PAT、Team Key、简历或模型密钥。

构建时下载以下官方公开源码，构建上下文只含 Dockerfile、Python 包和 MIT LICENSE。公开样例数据位于上下文之外，仅用于烟测。

| 来源 | 固定版本 |
|---|---|
| [Track 2](https://github.com/Agenthon-2026/track2-forecasting-public/tree/a2c93e7b66b8c6a6c429fda81d6bc7873b7c5dec) | `a2c93e7b66b8c6a6c429fda81d6bc7873b7c5dec` |
| [Toolkit v2.6.0](https://github.com/Agenthon-2026/Agenthon2026-public/tree/bd01548e34d21fd660d88fd06157078fb25ece4e) | `bd01548e34d21fd660d88fd06157078fb25ece4e` |

这是未修改预测逻辑的官方数值基线，不使用文本或 House 模型。临时 Dockerfile 把 toolkit 的 v2.6.0 下载地址固定为同一版本的 commit 地址，并把 MIT LICENSE 保留在镜像 `/opt/LICENSE.track2`。Python 基础镜像仍使用官方 `python:3.13-slim-bookworm` 标签，因此以后重新构建可能得到不同 digest；提交时必须保留实际产出的 digest。

流程先构建 `linux/amd64`，用官方公开样例 `t2-EXAMPLE-ust-curve-1m` 运行受限离线容器，再运行原版 `qfbench2-smoke`。Linux 的原始 manifest 校验及 g0-g3 检查全部保留，失败即停止发布。烟测用 2 CPU / 4 GiB 本地额度，不能证明正式平台性能。通过后推送独立的 `run-<run_id>-<attempt>` 标签，并在运行摘要输出真实 `ghcr.io/chendesong/paperagent-t2@sha256:…`。

## 执行与公开访问

`.github/workflows/build.yml` 和 `verify-anonymous.yml` 已放到专用仓库默认分支并手动运行成功。以后修改策略后，可在 **Actions → Build and smoke Paperagent T2 → Run workflow** 构建新版本；新 digest 应再通过独立的匿名访问检查。第一次匿名检查有配置错误，修复后成功运行，没有跳过实际拉取。

首次发布的 package 默认是 **Private**。公开仓库不会自动公开 GHCR package：检查内容只包含上述基线后，在该 package 的 **Package settings → Danger Zone → Change visibility → Public** 单独操作并确认。容器 package 的 Public 状态才允许匿名拉取。

公开后，在 Linux 用全新的空 Docker 配置验证实际 digest（替换下面的占位符，不要把 `…` 当成 digest）：

```bash
image='ghcr.io/chendesong/paperagent-t2@sha256:<Actions摘要中的64位digest>'
anonymous_config="$(mktemp -d)"
docker --config "$anonymous_config" pull --platform linux/amd64 "$image"
docker --config "$anonymous_config" run --rm --network=none "$image" forecast --help
```

只有烟测通过、推送成功且匿名拉取成功后，这个引用才适合交给后续提交打包。这里不创建 CodaBench 提交，也不上传附件。

## 回滚

保留上一次成功运行的完整 `@sha256:…` 引用及对应运行记录。需要回滚时，后续提交引用改回那个 digest；不要依赖可变标签，也不要删除仍被提交引用的旧版本。此工作流不改 `latest` 标签。

## 官方依据（2026-10-11 核对）

- [GitHub：发布 Docker 镜像及最小权限](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)
- [GitHub：GHCR、默认 Private、digest 与匿名访问](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [GitHub：独立配置 package visibility](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility)
- [Docker：官方构建 Actions](https://docs.docker.com/build/ci/github-actions/)
- [Docker：build-push-action 的 digest 输出](https://github.com/docker/build-push-action)
- [GitHub：公开仓库标准 runner 免费](https://docs.github.com/en/billing/concepts/product-billing/github-actions)；[GHCR 镜像存储及带宽当前免费](https://docs.github.com/en/billing/concepts/product-billing/github-packages)。本方案不使用 larger runner、付费构建服务或持久化 Actions 缓存/构建记录。
- [Track 2：官方本地烟测和降低本地资源额度的说明](https://github.com/Agenthon-2026/track2-forecasting-public/blob/a2c93e7b66b8c6a6c429fda81d6bc7873b7c5dec/README.md)
- [Toolkit：Development runtime](https://github.com/Agenthon-2026/Agenthon2026-public/blob/bd01548e34d21fd660d88fd06157078fb25ece4e/docs/DEVELOPMENT-RUNTIME.md)
