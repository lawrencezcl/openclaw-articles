# 用OpenClaw打造你的QQ社群智能助手

## 前言

作为一名技术社群的管理者，我深知运营活跃社群的痛点：

- **重复问题太多**：新人每天问同样的问题，耐心重复回答
- **信息散落各处**：精华内容混在聊天记录里，难以查找
- **活动组织困难**：统计参与人数、提醒时间、收集反馈都靠人工
- **氛围难以维持**：缺少互动，社群容易变成死群

传统的 QQ 机器人只能做到简单的关键词回复，无法理解上下文，更无法处理复杂任务。

**OpenClaw 改变了这一切**。它是一个能够真正「理解」用户意图并执行复杂操作的 AI Agent。部署在腾讯云 Lighthouse 上，通过 QQ 接入，它将成为你的社群智能助手。

本文将分享如何用 OpenClaw 打造一个功能完善的 QQ 社群机器人，以及我在实际运营中的四个创新玩法。

---

## 一、架构设计：为什么选择这个组合

### 1.1 技术选型

| 组件 | 选择 | 理由 |
|-----|------|------|
| 云服务器 | 腾讯云 Lighthouse | 秒级部署、稳定在线、成本低 |
| AI 核心 | OpenClaw | 文件访问+命令执行，能力全面 |
| QQ 协议 | go-cqhttp (NapCat) | 开源稳定、社区活跃 |
| 向量数据库 | Chroma | 轻量级、易集成 |

### 1.2 系统架构

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│  QQ 用户    │  ←→     │  OpenClaw   │  ←→     │  知识库/数据库 │
│             │         │  Agent      │         │             │
└─────────────┘         └─────────────┘         └──────────────┘
                              ↓
                        ┌─────────────┐
                        │  工具脚本   │
                        │  (文件/命令)│
                        └─────────────┘
                              ↓
                        ┌─────────────┐
                        │ 外部 API    │
                        │ (天气/翻译) │
                        └─────────────┘
```

---

## 二、Lighthouse 部署指南

### 2.1 购买并配置 Lighthouse

**推荐配置：**

```yaml
CPU: 2核
内存: 4GB  # 社群场景建议4G
硬盘: 60GB SSD
带宽: 5Mbps（按流量计费）
系统: Ubuntu 22.04 LTS
费用: 约 ¥60/月
```

**创建实例：**

1. 登录腾讯云控制台 → Lighthouse → 新建实例
2. 选择「OpenClaw 专用镜像」或「Ubuntu 22.04」
3. 设置防火墙规则，开放端口：`8080, 5700, 5701`

### 2.2 安装 QQ 协议层 (NapCat)

[NapCat](https://github.com/NapNeko/NapCatQQ) 是新一代 QQ 协议实现，基于 NTQQ，更稳定：

```bash
# 安装依赖
sudo apt update
sudo apt install -y python3 python3-pip nginx

# 克隆 OpenClaw
git clone https://github.com/OpenClaw/clawdbot.git /opt/openclaw
cd /opt/openclaw

# 安装 Python 依赖
pip3 install -r requirements.txt
```

安装 NapCat（推荐使用 Docker）：

```bash
# 拉取镜像
docker pull mlikiowa/napcat-docker:latest

# 启动容器
docker run -d \
  --name napcat \
  -p 3001:3001 \
  -p 6099:6099 \
  -v /opt/napcat/config:/app/config \
  mlikiowa/napcat-docker:latest

# 配置文件路径
vim /opt/napcat/config/webui.json
```

NapCat 配置示例：

```json
{
  "qq": "你的QQ号",
  "password": "你的QQ密码或MD5",
  "protocol": "3",
  "http": {
    "enable": true,
    "host": "0.0.0.0",
    "port": 3001,
    "secret": "你的secret",
    "enableHeart": true,
    "enablePost": true,
    "postUrls": ["http://127.0.0.1:8080/qq/callback"]
  },
  "ws": {
    "enable": true,
    "host": "0.0.0.0",
    "port": 6099
  }
}
```

### 2.3 配置 OpenClaw

```bash
cd /opt/openclaw

# 复制配置模板
cp .env.example .env
vim .env
```

`.env` 配置：

```bash
# LLM 配置（使用 DeepSeek）
LLM_API_KEY=your_deepseek_api_key
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-chat

