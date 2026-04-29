# CLI Demo 验证步骤

本文档用于复现 `encode-code-review` 的本地 CLI 审查流程。这里只验证 `### CLI命令` 中的本地审查，不验证 GitHub Actions 集成。

## 1. 准备环境

进入项目目录：

```cmd
cd /d e:\Edge\ai-code-review\ai-code-review
```

安装依赖：

```cmd
pnpm install
```

确认 Node.js 和 pnpm 可用：

```cmd
node --version
pnpm --version
```

本次验证使用的是：

```text
Node.js v22.22.0
pnpm 10.6.2
```

## 2. 制造一个可审查的本地 diff

CLI 的本地审查会读取 `git diff HEAD`，所以需要先让一个已被 Git 跟踪的文件产生差异。

在 `test_review_file/index.js` 末尾临时追加下面的代码：

```js
export function runUserScript(script) {
  return eval(script)
}
```

确认 Git 能看到这个 diff：

```cmd
git diff --name-status HEAD
```

预期输出中应该包含：

```text
M       test_review_file/index.js
```

## 3. 设置 DeepSeek 环境变量

不要把真实 API key 写入代码或提交到仓库。运行前在当前终端设置环境变量即可：

如果你使用的是 PowerShell：

```powershell
$env:AI_REVIEWER_OPENAI_KEY = "你的_deepseek_api_key"
$env:AI_REVIEWER_MODEL = "deepseek-chat"
$env:AI_REVIEWER_BASE_URL = "https://api.deepseek.com"
$env:AI_REVIEWER_MAX_TOKENS = "1200"
```

如果你使用的是 cmd：

```cmd
set AI_REVIEWER_OPENAI_KEY=你的_deepseek_api_key
set AI_REVIEWER_MODEL=deepseek-chat
set AI_REVIEWER_BASE_URL=https://api.deepseek.com
set AI_REVIEWER_MAX_TOKENS=1200
```

注意：PowerShell 里不能用 `set AI_REVIEWER_OPENAI_KEY=...` 来设置 Node.js 可读取的环境变量。那只是设置 PowerShell 变量/别名相关内容，`process.env.AI_REVIEWER_OPENAI_KEY` 仍然为空，因此 CLI 会提示“OpenAI API 密钥未配置”。

说明：

- `AI_REVIEWER_MODEL=deepseek-chat`：使用 DeepSeek Chat 模型。
- `AI_REVIEWER_BASE_URL=https://api.deepseek.com`：使用 DeepSeek 的 OpenAI-compatible API 地址。
- `AI_REVIEWER_MAX_TOKENS=1200`：demo 场景下限制输出长度，避免结果太长。

## 4. 运行本地 CLI 审查

当前实现中，`local --path` 接收的是 Git 仓库目录，不是单个文件路径。因此应传项目根目录：

```cmd
pnpm cli local --path .
```

不要使用 README 中的单文件路径写法：

```cmd
pnpm cli local --path ./test_review_file/index.js
```

该写法在当前实现中会把文件路径当作工作目录，导致本地 diff 获取失败。

## 5. 预期结果

命令成功时，会看到类似输出：

```text
配置来源:
- 配置文件: 未设置
- 环境变量: deepseek-chat
- 默认配置: deepseek/deepseek-chat-v3-0324:free
最终模型配置: deepseek-chat

i 开始代码审查...
i 获取本地路径 . 的代码差异
i 获取到 1 个文件差异
i 过滤后剩余 1 个文件需要审查
i 审查文件: test_review_file/index.js
```

随后 AI 会输出审查报告。对于上面的 `eval` demo，预期会识别到类似问题：

```text
使用 eval 执行用户提供的脚本存在严重的安全风险，可能导致任意代码执行。
```

最后会看到类似：

```text
√ 代码审查完成
```

审查结果也会保存到系统临时目录，例如：

```text
C:\Users\Administrator\AppData\Local\Temp\encode-code-review\review-results-*.json
```

## 6. 清理 demo 改动

验证完成后，删除 `test_review_file/index.js` 末尾临时追加的 `runUserScript` 函数。

再次确认没有 demo 代码残留：

```cmd
git diff -- test_review_file/index.js
```

如果只剩换行符提示，没有实际 diff 内容，说明 demo 代码已经清理。

## 7. 已知注意事项

- 本地 CLI 当前依赖 `git diff HEAD`，因此只会审查 Git 能看到的已跟踪文件变更。
- `local --path` 当前应传仓库目录，例如 `.`，不要传单个文件路径。
- AI 返回裸 JSON 时，当前解析器可能没有完全按结构化 JSON 解析，导致后续格式化统计显示偏乱；但真实模型调用和审查流程已经跑通。
