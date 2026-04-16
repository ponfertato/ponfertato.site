---
title: "AI-боты: Архитектура интеграции в мессенджеры"
description: "Четыре реализации: Discord (Python), Pony.Town (TypeScript/Puppeteer), Telegram Business (PHP). Модульность, rate limiting, обход Cloudflare."
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
layout: single
---

## Обзор

Четыре проекта, разные платформы, общая цель: интеграция языковых моделей в чат-интерфейсы с минимальной задержкой и максимальной надёжностью.

| Проект               | Язык        | Платформа           | Ключевая задача                                       |
| -------------------- | ----------- | ------------------- | ----------------------------------------------------- |
| Discord-AI-Chatbot   | Python 3.10 | Discord API         | Гибридная маршрутизация, мульти-провайдер изображений |
| discord-ai (Compose) | Python 3.10 | Discord API         | Мульти-инстанс через Docker Compose, Traefik-ready    |
| pt-ai                | TypeScript  | Pony.Town (browser) | Обход Cloudflare, плагин-система, контекстные ответы  |
| telegram-ai-replier  | PHP 8.1     | Telegram Business   | Webhook-first, модульный AI-интерфейс, rate limiting  |

---

## Discord-AI-Chatbot: Гибридная маршрутизация сообщений

### Обработка входящих событий

Бот определяет необходимость ответа через комбинацию триггеров:

```python
# main.py
is_active_channel = string_channel_id in active_channels
is_allowed_dm = allow_dm and is_dm_channel
contains_trigger_word = any(word in message.content for word in trigger_words)
is_bot_mentioned = bot.user.mentioned_in(message) and smart_mention

if is_active_channel or is_allowed_dm or contains_trigger_word or is_bot_mentioned:
    # Обработка через AI
```

**Особенности**:

- `smart_mention` - игнорирует `@everyone`, реагирует только на прямые упоминания
- `active_channels` - динамический список, управляемый `/toggleactive`, хранится в `channels.json`
- Контекст диалога ограничивается `MAX_HISTORY` сообщений для контроля токенов

### Система личностей через внешние файлы

```python
# utilities/config_loader.py
def load_instructions(instruction):
    for file_name in os.listdir("instructions"):
        if file_name.endswith('.txt'):
            with open(f"instructions/{file_name}", 'r', encoding='utf-8') as file:
                instruction[file_name.split('.')[0]] = file.read()
```

**Преимущества**:

- Добавление новой личности без изменения кода - создать `.txt` в `instructions/`
- Мгновенное переключение через `/toggleactive persona:<name>`
- Изоляция промптов от бизнес-логики

### Генерация изображений: мульти-бэкенд

Поддержка трёх провайдеров через единый интерфейс в `utilities/ai_utils.py`:

| Провайдер           | Модель             | Особенности                              |
| ------------------- | ------------------ | ---------------------------------------- |
| **Prodia**          | SD 1.5, AnythingV4 | Контроль семплера, seed, negative prompt |
| **Pollinations.ai** | SDXL               | Без авторизации, быстрый прототипинг     |
| **DALL-E 3**        | OpenAI             | Высокое качество, соблюдение промпта     |

```python
async def generate_image_prodia(prompt, model, sampler, seed, neg):
    # 1. Создание джоба через API
    # 2. Polling статуса до 'succeeded'
    # 3. Загрузка бинарных данных в io.BytesIO
    # 4. Возврат file-like объекта для Discord
```

**Оптимизация**:

- `negative prompt` по умолчанию содержит ~200 токенов блокировок (nsfw, low quality)
- NSFW-фильтрация: проверка по `blacklisted_words` + флаг канала `ctx.channel.nsfw`

### Деплой: Replit + Docker

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

- Авто-детект среды Replit через `REPL_OWNER`
- Flask-сервер для keep-alive пингов в бесплатных облачных средах
- `.env` не коммитится - конфигурация через secrets в CI/CD

---

## discord-ai (Compose): Мульти-инстанс через изоляцию

### Архитектура контейнеров

Запуск нескольких независимых ботов (Celestia, Luna) в одном стеке:

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

**Преимущества**:

- Один образ - несколько инстансов с разными личностями и токенами
- `watchtower.enable: false` - ручной контроль обновлений
- `traefik.enable: false` - боты не экспозируются наружу, только внутренняя сеть

### Миграция на async OpenAI

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

**Улучшения**:

- Неблокирующий I/O - бот не зависает при ожидании ответа от AI
- Поддержка custom `base_url` - работа с локальными прокси
- Graceful error handling: возврат сообщения об ошибке вместо краша

### Динамическая загрузка моделей

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

