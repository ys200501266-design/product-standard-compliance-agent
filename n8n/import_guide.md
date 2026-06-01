# n8n 导入指南

## 文件位置

工作流文件：

```text
n8n/product_standard_compliance_workflow.json
```

测试请求：

```text
n8n/test_payload.json
```

## 环境变量

建议在 n8n 运行环境中配置：

```bash
OPENAI_API_KEY=你的 OpenAI API Key
SERPAPI_API_KEY=你的 SerpAPI API Key
```

如果你使用 Tavily、Bing Search API 或 Firecrawl，可以把两个 Search 节点替换为对应 HTTP Request 节点。

## 手动导入

1. 打开 n8n。
2. 新建 workflow。
3. 点击右上角菜单。
4. 选择 `Import from File`。
5. 选择 `product_standard_compliance_workflow.json`。
6. 检查 Webhook、LLM、HTTP Request 节点。
7. 填入自己的 API Key 或确认环境变量可用。
8. 点击 `Execute workflow` 测试。
9. 测试成功后保存并激活。

## Docker 导入方式

如果 n8n 运行在 Docker 容器中，请不要直接在宿主机执行危险命令。推荐步骤：

```bash
docker cp product-standard-compliance-agent/n8n/product_standard_compliance_workflow.json n8n:/tmp/product_standard_compliance_workflow.json
docker exec -it n8n n8n import:workflow --input=/tmp/product_standard_compliance_workflow.json
```

如果容器名称不是 `n8n`，请先执行：

```bash
docker ps
```

找到真实容器名后替换命令中的 `n8n`。

## Webhook 测试

在 n8n 中点击 `Execute workflow` 后，使用测试 URL：

```bash
curl -X POST "http://localhost:5678/webhook-test/product-compliance-recommend" \
  -H "Content-Type: application/json" \
  -d @n8n/test_payload.json
```

工作流激活后，生产 URL 通常为：

```bash
curl -X POST "http://localhost:5678/webhook/product-compliance-recommend" \
  -H "Content-Type: application/json" \
  -d @n8n/test_payload.json
```

## 没有搜索 API 时

可以把 `Search 1 - Standard Search` 和 `Search 2 - Product Search` 暂时替换为 Set 节点，模拟字段：

```json
{
  "organic_results": [
    {
      "title": "示例页面标题",
      "link": "https://example.com",
      "snippet": "示例搜索摘要，仅用于 Demo。"
    }
  ]
}
```

模拟数据不能用于真实合规结论。

