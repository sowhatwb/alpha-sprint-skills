---
name: knowledge-ingest
description: >
  一键将任意 URL 采集入 Obsidian。自动识别 YouTube 视频、X/Twitter 推文、普通文章/网页，
  提取内容并格式化为带完整 frontmatter 的 Obsidian 笔记，保存到 Clippings 文件夹。

  当用户说"存到 Obsidian"、"采集这个链接"、"保存到笔记"、"ingest"、"clip"，
  或提供一个 URL 并提到 Obsidian、知识库、笔记时，必须触发此 Skill。
  即使用户只是粘贴一个 URL 没有多余说明，也应主动询问是否需要存入 Obsidian。
---

# Knowledge Ingest — 一键采集 URL 到 Obsidian

接收 URL → 识别类型 → 提取内容 → 格式化笔记 → 存入 Obsidian。

---

## Step 1：识别 URL 类型

```bash
URL="<用户提供的 URL>"

if [[ "$URL" =~ youtube\.com/watch || "$URL" =~ youtu\.be/ || "$URL" =~ youtube\.com/shorts ]]; then
    TYPE="youtube"
elif [[ "$URL" =~ x\.com/ || "$URL" =~ twitter\.com/ ]]; then
    TYPE="tweet"
else
    TYPE="article"
fi

echo "📍 识别类型：$TYPE"
```

---

## Step 2：提取内容

### YouTube 视频

使用 `yt-dlp` 提取字幕和元数据：

```bash
# 获取元数据
TITLE=$(yt-dlp --print "%(title)s" "$URL" 2>/dev/null | tr '/' '-' | tr ':' '-')
AUTHOR=$(yt-dlp --print "%(uploader)s" "$URL" 2>/dev/null)
DATE=$(yt-dlp --print "%(upload_date>%Y-%m-%d)s" "$URL" 2>/dev/null)
DURATION=$(yt-dlp --print "%(duration_string)s" "$URL" 2>/dev/null)
VIEW_COUNT=$(yt-dlp --print "%(view_count)s" "$URL" 2>/dev/null)

# 下载字幕（优先中文，其次英文）
yt-dlp --write-sub --sub-langs "zh-Hans,zh,en" --skip-download -o "/tmp/ki_temp" "$URL" 2>/dev/null \
  || yt-dlp --write-auto-sub --sub-langs "zh-Hans,zh,en" --skip-download -o "/tmp/ki_temp" "$URL" 2>/dev/null

# 将 VTT 转为干净文本（去重）
VTT_FILE=$(ls /tmp/ki_temp.*.vtt 2>/dev/null | head -1)
if [ -n "$VTT_FILE" ]; then
    CONTENT=$(python3 -c "
import re, sys
seen = set()
result = []
with open(sys.argv[1]) as f:
    for line in f:
        line = line.strip()
        if line and '-->' not in line and not any(line.startswith(h) for h in ['WEBVTT','Kind:','Language:']):
            clean = re.sub('<[^>]*>', '', line)
            clean = clean.replace('&amp;','&').replace('&gt;','>').replace('&lt;','<')
            if clean and clean not in seen:
                result.append(clean)
                seen.add(clean)
print('\n'.join(result))
" "$VTT_FILE")
    rm -f /tmp/ki_temp.*.vtt
else
    CONTENT="（字幕不可用）"
fi
```

### X / Twitter 推文

使用 `xreach` CLI（已通过 `xreach auth extract --browser chrome` 认证）：

