---
title: "Hugo: Архитектура статических сайтов и CI/CD"
description: "Практика использования Hugo: мультиязычность, cascade, Hugo Pipes, кастомные шорткоды, сборка в Docker."
tags: ["hugo", "static", "docker", "ci-cd", "blowfish", "papermod"]
---

Использование Hugo для генерации статических сайтов позволяет достичь максимальной производительности при минимальных накладных расходах. Все проекты на стеке `*.potatoenergy.ru` используют единую методологию сборки, конфигурации и деплоя.

**Ключевые принципы**:

- 🔹 Декларативная конфигурация через `hugo.toml` и `config/_default/`
- 🔹 Мультиязычность через `translationKey` без дублирования логики
- 🔹 Наследование параметров через `cascade` во фронтматтере
- 🔹 Оптимизация ассетов через Hugo Pipes (минификация, фолбэки, хеширование)
- 🔹 Изоляция окружения сборки через Docker

---

## 🗂 Структура проекта

```
project/
├── config/_default/
│   ├── hugo.toml          # Базовые настройки: baseURL, theme, outputs
│   ├── params.toml        # Глобальные параметры темы и кастомные переменные
│   ├── menus.toml         # Конфигурация навигации
│   └── languages.*.toml   # Локализация: displayName, copyright, author
├── content/
│   ├── {section}/
│   │   ├── _index.{ru,en}.md  # Страница-листинг с описанием раздела
│   │   └── {slug}/
│   │       ├── index.{ru,en}.md  # Контент с переводом через translationKey
│   │       └── assets/           # Ресурсы, привязанные к странице
│   └── _index.{ru,en}.md    # Главная страница
├── layouts/
│   ├── _default/            # Шаблоны по умолчанию: single.html, list.html
│   ├── partials/            # Переиспользуемые компоненты: head, footer, svg
│   ├── shortcodes/          # Кастомные шорткоды: audio, toggle, icon
│   └── shortcodes/          # Переопределение шаблонов темы (при необходимости)
├── static/                  # Файлы, копируемые «как есть»: robots.txt, favicons
├── assets/                  # Исходники для Hugo Pipes: CSS/JS для сборки
└── docker-compose.yml       # Окружение для локальной разработки
```

---

## 🌐 Мультиязычность без дублирования

Связь переводов реализуется через `translationKey` во фронтматтере:

```yaml
# content/blog/post/_index.ru.md
title: "Настройка NixOS"
translationKey: "nixos-setup"
lang: ru

# content/blog/post/_index.en.md
title: "NixOS Configuration"
translationKey: "nixos-setup"
lang: en
```

**Преимущества**:

- ✅ Единый `slug` для обеих версий - чистые URL без `/ru/`, `/en/` в пути
- ✅ Автоматическая генерация `hreflang` для поисковых систем
- ✅ Переключатель языков в теме работает «из коробки»
- ✅ Фоллбэк на язык по умолчанию, если перевод отсутствует

**Конфигурация в `hugo.toml`**:

```toml
defaultContentLanguage = "ru"
[languages]
  [languages.ru]
    disabled = false
    languageName = "Русский"
    weight = 1
  [languages.en]
    disabled = false
    languageName = "English"
    weight = 2
```

---

## 📦 Cascade: наследование параметров

Параметры, заданные в `_index.md` раздела, автоматически применяются ко всем вложенным страницам через `cascade`:

```yaml
# content/projects/_index.ru.md
---
title: "Проекты"
cascade:
  showDate: false
  showReadingTime: false
  layout: "single"
  tags: ["projects"]
---
```

**Результат**:

- Все статьи в `/projects/*` наследуют `showDate: false`, `layout: "single"`
- Не нужно дублировать параметры в каждом файле
- Легко менять поведение целого раздела правкой одного файла

**Приоритет параметров**:

```
Страница > _index раздела > config/_default/params.toml > тема по умолчанию
```

---

## 🎨 Кастомные шорткоды

Шорткоды позволяют встраивать сложный контент без изменения шаблонов.

### Пример: `layouts/shortcodes/audio.html`

```go
{{ $src := .Get "src" }}
{{ $caption := .Get "caption" }}
<figure class="audio">
  <audio controls preload="metadata">
    <source src="{{ $src }}" type="audio/mpeg">
  </audio>
  {{ with $caption }}<figcaption>{{ . | markdownify }}</figcaption>{{ end }}
</figure>
```

### Пример: `layouts/shortcodes/toggle.html`

```go
{{ $title := .Get "title" }}
{{ $content := .Inner }}
<details>
  <summary>{{ $title }}</summary>
  {{ $content | markdownify }}
</details>
```

**Преимущества**:

- ✅ Инкапсуляция HTML/CSS/JS внутри шорткода
- ✅ Переиспользование в любом месте контента
- ✅ Безопасность: контент рендерится через `markdownify`, экранируются скрипты

---

## ⚡ Оптимизация ассетов через Hugo Pipes

Hugo Pipes позволяет обрабатывать CSS/JS/Images на этапе сборки:

