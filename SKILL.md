---
name: yida-doc-fetcher-batch
description: "批量抓取宜搭文档。使用 Node.js v3 提取逻辑，支持表格解析、URL清理、串行处理。"
---

# 宜搭文档批量抓取 Skill (优化版)

批量从宜搭帮助文档网站抓取所有文档，保留表格和格式。

## 核心改进 (vs 原版)

| 改进点 | 原版 (bash) | 优化版 (Node.js) |
|--------|-------------|------------------|
| 表格处理 | ❌ 删除所有标签 | ✅ 解析 tr/td 为 Markdown |
| HTML 转换 | 一次性删除标签 | ✅ 递归处理嵌套标签 |
| URL 清理 | 无 | ✅ 去除 `?spm=xxx` 参数 |
| 重试机制 | 无 | ✅ 失败自动重试 |
| 并发控制 | 并行 (乱序) | ✅ 串行处理 |

## 使用方法

```bash
# 创建输出目录
mkdir -p yida-docs-batch/images

# 运行批量抓取
node yida-batch-v4.js
```

## 完整脚本 (yida-batch-v4.js)

```javascript
/**
 * 钉钉宜搭文档批量抓取 - 优化版 v4
 * 改进点：
 * 1. 表格解析 (tr/td → Markdown 表格)
 * 2. 嵌套标签递归处理
 * 3. URL 参数清理
 * 4. 串行处理 + 重试
 */

const https = require('https');
const http = require('http');
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');

const OUTPUT_DIR = './yida-docs-batch';
const IMAGES_DIR = './yida-docs-batch/images';

// 标题翻译映射
const TITLE_MAP = {
    '产品功能介绍': 'product-intro',
    '什么是宜搭': 'what-is-yida',
    '宜搭页面类型': 'page-types',
    '宜搭界面介绍': 'ui-intro',
    '宜搭数据安全': 'data-security',
    '宜搭名词释义': 'glossary',
    '产品计费': 'pricing',
    '快速开始': 'quick-start',
    '表单管理': 'form-management',
    '流程设计': 'workflow-design',
    '集成': 'integration',
    '自动化': 'automation',
    '门户设计': 'portal-design',
    '报表设计': 'report-design',
    '大屏设计': 'dashboard-design',
    '自定义页面': 'custom-page',
    '酷应用': 'cool-apps',
    '平台管理': 'platform-management',
    '应用管理': 'app-management',
    'AI专区': 'ai-zone',
    '插件中心': 'plugins',
    '开发者手册': 'developer-guide',
    '使用案例': 'use-cases',
    '更新日志': 'changelog',
    '用户手册': 'user-guide',
};

// 初始化目录
function init() {
    if (!fs.existsSync(OUTPUT_DIR)) fs.mkdirSync(OUTPUT_DIR, { recursive: true });
    if (!fs.existsSync(IMAGES_DIR)) fs.mkdirSync(IMAGES_DIR, { recursive: true });
}

// HTTP 请求
function httpGet(url, retries = 3) {
    return new Promise((resolve, reject) => {
        const protocol = url.startsWith('https')) ? https : http;
        
        const req = protocol.get(url, (res) => {
            let data = '';
            res.on('data', chunk => data += chunk);
            res.on('end', () => resolve(data));
        });
        
        req.on('error', err => {
            if (retries > 0) {
                setTimeout(() => httpGet(url, retries - 1).then(resolve).catch(reject), 1000);
            } else reject(err);
        });
        req.setTimeout(30000, () => { req.destroy(); reject(new Error('Timeout')); });
    });
}

/**
 * 表格 HTML 转 Markdown
 */
function tableToMarkdown(html) {
    const rows = [];
    const trRegex = /<tr[^>]*>([\s\S]*?)(?=<tr|<\/table|$)/gi;
    
    for (const trMatch of [...html.matchAll(trRegex)]) {
        const cells = [];
        // td
        for (const tdMatch of [...trMatch[1].matchAll(/<td[^>]*>([\s\S]*?)(?=<td|<\/tr|$)/gi)]) {
            cells.push(extractText(tdMatch[1]));
        }
        // th
        if (cells.length === 0) {
            for (const thMatch of [...trMatch[1].matchAll(/<th[^>]*>([\s\S]*?)(?=<th|<\/tr|$)/gi)]) {
                cells.push(extractText(thMatch[1]));
            }
        }
        if (cells.length > 0) rows.push(cells);
    }
    
    if (rows.length === 0) return '';
    
    // 构建 Markdown 表格
    let md = '';
    rows.forEach((row, i) => {
        const cells = row.map(c => c.replace(/\n/g, ' ')).join(' | ');
        md += `| ${cells} |\n`;
        if (i === 0) {
            md += '|' + row.map(() => '---').join('|') + '|\n';
        }
    });
    return md;
}

/**
 * 提取纯文本 (递归处理嵌套标签)
 */
function extractText(html) {
    let text = html;
    
    // 预处理表格
    const hasTable = text.includes('<table');
    let tableMd = '';
    if (hasTable) {
        const tableMatch = text.match(/<table[^>]*>([\s\S]*?)<\/table>/i);
        if (tableMatch) {
            tableMd = tableToMarkdown(tableMatch[1]);
            text = text.replace(/<table[^>]*>[\s\S]*?<\/table>/i, '\n__TABLE__\n');
        }
    }
    
    // 处理嵌套标签
    const tagHandlers = [
        { re: /<script[^>]*>[\s\S]*?<\/script>/gi, replace: '' },
        { re: /<style[^>]*>[\s\S]*?<\/style>/gi, replace: '' },
        { re: /<p[^>]*>([\s\S]*?)<\/p>/gi, replace: '$1\n' },
        { re: /<strong[^>]*>([\s\S]*?)<\/strong>/gi, replace: '**$1**' },
        { re: /<b[^>]*>([\s\S]*?)<\/b>/gi, replace: '**$1**' },
        { re: /<em[^>]*>([\s\S]*?)<\/em>/gi, replace: '*$1*' },
        { re: /<i[^>]*>([\s\S]*?)<\/i>/gi, replace: '*$1*' },
        { re: /<code[^>]*>([^<]+)<\/code>/gi, replace: '`$1`' },
        { re: /<br\s*\/?>/gi, replace: '\n' },
        { re: /<a[^>]+href="([^"]+)"[^>]*>([\s\S]*?)<\/a>/gi, replace: '[$2]($1)' },
    ];
    
    for (let i = 0; i < 3; i++) { // 多次迭代处理嵌套
        for (const h of tagHandlers) {
            text = text.replace(h.re, h.replace);
        }
    }
    
    // 移除剩余标签
    text = text.replace(/<[^>]+>/g, '');
    
    // HTML 实体
    const entities = { '&nbsp;': ' ', '&lt;': '<', '&gt;': '>', '&amp;': '&', '&quot;': '"', '&#39;': "'" };
    for (const [k, v] of Object.entries(entities)) {
        text = text.replace(new RegExp(k, 'g'), v);
    }
    
    // 恢复表格
    text = text.replace('__TABLE__', tableMd);
    
    // 清理
    text = text.replace(/\n{3,}/g, '\n\n').trim();
    return text;
}

/**
 * 处理单个文档
 */
async function processDoc(docPath) {
    const docId = docPath.split('/').pop().split('?')[0]; // 去除 URL 参数
    const url = `https://docs.aliwork.com/docs/yida_support/${docId}`;
    
    console.log(`\n📄 [${docId}] 正在处理...`);
    
    try {
        const html = await httpGet(url);
        
        // 提取标题
        const titleMatch = html.match(/<title[^>]*>([^<]+)<\/title>/);
        const title = titleMatch ? titleMatch[1].replace(/ \|.*/, '').trim() : docId;
        
        // 翻译 slug
        let slug = docId;
        for (const [key, val] of Object.entries(TITLE_MAP)) {
            if (title.includes(key)) { slug = val; break; }
        }
        
        // 提取正文 (yuque-content)
        const contentMatch = html.match(/id="yuque-content"[^>]*>([\s\S]*?)<\/div>/);
        if (!contentMatch) {
            console.log(`  ⚠️ 未找到内容区域`);
            return false;
        }
        
        let content = extractText(contentMatch[1]);
        
        // 移除噪音
        content = content.replace(/跳到主要内容/g, '').replace(/此文档对您是否有帮助[\s\S]*/g, '').replace(/Copyright ©[\s\S]*/g, '');
        
        // 保存
        const filename = `${slug}-${docId}.md`;
        const md = `---\ntitle: ${title}\nurl: ${url}\nsource: docs.aliwork.com\n---\n\n# ${title}\n\n${content}\n`;
        fs.writeFileSync(path.join(OUTPUT_DIR, filename), md);
        
        // 下载图片
        const imgUrls = [...html.matchAll(/yda?[-a-z0-9]*\.[a-z]+\/[-a-z0-9_]+\.(png|jpeg|jpg|webp)/gi)];
        let imgCount = 0;
        for (const [imgUrl] of new Set(imgUrls)) {
            try {
                const ext = imgUrl.match(/\.(png|jpeg|jpg|webp)$/)[0];
                const imgName = `${slug}-${imgCount++}${ext}`;
                await httpGet('https://' + imgUrl).then(d => 
                    fs.writeFileSync(path.join(IMAGES_DIR, imgName), d));
            } catch (e) { /* skip */ }
        }
        
        console.log(`  ✅ ${title} → ${filename} (${imgCount} 图)`);
        return true;
        
    } catch (err) {
        console.log(`  ❌ 失败: ${err.message}`);
        return false;
    }
}

