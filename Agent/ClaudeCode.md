# 1.CLI和MCP的区别

|类型|优点|缺点|
| :-- | :-- |:--|
| CLI Command Line Interface（命令行界面）| 1、Token消耗小；2、执行效率高|1、命令拼接灵活，操控更危险|
| MCP Model Context Protocol（模型上下文协议）|1、Token消耗大（需要传入元数据（名称、描述、入参格式等））；2、执行效率低|更可控；更安全|

# 2.Skills的本质：从提示词到组织知识库
Skill不是一个prompt，它是一个文件夹，它代表一种将组织内的隐形经验封装为可复用AI上下文的工程实践

核心设计哲学：
- 单一职责：最好的skill cleanly fit into one category（干净地落入一个类别）；试图覆盖多个领域的 skill 会让 agent 困惑

- 渐进式披露：通过文件系统结构实现上下文工程，而非把所有信息塞进一个文件

- 持续迭代：skill时活的文档，随实际使用中的失败点不断进化

## 2.1 9大Skill分类体系

|类别|核心功能|典型示例|
| :-- | :-- |:--|
| Library&API Reference| 内部库/CLI/SDK的正确用法与陷阱 |billing-lib，platform-cli|
| Product Verification | 驱动运行中的产品进行端到端验证 | signup-driver，checkout-verifier|
| Data&Analysis | 连接数据栈与监控体系 | funnel-query，grafana，datadog|
| Business Automation | 多工具工作流一键化 | standup-post，weekly-recap|
| Scaffolding& Templates | 框架正确的代码脚手架 | new-app，new-migration|
| Code Quality&Review | 代码质量方法论 | adversarial-review，code-style|
| CI/CD&Deploymemt | 安全地提交、推送、部署 | babysit-pr，deploy-service|
| Incident Runbooks | 症状->调查->报告的故障处理 | oncall-runner，queue-debug|
| infrastructure ops| 带安全门的运维操作| orphans，cost-investigation|