- Команда `/imagine` автоматически получает актуальный список моделей
- Нет хардкода - добавление новой модели на бэкенде сразу доступно в боте

---

## pt-ai: Browser automation для Pony.Town

### Архитектура: браузер как клиент

Взаимодействие с игрой через headless Chromium:

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

**Почему Puppeteer**:

- Обход Cloudflare через эмуляцию реального браузера
- Прямое взаимодействие с DOM - не зависит от внутренних изменений API игры
- Возможность визуальной отладки (`headless: false`)

### Cloudflare Bypass модуль

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

**Особенности**:

- Детект челленджа через `div#cf-challenge-running`
- Автоматический клик по чекбоксу внутри iframe
- Экспоненциальная задержка между попытками

### Плагин-система с приоритетами

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

| Класс                 | Приоритет | Задача                                                       |
| --------------------- | --------- | ------------------------------------------------------------ |
| `EmoteHandler`        | 1000      | Реакция на эмоции через `/nod`, `/laugh`                     |
| `CommandHandler`      | 900       | Обработка `.ai`, `.echo` команд                              |
| `ChatResponseHandler` | 800       | Спонтанные ответы с вероятностью `CHAT_RESPONSE_PROBABILITY` |
| `AdvancedChatHandler` | 100       | Генерация контекстных наблюдений за окружением               |

### Генерация ответов: правила и ограничения

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

**Результат**:

- Ответы выглядят как сообщения реального игрока
- Жёсткие лимиты длины предотвращают спам
- Эмодзи заменяют пунктуацию - естественный стиль чата

---

## telegram-ai-replier: Модульный бот для бизнеса

### Архитектура: разделение ответственности

```
src/
├── Config/      # Валидация конфигурации, загрузка .env
├── Core/        # Bot, WebhookHandler, StatusPage - ядро
├── AI/          # Интерфейс + реализации: OpenAIProvider, OllamaProvider
└── Exceptions/  # Типизированные ошибки
```

**Преимущества**:

- Замена AI-провайдера без изменения бизнес-логики
- Тестируемость: каждый модуль изолирован
- Расширяемость: добавление нового провайдера - реализация `AIInterface`

### Мультипровайдер AI через интерфейс

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

**Конфигурация через .env**:

```dotenv
AI_PROVIDER=openai          # или 'ollama'
OPENAI_API_KEY=sk-...
OPENAI_BASE_URL=https://proxy.local/v1  # опционально
OLLAMA_URL=http://host.docker.internal:11434
```

### Обработка вебхуков: валидация и маршрутизация

```php
// src/Core/WebhookHandler.php
public function handle(array $update): void {
    if (isset($update['message'])) {
        $chatId = $update['message']['chat']['id'];
        $text = $update['message']['text'] ?? '';

        if (!$this->rateLimiter->allow($chatId)) {
            return; // Игнорировать спам
        }

        $response = $this->aiProvider->generateResponse($text, []);
        $this->bot->sendMessage($chatId, $response);
    }
}
```

**Rate limiting**:

- Хранение счётчиков в `data/rate_limit.json`
- Окно `RATE_LIMIT_WINDOW` секунд, максимум `RATE_LIMIT_MAX_REQUESTS` запросов
- Автоматическая очистка устаревших записей

### Деплой: Docker с персистентными данными

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

- `db.json` - хранение состояния (опционально)
- `rate_limit.json` - персистентность лимитов между рестартами
- Порт 8008 - легко проксируется через Nginx/Traefik с HTTPS

---

## Общие принципы проектов

1. **Конфигурация вне кода** - все параметры через `.env`, никаких хардкодов
2. **Интерфейсы вместо наследования** - лёгкая замена компонентов (AI-провайдеры, хендлеры)
3. **Изоляция состояний** - каждый бот/инстанс работает независимо
4. **Безопасность по умолчанию** - rate limiting, валидация ввода, блокировка опасных паттернов
5. **Деплой-агностик** - работает локально, на VPS, в Docker, в облаке

---

## Ссылки

- 🐙 [Discord-AI-Chatbot](https://github.com/ponfertato/Discord-AI-Chatbot)
- 🐙 [discord-ai (Compose)](https://github.com/laspegasuscommunity/discord-ai)
- 🐙 [pt-ai](https://github.com/potatoenergy/pt-ai)
- 🐙 [telegram-ai-replier](https://github.com/potatoenergy/telegram-ai-replier)
- 🤖 [Telegram Bot API](https://core.telegram.org/bots/api)
- 🎭 [Puppeteer Docs](https://pptr.dev)
- 🤯 [Ollama API](https://github.com/ollama/ollama/blob/main/docs/api.md)
