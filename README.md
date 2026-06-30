# 商家退保 AI 审核系统

基于通义千问多模态大模型的商家退保材料智能审核系统。通过 4-Sub-Agent 流水线实现营业执照注销证明与退款申请函的自动化审核、工商信息比对、置信度评估和通知触达。

## 项目结构

```
merchant-withdrawal-review/
├── index.html          # 前端单文件 (Tab 决策树 + 材料提交 + 人工审核 + 通知 + 汇总)
├── server.py           # Python HTTP 后端 (端口 8766, 静态服务 + AI/QCC 代理)
├── start.sh            # 一键启动脚本
├── README.md           # 项目说明
├── CHANGELOG.md        # 版本更新日志
├── .gitignore          # Git 忽略规则
└── docs/
    └── architecture.md # 系统架构详解
```

## 架构概览

```
用户上传材料 → Agent 1 (AI材料审核员) → Agent 2 (置信度评估) → Agent 3 (工商信息核对) → Agent 4 (消息触达)
    🟢 视觉模型           🔴 文本模型            🟡 企查查MCP           🟣 后台静默
```

| Agent | 颜色 | 名称 | 模型 | 说明 |
|-------|------|------|------|------|
| Agent 1 | 🟢 绿 | AI材料审核员 | `qwen-vl-max` | 视觉模型分析上传图片，审查公章、签名、日期 |
| Agent 2 | 🔴 红 | 审核结果评估专家 | `qwen-turbo` | 3维度置信度评分，低分自动重试 |
| Agent 3 | 🟡 黄 | AI工商信息核对员 | 企查查 MCP | 查询工商登记状态，4场景比对 |
| Agent 4 | 🟣 紫 | 消息触达机器人 | `qwen-turbo` | 后台生成飞书/企微/钉钉通知模板 |

## 快速启动

```bash
# 方式一：直接启动
cd merchant-withdrawal-review
python3 server.py

# 方式二：后台启动
bash start.sh

# 访问
open http://localhost:8766
```

> **首次使用**：点击页面右上角 ⚙️ 设置图标，配置通义千问 API Key。

## 功能模块

### Tab 1 — 场景决策树
4 维度决策树（绑卡状态 × 商家类型 × 经营资质 × 对公卡状态），自动判断是否需要人工审核。5 种基础场景 × 2 种商家类型 = 完整材料清单自动匹配。

### Tab 2 — 商家提交 & AI 预审
- 商家信息填写（名称、信用代码、收款账户、资质状态）
- 双文件上传（营业执照注销证明 + 退款申请函，JPG/PNG/PDF）
- 4 Agent 流水线并发执行，实时展示各 Agent 状态与结果
- AI 工商信息 4 场景比对（绿/红结论卡片）

### Tab 3 — 人工审核
- 展示 AI 审核意见 + 维度置信度评分
- 企查查工商数据与商家填报的对比结果
- 一键通过 / 驳回，结果自动进入当日汇总

### Tab 4 — 流程失败通知
飞书卡片样式通知预览，根据审核结果动态生成通过/驳回通知模板。

### Tab 5 — 当日审批汇总
每日 22:00 自动触发 AI 汇总建议，展示当日审核统计 + 各案例快速回顾。

## 技术栈

| 层级 | 技术 |
|------|------|
| 前端 | 原生 HTML/CSS/JS (无框架依赖) |
| 后端 | Python 3 + `http.server` 标准库 |
| AI 视觉 | 通义千问 `qwen-vl-max` (DashScope API) |
| AI 文本 | 通义千问 `qwen-turbo/plus/max` |
| 工商查询 | 企查查 MCP (JSON-RPC over SSE) |
| 部署 | 单端口 8766 静态文件 + API 代理 |

## API Endpoints

| 端点 | 方法 | 说明 |
|------|------|------|
| `/` | GET | 静态文件服务 (index.html) |
| `/api/chat` | POST | 通义千问 API 代理 |
| `/api/qcc/query` | POST | 企查查 MCP 工商查询代理 |

## 文档

- [系统架构详解](docs/architecture.md)
- [版本更新日志](CHANGELOG.md)
- [WorkBuddy Skill 定义](.workbuddy/skills/merchant-withdrawal-review/SKILL.md)
