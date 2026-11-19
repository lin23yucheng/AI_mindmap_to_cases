# FreeMind 测试点生成测试用例

本项目只做一件事：读取 FreeMind `.mm` 测试点，通过大模型理解整图业务上下文，再逐个叶子测试点生成完整测试用例，导出 CSV。一个测试点可以生成多条独立场景用例；不自动执行用例。

## 安装依赖

推荐 Python 3.11 及以上（本次使用 Python 3.12 验证）。Windows PowerShell 在项目目录执行：

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

仅依赖 `requests` 和 `python-dotenv`，无需安装浏览器或 Node.js。

## 环境变量

项目已有自己的 `.env`，请保留。只有首次创建且 `.env` 不存在时，才复制模板：

```powershell
if (!(Test-Path .env)) { Copy-Item .env.example .env }
```

编辑 `.env`，以 DeepSeek 为例：

```dotenv
LLM_PROVIDER=deepseek
DEEPSEEK_API_KEY=填写自己的密钥
DEEPSEEK_API_URL=https://api.deepseek.com/chat/completions
LLM_REASONING_MODEL=deepseek-reasoner
LLM_GENERATION_MODEL=deepseek-chat
LLM_PROJECT_SUMMARY_MAX_TOKENS=24000
```

`LLM_REASONING_MODEL` 用于整图/模块摘要，`LLM_GENERATION_MODEL` 用于生成用例；请填写账号实际可用的模型名称。上述模板不代表当前账号的模型权限。本次整理保留了当前 `.env` 的模型设置。

当前仅支持 DeepSeek，`LLM_PROVIDER` 设置为 `deepseek`。`DEEPSEEK_API_URL` 为完整的聊天补全接口 URL，默认是 `https://api.deepseek.com/chat/completions`。

固定加载脚本所在项目的 `.env`，不会寻找旧项目配置；进程环境变量优先。密钥及 `.env` 不应提交到版本库。

不请求远程服务的配置检查：

```powershell
.\.venv\Scripts\python.exe freemind_to_cases_AI.py --check-config
```

该命令检查配置是否齐全，不验证密钥有效性、余额或远程模型是否可用。可选超时、重试次数与 token 上限见 `.env.example`。

## 运行

将测试点文件放入 `inputs/freemind/`，以下使用 `XXXXX.mm` 作为文件名示例，运行时请替换为实际文件名：

```powershell
.\.venv\Scripts\python.exe freemind_to_cases_AI.py --input "inputs/freemind/XXXXX.mm"
```

指定输入，默认导出到 `outputs/csv/<输入文件名>_full_cases.csv`：

```powershell
.\.venv\Scripts\python.exe freemind_to_cases_AI.py --input "inputs/freemind/XXXXX.mm"
```

指定输出：

```powershell
.\.venv\Scripts\python.exe freemind_to_cases_AI.py `
  --input "inputs/freemind/XXXXX.mm" `
  --output "outputs/csv/XXXXX.csv"
```

输入和输出的相对路径始终相对于项目根目录，也可传绝对路径；输出父目录自动创建。生成会向所配置的模型服务发送测试点内容并产生相应 API 用量。

CSV 使用带 BOM 的 UTF-8，包含：源测试点ID、所属模块、用例名称、前置条件、步骤、预期、关键词、优先级、用例类型、适用阶段。输出供人工审核或导入测试管理平台使用。

## 配置与目录

```text
freemind_to_cases_AI.py       生成器与命令行入口
requirements.txt            Python 依赖
.env.example                无密钥的配置模板
config/business-modules.json 业务模块清单和兜底模块
config/tester-names.json     测试人员署名过滤名单
inputs/freemind/             原始 .mm 测试点
outputs/csv/                生成的 CSV、摘要缓存与生成断点
```

`business-modules.json` 的 `moduleNames` 为合法业务模块，`fallbackModule` 用于未匹配测试点；更换业务时先调整该配置。`tester-names.json` 的 `testerNames` 用于去除节点中的人员署名，防止将姓名识别成模块或用例标题。

## 中断、失败与重跑

- 生成失败退出码为 `1`，手动中断为 `130`，全部成功为 `0`。失败保留已经生成的 CSV，并提示未完成。
- `*.csv.breakpoint` 只记录生成进度；同一输入和输出使用相同命令重试，会跳过已成功生成的测试点。不要在保留断点时修改输入或业务配置；有变更请使用新的输出文件名。
- `*.csv.project_context.json` 是按测试点内容、摘要模型等校验复用的业务摘要缓存。
- 完整成功后删除断点。**再次运行相同输出路径会重新生成并替换 CSV**；需要保留结果时，先另存或使用新的 `--output`。并发生成请使用不同输出路径。
- 本项目不包含用例执行、状态合并、自动化脚本或执行报告。输出中的断点和摘要只用于生成阶段。
