# Travel Planner v3 — AI 旅行助手架构设计

## 1. 核心问题

当前页面是**纯静态 HTML**，聊天面板里的"AI 对话"是硬编码的 if/else 规则引擎：
- 无法理解用户没预写过的自然语言
- 无法调整行程数据、推荐酒店、修改路线
- 用户觉得"完全不是在跟 AI 对话"

## 2. 目标

用户在聊天面板输入自然语言，后端 LLM 理解语义，实现：

| 能力 | 示例 |
|------|------|
| 调整行程 | "把屏山峡谷改到第二天" |
| 推荐酒店 | "推荐恩施的亲子酒店" |
| 景点问答 | "大峡谷门票多少钱" |
| 添加/删除 | "加一天去宜昌" |
| 修改路线 | "回程不走安康了" |
| 路线分析 | "第二天累不累" |
| 费用分析 | "总预算多少" |
| 自由对话 | "屏山峡谷好玩吗" |

## 3. 整体架构

```
┌─────────────────────────────────────────────────┐
│  Frontend (GitHub Pages)                        │
│  ┌─────────┐  ┌──────────────────────────────┐  │
│  │ Leaflet │  │  Chat Panel                   │  │
│  │  Map    │  │  ┌───────────────────────┐   │  │
│  │         │  │  │ User: "推荐恩施...   │   │  │
│  │  Locs   │◄─│  │ AI: [推荐结果]        │   │  │
│  │  S/I    │  │  └───────────────────────┘   │  │
│  │  route  │  │         ↕ POST /api/chat      │  │
│  └─────────┘  └──────────────┬───────────────┘  │
└──────────────────────────────┼──────────────────┘
                               │ HTTPS
┌──────────────────────────────┼──────────────────┐
│  Backend (Zeabur / Railway)  │                  │
│  ┌──────────────────────────────────────────┐   │
│  │  FastAPI Server                          │   │
│  │  POST /api/chat                          │   │
│  │  ┌──────────────────────────────────┐    │   │
│  │  │  1. 解析用户输入 + 当前行程JSON  │    │   │
│  │  │  2. 构造 System Prompt           │    │   │
│  │  │  3. 调 DeepSeek API              │    │   │
│  │  │  4. 解析 LLM 返回 (JSON)         │    │   │
│  │  │  5. 返回结构化响应               │    │   │
│  │  └──────────────────────────────────┘    │   │
│  │                                           │   │
│  │  POST /api/chat?mode=hotel               │   │
│  │  POST /api/recommend                     │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────┐                    │
│  │  hotel_db.json           │  ← 酒店/景点数据  │
│  │  (可选: 预置恩施酒店)    │                    │
│  └──────────────────────────┘                    │
└──────────────────────────────────────────────────┘
```

## 4. 后端 API 设计

### `POST /api/chat`

**Request:**
```json
{
  "message": "推荐恩施的亲子酒店",
  "itinerary": {
    "locs": { ... },
    "segments": { ... },
    "routeData": { ... }
  },
  "history": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ]
}
```

**Response (两种类型):**

**类型 A: 需要更新行程**
```json
{
  "type": "itinerary_update",
  "reply": "好的，已把屏山峡谷调整到第二天",
  "updates": {
    "locs": { ... },     // 修改后的 Locs
    "segments": { ... }, // 修改后的 S（路线坐标）
    "mapping": { ... }   // 修改后的 I（标记→路线关联）
  },
  "action": "reorder"
}
```

**类型 B: 仅文本回复**（推荐酒店、问答等）
```json
{
  "type": "text_reply",
  "reply": "恩施推荐以下亲子酒店：\n1. 恩施铂尔曼酒店 - 市中心，有儿童乐园\n2. 恩施紫荆国际酒店 - 靠近女儿城\n3. ...",
  "highlights": [
    { "name": "铂尔曼酒店", "lat": 30.27, "lng": 109.48, "type": "hotel" }
  ]
}
```

**类型 C: 错误**
```json
{
  "type": "error",
  "reply": "抱歉，我没理解你的意思，能再说一遍吗？"
}
```

## 5. Prompt 设计

```python
SYSTEM_PROMPT = """你是一个专业的中国自驾游AI旅行助手。

## 你的核心能力
1. 调整行程：重新排列景点、修改天数、添加/删除目的地
2. 推荐酒店/餐厅：根据预算、亲子需求等推荐
3. 回答景点问答：门票、时间、路线建议
4. 路线分析：每日驾车时间、距离、疲劳度
5. 费用估算：过路费、油费、住宿、门票
6. 天气/穿衣建议

## 当前行程数据
{itinerary_json}

## 用户消息
{user_message}

## 输出格式
你的回复必须是以下两种格式之一：

如果涉及行程数据变更（调整顺序、添加/删除、改天数）：
```json
{
  "type": "itinerary_update",
  "reply": "对用户说的人话",
  "updates": {
    "locs": { ... },  // 完整的新 Locs 对象
    "segments": { ... },
    "mapping": { ... }
  }
}
```

如果仅需文字回复（推荐、问答、分析）：
```json
{
  "type": "text_reply",
  "reply": "你的回答",
  "highlights": []  // 可选：要在地图上标记的点
}
```
"""
```

## 6. 前端改动

现有前端需要改的**最小**部分：

