---
name: sales-daily-brief
description: >-
  生成销售晨报：汇总今日日程、待审批、7天未跟进客户、14天内到期的进行中商机。
  用户说今天怎么安排、晨会、晨报、今天该跟进谁、今日待办概览时使用。只读，不创建也不修改记录。
---

# 销售晨报

只读。不创建、不修改、不审批、不写跟进。用户接着要求写操作时，先说明晨报已结束，再按对应工具单独确认。

本 Skill 面向已连接销售易（Neo CRM）MCP 的助手。正文里的工具名使用连字符形式，例如 `schedule-query`、`entity-search`。

## 何时使用

用户要今日安排、晨会材料、晨报，或问今天该跟进谁。

## 步骤

1. 调用 `user-responsibilities`，记下 `primaryDomain`。
2. 同一轮并行调用下面三项。时间用当前真实时间，时区 Asia/Shanghai。毫秒时间戳只做一次 UTC+8 换算。

### schedule-query

- `startTime`：今天 00:00:00
- `endTime`：明天 00:00:00

### process-workbench

- `viewType`: `todo`
- `mode`: `list`
- `page`: `{ "pageNo": 0, "pageSize": 5 }`

记下返回的 `total` / `flowCount`。列表只展示前 5 条。

### entity-search

只调用一次，用 `queries`（最多 5 项）。禁止再并发第二次 `entity-search`。

按 `primaryDomain` 选择子查询：

| primaryDomain | 子查询 |
|---------------|--------|
| sales、unknown | 客户 + 商机 |
| prm | 伙伴 |
| service | 工单 |

**客户：** `entity=account`，`dataView=account_view_noActivitiesDay`，`page={pageNo:1,pageSize:5}`。记下 `total`。视图不存在时改用 `account_view_all`，并说明该视图未开启。

**商机：** `entity=opportunity`，`dataView=opportunity_view_owner`，`sort=[{sortdatafield:closeDate,sortorder:asc}]`，`page={pageNo:1,pageSize:5}`，`conditions`：

- `{apiKey:status,type:1,value:"0"}`（进行中；本租户字段是 `status`，不是 `opportunityStatus`）
- `{apiKey:closeDate,type:12,value:"开始毫秒,结束毫秒"}`，从今天 00:00 到 14 天后 23:59:59.999

若响应 `warnings` 提示字段不存在，去掉该条件重查一次，并在结果里说明。14 天内无结果但名下有进行中商机时，追加一句：本窗口内无到期商机，名下另有进行中商机。

**伙伴：** `entity=partner`，`dataView=partner_view_owner`，`sort=[{sortdatafield:updatedAt,sortorder:desc}]`，`page={pageNo:1,pageSize:5}`。

**工单：** `entity=case`，`dataView=case_view_owner`，`sort=[{sortdatafield:createdAt,sortorder:desc}]`，`page={pageNo:1,pageSize:5}`。

视图不存在时降为 `{entity}_view_all` 并说明。

## 展示

有数据的区块才写。空结果整段省略，不要写「暂无」。

顺序：今日日程、待审批、然后是业务区块（未跟进客户、即将到期商机、伙伴或工单）。

每条只保留名称、时间和一句状态。选项字段用返回的 `*-label`，不展示 optionCode。

待审批、客户等若 `total` 大于本页条数，在区块标题写总数，例如「待审批（共 7 条，显示 5 条）」。待审批创建时间明显偏旧时，附带创建日期，避免看起来像今天的新待办。

末尾一句：「还想深入了解哪方面？」
