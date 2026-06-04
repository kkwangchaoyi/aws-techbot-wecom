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
4. 配置企业可信IP（见下方说明）

## 企业可信IP 配置说明

企业微信要求自建应用调用 API 时，请求来源 IP 必须在「企业可信IP」白名单中，否则会返回错误码 `60020`（not allow to access from your ip）。

### 问题原因

Worker Lambda 通过企业微信 API（`qyapi.weixin.qq.com`）发送消息时，使用的是 AWS Lambda 的公网出口 IP。该 IP 由 AWS 动态分配，**每次冷启动可能变化**，因此无法简单地添加一个固定 IP。

### 解决方案

#### 方案一：手动添加 IP（仅测试用）

1. 发送一条消息触发 Worker Lambda
2. 在 CloudWatch Logs（`/aws/lambda/<ProjectName>-worker`）中找到错误日志：
   ```
   Send message error: {'errcode': 60020, ..., from ip: xx.xx.xx.xx, ...}
   ```
3. 复制 `from ip` 后面的 IP 地址
4. 企业微信管理后台 → 应用管理 → 自建应用 → **企业可信IP** → 添加该 IP
5. 再次发送消息验证

> ⚠️ Lambda 冷启动后 IP 可能变化，需要重复上述步骤添加新 IP。仅适合快速验证，不适合生产环境。

#### 方案二：VPC + NAT Gateway（推荐生产方案）

为 Worker Lambda 配置 VPC + NAT Gateway，使所有出站流量经过固定的 Elastic IP：

1. 创建 VPC + 私有子网 + 公有子网
2. 创建 NAT Gateway（分配 Elastic IP）
3. 私有子网路由表指向 NAT Gateway
4. Worker Lambda 配置 VpcConfig 连接到私有子网
5. 将 NAT Gateway 的 Elastic IP 添加到企业可信IP

**成本**：NAT Gateway 约 $0.045/小时（≈ $32/月）+ 数据处理费

#### 方案三：使用 Lambda 函数 URL + CloudFront（替代方案）

如果不想使用 NAT Gateway，也可以考虑通过代理服务（如部署在 EC2 上的转发代理）固定出口 IP，但实现更复杂。

### 配置入口

企业微信管理后台 → **应用管理** → 选择你的自建应用 → 向下滚动找到 **「企业可信IP」** → 配置

## 注意事项

- 企业微信**不允许**设置 `0.0.0.0/0` 为可信IP
- 如需与已有飞书版共存（同区域），使用不同的 `ProjectName` 即可
- 模板大小超过 51.2KB，需通过控制台「Upload a template file」方式部署

## License

MIT-0 (基于原始项目 License)
