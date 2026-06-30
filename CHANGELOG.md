# Changelog

所有版本更新记录。版本号格式：`vYYYYMMDD` + 字母后缀。

---

## v20260630f — GitHub 打包发布

### 新增
- 完整项目仓库结构（README / CHANGELOG / docs / .gitignore）
- 项目级 WorkBuddy Skill 定义

### 修复
- 企查查空 apiKey 导致 401（`dict.get()` 空字符串陷阱）
- 工商比对卡片被材料审核结果覆盖（DOM 写入顺序）
- Agent 4 消息触达机器人 UI 隐藏

---

## v20260630e — 四场景工商状态 AI 复核结论

### 新增
- `displayBusinessQueryResult()` 四场景比对逻辑：
  - 场景 1: 填报已注销 + 实际已注销 → 🟢 一致
  - 场景 2: 填报未注销 + 实际未注销 → 🟢 一致
  - 场景 3: 填报已注销 + 实际未注销 → 🔴 不一致（纠错提示）
  - 场景 4: 填报未注销 + 实际已注销 → 🔴 不一致（纠错提示）
- AI 复核结论卡片（绿/红配色）替代原有简单状态显示
- Tab3 人工审核卡片同步四场景格式

### 修复
- 企查查 SSE 响应 401 (Authorization header 缺失)
- 工商比对卡片被后续渲染覆盖

---

## v20260630d — AI工商信息核对员 "Failed to fetch" 修复

### 修复
- Agent 3 QCC 查询改用 `API_BASE + '/api/qcc/query'` 替代裸相对路径
- file:// 协议下 QCC 查询现在与 AI 聊天使用相同的 fallback 策略

---

## v20260630c — Agent 命名 + 企查查 Bug 修复

### 变更
- Agent 3 → AI工商信息核对员 (企查查查询)
- Agent 4 → 消息触达机器人 (推送通知)
- Pipeline 执行顺序对调（Agent 3 先于 Agent 4）

### 修复
- 企查查 SSE 解析：从 `startswith('data: ')` 改为逐行扫描
- server.py 工具名：`search_company` → `get_company_registration_info`
- 多余 `}` 大括号导致页面完全无响应

---

## v20260630b — AI工商信息核对员结果集成

### 新增
- Agent 3 企查查查询结果集成到 Tab1 和 Tab3
- 工商信息与商家表单数据自动比对
- 法定代表人、注册资本、成立日期等扩展字段展示

### 变更
- Tab1 工商查询结果增强（比对 + 纠错提示）
- Tab3 审核卡片新增企查查查询 section

---

## v20260621m — 4-Sub-Agent Pipeline 架构

### 新增
- 4 Agent 流水线架构：
  - Agent 1 (绿): AI材料审核员 — 视觉模型 qwen-vl-max 图片分析
  - Agent 2 (红): 审核结果评估专家 — 置信度评分 + 自动重试
  - Agent 3 (黄): 新 agent 预留位
  - Agent 4 (紫): 新 agent 预留位
- `runAgentPipeline()` 流水线编排器
- Agent 2 低置信度自动重试机制（最多 1 次）
- 非关键路径 Agent 失败不阻塞主流程

### 技术约定
- 两份材料（营业执照注销证明 + 退款申请函）必须都上传才触发 AI
- 视觉模型审查图1公章、图2姓名+日期
- 决策树无「分公司」选项，收款账户仅「绑定对公卡」和「个人法人卡」
- API Key 存 localStorage，设置面板可配置

---

## v20260621a — 项目初始化

### 新增
- 单文件 frontend `index.html` (iPhone 15 Pro Max phone demo)
- Python backend `server.py` (port 8766 静态文件 + AI API 代理)
- 通义千问集成（vision `qwen-vl-max` + text `qwen-turbo`）
- 场景决策树 (Tab1)
- 材料提交表单 + 文件上传 (Tab2)
- 人工审核面板 (Tab3)
- 飞书通知卡片 (Tab4)
- 当日审批汇总 (Tab5)
- `start.sh` 启动脚本
