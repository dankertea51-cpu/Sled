<p align="center">
  <img src="https://img.shields.io/badge/version-1.0.0-blue" alt="Version">
  <img src="https://img.shields.io/badge/python-3.11%2B-blue" alt="Python">
  <img src="https://img.shields.io/badge/license-MIT-green" alt="License">
  <img src="https://img.shields.io/badge/status-production-brightgreen" alt="Status">
  <img src="https://img.shields.io/badge/OSINT-ready-orange" alt="OSINT">
</p>

<h1 align="center">След — OSINT-фреймворк для анализа цифровых следов</h1>

<p align="center">
  <b>Без API-ключей. Без настроек. Из коробки.</b><br>
  Поиск по email / username / телефону / криптокошелькам / IP / доменам
</p>

---

## Возможности

- **Email**: анализ формата, disposable-детекция, Gravatar, GitHub commits, paste-сайты, 9+ утечек, извлечение имени
- **Username**: проверка существования на **80+ платформах** (GitHub, Telegram, Instagram, YouTube, VK, TikTok, Reddit, Twitter, Steam, Twitch и др.)
- **Телефон**: нормализация, регион РФ, оператор, Telegram-аккаунт
- **Криптокошельки**: автоопределение блокчейна (BTC, ETH, TRON, BSC, SOL), баланс, транзакции
- **Интеллект**: риск-скоринг (5 уровней), детектор паттернов (мулы/дропы/организаторы), анализ графа (PageRank, кластеризация)
- **Отчёты**: Markdown, HTML (интерактивный граф), PDF, JSON

---

## Быстрый старт

```bash
# Установка
pip install sled

# Или из репозитория
git clone https://github.com/yourname/sled
cd sled && pip install -e .

# Расследование email
sled investigate email test@example.com

# Расследование username
sled investigate username torvalds

# Расследование телефона
sled investigate phone "+79123456789"

# Криптокошелёк
sled investigate crypto 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa

# Сохранить результат и сгенерировать отчёт
sled investigate email user@gmail.com -o case.json
sled report case-001 case.json -f md
```

---

## Пример работы

```bash
$ sled investigate email admin@company.com

┌──────────────────────────────────────────────┐
│  Расследование: email = admin@company.com     │
└──────────────────────────────────────────────┘
Модуль: email_provider — Анализ email: синтаксис, домен, Gravatar, ...
+ Найдено: 14 сущностей

Найдено сущностей: 14

Типы объектов: domain=1, email=1, leak_record=9, person=1, social_profile=2
Источники: email_provider(14)

Утечки данных:
┌──────────────┬──────────────────────┬──────────┐
│ Сервис       │ Что утекло           │ Дата     │
├──────────────┼──────────────────────┼──────────┤
│ Collection#1 │ email, password      │ 2019-01  │
│ LinkedIn     │ email, name, phone   │ 2021-06  │
│ Facebook     │ phone, email, name   │ 2021-04  │
│ ...          │                      │          │
└──────────────┴──────────────────────┴──────────┘

Профили в соцсетях:
┌──────────┬───────────────┬──────────────────────────────────┐
│ Платформа│ Username      │ Профиль                          │
├──────────┼───────────────┼──────────────────────────────────┤
│ gravatar │ admin@...     │ https://gravatar.com/...         │
│ github   │ admin@...     │ https://github.com/...           │
└──────────┴───────────────┴──────────────────────────────────┘

Все объекты:
┌──────────────┬────────────────────────┬──────────────┬──────────┬───────┐
│ Тип          │ Значение               │ Достоверность│ Источник │ Риск  │
├──────────────┼────────────────────────┼──────────────┼──────────┼───────┤
│ email        │ admin@company.com      │ 100%         │ email_pr.│ 23 CRIT│
│ domain       │ company.com            │ 100%         │ email_pr.│  0 SAFE│
│ person       │ Admin                  │  35%         │ email_pr.│ 12 MED │
│ social_pr.   │ admin@... gravatar     │  65%         │ email_pr.│ 10 LOW │
│ leak_record  │ admin@... Collection#1 │  35%         │ email_pr.│ 50 HIGH│
│ leak_record  │ admin@... LinkedIn     │  35%         │ email_pr.│ 50 HIGH│
└──────────────┴────────────────────────┴──────────────┴──────────┴───────┘
```

