---
name: yida-doc-fetcher-batch
description: "批量抓取宜搭文档。对每个文档 URL 调用 yida-doc-fetcher skill 进行处理。"
---

# 宜搭文档批量抓取 Skill

批量从宜搭帮助文档网站抓取所有文档。

## 重要原则

⚠️ **必须逐个处理**：每个文档都必须通过 yida-doc-fetcher 处理！

## 工作流程

### Step 1: 获取所有文档 URL

```bash
curl -s "https://docs.aliwork.com/sitemap.xml" | grep -o 'yida_support/[^<]*' | sort -u
```

### Step 2: 逐个调用 yida-doc-fetcher

对每个 URL：
1. 获取 HTML
2. 提取标题
3. 翻译为 slug
4. 提取正文（用 yuque-content）
5. 下载图片
6. 保存文件

### Step 3: 汇总结果

统计处理成功的文档数、图片数等。

## 实现代码

```bash
#!/bin/bash
OUTPUT_DIR="yida-docs"
mkdir -p "$OUTPUT_DIR/images"

# 标题翻译映射 (完整版)
declare -A TITLE_MAP=(
    # 常用词
    ["产品功能介绍"]="product-intro"
    ["什么是宜搭"]="what-is-yida"
    ["宜搭页面类型"]="page-types"
    ["宜搭界面介绍"]="ui-intro"
    ["宜搭数据安全"]="data-security"
    ["宜搭名词释义"]="glossary"
    ["产品计费"]="pricing"
    ["快速开始"]="quick-start"
    ["表单管理"]="form-management"
    ["流程设计"]="workflow-design"
    ["集成"]="integration"
    ["自动化"]="automation"
    ["门户设计"]="portal-design"
    ["报表设计"]="report-design"
    ["大屏设计"]="dashboard-design"
    ["自定义页面"]="custom-page"
    ["酷应用"]="cool-apps"
    ["平台管理"]="platform-management"
    ["应用管理"]="app-management"
    ["AI专区"]="ai-zone"
    ["插件中心"]="plugins"
    ["国际化"]="international"
    ["联系我们"]="contact-us"
    ["开发者手册"]="developer-guide"
    ["使用案例"]="use-cases"
    ["更新日志"]="changelog"
    ["用户手册"]="user-guide"
    ["基本信息"]="basic-info"
    ["企业域名"]="enterprise-domain"
    ["HTTP连接器"]="http-connector"
    ["流程表单"]="workflow-form"
    ["普通表单"]="standard-form"
    ["表单设计"]="form-design"
    ["应用复制"]="app-copy"
    ["应用分发"]="app-distribution"
    ["上下级组织"]="org-structure"
    ["上下游组织"]="upstream-downstream"
    ["专属宜搭"]="dedicated-yida"
    ["权限矩阵"]="permission-matrix"
    ["企业效能"]="enterprise-efficiency"
    ["计费概述"]="pricing-overview"
    ["增值服务"]="value-added-service"
    ["开发者功能"]="developer-features"
    ["连接器专题"]="connector-topics"
    ["公式专题"]="formula-topics"
    ["聚合表设计"]="aggregated-table"
    ["报表专题"]="report-topics"
    ["表单专题"]="form-topics"
    ["私有化宜搭"]="private-yida"
)

process_doc() {
    local doc_id="$1"
    local url="https://docs.aliwork.com/docs/yida_support/$doc_id"
    
    # 获取 HTML
    html=$(curl -s "$url")
    [[ -z "$html" ]] && echo "❌ $doc_id: 获取失败" && return 1
    
    # 提取标题
    title=$(echo "$html" | grep -o "<title[^>]*>[^<]*</title>" | head -1 | sed 's/<title[^>]*>//;s/<\/title>//' | sed 's/ |.*//' | xargs)
    [[ -z "$title" ]] && title="$doc_id"
    
    # 翻译为 slug
    slug=""
    for key in "${!TITLE_MAP[@]}"; do
        [[ "$title" == *"$key"* ]] && slug="${TITLE_MAP[$key]}" && break
    done
    [[ -z "$slug" ]] && slug="$doc_id"
    slug=$(echo "$slug" | sed 's/\//-/g')
    
    # 提取正文 (用 yuque-content)
    content=$(echo "$html" | grep -A 5000 'id=yuque-content' | head -500)
    
    # 清理 HTML
    content=$(echo "$content" | sed 's/<[^>]*>//g')
    content=$(echo "$content" | sed 's/&nbsp;/ /g; s/&amp;/\&/g; s/&lt;/</g; s/&gt;/>/g; s/&quot;/"/g')
    content=$(echo "$content" | sed 's/[ \t]\+/ /g; /^[[:space:]]*$/d')
    
    # 移除噪音
    content=$(echo "$content" | sed 's/跳到主要内容//g; s/此文档对您是否有帮助.*//g; s/Copyright ©.*//g; s/历史文章内容.*//g; s/首页帮助手册.*用户手册//g')
    
    # 限制长度
    content=$(echo "$content" | head -300)
    
    # 保存
    {
        echo "# $title"
        echo ""
        echo "$content"
    } > "$OUTPUT_DIR/${slug}.md"
    
    # 下载图片
    img_count=0
    for img_url in $(echo "$html" | grep -oE 'yida-support[^"]+\.(png|jpeg|jpg)' | sort -u); do
        img_count=$((img_count + 1))
        printf -v num "%02d" $img_count
        ext="${img_url##*.}"
        curl -sL "https://$img_url" -o "$OUTPUT_DIR/images/${slug}-${num}.$ext" &
    done
    wait
    
    echo "✓ $title -> ${slug}.md ($img_count 图)"
}

# 主流程
DOC_URLS=$(curl -s "https://docs.aliwork.com/sitemap.xml" | grep -o 'yida_support/[^<]*' | sort -u)
TOTAL=$(echo "$DOC_URLS" | wc -l)
echo "共 $TOTAL 个文档"

counter=0
for DOC_PATH in $DOC_URLS; do
    counter=$((counter + 1))
    DOC_ID="${DOC_PATH##*/}"
    process_doc "$DOC_ID"
    
    # 每 10 个汇报
    [ $((counter % 10)) -eq 0 ] && echo "--- 进度: $counter/$TOTAL ---"
done
```

## 关键约束

1. **必须用 yida-doc-fetcher**：每个文档都要调用 yida-doc-fetcher skill
2. **不能直接批量 curl**：禁止直接 curl 批量抓取页面
3. **顺序处理**：逐个处理，确保每个都被正确转换

## 输出

- 文档：`yida-docs/*.md`
- 图片：`yida-docs/images/`
- 日志：处理过程记录
