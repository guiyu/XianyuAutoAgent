# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

XianyuAutoAgent is an AI-powered customer service automation system for Xianyu (闲鱼, Alibaba's second-hand marketplace). It provides 24/7 intelligent customer support with multi-expert routing, price negotiation, and technical consultation capabilities.

**Tech Stack:** Python 3.8+, OpenAI SDK (compatible with Alibaba Qwen models), WebSockets, SQLite, asyncio

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run the application
python main.py

# Docker deployment
docker-compose up -d
```

## Architecture

### Core Files

- **main.py** - Entry point. `XianyuLive` class manages WebSocket connection to Xianyu's Dingding server, handles message routing, token refresh (1-hour interval), and heartbeat (15-second interval)
- **XianyuAgent.py** - LLM orchestration. Contains `XianyuReplyBot` (main orchestrator), `IntentRouter` (three-level routing: keyword → regex → LLM), and agent classes (`ClassifyAgent`, `PriceAgent`, `TechAgent`, `DefaultAgent`)
- **XianyuApis.py** - Xianyu platform API client for token retrieval, login validation, and product info fetching
- **context_manager.py** - SQLite-based conversation storage with bargain count tracking per chat

### Agent System

```
BaseAgent (template method pattern)
├── ClassifyAgent    # Intent detection → returns: 'tech', 'price', 'default', 'no_reply'
├── PriceAgent       # Price negotiation with dynamic temperature (0.3-0.9 based on bargain rounds)
├── TechAgent        # Technical support with web search capability
└── DefaultAgent     # General customer service inquiries
```

**Intent Routing Priority:** keyword matching (tech > price) → regex patterns → LLM classification fallback

### Prompt System

Prompts are in `prompts/` directory. The system loads custom prompts (`*_prompt.txt`) first, falling back to example templates (`*_prompt_example.txt`):
- `classify_prompt.txt` - Intent classification
- `price_prompt.txt` - Price negotiation expert
- `tech_prompt.txt` - Technical support expert
- `default_prompt.txt` - Default conversation agent

### Message Flow

1. WebSocket receives encrypted message (Base64 + MessagePack)
2. `utils/xianyu_utils.py` decrypts and deserializes
3. Message validated (age < 5 min, not system message, sender is customer)
4. `IntentRouter` determines agent
5. Selected agent generates response with conversation context
6. Safety filter applied (blocks external contact info)
7. Response sent via WebSocket, stored in SQLite

### Key Configuration (.env)

Required:
- `API_KEY` - LLM API key (default: Alibaba Qwen)
- `COOKIES_STR` - Xianyu web cookies

Optional:
- `MODEL_BASE_URL` - defaults to dashscope.aliyuncs.com
- `MODEL_NAME` - defaults to qwen-max
- `TOGGLE_KEYWORDS` - seller takeover trigger (default: "。")
- `SIMULATE_HUMAN_TYPING` - add typing delay (default: False)

### Manual Mode

Seller can send toggle keyword (default: "。") to take over a conversation. AI auto-resumes after timeout (default: 1 hour) or another toggle.

## Code Patterns

- Full async/await for WebSocket and API operations
- Template method pattern in `BaseAgent._build_messages()` / `_call_llm()` / `generate()`
- SQLite with indexed queries for conversation retrieval (max 100 messages per chat)
- Safety filtering blocks phrases like "微信", "QQ", "支付宝" to prevent off-platform transactions