---

## Архитектура

```
sled/                        # ← установка: pip install sled
├── core/                    # Ядро: Entity, Graph, Registry, Scoring
│   ├── entities.py          # 13 типов сущностей
│   ├── graph.py             # EntityGraph (NetworkX)
│   ├── registry.py          # TransformRegistry
│   └── scoring.py           # RiskScore (5 уровней)
├── modules/                 # Провайдеры данных (без API-ключей)
│   ├── email/provider.py    # Gravatar, GitHub, paste, утечки
│   ├── username/provider.py # 80+ платформ, GET+парсинг HTML
│   ├── phone/provider.py    # Регион РФ, оператор, Telegram
│   ├── crypto/provider.py   # BTC/ETH/TRON/BSC/SOL
│   ├── leaks/provider.py    # Встроенная база утечек
│   ├── social/provider.py   # VK API
│   └── devices/provider.py  # User-Agent парсинг
├── transforms/              # Трансформации между сущностями
├── intelligence/            # RiskEngine, PatternDetector, LinkAnalyzer
├── reporting/               # MD, HTML (Cytoscape.js), PDF, JSON
├── cli/                     # Typer (5 команд)
└── api/                     # FastAPI (3 эндпоинта)
```

### Конвейер

```
Ввод (CLI/API)
    │
    ▼
Seed Entity ──► Provider.Query ──► Transforms ──► EntityGraph
    │              │                    │              │
    │         Gravatar            email→person    NetworkX
    │         GitHub              email→domain    PageRank
    │         Pastebin            phone→tg        Clusters
    │         80+ платформ        username→prof
    │
    ▼
RiskEngine ──► PatternDetector ──► Output
   scoring         mule/drop        CLI (Rich)
   factors         organizer        MD/HTML/PDF/JSON
                                    API response
```

---

## Email: глубокий анализ

| Признак | Тег | Пример |
|---------|-----|--------|
| Одноразовый домен | `disposable` | `user@guerrillamail.com` |
| Содержит год/число | `contains_numeric` | `user2023@gmail.com` |
| Плюс-теггинг | `plus_tagged` | `user+spam@mail.com` |
| Ролевой адрес | `role_based` | `admin@company.com` |
| Имя-фамилия | `likely_name_format` | `ivan.ivanov@mail.ru` |
| Короткий local-part | `short_local` | `a@domain.com` |
| Случайный | `randomized_local` | `qwertyuiop@mail.com` |

