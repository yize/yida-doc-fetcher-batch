---
name: yida-doc-fetcher-batch
description: "批量抓取宜搭文档。对每个文档 URL 调用 yida-doc-fetcher skill 进行处理。"
---

# 宜搭文档批量抓取 Skill

批量从宜搭帮助文档网站抓取所有文档。

## 重要原则

⚠️ **必须逐个处理**：每个文档都必须通过 yida-doc-fetcher 处理，不能直接批量抓取！

## 工作流程

### Step 1: 获取所有文档 URL

```bash
# 从 sitemap 获取所有 yida_support 下的文档
curl -s "https://docs.aliwork.com/sitemap.xml" | grep -o 'yida_support/[^<]*' | sort -u
```

### Step 2: 逐个调用 yida-doc-fetcher

对每个 URL：
1. 调用 yida-doc-fetcher skill 处理
2. 等待处理完成
3. 记录结果
4. 继续下一个

### Step 3: 汇总结果

统计处理成功的文档数、图片数等。

## 实现示例

```bash
#!/bin/bash
# 宜搭文档批量抓取

OUTPUT_DIR="yida-docs"
mkdir -p "$OUTPUT_DIR"

echo "=== 开始批量抓取 ==="
START_TIME=$(date +%s)

# Step 1: 获取所有文档 URL
echo "获取文档列表..."
DOC_URLS=$(curl -s "https://docs.aliwork.com/sitemap.xml" | grep -o 'yida_support/[^<]*' | sort -u)
TOTAL=$(echo "$DOC_URLS" | wc -l)
echo "共找到 $TOTAL 个文档"

# Step 2: 逐个处理
PROCESSED=0
FAILED=0

for DOC_PATH in $DOC_URLS; do
    DOC_ID="${DOC_PATH##*/}"
    
    # ⚠️ 必须调用 yida-doc-fetcher 处理
    echo "[$((PROCESSED+FAILED+1))/$TOTAL] 处理: $DOC_ID"
    
    # 调用 yida-doc-fetcher (模拟，实际需要调用 skill)
    if process_with_yida_doc_fetcher "$DOC_ID"; then
        PROCESSED=$((PROCESSED + 1))
    else
        FAILED=$((FAILED + 1))
    fi
    
    # 每 10 个输出进度
    if [ $(( (PROCESSED+FAILED) % 10 )) -eq 0 ]; then
        echo "--- 进度: $((PROCESSED+FAILED))/$TOTAL (成功: $PROCESSED, 失败: $FAILED) ---"
    fi
done

# Step 3: 汇总结果
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

echo ""
echo "=== 完成 ==="
echo "成功: $PROCESSED"
echo "失败: $FAILED"
echo "耗时: ${DURATION}秒"
echo "输出目录: $OUTPUT_DIR"
```

## 关键约束

1. **必须用 yida-doc-fetcher**：每个文档都要调用 yida-doc-fetcher skill
2. **不能直接批量 curl**：禁止直接 curl 批量抓取页面
3. **顺序处理**：逐个处理，确保每个都被正确转换

## 输出

- 文档：`yida-docs/*.md`
- 图片：`yida-docs/images/`
- 日志：处理过程记录
