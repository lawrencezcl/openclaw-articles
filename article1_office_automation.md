# OpenClaw + 企业微信：打造你的24小时AI办公助理

## 前言

作为一名技术从业者，每天的工作中充满了大量重复性操作：整理会议纪要、汇总数据报表、监控服务器状态、定时发送通知...这些任务琐碎却必不可少，占用了大量本可以用于创造性工作的精力。

当我第一次接触到 OpenClaw 时，立刻意识到这就是我一直在寻找的解决方案——一个能够直接执行系统命令、操作文件、调用 API 的 AI Agent。将它部署在腾讯云 Lighthouse 上，并通过企业微信接入，意味着我可以拥有一个全天候在线的智能办公助理。

本文将详细介绍如何基于 Lighthouse 部署 OpenClaw 并接入企业微信，以及我设计的三个实用办公自动化场景。

---

## 一、为什么选择云端部署

在开始之前，我想先谈谈为什么选择腾讯云 Lighthouse 作为 OpenClaw 的运行环境。

**本地部署的问题：**
- 占用个人电脑资源，影响日常工作
- 关机后 Agent 无法响应
- 混合个人数据和工作数据，存在安全隐患
- 网络不稳定时可能掉线

**Lighthouse 云部署的优势：**
- **秒级部署**：一键镜像启动，无需复杂配置
- **7×24小时在线**：服务器永不关机，随时响应请求
- **环境隔离**：工作环境与个人环境完全分离
- **资源充足**：2核4G配置足以运行大部分轻量级任务
- **成本可控**：轻量应用服务器按需付费，性价比高

---

## 二、Lighthouse 部署 OpenClaw 全流程

### 2.1 购买并配置 Lighthouse 实例

1. 登录腾讯云控制台，选择「轻量应用服务器 Lighthouse」

2. 选择镜像：
   - 地域：选择距离你最近的区域（延迟更低）
   - 镜像：选择「OpenClaw」专用镜像（腾讯云已预制）
   - 配置：建议 2核4G 或以上
   - 时长：根据比赛周期选择（1个月起购）

3. 设置密码并购买，等待实例创建完成（约1-2分钟）

### 2.2 获取企业微信应用凭证

