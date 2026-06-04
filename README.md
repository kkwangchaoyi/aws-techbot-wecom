# AWS TechBot with WeCom (企业微信)

基于 [aws-samples/sample-aws-techbot](https://github.com/aws-samples/sample-aws-techbot) 改造，将飞书(Feishu)对接替换为**企业微信(WeCom)**。

## 架构

- **Amazon Bedrock AgentCore Runtime** — 运行 AI 助手容器
- **AgentCore Gateway (MCP)** — 统一工具接入（全球文档、中国区文档、定价、客户案例、Kiro）
- **API Gateway + Lambda** — 接收企业微信回调消息
- **DynamoDB** — 消息去重（防止企业微信重试推送导致重复回复）

## 功能特性

- ✅ 企业微信私聊对话
- ✅ 群聊 @机器人 触发回复
- ✅ Markdown 格式消息（自动适配企业微信支持的语法）
- ✅ 消息去重（DynamoDB + TTL 自动清理）
- ✅ AgentCore Memory 多轮对话记忆
- ✅ AES 消息加解密（使用 openssl，无需额外 Layer）

## 一键部署

> ⚠️ 仅支持 **us-west-2 (Oregon)** 区域

上传 `deploy/template-wecom.yaml` 文件到 CloudFormation 进行部署。

## 参数说明

| 参数 | 说明 |
|------|------|
| ProjectName | 项目名称前缀（小写字母，默认 techbot） |
| ModelId | AI 模型选择（GLM-5 / MiniMax M2.5） |
| WeComCorpId | 企业微信企业ID |
| WeComSecret | 应用 Secret |
| WeComAgentId | 应用 AgentId |
| WeComToken | 回调验证 Token |
| WeComEncodingAESKey | 回调加密密钥（43位） |

## 部署后配置

1. 从 CloudFormation Outputs 复制 `WeComCallbackUrl`
2. 企业微信管理后台 → 应用管理 → 自建应用 → 接收消息 → 设置API接收
3. 填入 URL、Token、EncodingAESKey
4. 在「企业可信IP」中添加 Lambda 出口 IP（或配置 VPC + NAT Gateway 固定IP）

## 注意事项

- 企业微信要求配置**企业可信IP**，Lambda 出口IP可能变化，建议使用 VPC + NAT Gateway 方案
- 如需与已有飞书版共存（同区域），使用不同的 `ProjectName` 即可
- 模板大小超过 51.2KB，需通过控制台「Upload a template file」方式部署

## License

MIT-0 (基于原始项目 License)
