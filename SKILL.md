---
name: icloud-calendar
version: 2.0.0
description: Manage iCloud calendars from natural language. Create, query, update, and delete events on your iPhone calendar via CalDAV. Trigger words: schedule, reminder, meeting, appointment, calendar, event, 日程, 提醒, 会议, 安排, 日历. Supports smart duration inference, auto calendar selection (work/personal), location parsing, keyword-based bulk delete/update, RRULE recurring events.
metadata:
  openclaw:
    requires:
      bins: [python]
      env: [ICLOUD_EMAIL, ICLOUD_APP_PASSWORD]
    emoji: "📅"
  author: RickHuang28
  license: MIT
  tags: [calendar, icloud, caldav, schedule, reminder, 日程, 日历]
---

# 📅 iCloud Calendar — Full CRUD via Natural Language

**Version 2.0.0** | Manage your iCloud calendar through natural language via CalDAV.

> ⚠️ 首次使用请先完成配置 → 见 [CONFIG.md](CONFIG.md)
> 
> 你需要：**iCloud 邮箱** + **App-Specific 密码** + **Python 3.8+** + **`pip install caldav`**

---

## 配置清单

使用前确认以下 3 项已完成：

1. ✅ **`pip install "caldav>=1.3.0,<2.0"`**
2. ✅ **`ICLOUD_EMAIL` 环境变量** = 你的 iCloud 邮箱
3. ✅ **`ICLOUD_APP_PASSWORD` 环境变量** = Apple 专用密码 (xxxx-xxxx-xxxx-xxxx)

> 如果脚本返回 `"Missing ICLOUD_EMAIL or ICLOUD_APP_PASSWORD environment variables"`，说明环境变量未正确注入。

---

## 解析规则

从自然语言中提取字段，未提及的按推断规则补全。

### 1. 标题 (summary)

从消息中提取事件核心，去掉时间词和口癖：
- "下周三下午3点和张总开会" → "和张总开会"
- "明天记得交报告" → "交报告"
- "周五晚上聚餐" → "聚餐"

### 2. 开始时间 (start)

必填。相对日期→绝对日期计算（Asia/Shanghai）：
- 今天/明天/后天/大后天 → 对应日期
- 下周一~下周日 → 下周对应星期几
- 时间："下午3点"→15:00 "上午九点半"→09:30 "晚上8点"→20:00
- 无具体时间 → 默认 09:00

### 3. 结束时间 (end)

永远不问用户，按事件类型推断时长：
- 会议/汇报/评审/面试 → 1小时
- 聚餐/饭局/火锅/吃饭 → 2小时
- 运动/健身/游泳/打球 → 1.5小时
- 电影 → 2.5小时
- 培训/课程/讲座 → 2小时
- 飞机/高铁/火车 → 3小时
- 全天 → 00:00~23:59
- 兜底 → 1小时
- 用户说了结束时间 → 用用户的

### 4. 日历选择 (calendar)

按事件性质自动选：
- 含"开会/汇报/评审/客户/项目/出差/面试"关键词 → 工作日历
- 其他 → 个人日历

### 5. 地点 (location)

提及则提取："在301会议室"→"301会议室" "去北京出差"→"北京"
未提及 → 空

### 6. 提醒 (alarm)

- 用户说"提醒我" → 开始前15分钟
- 指定时间"提前10分钟" → 用指定时间
- 未提及 → 默认 15 分钟

### 7. 备注 (description)

收集额外信息：参与者、特殊要求等。"和张总一起" → 备注写"参与者：张总"

---

## 执行（创建事件）

```bash
export ICLOUD_EMAIL="your@icloud.com"
export ICLOUD_APP_PASSWORD="xxxx-xxxx-xxxx-xxxx"
python scripts/add-event.py \
    --summary "标题" \
    --start "2026-04-08T15:00:00" \
    --end "2026-04-08T16:00:00" \
    --timezone "Asia/Shanghai" \
    --location "" \
    --calendar "个人" \
    --description "" \
    --alarm-minutes 15
```

**可选参数：**
- `--is-all-day`：纯日期事件，生成 `DTSTART;VALUE=DATE` 格式，跨时区不漂移
- `--rrule "FREQ=WEEKLY;BYDAY=MO"`：重复事件

### 回复示例

```
📅 已记录：和张总开会
🕐 4月8日 周三 15:00-16:00（工作日历）
📍 301会议室
⏰ 提前15分钟提醒
```

---

## 查询（反向查日历）

用户问"我明天有什么安排""这周日程""4月8号有什么"时：

```bash
python scripts/add-event.py --query "today|tomorrow|week|nextweek|YYYY-MM-DD|YYYY-MM-DD~YYYY-MM-DD"
```

query 值：
- `today` / `tomorrow` / `week` / `nextweek`
- 单日：`2026-04-08`
- 范围：`2026-04-01~2026-04-30`

### 回复示例

