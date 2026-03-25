# IMA Knowledge Extract

IMA 知识库内容深度抓取技能。通过多关键词轮询搜索并提取高亮片段（highlight_content），从知识库中拼凑完整内容。

## 功能

- 在 IMA 知识库中搜索相关内容
- 多关键词轮询搜索，收集高亮片段
- 提取并整理 `highlight_content` 片段
- 支持知识库 ID 查询和权限验证

## 使用场景

- "帮我搜一下知识库里XXX相关内容"
- "在知识库里找XXX"
- "扒出知识库里关于XXX的内容"
- "整理知识库中XXX的信息"

## 核心原理

IMA 知识库的 `search_knowledge` 接口会返回 `highlight_content` 字段，这是文档中被关键词匹配的片段。通过多轮、多角度关键词搜索，可以拼凑出完整内容。

## 工作流程

1. **确认知识库 ID** — 通过 `search_knowledge_base` 接口搜索或从分享链接提取
2. **多关键词轮询搜索** — 使用核心词、同义词、相关词、长尾词进行多轮搜索
3. **提取高亮片段** — 收集每次搜索的 `title`、`highlight_content`、`media_id`
4. **拼凑整理内容** — 按主题分类整理，标注来源

## 注意事项

- 高亮片段是从搜索索引中提取的，**不是完整原文**
- 如果返回"没有权限"，需要用户将机器人添加到知识库
- 多次搜索的 `highlight_content` 可能重复，需要去重整理

## 文件结构

```
ima-knowledge-extract/
├── SKILL.md                    # Skill 配置文件
└── references/
    └── search-patterns.md      # 搜索模式参考
```

## License

MIT
