# NABH Check

独立的实时 NABH 认证验证 Agent。输入医疗机构的名称和地址，实时查询 NABH 官方目录、
读取目录链接的认证 PDF，由火山方舟模型核对机构与分院，并返回带原文证据的 JSON。

不依赖 Databricks、Lakehouse、Lakebase 或其他项目。当前 Demo 是命令行程序。

## 快速启动 Demo

需要 Python 3.10+。在终端执行：

```bash
git clone https://github.com/RainLiu1015/NABH-check.git
cd NABH-check
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Windows PowerShell 激活命令为 `.venv\Scripts\Activate.ps1`。

### Demo 1：无需 API Key 的官网实时查询

```bash
python verify.py \
  --name 'Accord Hospital' \
  --address 'Sant Nagar, Sector No.4, Plot No.1/1, Moshi Pradhikaran, Pune, Maharashtra 412105' \
  --directory-only
```

输出候选机构的名称、地址、认证号、官方状态和 `pdf_url`。
`status: not_attempted` 表示只查询官网，尚未执行 PDF 与模型验证。
这是源站连通性 Demo，不是模拟数据，也不会产生模型调用。

### Demo 2：火山方舟完整 Agent 验证

在火山方舟控制台创建 API Key，并开通支持 Responses API 的模型或推理接入点。
在当前终端设置环境变量：

```bash
export ARK_API_KEY='你的真实 API Key'
export ARK_MODEL='已开通的模型 ID 或 endpoint ID'

python verify.py \
  --name 'Accord Hospital' \
  --address 'Sant Nagar, Sector No.4, Plot No.1/1, Moshi Pradhikaran, Pune, Maharashtra 412105'
```

`.env.example` 仅是配置参考，程序不会自动加载 `.env` 文件。
Key 只留在本地/后端环境。完整验证会把机构名称、地址和证书文本发送给火山方舟。
不需要开通联网搜索组件：Python 实时读取官网和 PDF，方舟模型只分析本次获取的证据。

保存结果：

```bash
python verify.py \
  --name 'Accord Hospital' \
  --address 'Sant Nagar, Moshi Pradhikaran, Pune, Maharashtra 412105' \
  > /tmp/nabh-result.json
```

可将输入地址换为 `Mumbai, Maharashtra 400001`，观察分院地址/邮编冲突的处理。
结果由实时官网记录和模型输出决定，不保证示例每次均为 `verified`。
一次完整验证会执行多次网络请求，可能需要数分钟。

### 方舟响应超时排查

默认使用 `thinking: disabled`、`max_output_tokens: 4096`，方舟 socket 超时为 180 秒。
这适用于 `doubao-seed-2-1-pro-260915` 的证据提取 Demo；原先未设置 thinking 的请求
可能启用深度思考，延长等待时间。网络延迟、服务排队也可能导致超时。
此超时是网络 socket 等待时间，不是整个任务的总时限。

```bash
export ARK_MODEL='doubao-seed-2-1-pro-260915'
export ARK_THINKING=disabled
export ARK_TIMEOUT_SECONDS=180
export ARK_MAX_OUTPUT_TOKENS=4096

python verify.py \
  --name 'Accord Hospital' \
  --address 'Sant Nagar, Sector No.4, Plot No.1/1, Moshi Pradhikaran, Pune, Maharashtra 412105' \
  --verbose
```

`--verbose` 将阶段日志写到 stderr，不影响 stdout 的 JSON。
若仍超时，可将 `ARK_TIMEOUT_SECONDS` 提高到 300（最大 600），并检查运行环境到
`ark.cn-beijing.volces.com` 的网络连接。脚本不自动重试模型请求，避免重复发起收费调用。
若返回 HTTP 401/403/404，应检查 Key、模型访问权限和模型 ID，而不是继续增大超时。
需要深度思考时可设 `ARK_THINKING=enabled`，并相应增加超时和输出 token 预算。

## 验证流程

```text
名称 + 地址
    ↓
NABH 官网 AJAX 查询
    ↓
