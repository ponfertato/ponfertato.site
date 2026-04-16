---
title: "AI Bots: Messenger and Games Integration Architecture"
description: "Four implementations: Discord (Python), Pony.Town (TypeScript/Puppeteer), Telegram Business (PHP). Modularity, rate limiting, Cloudflare bypass."
tags:
  [
    "ai",
    "bots",
    "discord",
    "telegram",
    "puppeteer",
    "python",
    "php",
    "typescript",
  ]
---

Four projects, different platforms, one goal: integrating language models into chat interfaces with minimal latency and maximum reliability.

| Project              | Language    | Platform            | Key Task                                               |
| -------------------- | ----------- | ------------------- | ------------------------------------------------------ |
| Discord-AI-Chatbot   | Python 3.10 | Discord API         | Hybrid routing, multi-provider image generation        |
| discord-ai (Compose) | Python 3.10 | Discord API         | Multi-instance via Docker Compose, Traefik-ready       |
| pt-ai                | TypeScript  | Pony.Town (browser) | Cloudflare bypass, plugin system, contextual responses |
| telegram-ai-replier  | PHP 8.1     | Telegram Business   | Webhook-first, modular AI interface, rate limiting     |

---

## Discord-AI-Chatbot: Hybrid Message Routing

### Incoming Event Processing

The bot determines response necessity through trigger combination:

```python
# main.py
is_active_channel = string_channel_id in active_channels
is_allowed_dm = allow_dm and is_dm_channel
contains_trigger_word = any(word in message.content for word in trigger_words)
is_bot_mentioned = bot.user.mentioned_in(message) and smart_mention

if is_active_channel or is_allowed_dm or contains_trigger_word or is_bot_mentioned:
    # Process via AI
```

**Features**:

- `smart_mention` — ignores `@everyone`, responds only to direct mentions
- `active_channels` — dynamic list managed via `/toggleactive`, stored in `channels.json`
- Dialogue context limited to `MAX_HISTORY` messages for token control

### Personality System via External Files

```python
# utilities/config_loader.py
def load_instructions(instruction):
    for file_name in os.listdir("instructions"):
        if file_name.endswith('.txt'):
            with open(f"instructions/{file_name}", 'r', encoding='utf-8') as file:
                instruction[file_name.split('.')[0]] = file.read()
```

**Benefits**:

- Add new personality without code changes — create `.txt` in `instructions/`
- Instant switching via `/toggleactive persona:<name>`
- Prompt isolation from business logic

### Image Generation: Multi-Backend

Support for three providers via unified interface in `utilities/ai_utils.py`:

| Provider            | Model              | Features                               |
| ------------------- | ------------------ | -------------------------------------- |
| **Prodia**          | SD 1.5, AnythingV4 | Sampler control, seed, negative prompt |
| **Pollinations.ai** | SDXL               | No auth, fast prototyping              |
| **DALL-E 3**        | OpenAI             | High quality, prompt adherence         |

```python
async def generate_image_prodia(prompt, model, sampler, seed, neg):
    # 1. Create job via API
    # 2. Poll status until 'succeeded'
    # 3. Load binary data into io.BytesIO
    # 4. Return file-like object for Discord
```

**Optimization**:

- `negative prompt` defaults to ~200 tokens of blocks (nsfw, low quality)
- NSFW filtering: check against `blacklisted_words` + channel flag `ctx.channel.nsfw`

### Deployment: Replit + Docker

```yaml
# docker-compose.yml
services:
  bot:
    build: .
    restart: unless-stopped
    env_file: .env
    command: >
      sh -c "if [ -n \"$REPL_OWNER\" ]; then
               python utilities/replit_flask_runner.py & python main.py;
             else
               python main.py;
             fi"
```

- Auto-detect Replit environment via `REPL_OWNER`
- Flask server for keep-alive pings in free cloud tiers
- `.env` not committed — config via secrets in CI/CD

---

## discord-ai (Compose): Multi-Instance via Isolation

### Container Architecture

Running multiple independent bots (Celestia, Luna) in one stack:

