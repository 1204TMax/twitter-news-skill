---
name: twitter-news
description: Twitter/X 专属资讯快照工作流：按账号池分批抓取新增帖子、批间落盘、支持断点续跑与失败恢复，输出中文可读简报。用于每日推特简报、长任务抓取、需要可审计执行清单的场景。
---

# Twitter News Snapshot（分批+断点版）

## 核心目标
稳定抓取 + 可恢复执行 + 可审计结果。避免“一口气全跑到超时失败”。

## 输入文件
1. 账号池：`/Users/xiaoshan/.openclaw/skills/twitter-news/twitter-following-ai.json`
2. 汇总状态：`/Users/xiaoshan/.openclaw/scene/每日简报/news/state/twitter-digest-state.json`
3. 执行检查点（新增）：`/Users/xiaoshan/.openclaw/scene/每日简报/news/state/twitter-checkpoint.json`

## 强制规则
1. 必须使用 `browser` 作为主抓取工具。
2. 必须遍历账号池 `active=true` 账号。
3. 必须按批执行：**worker 池并发抓取**（默认并发 6，每个 worker 一次处理 1 个账号）。
4. 必须分批提交结果并落盘 checkpoint（每完成一轮 worker 汇总就落盘）。
5. 必须支持断点续跑：从 `nextIndex` 继续。
6. 必须在**同一次任务中循环批次**，直到 `nextIndex >= totalActiveAccounts` 才能结束。
7. 必须输出“账号检查清单”。
8. 不得编造帖子；无数据就明确写无。

## 时间窗口
- 起始：昨天 09:30（Asia/Shanghai）
- 结束：当前执行时刻
- 只保留发布时间在窗口内的帖子。

## 去重
1. URL 去重（status 链接）
2. post_id/hash 去重
3. 与最近 1~3 份简报做标题/主题去重

## 执行流程（必须遵守）

### Step 0：初始化
- 读取账号池、状态文件。
- 加载 checkpoint；若不存在则创建：

```json
{
  "runId": "",
  "startedAt": "",
  "window": {"start": "", "end": ""},
  "concurrency": 6,
  "nextIndex": 0,
  "completedAccounts": [],
  "failedAccounts": [],
  "items": []
}
```

### Step 1：并发 worker 抓取
- 将 active 账号加入任务队列。
- 启动固定 worker 池：**并发 6**（可降到 4；不得高于 8）。
- 每个 worker 一次只处理 1 个账号，流程：
  1) 打开账号页
  2) 抓首屏
  3) 下拉一次再抓
  4) 提取帖子字段：时间、正文、status 链接、互动数据、外链
- worker 完成后立即领取下一个账号，直到队列为空。

### Step 2：单账号容错
- 单账号超时建议：90 秒。
- 出现临时错误（加载失败/执行中断）最多重试 2 次（指数退避）。
- 仍失败：记录到 `failedAccounts`，继续下一个账号，不中止整批。

### Step 3：轮次落盘（关键）
每一轮并发 worker 回收后，立即写 `twitter-checkpoint.json`：
- `nextIndex`（队列消费进度）
- `completedAccounts`
- `failedAccounts`
- 当前累计 `items`
- `lastBatchFinishedAt`
- `concurrency`

并立刻判断：
- 若 `nextIndex < totalActiveAccounts`：继续下一轮 worker（同一任务内继续循环）
- 若 `nextIndex >= totalActiveAccounts`：进入汇总阶段

### Step 4：断点续跑
- 重新执行时先读 checkpoint。
- 若 `nextIndex < totalActiveAccounts`，从该索引继续并循环到完成。
- 若已完成，则直接进入汇总。

### Step 5：汇总与输出
- 对 `items` 做时间窗口筛选 + 去重。
- 按价值排序（重要性 + 互动数据 + 时效性）。
- 按“输出格式要求（强制）”生成完整中文简报。
- 同时输出“账号检查清单”：
  - handle
  - checked（yes/no）
  - inWindowPosts
  - status（ok/failed/skipped）
  - reason（若异常）

### Step 5.5：交付顺序（强制）
1) **先把完整简报发送到当前聊天会话**（不是只写文件）
2) 再落盘到场景目录（Summary/state）
3) 最后回传“已发送+已落盘”的确认

### Step 6：更新最终状态
仅当 `nextIndex >= totalActiveAccounts`（整轮完成）时，才允许更新 `twitter-digest-state.json`：
- `lastRun`
- `lastWindow`
- `accountsChecked`
- `newItemsFound`
- `highlights`

若未完成整轮：
- 禁止写最终 digest state
- 只允许更新 checkpoint，并明确标记 `status: partial`

## 输出格式要求（强制）
- **禁止自由发挥格式**。
- **必须严格按照** ` /Users/xiaoshan/.openclaw/skills/twitter-news/example-output.md ` 的结构、标题层级、字段顺序与书写风格输出。
- 生成简报前必须先读取 `example-output.md`，并以其为唯一模板。

每条资讯必须包含：
- 标题
- 正文
- 来源（可点击 URL）
- 小山判断

附加约束：
- 用中文输出
- 不允许只给摘要不带清单
- 链接使用可点击 URL
- 若模板与其他描述冲突，以 `example-output.md` 为准

## 错误处理策略
- browser evaluate/snapshot 中断：短重试一次；失败则降级到 snapshot 解析。
- 页面要求登录：记录 `login_required`，跳过账号继续。
- 并发异常控制：
  - 若连续 3 个账号失败，自动把并发从 6 降到 4。
  - 若继续失败，再降到 2，并输出 degraded_mode。
- 整体超时：输出“已完成部分 + 未完成账号 + nextIndex”，保留下次续跑点。

## 禁止事项
- 禁止一口气跑完整账号池而不落盘。
- 禁止仅输出摘要不附账号检查清单。
- 禁止在未完成去重时直接产出最终简报。