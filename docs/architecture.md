# 商家退保 AI 审核系统 — 架构详解

## 1. 4-Sub-Agent Pipeline 详解

### Agent 1: AI材料审核员 (`#16a34a` 绿)

**函数**: `runSubAgent1_MaterialReview()`

- 使用视觉模型 `qwen-vl-max` 分析上传的图片
- 审查项：
  - **图1（营业执照注销证明）**：公章是否清晰、是否在有效位置
  - **图2（退款申请函）**：签名是否完整、日期是否填写
- 输入：`cert` 和 `letter` 两张图片的 base64
- 输出：JSON 格式的审核结果，包含 `passed`、`materialType`、`issues` 等字段
- 温度：`0.2`
- 失败策略：阻断整个流水线

### Agent 2: AI审核结果评估专家 (`#dc2626` 红)

**函数**: `runSubAgent2_ConfidenceEval()`

- 使用文本模型评估 Agent 1 的结果置信度
- 评分维度（0-100）：
  - `materialTypeAccuracy`：材料类型识别准确度
  - `keywordExtractionAccuracy`：关键词提取准确度
  - `complianceAssessmentAccuracy`：合规评估准确度
- 综合评分 → `high`（≥80）/ `medium`（50-79）/ `low`（低于 50）
- 低置信度时自动重试 Agent 1（最多 1 次），tracked by `state._agent1Retried`
- 温度：`0.2`
- 失败策略：降级为 `{confidence: "medium", overallScore: 50}`

### Agent 3: AI工商信息核对员 (`#d97706` 黄)

**函数**: `runSubAgent3_QccQuery()`

- **非 LLM 调用**，直接 HTTP POST 到 `/api/qcc/query`
- 输入：商家名称 (`companyName`)、统一社会信用代码 (`creditCode`)
- 返回：`{ isDeregistered, registrationStatus, legalRep, registeredCapital, establishDate, queryTime }`
- 失败策略：非关键路径，返回 `isDeregistered: null`

### Agent 4: 消息触达机器人 (`#7c3aed` 紫)

**函数**: `runSubAgent4_Notification()`

- 使用 LLM 生成通知消息模板
- 格式：飞书/企微/钉钉卡片消息 JSON
- 温度：`0.3`
- 失败策略：非关键路径，静默失败
- UI：`agentCard4` 默认 `display:none`

## 2. 4 场景工商信息比对

**函数**: `displayBusinessQueryResult(queryResult)`

| 场景 | 商家填报 | 企查查实况 | 结论颜色 | 提示 |
|------|----------|-----------|----------|------|
| 1 | 已注销 | 已注销 | 🟢 绿 `#16a34a` | AI已复核，检测到当前公司的工商信息已注销。 |
| 2 | 未注销 | 未注销 | 🟢 绿 `#16a34a` | AI已复核，检测到当前公司的工商信息未注销。 |
| 3 | 已注销 | 未注销 | 🔴 红 `#dc2626` | AI已复核，检测到当前公司的工商信息未注销，请核对你选择的信息。 |
| 4 | 未注销 | 已注销 | 🔴 红 `#dc2626` | AI已复核，检测到当前公司的工商信息已注销，请核对你选择的信息。 |
| - | 任意 | 查询失败 | ⬜ 灰 `#6b7280` | AI工商信息核对员查询失败，无法验证工商状态 |

## 3. 场景决策树（Tab 1）

4 个决策维度：

### Dimension 1: 绑卡状态 (`data-level="bind"`)
- `bound` — 已绑卡
- `unbound` — 未绑卡

### Dimension 2: 商家类型 (`data-level="type"`)
- `company` — 单个公司 / 总公司
- `individual` — 个体工商户

### Dimension 3: 经营资质状态 (`data-level="license"`)
- `active` — 未注销
- `deregistered` — 已注销

### Dimension 4: 对公卡状态 (`data-level="card"`)
- `normal` — 正常使用
- `deregistered` — 已注销
- `unavailable` — 未办理/无法使用

### 自动规则

