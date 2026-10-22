# npm-publish-oidc

在任意 npm 项目根目录执行的 OIDC 发布命令。本地负责检查项目、提升版本并推送 `vX.Y.Z` 标签；`npm publish` 由该项目的 GitHub Actions 通过 npm Trusted Publishing 完成。

当前工作目录就是项目根目录。脚本不会切换到自己所在的目录。

## 安装

依赖：

- bash、git、Node.js、npm
- 监控发布进度时需要已登录的 [GitHub CLI](https://cli.github.com/)（`gh auth login`）

给脚本执行权限，并把所在目录加入 `PATH`：

```bash
chmod +x /Users/hanqf/brew_workspace/npm-publish-oidc/npm-publish-oidc
```

在 `~/.zshrc` 中加入：

```bash
export PATH="/Users/hanqf/brew_workspace/npm-publish-oidc:$PATH"
```

新开一个终端后确认：

```bash
npm-publish-oidc --help
```

`publish.yml.tmpl` 必须和 `npm-publish-oidc` 放在同一目录。`--init` 通过脚本的真实路径读取它，符号链接调用也能找到。

## 项目要求

除 `--help` 外，命令会先确认这两件事：

1. 当前目录可以发布到 npm：存在合法的 `package.json`，包含可发布的 `name` 和语义化 `version`，且 `private` 不为 `true`。
2. 当前目录是 git 仓库根目录。

正式发布和 `--dry-run` 还要求：

- 工作区干净
- `origin` 指向 GitHub 仓库
- `.github/workflows` 中有执行 `npm publish`、并声明 `id-token: write` 的工作流。存在 `publish.yml` 时优先使用它

用 HTTPS 和个人访问令牌推送 `.github/workflows/` 时，令牌需要 `workflow` 权限。

## 初始化工作流

项目还没有 `.github/workflows/publish.yml` 时：

```bash
cd /path/to/your-package
npm-publish-oidc --init
```

这会根据 `publish.yml.tmpl` 生成一份基础工作流。scope 包且未设置 `publishConfig.access` 时，发布命令写成 `npm publish --access public`。文件已存在时不会覆盖。

生成结果只是基础模板，可以按项目修改构建、测试和安装步骤。模板中的 `npm ci` 需要 `package-lock.json`。没有可用的 `test` 脚本时，需要删掉或改掉模板里的 `npm test`。`--init` 不提交、不推送、不发布，也不能和其他参数一起使用。

## npm Trusted Publisher

在 npm 包的 Trusted Publisher 中配置 GitHub Actions：

| 字段 | 值 |
| --- | --- |
| Organization or user | GitHub 仓库所有者 |
| Repository | 仓库名 |
| Workflow filename | `publish.yml`（只写文件名） |
| Environment name | 留空 |

仓库所有者、仓库名和工作流文件名必须与 `origin`、实际使用的工作流一致。

## 发布

在项目根目录执行：

```bash
npm-publish-oidc --check          # 本地检查：已有的 typecheck、build、test，以及 npm pack --dry-run
npm-publish-oidc --dry-run        # 检查并预览将推送的提交和标签
npm-publish-oidc patch            # 0.1.0 → 0.1.1，推送后监控 Actions
npm-publish-oidc minor            # 0.1.0 → 0.2.0
npm-publish-oidc major            # 0.1.0 → 1.0.0
npm-publish-oidc                  # 不升版本，推送当前版本的 vX.Y.Z 标签
npm-publish-oidc --no-watch patch # 推送后不等待 Actions
npm-publish-oidc --workflow release.yml patch
```

`patch`、`minor`、`major` 可与 `--dry-run`、`--no-watch`、`--workflow` 组合，顺序不限。

有多个 OIDC 发布工作流、且其中没有 `publish.yml` 时，用 `--workflow` 指定文件名。

正式发布会：

1. 运行项目里已定义的 `typecheck`、`build`，以及 `test:unit` 或 `test`
2. 预览 `npm pack --dry-run`
3. 按需执行 `npm version`，提交信息为 `chore: release v%s`
4. 推送当前提交和 `vX.Y.Z` 标签
5. 等待 GitHub Actions 结束，再查询 npm 上的版本

本地不执行 `npm publish`。