```bash
TWEET_JSON=$(xreach tweet "$URL" --json 2>/dev/null)
if [ $? -ne 0 ]; then
    echo "⚠️  xreach 未认证，请先运行：xreach auth extract --browser chrome"
    exit 1
fi

# 解析推文字段
TITLE=$(python3 -c "import json,sys; d=json.load(sys.stdin); print(d['text'][:60].replace('\n',' '))" <<< "$TWEET_JSON")
AUTHOR=$(python3 -c "import json,sys; d=json.load(sys.stdin); print(d['user']['name'] + ' (@' + d['user']['screenName'] + ')')" <<< "$TWEET_JSON")
DATE=$(python3 -c "
import json, sys
from datetime import datetime
d = json.load(sys.stdin)
dt = datetime.strptime(d['createdAt'], '%a %b %d %H:%M:%S +0000 %Y')
print(dt.strftime('%Y-%m-%d'))
" <<< "$TWEET_JSON")
CONTENT=$(python3 -c "import json,sys; print(json.load(sys.stdin)['text'])" <<< "$TWEET_JSON")
LIKES=$(python3 -c "import json,sys; print(json.load(sys.stdin).get('likeCount',0))" <<< "$TWEET_JSON")
RETWEETS=$(python3 -c "import json,sys; print(json.load(sys.stdin).get('retweetCount',0))" <<< "$TWEET_JSON")
VIEWS=$(python3 -c "import json,sys; print(json.load(sys.stdin).get('viewCount',0))" <<< "$TWEET_JSON")
BOOKMARKS=$(python3 -c "import json,sys; print(json.load(sys.stdin).get('bookmarkCount',0))" <<< "$TWEET_JSON")
```

### 文章 / 网页

按优先级尝试以下工具：

```bash
DATE=$(date +%Y-%m-%d)

if command -v defuddle &>/dev/null; then
    # 首选：defuddle（安装：npm install -g defuddle-cli）
    CONTENT=$(defuddle parse "$URL" --md 2>/dev/null)
    TITLE=$(defuddle parse "$URL" -p title 2>/dev/null)
    AUTHOR=$(defuddle parse "$URL" -p author 2>/dev/null)
    RAW_DATE=$(defuddle parse "$URL" -p published 2>/dev/null)
    [ -n "$RAW_DATE" ] && DATE="${RAW_DATE:0:10}"

elif command -v trafilatura &>/dev/null; then
    # 次选：trafilatura
    META=$(trafilatura --URL "$URL" --json 2>/dev/null)
    TITLE=$(python3 -c "import json,sys; print(json.load(sys.stdin).get('title',''))" <<< "$META")
    AUTHOR=$(python3 -c "import json,sys; print(json.load(sys.stdin).get('author',''))" <<< "$META")
    RAW_DATE=$(python3 -c "import json,sys; print((json.load(sys.stdin).get('date') or '')[:10])" <<< "$META")
    [ -n "$RAW_DATE" ] && DATE="$RAW_DATE"
    CONTENT=$(trafilatura --URL "$URL" --output-format txt --no-comments 2>/dev/null)

else
    # 兜底：Jina Reader（无需安装，网络可用即可）
    # 注意：不要加 -H "Accept: text/markdown"，否则返回空
    RAW=$(curl -s --max-time 20 "https://r.jina.ai/$URL" 2>/dev/null)
    # Jina Reader 返回格式：第一行 "Title: <title>"，后续为正文
    TITLE=$(echo "$RAW" | grep -m1 "^Title:" | sed 's/^Title: //')
    [ -z "$TITLE" ] && TITLE=$(echo "$RAW" | grep -m1 "^# " | sed 's/^# //')
    [ -z "$TITLE" ] && TITLE=$(echo "$URL" | sed 's|.*/||' | sed 's/[?#].*//')
    AUTHOR=$(echo "$RAW" | grep -m1 "^Author:" | sed 's/^Author: //')
    RAW_DATE=$(echo "$RAW" | grep -m1 "^Published Time:" | sed 's/^Published Time: //' | cut -c1-10)
    [ -n "$RAW_DATE" ] && DATE="$RAW_DATE"
    # 去掉 Jina Reader 的 header 元数据行，只保留正文
    CONTENT=$(echo "$RAW" | sed '1,/^Markdown Content:/d')
fi

[ -z "$TITLE" ] && TITLE="$(date '+%Y-%m-%d') 采集"
```

---

## Step 3：格式化 Obsidian 笔记

根据类型生成不同的 frontmatter 和正文结构。

### YouTube 格式

```markdown
---
source: <URL>
author: "<AUTHOR>"
date: <DATE>
type: clipping
content_type: youtube
duration: "<DURATION>"
views: <VIEW_COUNT>
tags:
  - clipping
  - youtube
---

# <TITLE>

## 字幕内容

<CONTENT>

---
**频道：** <AUTHOR>
**时长：** <DURATION> | **播放量：** <VIEW_COUNT>
**链接：** <URL>
```

