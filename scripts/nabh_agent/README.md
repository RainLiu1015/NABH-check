# 实时 NABH 验证 Agent

独立 Python 脚本，不连接 Databricks、Lakehouse 或 Lakebase，也不改变现有前端。
输入机构名称和地址；实时读取 NABH 官方目录及其链接的 PDF，由火山方舟 Responses
API 模型核对机构和分院，代码检查证据引用、证书编号、认证类型和有效期，输出 JSON。

## 安装与运行

在仓库根目录运行（Python 3.10+）：

```bash
python3 -m venv .venv-nabh
.venv-nabh/bin/pip install -r scripts/nabh_agent/requirements.txt
export ARK_API_KEY='你的火山方舟 API Key'
export ARK_MODEL='你已开通的支持 Responses API 的模型或 endpoint ID'
.venv-nabh/bin/python scripts/nabh_agent/verify.py \
  --name 'Accord Hospital' \
  --address 'Sant Nagar, Sector No.4, Plot No.1/1, Moshi Pradhikaran, Pune, Maharashtra 412105'
```

API Key 只保留在后端环境，不发送给前端。无需联网搜索组件：此 Agent 的网页读取工具
由 Python 直接请求官方目录/PDF，模型只接收本次实际获取的证据。这比让模型凭搜索摘要
生成 PDF 地址更可追溯；模型调用的是火山方舟，不是 Databricks Foundation Models。
机构名称、地址和候选证书文本会发送给火山方舟。

不配置 Key 也可检查官方查询链路（不会读取 PDF 或给出认证结论）：

```bash
.venv-nabh/bin/python scripts/nabh_agent/verify.py \
  --name 'Accord Hospital' --address 'Pune, Maharashtra 412105' --directory-only
```

## 执行流程与来源

1. `POST https://nabh.co/wp-admin/admin-ajax.php`，表单为
   `action=get_hospitals&q=<名称>&page=1`，附带官网表单的空位置/筛选字段。
2. 解析返回的 `html` 与 `pagination`：机构名称、地址、认证号、状态、实际 PDF 链接。
3. 最多读取 2 页，按邮编/名称排序，处理前 4 个候选；达到限制时结果保持待审核。
4. 实时下载候选 PDF，检查允许域名、重定向、文件签名和体积，使用 pypdf 提取文本。
   官方目录可能链接到第三方 S3 证书桶，仅允许代码中列出的具体主机。
5. 调用火山方舟，让模型判断是否为同一分院并逐项提供原文引用。
6. 检查引用出现在本次抓取的来源中、PDF 与目录证书号一致、有效期覆盖查询日期。
   只对目录明确为 accredited 的记录输出 `verified`。

2026-11-21 已检查的官网前端实现：
- [NABH 查询页](https://nabh.co/find-a-healthcare-organisation/)
- [官网查询 JavaScript](https://nabh.co/wp-content/themes/nabh/js/my-map.js)
- [火山方舟 Responses API](https://docs.volcengine.com/docs/ark/create-model-responses-api?lang=zh)

这是官网当前使用的公开 AJAX 请求，未发现具有稳定性承诺的公开 REST API 文档。
字段、HTML 选择器、下载域名可能变化；发生变化、403、超时或验证码时返回查询失败/待审核，
不绕过访问限制，不推断“没有认证”。只用名称搜索可能因拼写、旧名称漏召回，
`no_match` 也不能作为没有认证的证据。

## 输入与输出

后端可导入 `verify_facility(name, address)`；该函数是同步网络任务，应放在线程池或任务队列，
避免阻塞 Express/异步服务的请求循环。亦可用 `execFile` 启动 CLI，通过参数数组传入名称地址，
不要拼接 shell 命令。单次可能包含多次网络请求，前端适合显示验证进度并轮询任务结果。

Python 服务在仓库根目录可直接调用：

```python
from scripts.nabh_agent.verify import verify_facility

result = verify_facility(
    name="Accord Hospital",
    address="Sant Nagar, Moshi Pradhikaran, Pune, Maharashtra 412105",
)
```

JSON 的主要字段：

| 字段 | 含义 |
| --- | --- |
| `facility` | 输入名称和地址 |
| `checked_at` / `as_of_date` | 本次抓取时间和有效期判断日期，UTC |
| `status` | 整体验证结果 |
| `certificate_urls` | 整体验证成功时的证书链接 |
| `candidates[]` | 机构名称、地址、认证号、官网状态、PDF 链接 |
| `candidates[].pdf` | 下载错误、PDF SHA-256、实际响应 URL |
| `candidates[].verification` | 分院匹配、日期判断、模型理由与原文引用 |
| `warnings` | 候选截断等限制 |

| status | 后台/前端处理 |
| --- | --- |
| `verified` | 身份匹配、PDF 证据齐备、认证在有效期内 |
| `expired` / `not_yet_valid` | 匹配到了证书，但当前日期不在有效期内 |
| `listed_only` | 有目录记录，但不能由此认定 active accreditation，例如 empanelment |
| `needs_review` | 身份不确定、PDF 不可读、证据缺失、候选截断等 |
| `no_match` | 本次名称搜索没有结果，不等于无认证 |
| `source_unavailable` | 官网请求失败或结构变化 |
| `agent_error` / `configuration_error` | 方舟调用失败、响应不合规或配置缺失 |
| `not_attempted` | 仅运行 directory-only 源站检查 |

`verified` 表示本次证据支持该分院的机构认证，不代表所有服务、床位或医生实时可用。
当前脚本不逐项验证服务范围；可后续加入服务名称输入和 Scope 原文引用。
原文引用检查能阻止不存在的引用，但分院语义匹配与日期标签的理解仍依赖模型；
这不是密码学证书验签，也不能仅凭旧 PDF 确认官方尚未更新到目录的撤销状态。
扫描件暂不做 OCR，保留待审核。无缓存，每次重新查询；PDF 不落盘，只返回哈希和证据。

## 测试

```bash
.venv-nabh/bin/python -m unittest discover -s scripts/nabh_agent -v
```

测试覆盖有效期、分院邮编冲突、Empaneled 与认证区别、虚构引用、证书号不一致、
扫描 PDF、域名限制、源站失败和候选截断。目录实时查询可用上面的 `--directory-only` 命令检查。
