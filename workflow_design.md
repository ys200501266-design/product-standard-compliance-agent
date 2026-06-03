# Workflow Design：n8n 交付说明

## 1. 工作流目标

将用户的商品选购需求自动转化为标准检索、商品证据判断和结构化推荐报告。工作流强调证据链、风险提示和可解释排序。

## 2. 节点总览

| 序号 | 节点名称 | 节点类型 | 作用 |
|---:|---|---|---|
| 1 | Webhook | Webhook | 接收用户输入 |
| 2 | Input Validation | Code | 标准化输入并检查必填字段 |
| 3 | IF - Missing product_name | IF | 判断是否缺少产品名称 |
| 4 | Error Response - Missing product_name | Respond to Webhook | 返回输入错误 |
| 5 | LLM 1 - Product Category Identification | HTTP Request | 识别品类和标准检索关键词 |
| 6 | Search 1 - Standard Search | HTTP Request | 检索标准、认证和监管信息 |
| 7 | IF - Standard Search Empty | IF | 判断标准搜索是否为空 |
| 8 | Fallback Response - No Standard Results | Respond to Webhook | 标准线索不足时返回兜底报告 |
| 9 | LLM 2 - Standard Extraction | HTTP Request | 整理标准信息 |
| 10 | LLM 3 - Product Search Strategy | HTTP Request | 生成商品搜索策略 |
| 11 | Search 2 - Product Search | HTTP Request | 检索商品和证据页面 |
| 12 | IF - Product Search Empty | IF | 判断商品证据搜索是否为空 |
| 13 | Fallback Response - No Product Evidence | Respond to Webhook | 商品证据不足时返回兜底报告 |
| 14 | LLM 4 - Compliance Analysis | HTTP Request | 分析商品证据等级 |
| 15 | LLM 5 - Product Ranking | HTTP Request | 评分排序 |
| 16 | LLM 6 - Final Report | HTTP Request | 生成最终报告 |
| 17 | Respond to Webhook | Respond to Webhook | 返回 Markdown 报告 |

## 3. 节点输入输出

| 节点 | 输入 | 输出 |
|---|---|---|
| Webhook | HTTP POST JSON | 用户需求 |
| Input Validation | Webhook body | 标准化字段、`is_missing_product_name` |
| LLM 1 | 标准化用户需求 | 品类、风险、检索关键词 |
| Search 1 | 标准检索关键词 | 标准搜索结果 |
| LLM 2 | 品类信息、标准搜索结果 | 标准清单、证据等级 |
| LLM 3 | 用户需求、标准清单 | 商品搜索 query |
| Search 2 | 商品搜索 query | 商品搜索结果 |
| LLM 4 | 标准清单、商品搜索结果 | 商品证据分析 |
| LLM 5 | 商品证据分析 | 推荐排序和评分 |
| LLM 6 | 标准清单、排序结果 | Markdown 报告 |

## 4. 连接关系

主流程为：

`Webhook -> Input Validation -> IF -> LLM 1 -> Search 1 -> IF -> LLM 2 -> LLM 3 -> Search 2 -> IF -> LLM 4 -> LLM 5 -> LLM 6 -> Respond`

异常流程：

| 异常 | 连接 |
|---|---|
| 缺少 `product_name` | `IF - Missing product_name -> Error Response` |
| 标准搜索无结果 | `IF - Standard Search Empty -> Fallback Response - No Standard Results` |
| 商品搜索无结果 | `IF - Product Search Empty -> Fallback Response - No Product Evidence` |

## 5. 搜索 API 配置方式

默认示例使用 SerpAPI：

| 参数 | 示例 |
|---|---|
| URL | `https://serpapi.com/search.json` |
| engine | `bing` |
| api_key | `{{$env.SERPAPI_API_KEY}}` |
| q | 来自上游 LLM 输出 |

也可以替换为 Tavily、Bing Search API 或 Firecrawl。只要返回结果包含标题、摘要和链接即可。

## 6. OpenAI API 配置方式

LLM 节点使用 HTTP Request 调用：

| 参数 | 值 |
|---|---|
| Method | POST |
| URL | `https://api.openai.com/v1/chat/completions` |
| Authorization | `Bearer {{$env.OPENAI_API_KEY}}` |
| Content-Type | `application/json` |

## 7. Mock Demo 模式

如果没有搜索 API，可以临时把 Search 节点替换为 Set 节点，并使用：

[examples/mock_search_results.json](examples/mock_search_results.json)

Mock 数据只用于演示结构，不能作为真实推荐依据。

## 8. Webhook 测试方法

```bash
curl -X POST "http://localhost:5678/webhook/product-compliance-recommend" \
  -H "Content-Type: application/json" \
  -d @n8n/test_payload.json
```

## 9. 常见错误

| 错误 | 可能原因 | 处理方式 |
|---|---|---|
| Webhook 无响应 | workflow 未激活或 URL 用错 | 检查 test/production webhook URL |
| OpenAI 节点失败 | API Key 缺失或额度不足 | 检查环境变量和账户额度 |
| 搜索节点为空 | 搜索 API 未配置或 query 过窄 | 使用 Mock 数据或更换搜索关键词 |
| JSON 解析失败 | LLM 未按 JSON 输出 | 检查 Prompt 和 response_format |
| 导入失败 | n8n 版本差异 | 参考 import_guide 手动调整节点 |

## 10. 调试方法

1. 先只执行 Webhook 和 Input Validation。
2. 检查 LLM 1 是否输出标准关键词。
3. 单独运行 Search 1 看是否有结果。
4. 如果搜索为空，使用 Mock 数据验证后续流程。
5. 检查每个 LLM 节点是否输出约定字段。
6. 最终报告必须包含证据等级和不确定性。

## 11. 导入 n8n 后需要手动检查的地方

| 检查项 | 说明 |
|---|---|
| API Key | 确认 OpenAI 和搜索 API 可用 |
| Search 节点 | 根据实际搜索服务调整 URL 和参数 |
| Webhook URL | 测试模式和生产模式 URL 不同 |
| IF 条件 | 不同 n8n 版本条件编辑器展示可能不同 |
| Mock 模式 | 如果没有搜索 API，临时使用 Set 节点 |
| 节点名称 | 保持名称一致，方便表达式引用 |