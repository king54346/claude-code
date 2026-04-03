每次回答需要模型输出摘要
如果上下空间加输出摘要满了的话，需要提前触发上下文压缩
2w个token给执行压缩使用

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