**Внешние источники** (без ключей):
- Gravatar: GET `gravatar.com/avatar/{md5}?d=404` + JSON-профиль
- GitHub: GET `api.github.com/search/commits?q={email}`
- Pastebin: GET `psbdmp.ws/api/search/{email}`
- Утечки: 9 известных бречей (Collection #1, LinkedIn, Facebook, Adobe, Dropbox и др.)

---

## Username: 80+ платформ

Реальная проверка через GET + парсинг HTML. Кастомные детекторы:

| Платформа | Детектор |
|-----------|----------|
| GitHub | `itemprop="name"` в HTML |
| Telegram | `<meta property="tgme_page"` |
| Instagram | `__INITIAL_STATE__` в script |
| YouTube | `channel` / `subscriber` в title |
| Twitter/X | `profile_sidebar` |
| VK | не содержит "не найдено" |
| TikTok | `webapp.user-detail` |
| Reddit | `profile-header` |
| Steam | `profile_content` |

Полный список: GitHub, GitLab, BitBucket, Telegram, Discord, Instagram,
Twitter/X, TikTok, YouTube, Reddit, Pinterest, LinkedIn, Facebook, VK, OK,
Twitch, Steam, Snapchat, Medium, Docker Hub, Keybase, Replit, Glitch,
Codepen, Behance, Dribbble, Flickr, DeviantArt, SoundCloud, Mixcloud,
Bandcamp, Chess.com, Codecademy, Coursera, Wikipedia, Habr, Pikabu, Boosty и др.

---

## Phone

```bash
sled investigate phone "+79123456789"
```

**Определение региона РФ**:
| DEF | Регион |
|-----|--------|
| 495 | Москва |
| 499 | Москва |
| 812 | Санкт-Петербург |
| 9.. | Мобильный |
| 800 | Бесплатный |

**Определение оператора**:
| Коды | Оператор |
|------|----------|
| 91x, 98x | МТС |
| 92x, 96x | Билайн |
| 93x, 99x | Мегафон |
| 90x, 95x, 97x | Tele2 / Yota |

---

## Crypto

```bash
sled investigate crypto 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
```

Автоопределение блокчейна:
- `1`, `3`, `bc1...` → Bitcoin
- `0x...` → Ethereum
- `T...` → TRON
- `bnb...` → BSC
- `S...` → Solana

Данные: Blockchain.info, Etherscan (парсинг HTML, без ключей).

---

## CLI

```bash
# Расследование
sled investigate <type> <value> [options]

# Отчёт
sled report <case_id> <input.json> -f md|html|pdf|json

# Список трансформаций
sled list-transforms [--module email]

# Анализ графа
sled analyze-graph input.json

# Конфигурация
sled config
```

**Опции investigate**:
| Опция | По умолч. | Описание |
|-------|-----------|----------|
| `--depth, -d` | 1 | Глубина поиска (1–5) |
| `--output, -o` | — | Сохранить JSON |
| `--operator, -op` | "default" | Оператор (аудит) |

---

## API

```bash
uvicorn sled.api.server:app --port 8000
```

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/investigate` | Запуск расследования |
| GET | `/transforms` | Список трансформаций |
| POST | `/report` | Генерация отчёта |

---

## Трансформации

Из одной сущности система строит связанные:

| Вход | Выход | Трансформация |
|------|-------|---------------|
| Email | Person | Извлечение имени из local-part |
| Email | Domain | Классификация домена |
| Email | SocialProfile | Gravatar MD5 |
| Email | Username | Local-part как username |
| Phone | SocialProfile | Telegram-аккаунт |
| Username | SocialProfile[] | Профили на всех платформах |

---

## Риск-скоринг

| Уровень | Score | Описание |
|---------|-------|----------|
| 🔴 Critical | 80–100 | Утечки, компрометация |
| 🟠 High | 60–79 | Множественные факторы |
| 🟡 Medium | 40–59 | Подозрительные признаки |
| 🟢 Low | 20–39 | Незначительные риски |
| ⚪ Safe | 0–19 | Чисто |

**Факторы**: disposable_email, found_in_breach, github_commit_public,
paste_dump_found, role_based_email, plus_tagged, short_local_part

---

## Установка

```bash
# Через pip
pip install sled

# Локально
git clone https://github.com/dankertea51-cpu/sled
cd sled
pip install -e .

# Зависимости
pip install -r requirements.txt

# Запуск тестов
pytest tests/ -v
```

**Python**: 3.11+

---

## Отчёты

| Формат | Команда | Особенности |
|--------|---------|-------------|
| Markdown | `-f md` | 7 разделов, таблицы, риск-факторы |
| HTML | `-f html` | Интерактивный граф (Cytoscape.js) |
| JSON | `-f json` | Для СМЭВ / внешних систем |
| PDF | `-f pdf` | ReportLab, кириллица |

---

## Лицензия

MIT

---

*След v1.0.0 — OSINT-фреймворк для анализа цифровых следов*