1. 登录[企业微信管理后台](https://work.weixin.qq.com/)
2. 进入「应用管理」→「应用」→「自建应用」
3. 创建新应用，记录以下信息：
   - `corp_id`：企业ID
   - `agent_id`：应用AgentId
   - `secret`：应用Secret

4. 配置应用接收消息：
   - 设置回调URL：`http://你的Lighthouse公网IP:端口/callback`
   - 设置Token和EncodingAESKey

### 2.3 配置 OpenClaw 连接企业微信

SSH 连接到 Lighthouse 实例后，编辑配置文件：

```bash
# 进入项目目录
cd /opt/openclaw

# 编辑配置文件
vim config/wechat_work.json
```

填入企业微信凭证：

```json
{
  "corp_id": "你的企业ID",
  "agent_id": 1000002,
  "secret": "你的应用Secret",
  "token": "你设置的Token",
  "encoding_aes_key": "你设置的EncodingAESKey",
  "callback_port": 8080
}
```

### 2.4 启动服务

```bash
# 启动 OpenClaw 服务
docker-compose up -d

# 查看运行状态
docker-compose ps

# 查看日志
docker-compose logs -f openclaw
```

**部署完成截图示例：**

```
NAME                COMMAND                  SERVICE             STATUS              PORTS
openclaw            "/usr/bin/openclaw ..."   openclaw            running             0.0.0.0:8080->8080/tcp
```

在企业微信中向应用发送「你好」，如果收到回复，则部署成功！

---

## 三、场景一：自动会议纪要生成器

### 痛点描述

每次开完会后，都需要花费半小时整理会议记录：提取关键决策、分配行动项、设置提醒...既耗时又容易遗漏。

### 解决方案

利用 OpenClaw 的文件处理能力和 LLM 的理解能力，实现半自动化的会议纪要生成。

**工作流程：**

1. 用户在企业微信中发送会议录音转写的文本文件
2. OpenClaw 接收文件，调用 LLM 进行结构化提取
3. 生成包含「会议时间、参会人员、讨论要点、决策事项、行动项」的标准格式纪要
4. 将纪要保存到共享目录，并@相关负责人发送提醒

**关键代码片段（OpenClaw 脚本）：**

```python
import json
import re
from datetime import datetime

def process_meeting_transcript(content):
    """
    处理会议转录文本，提取结构化信息
    """
    system_prompt = """
    你是一个会议记录助手。请从以下会议转录中提取：
    1. 会议时间
    2. 参会人员
    3. 讨论主题
    4. 决策事项（格式：[决策] XXX）
    5. 行动项（格式：[行动] @谁 - 完成什么 - 截止日期）
    """

    # 调用 LLM 进行结构化提取
    response = call_llm(content, system_prompt)

    # 格式化输出
    template = f"""
# 会议纪要

**时间**：{extract_time(content)}
**参会人**：{extract_attendees(content)}

## 讨论要点
{response['discussion']}

## 决策事项
{response['decisions']}

## 行动项
| 负责人 | 任务内容 | 截止日期 |
|--------|----------|----------|
{response['action_items']}

---
*生成时间：{datetime.now().strftime('%Y-%m-%d %H:%M')}*
"""

    return template

# 配置 OpenClaw 指令映射
command_handlers = {
    "会议纪要": process_meeting_transcript
}
```

**使用效果：**

```
用户：[上传 transcript.txt 文件]
用户：帮我把这个会议内容整理成纪要

OpenClaw：正在处理会议记录...
OpenClaw：[发送格式化的会议纪要]
OpenClaw：已将纪要保存到 /meetings/2024-02-07-product-review.md
OpenClaw：@张三 @李五 请确认各自的行动项
```

---

## 四、场景二：定时业务数据汇总

### 痛点描述

每天早上9点，需要从多个系统导出数据、生成报表、发送给团队。重复且容易出错。

### 解决方案

使用 OpenClaw 的定时任务能力，自动完成数据汇总和推送。

**实现方式：**

1. 创建定时脚本 `daily_report.sh`：

```bash
#!/bin/bash

# 每日业务数据汇总脚本

# 定义日期
TODAY=$(date +%Y-%m-%d)
REPORT_DIR="/var/www/reports"

# 1. 从数据库导出昨日数据
docker exec mysql mysql -uuser -ppassword db -e "
SELECT DATE(created_at) as date,
       COUNT(*) as orders,
       SUM(amount) as revenue
FROM orders
WHERE DATE(created_at) = DATE_SUB(CURDATE(), INTERVAL 1 DAY)
GROUP BY DATE(created_at);
" > $REPORT_DIR/orders_$TODAY.csv

# 2. 生成图表
python3 /scripts/generate_chart.py $REPORT_DIR/orders_$TODAY.csv

# 3. 上传到企业微信
curl -X POST "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "msgtype": "markdown",
    "markdown": {
      "content": "## 📊 每日业务数据报告\n\n**日期**：'"$TODAY"'\n\n请查看附件中的详细数据。"
    }
  }'

# 4. 发送文件
curl -X POST "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY" \
  -F media=@$REPORT_DIR/report_$TODAY.png

echo "报告发送完成：$TODAY"
```

2. 配置 crontab 定时执行：

```bash
# 每天早上8:50执行
50 8 * * * /opt/scripts/daily_report.sh >> /var/log/daily_report.log 2>&1
```

3. 通过企业微信接收报告：

```
OpenClaw：
📊 每日业务数据报告

日期：2026-02-07

请查看附件中的详细数据。

[图片：订单趋势图]
[文件：orders_2026-02-07.csv]
```

---

## 五、场景三：服务器健康监控与告警

### 痛点描述

服务器偶尔会出现异常，但总是等到影响业务了才发现。需要有一个主动监控和告警的机制。

### 解决方案

OpenClaw 作为监控 Agent，实时检查服务器状态，异常时主动通知。

**监控脚本：**

```python
#!/usr/bin/env python3
import psutil
import requests
import json

def check_server_health():
    """
    检查服务器健康状态
    """
    alerts = []

    # 1. 检查CPU使用率
    cpu_percent = psutil.cpu_percent(interval=1)
    if cpu_percent > 80:
        alerts.append(f"⚠️ CPU使用率过高：{cpu_percent}%")

    # 2. 检查内存使用率
    memory = psutil.virtual_memory()
    if memory.percent > 85:
        alerts.append(f"⚠️ 内存使用率过高：{memory.percent}%")

    # 3. 检查磁盘空间
    disk = psutil.disk_usage('/')
    if disk.percent > 90:
        alerts.append(f"⚠️ 磁盘空间不足：{disk.percent}%")

    # 4. 检查关键进程
    critical_processes = ['nginx', 'mysql', 'redis']
    for proc in critical_processes:
        if not is_process_running(proc):
            alerts.append(f"🔴 关键进程 {proc} 未运行")

    # 5. 发送告警
    if alerts:
        send_wechat_alert(alerts)

def is_process_running(process_name):
    """检查进程是否运行"""
    for proc in psutil.process_iter(['name']):
        if process_name.lower() in proc.info['name'].lower():
            return True
    return False

def send_wechat_alert(alerts):
    """发送企业微信告警"""
    webhook_url = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY"

    content = "## 🔔 服务器告警\n\n" + "\n".join(alerts)

    requests.post(webhook_url, json={
        "msgtype": "markdown",
        "markdown": {"content": content}
    })

if __name__ == "__main__":
    check_server_health()
```

**配置定时检查（每5分钟）：**

```bash
*/5 * * * * /opt/scripts/health_check.py
```

**实际效果：**

```
OpenClaw：
🔔 服务器告警

⚠️ CPU使用率过高：92%
⚠️ 内存使用率过高：88%

建议：检查是否有异常进程占用资源
```

---

## 六、使用技巧与最佳实践

### 6.1 安全建议

1. **权限最小化**：OpenClaw 运行在专用容器中，只授予必要的系统权限
2. **敏感数据加密**：API Key、数据库密码等使用环境变量存储
3. **命令白名单**：限制 OpenClaw 可以执行的命令范围
4. **日志审计**：记录所有操作日志，便于问题排查

### 6.2 提升响应速度

1. **使用 SSD 云硬盘**：减少文件 I/O 延迟
2. **选择就近地域**：降低 IM 平台回调延迟
3. **启用缓存**：常用查询结果缓存 5-10 分钟

### 6.3 提高回答质量

1. **明确的指令模板**：为每个场景设计固定的指令格式
2. **上下文管理**：设置合理的上下文窗口，避免信息混乱
3. **多轮对话优化**：通过追问澄清用户意图

---

## 七、总结与展望

通过将 OpenClaw 部署在腾讯云 Lighthouse 并接入企业微信，我成功构建了一个 7×24 小时在线的智能办公助理。从会议纪要自动生成，到业务数据定时汇总，再到服务器健康监控，三个场景都已经在实际工作中稳定运行。

**关键成果：**
- 每周节省约 5 小时的重复性工作时间
- 会议纪要质量提升，遗漏率从 30% 降至 5%
- 服务器故障发现时间缩短至 5 分钟以内

**下一步计划：**
- 接入更多企业内部系统（OA、CRM）
- 增加多轮对话能力，支持更复杂的任务
- 探索与其他 IM 平台的集成

OpenClaw 的强大之处在于它不仅是一个对话机器人，更是一个能够真正「动手」的 AI Agent。结合 Lighthouse 的稳定性和企业微信的便捷性，这样的组合为办公自动化打开了无限可能。

---

## 部署资源清单

| 资源类型 | 配置建议 | 费用参考 |
|---------|---------|---------|
| Lighthouse 实例 | 2核4G，50GB SSD | 约 ¥60/月 |
| 峰值带宽 | 4Mbps（按流量计费） | 约 ¥10-30/月 |
| 企业微信 | 免费版 | ¥0 |
| **月度成本** | - | **约 ¥70-90/月** |

---

*本文首发于腾讯云开发者社区，标签：玩转OpenClaw云端创意实践*