提取候选机构、地址、认证号、状态和证书链接
    ↓
实时下载 PDF → 提取文本 → 计算 SHA-256
    ↓
火山方舟：核对同一分院，提供名称、地址、证书号、有效期原文
    ↓
代码：检查引用、邮编冲突、证书号、认证类型和有效期
    ↓
JSON 结果 + 证据 + PDF URL
```

官网当前使用 `POST https://nabh.co/wp-admin/admin-ajax.php`，主要表单字段为
`action=get_hospitals`、`q=<机构名称>` 和 `page=1`。
这是官网前端使用的 AJAX 接口，未发现具有稳定性承诺的公开 REST API 文档。

脚本最多查询 2 页、处理前 4 个候选。达到限制时保留待审核，不自动通过。
只接受允许名单中的来源主机，包括官网实际链接的一个具体 S3 证书桶；
检查重定向、PDF 签名、体积，并拒绝未知下载主机。
官网请求失败、验证码、响应结构变化不会被判为“没有认证”。

参考：
- [NABH 官方机构查询](https://nabh.co/find-a-healthcare-organisation/)
- [官网查询 JavaScript](https://nabh.co/wp-content/themes/nabh/js/my-map.js)
- [火山方舟 Responses API](https://docs.volcengine.com/docs/ark/create-model-responses-api?lang=zh)

## 输出

| 字段 | 含义 |
| --- | --- |
| `facility` | 输入的名称和地址 |
| `checked_at` / `as_of_date` | 本次抓取时间与有效期判断日期，UTC |
| `status` | 整体验证结果 |
| `certificate_urls` | 整体验证成功时的 PDF 链接 |
| `candidates[]` | 候选名称、地址、认证号、官网状态、PDF 链接 |
| `candidates[].pdf` | PDF 哈希、最终响应 URL 或下载错误 |
| `candidates[].verification` | 匹配结论、日期判断、模型理由与原文引用 |
| `warnings` | 候选截断等限制 |

| status | 含义 |
| --- | --- |
| `verified` | 分院匹配、证据齐备、目录表明已认证且证书在有效期内 |
| `expired` / `not_yet_valid` | 当前日期不在证书有效期内 |
| `listed_only` | 有目录记录，但不能据此认定 active accreditation，例如 Empaneled |
| `needs_review` | 身份不确定、证据不足、PDF 不可读或候选被截断 |
| `no_match` | 本次名称搜索没有结果，不代表没有认证 |
| `source_unavailable` | 官方查询失败或响应结构变化 |
| `agent_error` | 方舟调用失败或响应不合规 |
| `configuration_error` | 未设置 `ARK_API_KEY` 或 `ARK_MODEL` |
| `not_attempted` | 仅运行了 `--directory-only` |

## 在自己的后端调用

将仓库目录作为 Python 模块路径，在工作线程/任务队列中调用：

```python
from verify import verify_facility

result = verify_facility(
    name="Accord Hospital",
    address="Sant Nagar, Moshi Pradhikaran, Pune, Maharashtra 412105",
)
```

这是同步网络任务，不应直接阻塞异步服务的事件循环。
Node.js 后端亦可使用 `execFile` 启动 CLI，并通过参数数组传入名称和地址；不要拼接 shell 命令。

## 测试

```bash
python -m unittest -v test_verify
```

离线测试覆盖有效期、分院邮编冲突、Empaneled 与认证的区别、虚构引用、证书号不一致、
扫描 PDF、下载域名限制、源站失败和候选截断。
真实官网查询和 PDF 提取已检查；火山方舟完整链路需要自行配置有效 Key 后运行。

## 限制

`verified` 只表示本次证据支持该分院的机构认证，不代表所有服务、医生或床位实时可用。
当前不逐项验证服务范围，不对扫描件做 OCR。没有缓存，每次重新查询，PDF 不落盘。
名称搜索可能因拼写或旧名称漏召回；分院语义匹配及日期标签理解仍依赖模型。
引用检查不是证书数字签名验证，也不能发现官网尚未更新的撤销信息。
