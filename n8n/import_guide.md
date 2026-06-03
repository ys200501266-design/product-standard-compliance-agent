# n8n 导入指南

## 1. 适用对象

这份指南适合第一次使用 n8n 的用户。你不需要先理解所有节点，只需要完成导入、配置 API Key、执行测试。

## 2. 导入前准备

| 准备项 | 说明 |
|---|---|
| n8n | 本地、Docker 或云端版本均可 |
| OpenAI API Key | 用于 LLM 节点 |
| 搜索 API Key | 默认示例使用 SerpAPI，也可替换为 Tavily/Bing/Firecrawl |
| workflow JSON | `n8n/product_standard_compliance_workflow.json` |
| 测试 payload | `n8n/test_payload.json` |

## 3. 手动导入步骤

1. 打开 n8n。
2. 新建 workflow。
3. 点击右上角菜单。
4. 选择 `Import from File`。
5. 选择 `product_standard_compliance_workflow.json`。
6. 导入后检查节点是否完整。
7. 配置 OpenAI 和搜索 API。
8. 点击 `Execute workflow` 测试。
9. 测试通过后保存并激活。

## 4. CLI 导入步骤

如果本机已经安装 n8n CLI，可以运行：

```bash
n8n import:workflow --input=n8n/product_standard_compliance_workflow.json
```

导入后仍需要进入 n8n UI 检查 API Key、Webhook URL 和搜索节点参数。

## 5. Docker 环境导入思路

如果 n8n 运行在 Docker 中，建议先把 JSON 文件复制到容器，再在容器中执行导入。不同用户的容器名称可能不同，请先确认容器名。

示例思路：

```bash
docker cp n8n/product_standard_compliance_workflow.json n8n:/tmp/product_standard_compliance_workflow.json
docker exec n8n n8n import:workflow --input=/tmp/product_standard_compliance_workflow.json
```

不要删除容器中的数据目录，也不要运行会清空数据的命令。

## 6. API Key 配置

复制 `.env.example` 为 `.env`：

```bash
OPENAI_API_KEY=your_openai_api_key
SERPAPI_API_KEY=your_serpapi_api_key
N8N_WEBHOOK_URL=http://localhost:5678/webhook/product-compliance-recommend
```

注意：

- 不要把真实 Key 上传到 GitHub。
- `.env` 已被 `.gitignore` 忽略。
- 如果使用 n8n 凭据管理，也可以不使用 `.env`，直接在节点中选择 credential。

## 7. Webhook 测试

测试模式：

```bash
curl -X POST "http://localhost:5678/webhook-test/product-compliance-recommend" \
  -H "Content-Type: application/json" \
  -d @n8n/test_payload.json
```

激活后生产模式：

```bash
curl -X POST "http://localhost:5678/webhook/product-compliance-recommend" \
  -H "Content-Type: application/json" \
  -d @n8n/test_payload.json
```

## 8. 常见问题

| 问题 | 解决方式 |
|---|---|
| 导入后节点报红 | 检查 n8n 版本和节点类型 |
| OpenAI 调用失败 | 检查 `OPENAI_API_KEY` 是否有效 |
| 搜索结果为空 | 检查 `SERPAPI_API_KEY` 或改用 Mock 数据 |
| Webhook 没有返回 | 确认工作流处于执行或激活状态 |
| LLM 输出不是 JSON | 检查 Prompt 和 response_format |

## 9. 如果导入失败怎么办

1. 确认 JSON 文件没有被编辑器破坏。
2. 使用 n8n UI 的 Import from File。
3. 如果某个 IF 节点不兼容，可以手动新建 IF 节点并按节点名称连接。
4. 如果搜索节点不兼容，可以用 HTTP Request 节点重新配置搜索 API。
5. 如果仍失败，可以先搭建主流程，再逐步补异常处理节点。

## 10. 如何确认 workflow 已经跑通

| 检查项 | 通过标准 |
|---|---|
| Webhook | 能收到测试 JSON |
| LLM 1 | 能输出品类和关键词 |
| Search 1 | 能返回标准搜索结果或进入兜底 |
| LLM 2 | 能输出标准表格数据 |
| Search 2 | 能返回商品线索或进入兜底 |
| Final Report | 能返回 Markdown 报告 |