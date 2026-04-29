# 如何用本工具审查「其他项目」的代码

本工具的 CLI **在你执行命令时所处的工作目录**里读取配置（`.encode_review.yml` 等），并依赖 **Git** 计算本地差异（`encode-code-review local`）。审查「别的仓库」本质上就是：**在目标仓库根目录带好配置与密钥**，再调用 CLI。

下文中的命令名也可用别名 `encode-cr`，与 `encode-code-review` 等价。

---

## 你需要满足的前提

1. **目标代码已是一个 Git 仓库**（有 `.git/`），且当前分支你希望用来对比的版本可用（本地审查默认对比 `HEAD`）。
2. **本地审查只会看到 `git diff HEAD` 能列出的变更**：
   - 修改、删除的已跟踪文件：会出现在 diff 里。
   - **新建文件若仍是未追踪状态（`git status` 里是 `??`）**：默认**不会**出现在 `git diff HEAD` 里，需要先 `git add <文件>` 再跑审查。
3. **当前实现里 `local --path` 表示「Git 仓库目录」**，不是单个文件路径。通常在你已经 `cd` 到目标项目根目录时，使用：

   ```bash
   encode-code-review local --path .
   ```

---

## 方式 A：全局安装（适合经常审查多个仓库）

在任意目录执行一次：

```bash
npm install -g encode-code-review
# 或: pnpm add -g encode-code-review
```

以后审查某个项目时：

```bash
cd /path/to/other-project
```

### 1）配置 AI（二选一）

**推荐：环境变量**（不把密钥写进对方仓库文件）

下面以 **DeepSeek** 为例（OpenAI 兼容接口）：

**PowerShell：**

```powershell
$env:AI_REVIEWER_OPENAI_KEY = "你的_api_key"
$env:AI_REVIEWER_MODEL = "deepseek-chat"
$env:AI_REVIEWER_BASE_URL = "https://api.deepseek.com"
```

**cmd：**

```cmd
set AI_REVIEWER_OPENAI_KEY=你的_api_key
set AI_REVIEWER_MODEL=deepseek-chat
set AI_REVIEWER_BASE_URL=https://api.deepseek.com
```

**或在目标项目根目录放 `.encode_review.yml`**（注意不要把含密钥的该文件提交到公开仓库；可配合 `.gitignore`）：

```yaml
ai:
  provider: openai
  model: deepseek-chat
  apiKey: 你的_api_key
  baseUrl: https://api.deepseek.com
```

也可用 OpenRouter 等：`model`、`baseUrl` 与所选服务商文档一致即可。环境变量名仍以 `AI_REVIEWER_OPENAI_KEY`、`AI_REVIEWER_BASE_URL`、`AI_REVIEWER_MODEL` 等为准（参见主 README「配置」）。

### 2）让 Git「看到」你想审的改动

按需执行：

```bash
git status
git add .
# 若有新文件，务必先 git add，否则可能不会进入审查列表
encode-code-review local --path .
```

---

## 方式 B：不把工具装全局（装进「目标项目」里）

在**要被审查的那个项目**里作为开发依赖安装：

```bash
cd /path/to/other-project
pnpm add -D encode-code-review
# 或: npm install -D encode-code-review
```

同样先设置环境变量或使用 `.encode_review.yml`，然后通过 `pnpm exec` / `npx` 调用：

```bash
pnpm exec encode-code-review local --path .
# 或
npx encode-code-review local --path .
```

这样每个仓库自带一个 CLI 版本，互不干扰。

---

## 方式 C：从本仓库源码开发版运行（你已 clone 本 monorepo 时）

如果你在开发本仓库，并希望用当前源码跑在别的项目上：

```bash
cd /path/to/encode-code-review-repo/ai-code-review
pnpm install
pnpm build   # 生成 dist/cli.mjs
```

在**目标项目**目录中，用 Node 直接跑构建产物（注意路径写成你机器上的绝对路径）：

```bash
cd /path/to/other-project
node /path/to/encode-code-review-repo/ai-code-review/dist/cli.mjs local --path .
```

（同样需要设置 `AI_REVIEWER_*` 环境变量或把 `.encode_review.yml` 放在目标项目根目录。）

---

## 审查 GitHub 上的 PR（不跑本地 diff）

若你要审的是远程 PR，而不是本机工作区，应使用 `github-pr`（需要 GitHub Token 与网络访问）：

```bash
$env:AI_REVIEWER_GITHUB_TOKEN = "ghp_xxx"   # PowerShell 示例
encode-code-review github-pr --owner 组织或用户 --repo 仓库名 --pr-id 123
```

具体参数与 Secrets 命名见主仓库 `README.md` 中的「GitHub Actions 集成」与 CLI 说明。

---

## 常见坑（跨项目使用时最容易踩）

| 现象 | 原因 |
|------|------|
| 新加了文件但没被审查 | 文件仍是 `??` 未追踪，需 `git add` |
| 提示未配置 API 密钥 | 环境变量未在当前终端生效；PowerShell 勿用 cmd 的 `set KEY=value`，请用 `$env:KEY="..."` |
| `local --path` 传了单文件路径报错 | 当前实现期望「仓库目录」；请 `cd` 到项目根再用 `--path .` |
| 配置不生效 | CLI 在**当前工作目录**找 `.encode_review.yml`；请确保在目标项目根目录执行命令 |

---

## 配置优先级（复习）

**CLI 参数 > 环境变量 > 配置文件（`.encode_review.yml` 等）> 默认配置**

把敏感信息放在环境变量里，一般比写进对方仓库的 YAML 更安全。