```yaml
# compose.yml
services:
  celestia:
    image: ponfertato/discord-ai:main
    env_file: celestia.env
    labels:
      com.centurylinklabs.watchtower.enable: "false"
      traefik.enable: "false"

  luna:
    image: ponfertato/discord-ai:main
    env_file: luna.env
```

**Benefits**:

- One image — multiple instances with different personalities and tokens
- `watchtower.enable: false` — manual update control
- `traefik.enable: false` — bots not exposed externally, internal network only

### Migration to Async OpenAI

```python
# bot_utilities/ai_utils.py
from openai import AsyncOpenAI

openai_client = AsyncOpenAI(
    api_key=os.getenv('CHIMERA_GPT_KEY'),
    base_url="https://api.naga.ac/v1"  # Custom proxy endpoint
)

async def generate_response(instructions, search, history):
    messages = [
        {"role": "system", "name": "instructions", "content": instructions},
        *history,
        {"role": "system", "name": "search_results", "content": search_results},
    ]
    response = await openai_client.chat.completions.create(
        model=os.getenv('GPT_MODEL'),
        messages=messages
    )
    return response.choices[0].message.content
```

**Improvements**:

- Non-blocking I/O — bot doesn't hang waiting for AI response
- Custom `base_url` support — work with local proxies
- Graceful error handling: return error message instead of crash

### Dynamic Model Loading

```python
def fetch_chat_models():
    headers = {'Authorization': f'Bearer {CHIMERA_GPT_KEY}'}
    response = requests.get('https://api.naga.ac/v1/models', headers=headers)
    if response.status_code == 200:
        return [
            model['id'] for model in response.json().get('data')
            if "chat" in model['endpoints'][0]
        ]
    return []
```

- `/imagine` command automatically receives current model list
- No hardcoded values — new backend models immediately available in bot

---

## pt-ai: Browser Automation for Pony.Town

### Architecture: Browser as Client

Interaction with game via headless Chromium:

```typescript
// src/services/browser/index.ts
export class BrowserService {
  static async launch() {
    return await puppeteer.launch({
      executablePath: CONFIG.CHROMIUM_PATH,
      headless: false,
      args: [
        "--no-sandbox",
        "--disable-setuid-sandbox",
        "--disable-dev-shm-usage",
        `--user-agent=${CONFIG.BROWSER.USER_AGENT}`,
      ],
    });
  }
}
```

**Why Puppeteer**:

- Cloudflare bypass via real browser emulation
- Direct DOM interaction — independent of game API changes
- Visual debugging capability (`headless: false`)

### Cloudflare Bypass Module

```typescript
// src/modules/cloudflare/bypass.ts
export class CloudflareBypasser {
  async bypass(): Promise<boolean> {
    for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
      if (await this.isCloudflareChallengePresent()) {
        await this.solveChallenge();
        await sleep(CHECK_INTERVAL);
        continue;
      }
      return true;
    }
    return false;
  }
}
```

**Features**:

- Challenge detection via `div#cf-challenge-running`
- Automatic checkbox click inside iframe
- Exponential backoff between attempts

### Plugin System with Priorities

```typescript
// src/modules/plugins/manager.ts
async handleMessage(message: ChatMessage): Promise<boolean> {
  const sortedHandlers = [...this.handlers].sort((a, b) =>
    b.getPriority() - a.getPriority()
  );
  for (const handler of sortedHandlers) {
    if (await handler.shouldHandle(message)) {
      this.lastHandler = handler.constructor.name;
      const result = await handler.handle(message);
      if (result) return true;
    }
  }
  return false;
}
```

| Class                 | Priority | Task                                                   |
| --------------------- | -------- | ------------------------------------------------------ |
| `EmoteHandler`        | 1000     | React to emotions via `/nod`, `/laugh`                 |
| `CommandHandler`      | 900      | Handle `.ai`, `.echo` commands                         |
| `ChatResponseHandler` | 800      | Spontaneous responses with `CHAT_RESPONSE_PROBABILITY` |
| `AdvancedChatHandler` | 100      | Generate contextual environment observations           |

### Response Generation: Rules and Constraints

