---
name: twitter-news
description: Twitter/X 专属资讯快照工作流：遍历固定账号池抓取新增帖子，输出中文可读简报。
---

# Twitter News Snapshot

## 核心目标
给用户提供**有价值、可决策**的资讯快报。

---

## 输入与范围
1. 账号池：`/Users/xiaoshan/.openclaw/skills/twitter-news/twitter-following-ai.json`
2. 状态：`/Users/xiaoshan/.openclaw/skills/twitter-news/twitter-digest-state.json`
3. **时间窗口：昨天 09:30 → 当前时间（Asia/Shanghai）**

---

## ⚠️ 强制工具要求

### 必须使用 browser 工具 + profile=默认

**这是唯一正确的抓取方式**：

```
browser action=open profile=默认 targetUrl=https://x.com/OpenAI
browser action=snapshot profile= 默认 refs=aria
```

**禁止使用的替代方案**：
- ❌ `web_search` - 只能用于补充验证，不能作为主要抓取方式
- ❌ `web_fetch` - 会被 Twitter 登录墙拦截
- ❌ `canvas` - 这是 UI 展示工具，不是抓取工具
- ❌ `browser profile=chrome` - 需要 Chrome 扩展，不可用

---

## ⚠️ 时间窗口规则（重要）

### 窗口定义
- **起始**：昨天00:00
- **结束**：当前时间（执行时刻）

### 示例
- 上一次执行是：2026-03-02 000;00
- 今天是 2026-03-03 10:00
- 窗口：2026-03-02 00:00 → 2026-03-03 10:00

### 筛选逻辑
```bash
# 获取时间边界
YESTERDAY=$(date -v-1d "+%Y-%m-%d")  # 昨天
TODAY=$(date "+%Y-%m-%d")             # 今天
START_TIME="${YESTERDAY}T09:30:00+08:00"
END_TIME=$(date -Iseconds)            # 当前时间
```

**只保留发布时间在 [START_TIME, END_TIME] 范围内的帖子**

---

## ⚠️ 去重规则（必须执行）

### Step 0: 读取昨天的简报，排除已报道内容

**在开始抓取之前，必须先读取昨天的简报文件**：

```bash
# 构建昨天简报文件路径
YESTERDAY=$(date -v-1d "+%Y-%m-%d")
YESTERDAY_BRIEF="/Users/dashan/.openclaw/workspace/news/${YESTERDAY}-资讯速递.md"

# 读取前三天的简报（如果存在）
read /Users/xiaoshan/.openclaw/scene/每日简报/news/Summary/YYYY-MM-DD-资讯速递.md(读取前三天的）

# 提取之前已报道的完整内容。
# 在今天抓取时排除完全一致的内容
```

### 去重维度
1. **标题相似度**：如果今天抓取的帖子标题与昨天简报中的标题高度相似（>80%），排除
2. **URL 去重**：如果帖子链接已出现在昨天的简报中，排除
3. **主题去重**：如果昨天已报道过同一事件/话题，今天只报道新的进展

---

## 强制规则
1. **遍历账号池**：必须访问每个账号的时间线
2. **时间窗口筛选**：只保留窗口内发布的帖子
3. **去重**：排除昨天已报道的内容
4. **互动数据评分**：replies/reposts/likes/views 纳入价值评估
5. **外链阅读**：帖子含外链时，打开阅读后再总结
6. **中文输出**：可读、少术语；术语需解释

## 内容格式要求

每条必须有：
- 标题：
- 正文：
- 来源：`<可点击URL>`
- 小山判断：

---

## 执行步骤（完整版）

### Step 0: 前置准备

```bash
# 1. 确保浏览器启动
browser action=status profile=默认
# 如果未运行：
browser action=start profile=默认

# 2. 获取日期
date "+%Y-%m-%d"           # 今天
date -v-1d "+%Y-%m-%d"     # 昨天

# 3. 读取昨天简报（去重用）
YESTERDAY=$(date -v-1d "+%Y-%m-%d")
read file_path=/Users/dashan/.openclaw/workspace/news/${YESTERDAY}-资讯速递.md

# 4. 读取账号池
read file_path=/Users/dashan/.openclaw/workspace/news/sources/twitter-following-ai.json

# 5. 读取状态文件
read file_path=/Users/dashan/.openclaw/workspace/news/state/twitter-digest-state.json
```

### Step 1: 遍历账号抓取

对账号池中的每个账号：

```bash
# 1. 打开账号主页
browser action=open profile=openclaw targetUrl=https://x.com/{handle}

# 2. 等待页面加载，获取快照
browser action=snapshot profile=openclaw refs=aria

# 3. 从快照中提取帖子信息：
#    - 帖子文本内容
#    - 发布时间（必须解析）
#    - 互动数据（likes/reposts/replies/views）
#    - 外链（如有）

# 4. 滚动加载更多（如首屏帖子数量不足）
browser action=act profile=openclaw request='{"kind":"evaluate","fn":"window.scrollTo(0, document.body.scrollHeight)"}'

# 5. 再次获取快照
browser action=snapshot profile=openclaw refs=aria
```

### Step 2: 时间窗口筛选

```bash
# 时间边界
START="昨天 09:30:00"  # 如 2026-03-02 09:30:00
END="当前时间"        # 如 2026-03-03 10:15:00

# 对每个帖子，检查发布时间
if [ 帖子时间 >= START ] && [ 帖子时间 <= END ]; then
    保留该帖子
else
    排除该帖子
fi
```

### Step 3: 去重筛选

```bash
# 对每个候选帖子，检查是否在昨天简报中出现过
if 帖子标题/主题 已在昨天简报中; then
    排除该帖子
fi

# 检查状态文件中的 dedupeHashes
if 帖子哈希 在 dedupeHashes 中; then
    排除该帖子
fi
```

### Step 4: 打开外链阅读

对含有外链的帖子：
```bash
browser action=open profile=openclaw targetUrl={外链URL}
browser action=snapshot profile=openclaw refs=aria
# 阅读内容，补充信息
```

### Step 5: 价值排序

评分维度：
- 重要性（重大发布/政策变化/行业影响）
- 互动数据（likes/reposts/replies/views）
- 时效性（越新越优先）

### Step 6: 生成简报

- 阅读/Users/xiaoshan/.openclaw/skills/twitter-news/example-output.md
- 参考这个格式，输出今日的推特简报。

### Step 7: 更新状态文件

更新 `twitter-digest-state.json`：
- `lastRun`: 当前时间
- `timeWindow`: 本次窗口的 start/end
- `accountsChecked`: 已检查的账号列表
- `postsFound`: 发现的帖子数
- 更新各账号的 `lastSeenPostTime`
- 新增 `dedupeHashes`

---

## 返回格式

```json
{
  "type": "简报结果",
  "source": "推特",
  "content": "简报文本（Markdown格式）",
  "window": {
    "start": "2026-03-02T09:30:00+08:00",
    "end": "2026-03-03T10:15:00+08:00"
  },
  "accountsChecked": ["OpenAI", "AnthropicAI", ...],
  "postsFound": 12,
  "dedupeHashes": ["hash1", "hash2", ...],
  "status": "success|failed",
  "error": "如果失败，错误信息"
}
```

---

## 错误处理

| 场景 | 处理方式 |
|------|---------|
| 浏览器启动失败 | 重试一次，仍失败则返回错误 |
| 页面加载超时 | 等待5秒重试，跳过当前账号 |
| 需要登录 | 跳过当前账号，记录警告 |
| 无新帖子 | 返回空结果，不编造 |