**无需审核**：
- `bound + card(normal)` → 自动退款
- `unbound + license(active)` → 引导绑卡

**需要审核的材料清单**：

| 场景 | 公司 (company) | 个体户 (individual) |
|------|---------------|-------------------|
| 已注销 | 营业执照注销证明 + 退款申请函(全体股东+法代签字按手印) + 手持身份证照片 | 营业执照注销证明 + 退款申请函(经营者签字按手印) + 手持身份证照片 |
| 未注销+卡已注销 | 对公账户注销证明 + 退款申请函(加盖公章) | 退款申请函(经营者签字按手印) |
| 未注销+卡未办理 | 退款申请函(加盖公章) | 退款申请函(经营者签字按手印) + 手持身份证照片 |

## 4. 页面 Tab 结构

| Tab | ID | 内容 |
|-----|----|------|
| Tab 1 | `tab-scenario` | 场景决策树，根据商家属性自动判断是否需要人审 |
| Tab 2 | `tab-submit` | 商家提交表单 + 文件上传 + AI 审核流水线展示 |
| Tab 3 | `tab-review` | 人工审核面板，展示 AI 意见 + 置信度 + 工商比对 |
| Tab 4 | `tab-feishu` | 飞书卡片通知模板（动态生成） |
| Tab 5 | `tab-summary` | 当日审批汇总（22:00 自动触发 AI 汇总建议） |

## 5. 后端 API 端点

### `POST /api/chat`

AI 对话代理，转发到通义千问。

```json
// 请求
{
  "provider": "qwen",
  "apiKey": "sk-xxx",
  "body": {
    "model": "qwen-vl-max",
    "messages": [...],
    "temperature": 0.2,
    "max_tokens": 2048,
    "stream": false
  }
}

// 响应
{
  "status": 200,
  "body": {
    "choices": [{ "message": { "content": "..." } }]
  }
}
```

### `POST /api/qcc/query`

企查查工商查询代理。

```json
// 请求
{
  "companyName": "星辰科技有限公司",
  "creditCode": "91110108MA01XXXXX"
}

// 响应
{
  "success": true,
  "source": "qcc_mcp",
  "companyName": "星辰科技有限公司",
  "creditCode": "91110108MA01XXXXX",
  "isDeregistered": true,
  "registrationStatus": "注销",
  "legalRep": "张三",
  "registeredCapital": "100万元",
  "establishDate": "2015-03-15",
  "queryTime": "2026-06-30 15:00:00",
  "detail": "登记状态：注销"
}
```

## 6. 前端状态管理

```javascript
const state = {
  isChecking: false,           // 流水线执行锁
  aiPassed: false,             // AI 审核结果
  reviewSubmitted: false,      // 是否已提交审核
  _agent1Retried: false,       // Agent 1 重试标记
  businessQueryResult: null,   // QCC 查询结果缓存
  dailyRecords: [],            // 当日审核记录
  summaryDelayMs: 10000        // 汇总延迟（10s）
};
```

### 文件上传

```javascript
uploadedFiles = { cert: null, letter: null }     // File 对象
uploadedFileB64 = { cert: null, letter: null }   // base64 数据
```

两者必须都有值才触发 AI 流水线（`checkAndTriggerAutoAI()` 函数）。

## 7. 数据处理流程

```
用户填写表单 → 上传 2 份材料 → 点击提交
  ↓
checkAndTriggerAutoAI() 验证：
  - 2 份材料都已上传?
  - 不是重复提交?
  ↓
runAgentPipeline():
  1. collectFormData() 收集表单数据
  2. runSubAgent1_MaterialReview() → 视觉模型分析图片
  3. runSubAgent2_ConfidenceEval() → 评估置信度
     - low → 重试 Agent 1
  4. runSubAgent3_QccQuery() → 企查查查询
  5. runSubAgent4_Notification() → 生成通知模板
  6. 渲染结果 + displayBusinessQueryResult()
  7. state.businessQueryResult = queryResult (缓存供 Tab 3)
```
