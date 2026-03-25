---
name: ima-knowledge-extract
description: |
  IMA 知识库内容深度抓取技能。当用户要求在知识库中搜索某些知识、扒取内容、整理信息时使用。
  结合 IMA 知识库 API 的 search_knowledge 接口，通过多关键词轮询搜索并提取高亮片段（highlight_content），拼凑出完整内容。
  
  触发场景：
  - "帮我搜一下知识库里XXX相关内容"
  - "在知识库里找XXX"
  - "扒出知识库里关于XXX的内容"
  - "整理知识库中XXX的信息"
  - 任何需要在 IMA 知识库中搜索并提取内容的需求
---

# IMA 知识库内容抓取

本 skill 依赖 `ima-skill` 的知识库模块，通过**多关键词搜索 + 高亮片段拼凑**的方法，从知识库中扒取内容。

## 核心原理

IMA 知识库的 `search_knowledge` 接口会返回 `highlight_content` 字段，这是文档中被关键词匹配的片段。通过多轮、多角度关键词搜索，可以拼凑出完整内容。

## 工作流程

### 第一步：确认知识库 ID

如果用户提供了知识库名称（如"国家法律法规数据库"），先用 `search_knowledge_base` 搜索获取 ID：

```bash
# 搜索知识库
ima_api "openapi/wiki/v1/search_knowledge_base" '{"query": "知识库名称", "cursor": "", "limit": 20}'

# 如果已知道识库 ID，直接使用
```

如果用户提供了知识库分享链接，从 URL 中提取 `shareId`，尝试直接用于搜索（可能返回"没有权限"）。

如果提示没有权限，需要用户将机器人添加到知识库。

### 第二步：多关键词轮询搜索

对同一个知识库，用多个相关关键词搜索，收集所有 `highlight_content` 片段。

**搜索函数**（每次搜索调用）：

```bash
search_knowledge() {
  local query="$1"
  local kb_id="$2"
  curl -s -X POST "https://ima.qq.com/openapi/wiki/v1/search_knowledge" \
    -H "ima-openapi-clientid: $IMA_CLIENT_ID" \
    -H "ima-openapi-apikey: $IMA_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"query\": \"$query\", \"knowledge_base_id\": \"$kb_id\", \"cursor\": \"\"}"
}
```

**关键词策略**：
- 核心词：如"生态环境""破坏""保护"
- 同义词：如"环境保护""生态破坏""污染"
- 相关词：如"法律法规""条例""规定"
- 长尾词：如"生态环境破坏""生态保护措施"

**执行示例**（循环搜索多个关键词）：

```bash
for query in "核心词" "同义词" "相关词1" "相关词2"; do
  echo "=== 查询: $query ==="
  search_knowledge "$query" "$KB_ID"
  echo ""
done
```

### 第三步：提取和整理高亮片段

从每次搜索结果中提取：
- `title` — 文件/文档标题
- `highlight_content` — 匹配的高亮片段（已去除 `<em>` 标签）
- `media_id` — 内容 ID

### 第四步：拼凑和组织内容

将收集到的高亮片段按主题分类整理，标注来源。向用户展示时说明：
- 这是从知识库中高亮片段拼凑的内容，可能不完整
- 建议用户核实完整原文

## 完整示例

```bash
# 1. 加载凭证
IMA_CLIENT_ID="${IMA_OPENAPI_CLIENTID:-$(cat ~/.config/ima/client_id 2>/dev/null)}"
IMA_API_KEY="${IMA_OPENAPI_APIKEY:-$(cat ~/.config/ima/api_key 2>/dev/null)}"

KB_ID="CjgilyA9Ci7ea0N2lOrawqAu9d-76lBF_sDUEHJwKvQ="

# 2. 多关键词搜索
for query in "生态环境 破坏" "生态 保护" "环境 污染" "环境保护法" "生态 条例"; do
  echo "=== 查询: $query ==="
  curl -s -X POST "https://ima.qq.com/openapi/wiki/v1/search_knowledge" \
    -H "ima-openapi-clientid: $IMA_CLIENT_ID" \
    -H "ima-openapi-apikey: $IMA_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"query\": \"$query\", \"knowledge_base_id\": \"$KB_ID\", \"cursor\": \"\"}"
  echo ""
done
```

## 注意事项

- 高亮片段是从搜索索引中提取的，**不是完整原文**，长度和完整性有限
- 如果权限不足（返回"没有权限"或"内部错误"），需要用户将机器人添加到知识库
- 如果搜索结果为空，说明知识库中可能没有相关内容
- 每次搜索的 `highlight_content` 可能重复，需要去重整理