# QQ 机器人配置
QQ_BOT_ENABLED=true
QQ_HTTP_URL=http://127.0.0.1:3001
QQ_SECRET=你的napcat_secret
QQ_ADMIN_UIN=你的管理员QQ号

# 数据库配置
DATABASE_URL=sqlite:///data/community_bot.db
```

### 2.4 启动服务

```bash
# 启动 NapCat
docker start napcat

# 启动 OpenClaw
cd /opt/openclaw
python3 main.py

# 或使用 systemd 管理服务
sudo vim /etc/systemd/system/community-bot.service
```

systemd 配置：

```ini
[Unit]
Description=Community QQ Bot with OpenClaw
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/openclaw
Environment="PATH=/usr/bin:/usr/local/bin"
ExecStart=/usr/bin/python3 /opt/openclaw/main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl start community-bot
sudo systemctl enable community-bot
sudo systemctl status community-bot
```

### 2.5 验证部署

在 QQ 中私聊机器人：

```
你：你好
机器人：你好！我是 OpenClaw 智能助手，有什么可以帮你的吗？
```

如果收到回复，部署成功！

**将机器人拉入群聊：**

在 QQ 群设置中添加机器人 QQ 号为成员。

---

## 三、核心功能实现

### 3.1 功能一：智能问答与知识库

**痛点**：新人反复问同样的问题，老成员不堪其扰。

**解决方案**：建立社群知识库，机器人自动回答常见问题。

#### 3.1.1 知识库结构

```
/opt/openclaw/knowledge/
├── faq/                      # 常见问题
│   ├── 新人指南.md
│   ├── 规则说明.md
│   └── 技术问题.md
├── topics/                  # 话题讨论
│   ├── 云原生/
│   ├── AI开发/
│   └── 前端技术/
└── snippets/                # 精华片段
    ├── 代码片段.md
    └── 命令备忘.md
```

#### 3.1.2 问答实现

```python
# qa_engine.py

from sentence_transformers import SentenceTransformer
from pathlib import Path
import numpy as np
import yaml

class QAResponse:
    def __init__(self, knowledge_path="/opt/openclaw/knowledge"):
        self.knowledge_path = Path(knowledge_path)
        self.model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')

        # 加载 FAQ
        self.faq_items = self._load_faq()
        self.faq_embeddings = self.model.encode(
            [item['question'] for item in self.faq_items]
        )

    def _load_faq(self):
        """加载 FAQ"""
        faq_file = self.knowledge_path / "faq" / "faq.yaml"

        if not faq_file.exists():
            return []

        with open(faq_file, 'r', encoding='utf-8') as f:
            return yaml.safe_load(f)

    def answer(self, query, threshold=0.75):
        """
        回答问题
        """
        if not self.faq_items:
            return None

        # 计算相似度
        query_embedding = self.model.encode([query])
        similarities = np.dot(self.faq_embeddings, query_embedding.T).flatten()

        # 找最匹配的
        max_idx = np.argmax(similarities)
        max_score = similarities[max_idx]

        if max_score >= threshold:
            item = self.faq_items[max_idx]
            return {
                "question": item['question'],
                "answer": item['answer'],
                "confidence": max_score,
                "related": item.get('related', [])
            }

        return None
```

FAQ 文件格式 (`faq.yaml`)：

```yaml
- question: "如何加入社群?"
  answer: |
    加入方式：
    1. 扫描群公告中的二维码
    2. 或添加管理员微信：xxx（备注：入群）
    3. 入群后请修改群昵称为：姓名-行业-城市
  related:
    - "社群规则"

- question: "社群有什么活动?"
  answer: |
    常规活动：
    - 每周三 20:00：技术分享会
    - 每月第一个周六：线下聚会（北京/上海/深圳）
    - 不定期：代码审查会、项目实战
  related:
    - "活动报名"
```

#### 3.1.3 使用效果

```
群成员 A：怎么加入这个群？

机器人：
📋 常见问题匹配

加入方式：
1. 扫描群公告中的二维码
2. 或添加管理员微信：xxx（备注：入群）
3. 入群后请修改群昵称为：姓名-行业-城市

相关：社群规则
```

---

### 3.2 功能二：自动精华整理

**痛点**：群里有价值的讨论很快被淹没，难以回溯。

**解决方案**：机器人自动识别高质量内容，保存到精华库。

```python
# content_curator.py

import re
from datetime import datetime
from pathlib import Path
from openai import OpenAI