```
📅 明天（4月2日）有 3 个安排：
1. 10:00-11:00 日历功能测试（个人）
2. 14:00-15:00 项目评审（工作）
3. 全天 ios退款请加扣扣群...（工作）
```

---

## 搜索（按关键词）

用户说"我下周有和张总的安排吗""查一下开会的日程"时：

```bash
python scripts/add-event.py \
    --search "关键词" \
    --search-range "2026-04-01~2026-04-30"
```

- `--search`：必填，模糊匹配 title/location/description
- `--search-range`：可选日期范围，不传默认搜索 ±180 天
- 返回 `{"events": [...], "count": N, "keyword": "关键词", "errors": [...]}`

---

## 修改事件

用户说"改到4点""换到工作日历""地点改成301"时：

```bash
python scripts/add-event.py \
    --update-find "关键词" \
    [--update-set-summary "新标题"] \
    [--update-set-start "2026-04-08T16:00:00"] \
    [--update-set-end "2026-04-08T17:00:00"] \
    [--update-set-location "新地点"] \
    [--update-set-calendar "工作"] \
    [--update-start "2026-04-01"] \
    [--update-end "2026-04-30"]
```

- 匹配到多个事件时会报错并列出所有匹配项
- 返回 `{"updated": true, "changes": {...}}`

---

## 删除（按关键词）

用户说"删掉xxx相关的""删除垃圾事件"时：

```bash
export CONFIRM_DELETE=1
# 可选：预览不执行
export DELETE_DRY_RUN=1
python scripts/add-event.py \
    --delete "关键词" \
    --delete-start "2026-01-01" \
    --delete-end "2026-12-31"
```

- `--delete`：必填，关键词模糊匹配
- `--delete-start` / `--delete-end`：可选搜索范围
- 需要 `CONFIRM_DELETE=1` 环境变量确认
- 可设 `DELETE_DRY_RUN=1` 预览（不真正删除）
- 返回三态 `status: "success"/"partial_success"/"error"` + `dry_run: true/false`

---

## 重复事件（RRULE）

```bash
python scripts/add-event.py \
    --summary "每周例会" \
    --start "2026-04-07T09:00:00" \
    --rrule "FREQ=WEEKLY;BYDAY=MO"
```

常用模式：

| 频率 | RRULE |
|------|-------|
| 每天 | `FREQ=DAILY` |
| 每周一三五 | `FREQ=WEEKLY;BYDAY=MO,WE,FR` |
| 每两周周五 | `FREQ=WEEKLY;INTERVAL=2;BYDAY=FR` |
| 每月1号 | `FREQ=MONTHLY;BYMONTHDAY=1` |
| 限次10次 | `FREQ=WEEKLY;BYDAY=MO;COUNT=10` |
| 限日期 | `FREQ=WEEKLY;BYDAY=MO;UNTIL=20260630T000000Z` |

> RRULE 经过白名单校验，仅允许安全字符，拒绝 CRLF 注入。

---

## 安全特性

| 特性 | 说明 |
|------|------|
| **凭据零泄露** | 仅环境变量传递，异常消息脱敏，进程列表不可见 |
| **操作确认门** | `CONFIRM_DELETE=1` / `CONFIRM_UPDATE=1` 防误操作 |
| **Dry Run** | `DELETE_DRY_RUN=1` 预览删除不执行 |
| **日志轮转** | 最多 512KB × 5 备份，自动清理 |
| **内容截断** | 日志中摘要截断至 30 字符 |
| **重试保护** | Dual-Client 架构，4xx 错误立即放弃不盲目重试 |
| **幂等创建** | UID 唯一，弱网重试不发重复事件 |
| **协议安全** | iCal 字段全转义，RRULE 白名单校验 |

---

## 时区支持

默认 `Asia/Shanghai`，支持任意 IANA 时区名。VTIMEZONE 组件根据 `ZoneInfo` 自动计算，覆盖 DST 夏令时。

---

## 变更记录

完整变更记录见 [CHANGELOG.md](CHANGELOG.md)（v1.4.0 → v2.0.0，36 项改进）。

### v2.0.0 亮点
- ⚡ Dual-Client 架构（10s/60s 超时分离）
- 🔒 异常消息全面脱敏
- 🛡️ 删除/更新双确认门 + dry-run
- 🧹 日志全覆盖 sanitize

---

## 故障排查

| 错误 | 原因 | 解决 |
|------|------|------|
| `Missing ICLOUD_EMAIL...` | 环境变量未设置 | 检查 openclaw.json 并重启 |
| `Authentication failed` | App Password 错误 | 重新生成专用密码 |
| `Connection failed` | iCloud 不可达 | 检查网络/防火墙 |
| `Delete requires CONFIRM_DELETE` | 未设确认标志 | `export CONFIRM_DELETE=1` |

完整配置指南 → [CONFIG.md](CONFIG.md)