/**
 * 主流程
 */
async function main() {
    init();
    console.log('🔄 获取文档列表...');
    
    // 获取 sitemap
    const sitemap = await httpGet('https://docs.aliwork.com/sitemap.xml');
    const urls = [...new Set([...sitemap.matchAll(/yida_support\/([^<"]+)/g)].map(m => m[1]))];
    
    console.log(`📚 共 ${urls.length} 个文档\n`);
    
    let success = 0, fail = 0;
    for (let i = 0; i < urls.length; i++) {
        const result = await processDoc(urls[i]);
        if (result) success++; else fail++;
        
        if ((i + 1) % 50 === 0) {
            console.log(`\n📊 进度: ${i + 1}/${urls.length} (✅${success} ❌${fail})`);
        }
    }
    
    console.log(`\n🎉 完成! 成功: ${success}, 失败: ${fail}`);
}

main().catch(console.error);
```

## 关键改进说明

### 1. 表格解析
```javascript
// 原来 (bash): 全部删除
content=$(echo "$content" | sed 's/<[^>]*>//g')

// 现在 (Node.js):
function tableToMarkdown(html) {
    const trRegex = /<tr[^>]*>([\s\S]*?)(?=<tr|<\/table|$)/gi;
    // 提取 tr → td → | 表格
}
```

### 2. URL 清理
```javascript
// 去除 ?spm=xxx 参数
const docId = docPath.split('/').pop().split('?')[0];
```

### 3. 串行处理
```javascript
// 逐个处理，避免并发问题
for (let i = 0; i < urls.length; i++) {
    const result = await processDoc(urls[i]); // await 确保顺序
}
```

### 4. 重试机制
```javascript
function httpGet(url, retries = 3) {
    // 失败自动重试 3 次
}
```

## 输出

- 文档: `yida-docs-batch/*.md`
- 图片: `yida-docs-batch/images/`
- 格式: 保留表格、链接、加粗等

## 运行

```bash
cd /root/.openclaw/workspace
node -e "
const https = require('https'), http = require('http'), fs = require('fs'), path = require('path');
// ... 粘贴上面的脚本代码 ...
"
```

或保存为文件后运行:
```bash
node yida-batch-v4.js
```
