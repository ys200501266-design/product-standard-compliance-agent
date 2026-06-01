# n8n 工作流设计说明

## 总览

工作流路径：`n8n/product_standard_compliance_workflow.json`

设计思路：接收用户需求后，先识别品类，再搜索标准，再搜索商品，最后基于公开证据进行合规分析和推荐排序。

## 节点 1：Webhook

- 类型：Webhook
- 作用：接收用户输入。
- 输入：`product_name`、`use_case`、`budget`、`region`、`extra_requirements`
- 输出：原始用户需求 JSON。
- 连接：输出给 LLM 1。

## 节点 2：LLM 1 - Product Category Identification

- 类型：HTTP Request
- 作用：调用 OpenAI API，识别产品品类并生成标准检索关键词。
- 输入：Webhook JSON。
- 输出：品类、子品类、风险等级、标准检索关键词、商品检索关键词。
- 连接：输出给 Search 1。

## 节点 3：Search 1 - Standard Search

- 类型：HTTP Request
- 作用：调用搜索 API 检索标准和认证信息。
- 输入：LLM 1 生成的标准关键词。
- 输出：搜索结果列表。
- 连接：输出给 LLM 2。
- 推荐 API：SerpAPI、Tavily、Bing Search API、Firecrawl。

## 节点 4：LLM 2 - Standard Extraction

- 类型：HTTP Request
- 作用：从搜索结果提取标准编号、名称、类型、适用范围、关键要求和来源链接。
- 输入：产品品类信息和标准搜索结果。
- 输出：结构化标准清单。
- 连接：输出给 LLM 3。

## 节点 5：LLM 3 - Product Search Strategy

- 类型：HTTP Request
- 作用：基于用户需求和标准清单生成商品搜索关键词组合。
- 输入：用户需求、品类判断、标准清单。
- 输出：商品搜索 query。
- 连接：输出给 Search 2。

## 节点 6：Search 2 - Product Search

- 类型：HTTP Request
- 作用：搜索商品、品牌官网、电商详情页、检测报告和认证信息。
- 输入：LLM 3 生成的搜索 query。
- 输出：商品搜索结果。
- 连接：输出给 LLM 4。

## 节点 7：LLM 4 - Compliance Analysis

- 类型：HTTP Request
- 作用：分析候选商品是否公开标注标准或认证信息。
- 输入：用户需求、标准清单、商品搜索结果。
- 输出：商品合规证据分析。
- 连接：输出给 LLM 5。

## 节点 8：LLM 5 - Product Ranking

- 类型：HTTP Request
- 作用：根据评分规则排序推荐商品。
- 输入：商品合规分析。
- 输出：排序结果、分数、推荐理由和风险说明。
- 连接：输出给 LLM 6。

## 节点 9：LLM 6 - Final Report

- 类型：HTTP Request
- 作用：生成最终 Markdown 报告。
- 输入：标准整理和推荐排序结果。
- 输出：Markdown 报告。
- 连接：输出给 Respond to Webhook。

## 节点 10：Respond to Webhook

- 类型：Respond to Webhook
- 作用：把 Markdown 报告返回给用户。
- 输入：LLM 6 输出。
- 输出：HTTP Response。

## 需要配置的 API

| API | 用途 |
|---|---|
| OPENAI_API_KEY | 调用 LLM 节点 |
| SERPAPI_API_KEY | 调用搜索节点 |

如果不使用 SerpAPI，可以把搜索节点替换为 Tavily、Bing Search API 或 Firecrawl。只要输出包含标题、摘要和链接即可。

## 没有搜索 API 时如何测试

可以临时把 Search 1 和 Search 2 替换为 Set 节点，手动放入模拟搜索结果。示例字段：

```json
{
  "organic_results": [
    {
      "title": "示例标准页面标题",
      "link": "https://example.com/standard",
      "snippet": "此处为模拟标准摘要。"
    }
  ]
}
```

注意：模拟数据只用于演示流程，不能作为真实合规结论。

## 使用 Postman 测试 Webhook

1. 在 n8n 中点击 `Execute workflow`。
2. 复制 Webhook Test URL。
3. Postman 选择 POST。
4. Body 选择 raw JSON。
5. 粘贴 `n8n/test_payload.json` 内容。
6. 点击 Send。
7. 查看返回的 Markdown 报告。