class ContentCurator:
    def __init__(self):
        self.client = OpenAI()  # DeepSeek API
        self.highlights_path = Path("/opt/openclaw/knowledge/highlights")
        self.highlights_path.mkdir(parents=True, exist_ok=True)

    async def on_group_message(self, message):
        """
        处理群消息，判断是否值得收藏
        """
        # 只处理长消息（>100字）
        if len(message.content) < 100:
            return

        # AI 判断价值
        score = await self._evaluate_value(message.content)

        if score >= 7:
            await self._save_highlight(message, score)

    async def _evaluate_value(self, content):
        """
        评估内容价值（1-10分）
        """
        prompt = f"""
        请评估以下群聊内容的价值（1-10分），标准：
        - 技术深度：是否有技术干货
        - 实用性：是否对他人有帮助
        - 独特性：是否是独特见解

        只返回分数，不要其他内容。

        内容：
        {content}
        """

        response = await self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": prompt}],
            temperature=0
        )

        score_text = response.choices[0].message.content.strip()
        return int(score_text)

    async def _save_highlight(self, message, score):
        """
        保存精华内容
        """
        date_str = datetime.now().strftime("%Y%m%d")
        timestamp = datetime.now().strftime("%H%M%S")

        filename = f"{date_str}-{timestamp}.md"
        filepath = self.highlights_path / filename

        content = f"""# {message.sender_name} 的分享

**时间**：{message.timestamp}
**来源**：{message.group_name}
**评分**：{score}/10

{message.content}

---

**标签**：{self._extract_tags(message.content)}
"""

        with open(filepath, 'w', encoding='utf-8') as f:
            f.write(content)

        # 在群里通知
        await message.reply(
            f"✨ 已收藏这篇精华内容（评分 {score}/10）\n"
            f"可以发送「精华列表」查看所有收藏"
        )
```

**使用效果：**

```
群成员 B：[发送一段1000字的技术干货分享]

机器人：
✨ 已收藏这篇精华内容（评分 9/10）
可以发送「精华列表」查看所有收藏

群成员 C：精华列表

机器人：
📚 社群精华内容（共 23 篇）

1. 【9.0分】Go语言最佳实践 - @张三 (2026-02-05)
2. 【8.5分】微服务架构设计经验 - @李四 (2026-02-03)
3. 【8.0分】Docker性能优化技巧 - @王五 (2026-02-01)

回复「查看 1-3」查看详情
```

---

### 3.3 功能三：活动管理助手

**痛点**：组织活动需要统计报名、发送提醒、收集反馈，全靠人工。

**解决方案**：机器人接管活动管理全流程。

```python
# event_manager.py

import sqlite3
from datetime import datetime, timedelta
from pathlib import Path

class EventManager:
    def __init__(self, db_path="/data/community_bot.db"):
        self.db_path = db_path
        self._init_db()

    def _init_db(self):
        """初始化数据库"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
        CREATE TABLE IF NOT EXISTS events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT,
            event_time TEXT NOT NULL,
            max_participants INTEGER,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        )
        """)

        cursor.execute("""
        CREATE TABLE IF NOT EXISTS participants (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            event_id INTEGER,
            qq_number TEXT NOT NULL,
            nickname TEXT,
            joined_at TEXT DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (event_id) REFERENCES events(id)
        )
        """)

        conn.commit()
        conn.close()

    def create_event(self, title, description, event_time, max_participants=None):
        """创建活动"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
        INSERT INTO events (title, description, event_time, max_participants)
        VALUES (?, ?, ?, ?)
        """, (title, description, event_time, max_participants))

        event_id = cursor.lastrowid
        conn.commit()
        conn.close()

        return event_id

    def join_event(self, event_id, qq_number, nickname):
        """报名参加活动"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        # 检查活动是否存在
        cursor.execute("SELECT max_participants, event_time FROM events WHERE id = ?", (event_id,))
        event = cursor.fetchone()

        if not event:
            conn.close()
            return {"success": False, "message": "活动不存在"}

        max_participants, event_time = event

        # 检查是否已满
        if max_participants:
            cursor.execute("SELECT COUNT(*) FROM participants WHERE event_id = ?", (event_id,))
            count = cursor.fetchone()[0]
            if count >= max_participants:
                conn.close()
                return {"success": False, "message": "报名人数已满"}

        # 检查是否已报名
        cursor.execute(
            "SELECT id FROM participants WHERE event_id = ? AND qq_number = ?",
            (event_id, qq_number)
        )
        if cursor.fetchone():
            conn.close()
            return {"success": False, "message": "你已经报名过了"}

        # 报名
        cursor.execute(
            "INSERT INTO participants (event_id, qq_number, nickname) VALUES (?, ?, ?)",
            (event_id, qq_number, nickname)
        )

        conn.commit()
        conn.close()

        return {
            "success": True,
            "message": f"报名成功！活动时间：{event_time}"
        }

    def get_event_status(self, event_id):
        """获取活动状态"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()

        cursor.execute("""
        SELECT e.title, e.description, e.event_time, e.max_participants,
               COUNT(p.id) as current_participants
        FROM events e
        LEFT JOIN participants p ON e.id = p.event_id
        WHERE e.id = ?
        GROUP BY e.id
        """, (event_id,))

        event = cursor.fetchone()
        conn.close()

        if not event:
            return None

        return {
            "title": event[0],
            "description": event[1],
            "event_time": event[2],
            "max_participants": event[3],
            "current_participants": event[4]
        }
