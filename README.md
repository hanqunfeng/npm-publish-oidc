# npm-publish-oidc

[中文](README.zh-CN.md)

OIDC publish command for any npm project. Run it from the project root. It checks the project, bumps the version, and pushes a `vX.Y.Z` tag. `npm publish` is done by that project's GitHub Actions workflow through npm Trusted Publishing.

The current working directory is the project root. The script does not change to its own directory.

## Install

Requirements:

- bash, git, Node.js, and npm
- A signed-in [GitHub CLI](https://cli.github.com/) (`gh auth login`) when you want to watch the publish run

Clone the repository, make the script executable, and add the clone directory to `PATH`:

```bash
git clone https://github.com/hanqunfeng/npm-publish-oidc.git ~/npm-publish-oidc
chmod +x ~/npm-publish-oidc/npm-publish-oidc
```

Add this to `~/.zshrc`:

```bash
export PATH="$HOME/npm-publish-oidc:$PATH"
```

Open a new terminal and confirm:

```bash
npm-publish-oidc --help
```

`publish.yml.tmpl` must stay in the same directory as `npm-publish-oidc`. `--init` loads it from the script's real path, including when the command is a symlink.

## Project requirements

Every command except `--help` first checks two things:

1. The current directory can be published to npm: it has a valid `package.json` with a publishable `name` and a semantic `version`, and `private` is not `true`.
2. The current directory is the git repository root.

A real publish and `--dry-run` also require:

- A clean working tree
- `origin` pointing at a GitHub repository
- A workflow under `.github/workflows` that runs `npm publish` and declares `id-token: write`. `publish.yml` is preferred when it exists

Pushing `.github/workflows/` over HTTPS with a personal access token requires the `workflow` scope.

## Initialize a workflow

When the project does not yet have `.github/workflows/publish.yml`:

```bash
cd /path/to/your-package
npm-publish-oidc --init
```

This writes a basic workflow from `publish.yml.tmpl`. For a scoped package with no `publishConfig.access`, the publish command is `npm publish --access public`. An existing file is left unchanged.

The result is only a basic template. Change the build, test, and install steps for your project. `npm ci` in the template needs `package-lock.json`. Remove or replace the `npm test` step when the project has no usable `test` script. `--init` does not commit, push, or publish, and it cannot be combined with other arguments.

## npm Trusted Publisher

Configure GitHub Actions under the package's Trusted Publisher settings:

| Field | Value |
| --- | --- |
| Organization or user | GitHub repository owner |
| Repository | Repository name |
| Workflow filename | `publish.yml` (filename only) |
| Environment name | Leave empty |

The owner, repository, and workflow filename must match `origin` and the workflow you actually use.

## Publish

Run these from the project root:

```bash
npm-publish-oidc --check          # Local checks: existing typecheck, build, and test, then npm pack --dry-run
npm-publish-oidc --dry-run        # Check and preview the commit and tag that would be pushed
npm-publish-oidc patch            # 0.1.0 → 0.1.1, then watch Actions
npm-publish-oidc minor            # 0.1.0 → 0.2.0
npm-publish-oidc major            # 0.1.0 → 1.0.0
npm-publish-oidc                  # Do not bump; push the vX.Y.Z tag for the current version
npm-publish-oidc --no-watch patch # Push without waiting for Actions
npm-publish-oidc --workflow release.yml patch
```

`patch`, `minor`, and `major` can be combined with `--dry-run`, `--no-watch`, and `--workflow` in any order.

If several OIDC publish workflows exist and none of them is `publish.yml`, pass the filename with `--workflow`.

A real publish:

1. Runs `typecheck`, `build`, and `test:unit` or `test` when those scripts exist
2. Previews the package with `npm pack --dry-run`
3. Runs `npm version` when a bump was requested, with the commit message `chore: release v%s`
4. Pushes the current commit and the `vX.Y.Z` tag
5. Waits for GitHub Actions to finish, then checks the version on npm

The local command does not run `npm publish`.