| 文件 | 改动 |
|------|------|
| `index.html` | `sendChat()` 改为调 `POST /api/chat`，不再用规则引擎 |
| | 新增 `applyItineraryUpdate(updates)` 函数，接受后端返回的行程 JSON，更新 Locs/S/I，重新渲染地图和列表 |
| | 新增 `addMapHighlights(points)` 函数，在地图上标记推荐酒店等 |

**`sendChat()` 新逻辑：**
```javascript
window.sendChat = async function() {
  const text = chatInput.value.trim();
  if (!text) return;

  addMessage('你', text, 'user');
  chatInput.value = '';
  chatSend.disabled = true;
  chatSend.textContent = '...';

  // 收集当前行程数据
  const payload = {
    message: text,
    itinerary: {
      locs: Locs,
      segments: S,
      mapping: I,
      routeData: routeData
    }
  };

  try {
    const res = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
    const data = await res.json();

    if (data.type === 'itinerary_update') {
      applyItineraryUpdate(data.updates);
      addMessage('🤖 助手', data.reply, 'assistant');
    } else if (data.type === 'text_reply') {
      addMessage('🤖 助手', data.reply, 'assistant');
      if (data.highlights?.length) {
        addMapHighlights(data.highlights);
      }
    }
  } catch(e) {
    addMessage('🤖 助手', '⚠️ 网络错误，请稍后再试', 'assistant');
  }

  chatSend.disabled = false;
  chatSend.textContent = '➤';
};
```

**`applyItineraryUpdate()` 新函数：**
```javascript
function applyItineraryUpdate(updates) {
  // 1. 清空现有地图标记和路线
  Object.values(mks).forEach(m => map.removeLayer(m));
  Object.values(segs).forEach(s => map.removeLayer(s));
  if (full) map.removeLayer(full);
  Object.values(routeLabels).forEach(l => map.removeLayer(l));

  // 2. 更新数据
  Object.assign(Locs, updates.locs);
  Object.assign(S, updates.segments);
  Object.assign(I, updates.mapping);

  // 3. 重新初始化——调用已有的 mkIcon/starIcn 等逻辑重建
  rebuildMap();

  // 4. 更新左侧行程列表
  rebuildItineraryList();

  // 5. 更新每日统计
  renderDailyStats();

  // 6. 更新总里程
  updateTotalStats();
}
```

## 7. 后端实现（Zeabur 部署）

### 目录结构
```
travel-planner-api/
├── main.py              # FastAPI 入口
├── requirements.txt     # fastapi, uvicorn, httpx, pydantic
├── prompts.py           # System prompt 模板
├── llm_client.py        # DeepSeek API 调用封装
├── hotel_db.py          # 酒店/景点数据（可选）
└── zeabur.json          # Zeabur 部署配置
```

### main.py (~80行)
```python
from fastapi import FastAPI
from pydantic import BaseModel
import httpx, json, os

app = FastAPI()

DEEPSEEK_API_KEY = os.environ["DEEPSEEK_API_KEY"]

class ChatRequest(BaseModel):
    message: str
    itinerary: dict
    history: list = []

@app.post("/api/chat")
async def chat(req: ChatRequest):
    # 1. 构造 prompt
    system = build_prompt(req.itinerary)
    
    # 2. 调 DeepSeek
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "https://api.deepseek.com/v1/chat/completions",
            headers={"Authorization": f"Bearer {DEEPSEEK_API_KEY}"},
            json={
                "model": "deepseek-chat",
                "messages": [
                    {"role": "system", "content": system},
                    {"role": "user", "content": req.message}
                ],
                "temperature": 0.7
            },
            timeout=30
        )
    
    # 3. 解析回复
    content = resp.json()["choices"][0]["message"]["content"]
    result = parse_llm_response(content)
    return result
```

### 部署
```bash
# Zeabur CLI
zeabur deploy
# 或 git push 到 Zeabur 关联的仓库
```

## 8. 酒店数据

在后端预置一份恩施酒店 JSON，LLM 可以根据用户需求从中推荐：

```json
{
  "hotels": [
    {"name": "恩施铂尔曼酒店", "stars": 5, "price": "¥500-800", 
     "tag": "亲子、市中心、有儿童乐园", "lat": 30.275, "lng": 109.478},
    {"name": "恩施紫荆国际酒店", "stars": 4, "price": "¥350-550",
     "tag": "近女儿城、性价比高", "lat": 30.268, "lng": 109.485},
    {"name": "安康江景酒店", "stars": 3, "price": "¥200-350",
     "tag": "江景房、近汉江", "lat": 32.683, "lng": 109.025}
  ]
}
```

LLM 回答时如果有酒店推荐，在 `highlights` 中返回坐标，前端在地图上标记。

## 9. 安全考虑

- API key 放在后端环境变量，不暴露到前端
- 加 rate limit 防止滥用
- 加 CORS 只允许 GitHub Pages 域名
- 加请求大小限制（itinerary JSON 可能很大）

## 10. 总结

| 组件 | 改动量 | 技术 |
|------|--------|------|
| 前端聊天面板 | ~50行 JS 修改 | 从规则引擎改为调 API |
| 前端数据更新 | ~60行 JS 新增 | applyItineraryUpdate() |
| 后端 API | ~150行 Python | FastAPI + DeepSeek |
| 部署 | 一次配置 | Zeabur / Railway |
| 酒店数据 | ~50行 JSON | 预置恩施数据 |

**总代码量：约 300 行新代码，不改原有地图逻辑。**