```go
{{ $css := resources.Get "css/main.css" }}
{{ $css = $css | resources.ToCSS | resources.Minify | resources.Fingerprint }}
<link rel="stylesheet" href="{{ $css.RelPermalink }}" integrity="{{ $css.Data.Integrity }}">
```

**Этапы обработки**:

1. `resources.Get` - загрузка исходного файла из `assets/`
2. `ToCSS` - компиляция SCSS/Sass (если используется)
3. `Minify` - удаление пробелов, комментариев, сокращение имен
4. `Fingerprint` - добавление хеша к имени файла для cache-busting
5. `RelPermalink` - генерация относительного пути для HTML

**Результат**:

- Финальный файл: `/css/main.min.abc123.css`
- `integrity`-атрибут для Subresource Integrity (SRI)
- Автоматическая инвалидация кэша при изменении контента

---

## 🐳 Сборка в Docker: изоляция и воспроизводимость

### `docker-compose.yml` для локальной разработки

```yaml
services:
  hugo:
    command: server --appendPort=false --baseURL=http://localhost:1313 --buildDrafts --disableFastRender
    container_name: hugo
    image: ghcr.io/hugomods/hugo:base
    ports:
      - "1313:1313"
    restart: unless-stopped
    user: "1000:1002"
    volumes:
      - .:/src
```

**Разбор параметров**:

| Параметр              | Описание                                                                |
| --------------------- | ----------------------------------------------------------------------- |
| `--appendPort=false`  | Не добавлять `:1313` к `baseURL` - чистые ссылки в превью               |
| `--buildDrafts`       | Включать черновики (`draft: true`) в локальной сборке                   |
| `--disableFastRender` | Полная пересборка при изменениях - надёжнее для отладки шаблонов        |
| `user: '1000:1002'`   | Запуск от обычного пользователя - файлы создаются с правильными правами |
| `volumes: - .:/src`   | Монтирование проекта в контейнер - изменения видны мгновенно            |

**Почему `ghcr.io/hugomods/hugo:base`**:

- ✅ Официальный образ с поддержкой Hugo Extended (нужен для Sass, WebP)
- ✅ Минимальный размер (~150 МБ) - быстрая загрузка
- ✅ Стабильные теги версий - воспроизводимость сборок

### Деплой: одна команда

```bash
# Локальный запуск
docker compose up -d

# Сборка продакшена
docker compose run --rm hugo --minify --gc

# Копирование артефактов
rsync -avz public/ user@server:/var/www/site/
```

---

## 🔧 CI/CD: автоматизация через GitHub Actions

### `.github/workflows/deploy.yml`

```yaml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive # Для тем как git-submodule
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.125.0"
          extended: true
      - run: hugo --minify --gc
      - uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: deploy
          key: ${{ secrets.SSH_KEY }}
          source: "public/*"
          target: "/var/www/site"
          rm: true
```

**Особенности**:

- `submodules: recursive` - автоматическая инициализация тем как подмодулей
- `--gc` - удаление неиспользуемых ресурсов (очистка кэша)
- `scp-action` - атомарный деплой через SSH с удалением старых файлов

---

## 🛠 Отладка и диагностика

```bash
# Проверка конфигурации
hugo config --format yaml | grep -A5 languages

# Просмотр сгенерированных путей
hugo --printPathWarnings

# Поиск битых ссылок
hugo --minify 2>&1 | grep -i "error\|warning"

# Проверка целостности ассетов
grep -r "integrity=" public/ | wc -l

# Локальный просмотр с черновиками
docker compose run --rm hugo server --buildDrafts --bind 0.0.0.0
```

---

## ⚠️ Частые проблемы

```bash
# Шаблон не применяется
→ Проверить имя файла: layouts/{section}/single.html, не layouts/{section}.html
→ Убедиться, что в front matter нет `layout: "другой"`

# Переводы не переключаются
→ Проверить `translationKey`: должен совпадать в обеих версиях
→ Убедиться, что `defaultContentLanguage` задан в `hugo.toml`

# Ассеты не минифицируются
→ Проверить, что используется Hugo Extended: `hugo version | grep extended`
→ Убедиться, что в шаблоне есть `resources.Minify`

# Docker: файлы создаются от root
→ Добавить `user: '1000:1002'` в docker-compose.yml
→ Или запустить: `docker compose run --user $(id -u):$(id -g) ...`

# Кэш браузера не сбрасывается
→ Убедиться, что в шаблоне используется `resources.Fingerprint`
→ Проверить заголовки сервера: `Cache-Control: no-cache` для `*.css`, `*.js`
```

---

## Ссылки

- 📘 [Hugo Documentation](https://gohugo.io/documentation/)
- 🎨 [Blowfish Theme Docs](https://blowfish.page/docs/)
- 🧩 [PaperMod Theme](https://github.com/adityatelange/hugo-PaperMod)
- 🐳 [Hugo Docker Images](https://github.com/hugomods/docker-hugo)
- 🔧 [Hugo Pipes Guide](https://gohugo.io/hugo-pipes/)