### 推文格式

```markdown
---
source: <URL>
author: "<AUTHOR>"
date: <DATE>
type: clipping
content_type: tweet
likes: <LIKES>
retweets: <RETWEETS>
views: <VIEWS>
tags:
  - clipping
  - tweet
---

# <TITLE（推文前60字）>

<CONTENT>

---
**作者：** <AUTHOR>
**互动：** 👁️ <VIEWS> · ❤️ <LIKES> · 🔁 <RETWEETS> · 🔖 <BOOKMARKS>
**链接：** <URL>
```

### 文章格式

```markdown
---
source: <URL>
author: "<AUTHOR>"
date: <DATE>
type: clipping
content_type: article
tags:
  - clipping
  - article
---

# <TITLE>

<CONTENT>

---
**作者：** <AUTHOR>
**链接：** <URL>
**采集时间：** <TODAY>
```

---

## Step 4：发现 Vault 并保存

### 自动发现 Vault

```bash
VAULT=$(python3 -c "
import json, os, sys
config = os.path.expanduser('~/Library/Application Support/obsidian/obsidian.json')
try:
    with open(config) as f:
        d = json.load(f)
    vaults = [v.get('path','') for v in d.get('vaults',{}).values()
              if v.get('path') and 'Sandbox' not in v.get('path','')]
    if vaults:
        print(vaults[0])
except:
    pass
" 2>/dev/null)

if [ -z "$VAULT" ]; then
    echo "⚠️  未找到 Obsidian Vault，请提供路径："
    # 询问用户
fi
```

如果用户有多个 Vault（非 Sandbox），询问存入哪个。

### 写入文件

```bash
CLIPPINGS="$VAULT/Clippings"
mkdir -p "$CLIPPINGS"

# 清理文件名（去除特殊字符，限制长度）
SAFE_NAME=$(echo "$TITLE" | tr '/' '-' | tr ':' '-' | tr '?' '' | tr '"' '' | tr '<>' '' | tr '|' '-' | cut -c1-80 | sed 's/[[:space:]]*$//')
FILEPATH="$CLIPPINGS/${SAFE_NAME}.md"

# 写入笔记内容
cat > "$FILEPATH" << 'NOTEEOF'
<格式化后的笔记内容>
NOTEEOF

echo "✅ 已存入 Obsidian"
echo "   📁 $FILEPATH"
```

---

## 完成后汇报

存入成功后，始终输出以下摘要：

```
✅ 已存入 Obsidian

📌 标题：<TITLE>
👤 作者：<AUTHOR>
📅 日期：<DATE>
🏷️  类型：<youtube / tweet / article>
📁 路径：Clippings/<FILENAME>.md
```

---

## 错误处理

| 情况 | 处理方式 |
|------|---------|
| xreach 未认证 | 提示运行 `xreach auth extract --browser chrome` |
| yt-dlp 未安装 | 提示运行 `brew install yt-dlp` |
| defuddle 未安装 | 自动降级到 Jina Reader，提示可安装 `npm install -g defuddle-cli` 以获取更好效果 |
| 文章提取为空 | 告知用户可能是付费墙或需要登录，建议用 `baoyu-url-to-markdown` skill（支持 Chrome CDP）|
| 找不到 Vault | 询问用户 Vault 路径 |
| 文件名冲突 | 在文件名末尾追加时间戳 `_YYYYMMDD-HHMMSS` |

---

## 依赖工具

| 工具 | 用途 | 安装 |
|------|------|------|
| `yt-dlp` | YouTube 字幕 | `brew install yt-dlp` ✅ 已安装 |
| `xreach` | X/Twitter 推文 | 随 agent-reach 安装 ✅ 已安装 |
| `defuddle` | 文章提取（首选） | `npm install -g defuddle-cli` |
| Jina Reader | 文章提取（兜底） | 无需安装，网络请求 ✅ |
| Python 3 | 数据处理 | 系统自带 ✅ |