```typescript
// src/modules/chat/generator.ts
private buildPrompt(text: string, context: string): string {
  return [
    `${CONFIG.PERSONALITY_NAME} Context-Aware Response Rules:`,
    `Response MUST:`,
    `1. Be 1-2 casual phrases (110-140 chars)`,
    `2. Use spoken language with flow errors`,
    `3. Reflect environment context: ${context}`,
    `4. Add ONE relevant emoji at end (replaces punctuation)`,
    `5. Format: single line without markdown`,
  ].join('\n');
}
```

**Result**:

- Responses look like real player messages
- Strict length limits prevent spam
- Emojis replace punctuation — natural chat style

---

## telegram-ai-replier: Modular Bot for Business

### Architecture: Separation of Concerns

```
src/
├── Config/      # Config validation, .env loading
├── Core/        # Bot, WebhookHandler, StatusPage — core
├── AI/          # Interface + implementations: OpenAIProvider, OllamaProvider
└── Exceptions/  # Typed errors
```

**Benefits**:

- Swap AI provider without changing business logic
- Testability: each module isolated
- Extensibility: add new provider by implementing `AIInterface`

### Multi-Provider AI via Interface

```php
// src/AI/AIInterface.php
interface AIInterface {
    public function generateResponse(string $prompt, array $context): string;
}

// src/AI/OpenAIProvider.php
class OpenAIProvider implements AIInterface {
    public function generateResponse(string $prompt, array $context): string {
        $client = new OpenAI(getenv('OPENAI_API_KEY'));
        $response = $client->chat()->create([
            'model' => getenv('OPENAI_MODEL'),
            'messages' => [
                ['role' => 'system', 'content' => getenv('AI_SYSTEM_PROMPT')],
                ['role' => 'user', 'content' => $prompt]
            ]
        ]);
        return $response->choices[0]->message->content;
    }
}
```

**Configuration via .env**:

```dotenv
AI_PROVIDER=openai          # or 'ollama'
OPENAI_API_KEY=sk-...
OPENAI_BASE_URL=https://proxy.local/v1  # optional
OLLAMA_URL=http://host.docker.internal:11434
```

### Webhook Handling: Validation and Routing

```php
// src/Core/WebhookHandler.php
public function handle(array $update): void {
    if (isset($update['message'])) {
        $chatId = $update['message']['chat']['id'];
        $text = $update['message']['text'] ?? '';

        if (!$this->rateLimiter->allow($chatId)) {
            return; // Ignore spam
        }

        $response = $this->aiProvider->generateResponse($text, []);
        $this->bot->sendMessage($chatId, $response);
    }
}
```

**Rate Limiting**:

- Counters stored in `data/rate_limit.json`
- Window `RATE_LIMIT_WINDOW` seconds, max `RATE_LIMIT_MAX_REQUESTS` per chat
- Automatic cleanup of expired entries

### Deployment: Docker with Persistent Data

```yaml
# docker-compose.yml
services:
  telegram-ai-replier:
    build: .
    volumes:
      - ./data/db.json:/app/data/db.json
      - ./data/rate_limit.json:/app/data/rate_limit.json
    ports:
      - "8008:80"
```

- `db.json` — state storage (optional)
- `rate_limit.json` — limit persistence across restarts
- Port 8008 — easily proxied via Nginx/Traefik with HTTPS

---

## Common Project Principles

1. **Config outside code** — all parameters via `.env`, no hardcoded values
2. **Interfaces over inheritance** — easy component replacement (AI providers, handlers)
3. **State isolation** — each bot/instance operates independently
4. **Security by default** — rate limiting, input validation, dangerous pattern blocking
5. **Deployment-agnostic** — runs locally, on VPS, in Docker, in cloud

---

## Links

- 🐙 [Discord-AI-Chatbot](https://github.com/ponfertato/Discord-AI-Chatbot)
- 🐙 [discord-ai (Compose)](https://github.com/laspegasuscommunity/discord-ai)
- 🐙 [pt-ai](https://github.com/potatoenergy/pt-ai)
- 🐙 [telegram-ai-replier](https://github.com/potatoenergy/telegram-ai-replier)
- 🤖 [Telegram Bot API](https://core.telegram.org/bots/api)
- 🎭 [Puppeteer Docs](https://pptr.dev)
- 🤯 [Ollama API](https://github.com/ollama/ollama/blob/main/docs/api.md)
