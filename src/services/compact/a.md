每次回答需要模型输出摘要


用户消息 → [第1层：微压缩] → [第2层：自动压缩] → API 调用
              ↓                    ↓
         细粒度清理旧工具输出    上下文即将超限时触发
         (不丢语义，<1ms)       (调用 LLM 或 Session Memory)
                                   ↓
                    ┌──────────────┴──────────────┐
              [第4层：SM压缩]              [第3层：传统压缩]
              (用已有摘要，<10ms)          (Fork Agent 生成摘要，5-30s)



涉及的源文件
文件	行数	职责
microCompact.ts	~400	微压缩：规则清理旧工具结果
不调用 LLM，纯规则操作——清理旧的、大块的工具输出结果，保留语义信息。这是每轮查询前都会执行的最轻量操作
可压缩工具白名单：
const COMPACTABLE_TOOLS = new Set([
  'Read',      // 文件读取结果可能很大
  'Bash',      // Shell 输出可能很长
  'Grep',      // 搜索结果
  'Glob',      // 文件列表
  'WebSearch', // 网页搜索结果
  'WebFetch',  // 网页抓取结果
  'Edit',      // 文件编辑的 diff
  'Write',     // 文件写入确认
])
压缩触发点： 
1. 触发条件：距上次助手消息的时间间隔超过阈值（API 缓存已过期）
执行逻辑：
1. 收集所有可压缩工具的 tool_use ID
2. 保留最近 N 个工具结果
3. 将更早的 tool_result 内容替换为：
   "[Old tool result content cleared]"
4. 不修改 tool_use 块（保持 API 配对完整性）

特点：
- 缓存已过期，所以无需保护缓存
- 直接修改本地消息内容
- 减少重传时的 token 消耗

2. 触发条件： 特性开关 CACHED_MICROCOMPACT 开启，模型支持缓存编辑 API，当前是主线程查询（非 fork agent）

执行逻辑：
1. collectCompactableToolIds(): 收集所有可压缩的 tool_use ID
2. registerToolResult(): 注册每个工具结果（按用户消息分组）
3. registerToolMessage(): 记录工具消息组
4. getToolResultsToDelete(): 根据 count/keep 阈值决定删除哪些
5. createCacheEditsBlock(): 生成 cache_edits API 块

关键区别：
- 不修改本地消息内容！
- 通过 API 的 cache_edits 字段告诉服务端删除特定工具结果的缓存
- 保持 prompt cache 命中率
- 状态通过 pendingCacheEdits / pinnedCacheEdits 管理



apiMicrocompact.ts	—	API 层缓存编辑集成
timeBasedMCConfig.ts	—	时间触发微压缩配置
autoCompact.ts	~350	自动压缩：阈值判断 + 断路器
2w个token给执行压缩使用
// 举例（Opus 200K 上下文）：
// 有效窗口 = 200,000 - 20,000 = 180,000
<!-- 自动压缩阈值 = 有效上下文窗口 - 13K 缓冲 -->
// 自动压缩阈值 = 180,000 - 13,000 = 167,000 tokens


如果上下空间加输出摘要满了的话，需要提前触发上下文压缩




compact.ts	~600+	传统压缩：Fork Agent 摘要
prompt.ts	~375	压缩提示词模板
核心机制：Fork Agent
传统压缩使用一个 Fork Agent——创建当前会话的一个分支，让它生成摘要。关键优势是共享主会话的 prompt cache。



主agent loop    压缩模块       fork子查询maxturn=1      api

        触发压缩
        -------->
                    runforkedagent
                    querySource=compact
                    ------------>
                                             全部历史消息+压缩提示词
                                            ------------->



                                             analysis + summary           
                                            <---------------
                        只取summary
                     <--------------
        插入compactBoundaryMessage
        <---------      
                 
边界前消息全丢只保留摘要


sessionMemoryCompact.ts	~630	Session Memory 压缩路径
不调用 LLM 生成新摘要，而是直接使用已经通过后台记忆提取（extractMemories）积累的 Session Memory 作为"摘要"。




grouping.ts	~63	消息按 API 轮次分组
postCompactCleanup.ts	—	压缩后清理
compactWarningHook.ts	—	压缩警告钩子
compactWarningState.ts	—	压缩警告状态









skill的 四级优先级压缩，inline和fork模式 frontmatter配置

P0（最高）
system prompt
当前任务指令
核心约束

👉 绝对不能丢

🥈 P1
当前 skill 的输入
用户当前问题
当前执行上下文

👉 尽量保留

🥉 P2
历史对话（近期）
中间推理结果
相关上下文

👉 可以压缩（summarize）

🪨 P3（最低）
很久之前的对话
无关上下文
冗余信息

👉 优先删除


skill 执行方式
inline 在当前上下文里直接执行：共享当前 context

fork  开一个“子上下文”（子 Agent） 独立 context

frontmatter 配置：  

---
name: summarize_code
mode: fork
priority: P1
max_tokens: 2000
compression: aggressive
---

frontmatter = “这个 skill 在系统里的运行规则”