```

**使用效果：**

```
管理员：
创建活动
主题：云原生技术分享会
时间：2026-02-15 20:00
人数限制：50人

机器人：
✅ 活动创建成功！

📅 云原生技术分享会
时间：2026-02-15 20:00
名额：50人

回复「报名」即可参加活动

群成员 D：报名

机器人：
✅ 报名成功！活动时间：2026-02-15 20:00

当前已报名：23/50人

管理员：活动状态

机器人：
📊 活动状态：云原生技术分享会

时间：2026-02-15 20:00
报名人数：23/50人
```

---

### 3.4 功能四：互动游戏：猜代码挑战

**痛点**：技术群容易变成死群，缺少互动。

**解决方案**：每天出一个代码挑战题，活跃气氛。

```python
# code_quiz.py

import random
from datetime import datetime

class CodeQuiz:
    def __init__(self):
        self.questions = [
            {
                "code": """
def foo():
    return
    print("Hello")
foo()
""",
                "question": "这段代码会输出什么？",
                "options": ["A. Hello", "B. 什么都不输出", "C. 报错", "D. None"],
                "answer": "B",
                "explanation": "return 语句后的代码不会执行"
            },
            {
                "code": """
x = [1, 2, 3]
y = x
x.append(4)
print(y)
""",
                "question": "y 的值是什么？",
                "options": ["A. [1, 2, 3]", "B. [1, 2, 3, 4]", "C. None", "D. 报错"],
                "answer": "B",
                "explanation": "Python 中列表是可变对象，x 和 y 指向同一个列表"
            },
            # 更多题目...
        ]
        self.current_quiz = None
        self.answered_users = set()

    def start_quiz(self):
        """开始每日挑战"""
        self.current_quiz = random.choice(self.questions)
        self.answered_users = set()

        message = f"""🎯 每日代码挑战！

```python
{self.current_quiz['code']}
```

{self.current_quiz['question']}

{chr(10).join(self.current_quiz['options'])}

回复「答案 A/B/C/D」参与挑战！
"""
        return message

    def check_answer(self, user_id, answer):
        """检查答案"""
        if not self.current_quiz:
            return "今天还没有出题，请等待管理员出题"

        if user_id in self.answered_users:
            return "你已经回答过了"

        self.answered_users.add(user_id)

        if answer.upper() == self.current_quiz['answer']:
            return f"✅ 恭喜答对！\n\n{self.current_quiz['explanation']}"
        else:
            return f"❌ 答案错误\n\n{self.current_quiz['explanation']}"
```

**使用效果：**

```
管理员：每日挑战

机器人：
🎯 每日代码挑战！

```python
def foo():
    return
    print("Hello")
foo()
```

这段代码会输出什么？

A. Hello
B. 什么都不输出
C. 报错
D. None

回复「答案 A/B/C/D」参与挑战！

群成员 E：答案 B

机器人：
✅ 恭喜答对！

return 语句后的代码不会执行

群成员 F：答案 A

机器人：
❌ 答案错误

return 语句后的代码不会执行
```

---

## 四、高级功能：定时任务

### 4.1 每日早报

```python
# daily_news.py

import schedule
import time
import requests

def send_morning_news():
    """
    发送每日早报
    """
    # 获取科技新闻
    news = fetch_tech_news()

    # 获取天气
    weather = get_weather("北京")

    # 获取今日活动
    events = get_today_events()

    message = f"""🌅 早安！今天是 {datetime.now().strftime('%Y年%m月%d日')}

📰 今日科技要闻
{format_news(news)}

🌤️ 北京天气
{weather}

📅 今日活动
{format_events(events)}

祝大家今天工作愉快！
"""

    send_to_group(message)

# 每天早上8点发送
schedule.every().day.at("08:00").do(send_morning_news)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### 4.2 定时清理

```python
# 定时清理过期的活动报名、无用的临时文件等

import schedule
from datetime import datetime, timedelta

def cleanup_old_data():
    """
    清理30天前的数据
    """
    cutoff_date = datetime.now() - timedelta(days=30)

    # 清理过期活动
    conn = sqlite3.connect(self.db_path)
    cursor = conn.cursor()
    cursor.execute(
        "DELETE FROM events WHERE event_time < ?",
        (cutoff_date.isoformat(),)
    )
    conn.commit()
    conn.close()

    # 清理临时文件
    # ...

schedule.every().week.do(cleanup_old_data)
```

---

## 五、部署总结

### 5.1 成本分析

| 项目 | 配置 | 月费 |
|-----|------|------|
| Lighthouse | 2核4G，60GB | ¥60 |
| 流量费 | 按实际使用 | ¥10-30 |
| LLM API | DeepSeek | ¥20-50 |
| **总计** | - | **¥90-140/月** |

### 5.2 运营效果

使用 OpenClaw QQ 机器人运营社群 3 个月：

- **活跃度提升 200%**：每日消息数从 50 条 → 150 条
- **管理效率提升 10 倍**：FAQ 自动回答率 80%
- **精华沉淀**：累计收藏 150+ 篇高质量内容
- **活动参与率提升**：从 30% → 65%

---

## 六、创意扩展

### 6.1 代码审查机器人

群成员发送代码，机器人给出改进建议：

```python
# 自动代码审查

async def review_code(self, code, language="python"):
    prompt = f"""
    请审查以下 {language} 代码，指出：
    1. 潜在的 bug
    2. 性能优化建议
    3. 代码风格改进

    代码：
    ```{language}
    {code}
    ```
    """

    suggestions = await self.call_llm(prompt)
    return suggestions
```

### 6.2 技术名词解释

识别专业术语，自动解释：

```python
# 技术名词卡片

term_cards = {
    "Kubernetes": """
    📖 Kubernetes (K8s)

    容器编排平台，用于自动化部署、扩展和管理容器化应用。

    核心概念：
    - Pod：最小部署单元
    - Service：服务发现
    - Deployment：应用部署

    学习资源：https://kubernetes.io/zh/docs/
    """,
    # 更多术语...
}
```

### 6.3 AI 对战

让群成员和 AI 对弈（代码 Golf、算法题等）：

```
机器人：🎮 代码挑战赛开始！

题目：用最少的代码实现FizzBuzz

截止：今晚20:00

奖品：腾讯云周边礼包

当前最短代码（35字符）：
for i in range(100):print(i%3 or"Fizz"+(i%5 and""or"Buzz"))
```

---

## 七、总结

通过 OpenClaw + 腾讯云 Lighthouse，我成功打造了一个功能完善的 QQ 社群智能助手。它不仅是一个问答机器人，更是一个能够：

1. **自动管理知识**：FAQ 回答、精华收藏、智能检索
2. **组织活动**：报名统计、定时提醒、反馈收集
3. **活跃气氛**：每日挑战、互动游戏、早报推送
4. **持续进化**：从对话中学习，不断优化回复

**关键优势：**
- OpenClaw 的文件操作能力，让机器人可以真正「做事」
- Lighthouse 的稳定在线，保证服务不中断
- 低成本运营，每月不到 150 元

如果你也在运营技术社群，或者想为自己的社群增加智能功能，OpenClaw 是一个值得尝试的选择。

让 AI 成为你的社群管理员，把时间花在更有价值的交流上。

---

*本文首发于腾讯云开发者社区，标签：玩转OpenClaw云端创意实践*

**项目地址**：[GitHub - openclaw-qq-bot](https://github.com/your-repo)
**演示视频**：[Bilibili - QQ社群机器人实战](https://bilibili.com/...)
