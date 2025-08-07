# SecureTG - E2E Encrypted Telegram Bot

> Telegram бот для зашифрованного общения с использованием Signal Protocol и современных криптографических стандартов.

## 🏗️ Архитектура проекта

```
secure-tg/
├── package.json              # Root workspace manager
├── pnpm-workspace.yaml       # PNPM workspace config
├── .prettierrc.json         # Prettier config
├── .gitignore
├── README.md
├── scripts/
│   ├── dev.sh              # Development launcher
│   ├── build.sh            # Production builder
│   └── deploy.sh           # Deployment script
├── backend/                # Crypto & Database Service
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── index.ts        # Entry point
│   │   ├── config/
│   │   │   ├── database.ts
│   │   │   ├── redis.ts
│   │   │   └── env.ts
│   │   ├── crypto/         # Криптографический модуль
│   │   │   ├── signal/
│   │   │   │   ├── protocol.ts    # Signal Protocol implementation
│   │   │   │   ├── keys.ts        # Key management
│   │   │   │   └── session.ts     # Session management
│   │   │   ├── cipher.ts          # AES-GCM encryption
│   │   │   └── kdf.ts            # Key derivation
│   │   ├── database/       # Данные (НЕ ключи!)
│   │   │   ├── models/
│   │   │   │   ├── user.ts
│   │   │   │   ├── session.ts
│   │   │   │   └── message.ts
│   │   │   └── repositories/
│   │   │       ├── userRepo.ts
│   │   │       └── sessionRepo.ts
│   │   ├── cache/          # Redis operations
│   │   │   ├── sessionCache.ts
│   │   │   └── messageQueue.ts
│   │   ├── services/       # Бизнес-логика
│   │   │   ├── cryptoService.ts
│   │   │   ├── sessionService.ts
│   │   │   └── messageService.ts
│   │   ├── api/           # HTTP API для frontend
│   │   │   ├── routes/
│   │   │   │   ├── crypto.ts
│   │   │   │   └── session.ts
│   │   │   └── middleware/
│   │   │       ├── auth.ts
│   │   │       └── validation.ts
│   │   └── utils/
│   │       ├── logger.ts
│   │       └── errors.ts
│   └── prisma/            # Database schema
│       ├── schema.prisma
│       └── migrations/
└── frontend/              # Telegram Bot Interface
    ├── package.json
    ├── tsconfig.json
    └── src/
        ├── index.ts       # Bot entry point
        ├── bot/
        │   ├── handlers/  # Command handlers
        │   │   ├── start.ts
        │   │   ├── register.ts
        │   │   ├── chat.ts
        │   │   └── settings.ts
        │   ├── keyboards/ # Inline keyboards
        │   │   ├── main.ts
        │   │   ├── chat.ts
        │   │   └── settings.ts
        │   ├── middleware/
        │   │   ├── auth.ts
        │   │   ├── rateLimit.ts
        │   │   └── logging.ts
        │   └── scenes/    # Conversation flows
        │       ├── registration.ts
        │       ├── chatSetup.ts
        │       └── messaging.ts
        ├── services/      # Backend communication
        │   ├── apiClient.ts
        │   └── cryptoClient.ts
        └── utils/
            ├── formatter.ts
            └── validator.ts
```

## 🔧 Tech Stack

### Backend
- **Node.js** + **TypeScript** - основа
- **Fastify** - быстрый HTTP сервер
- **Prisma** - типобезопасная ORM
- **PostgreSQL** - production-ready БД с ACID гарантиями
- **Redis** - кэширование и временные сессии
- **libsignal-protocol** - криптография
- **node-forge** - дополнительная криптография
- **zod** - валидация схем

### Frontend (Bot)
- **Grammy** - современный Telegram Bot framework
- **node-cron** - задачи по расписанию
- **axios** - HTTP клиент для backend

### DevOps
- **PNPM** - package manager с workspaces
- **tsx** - TypeScript runner
- **Prettier** - code formatting
- **ESLint** - linting

## 🔐 Криптографическая схема

### Signal Protocol Implementation
```typescript
// Double Ratchet + X3DH key agreement
User A                    Bot Relay                    User B
  |                         |                          |
  |--- Identity Keys ------>|<----- Identity Keys -----|
  |                         |                          |
  |--- Prekeys Exchange ----|                          |
  |                         |---- Prekeys Exchange --->|
  |                         |                          |
  |=== E2E Encrypted ======>|                          |
  |    Messages             |=== Forward Encrypted ===>|
  |                         |    (Bot can't decrypt)   |
```

### Ключевые компоненты:
1. **Identity Keys** - долгосрочные Ed25519 ключи
2. **Ephemeral Keys** - временные X25519 ключи
3. **Prekeys** - набор одноразовых ключей
4. **Message Keys** - уникальные AES-256-GCM ключи для каждого сообщения
5. **Chain Keys** - для forward secrecy

## 🚀 Development Workflow

### Запуск разработки:
```fish
# Установка зависимостей
pnpm install

# Запуск в dev режиме (оба сервиса)
pnpm dev

# Только backend
pnpm dev:backend

# Только frontend (bot)
pnpm dev:frontend
```

### Продакшн:
```fish
# Билд всего проекта
pnpm build

# Запуск в продакшн
pnpm start
```

## 📦 Workspace Configuration

### Root `package.json`:
```json
{
  "name": "secure-tg",
  "private": true,
  "scripts": {
    "dev": "concurrently \"pnpm dev:backend\" \"pnpm dev:frontend\"",
    "dev:backend": "pnpm --filter backend dev",
    "dev:frontend": "pnpm --filter frontend dev",
    "build": "pnpm --filter backend build && pnpm --filter frontend build",
    "start": "concurrently \"pnpm --filter backend start\" \"pnpm --filter frontend start\"",
    "lint": "pnpm --filter backend lint && pnpm --filter frontend lint",
    "format": "prettier --write ."
  },
  "devDependencies": {
    "concurrently": "^8.2.2",
    "prettier": "^3.1.0"
  }
}
```

### Backend `package.json`:
```json
{
  "name": "@secure-tg/backend",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "db:generate": "prisma generate",
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:studio": "prisma studio"
  },
  "dependencies": {
    "fastify": "^4.24.3",
    "@fastify/cors": "^8.4.0",
    "@fastify/redis": "^6.1.1",
    "prisma": "^5.6.0",
    "@prisma/client": "^5.6.0",
    "pg": "^8.11.3",
    "redis": "^4.6.10",
    "libsignal-protocol": "^2.3.3",
    "node-forge": "^1.3.1",
    "zod": "^3.22.4",
    "pino": "^8.16.2"
  },
  "devDependencies": {
    "@types/pg": "^8.10.7"
  }
}
```

### Frontend `package.json`:
```json
{
  "name": "@secure-tg/frontend",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "grammy": "^1.20.3",
    "@grammyjs/conversations": "^1.2.0",
    "@grammyjs/menu": "^1.2.1",
    "axios": "^1.6.2",
    "node-cron": "^3.0.3"
  }
}
```

## 🔄 Message Flow

### Регистрация пользователя:
1. `/start` → генерация Identity Key + Prekeys
2. Сохранение публичных ключей в backend
3. Приватные ключи остаются только у пользователя

### Установка сеанса:
1. User A выбирает User B для чата
2. Backend отдает prekey bundle для User B
3. User A выполняет X3DH key agreement
4. Создается Double Ratchet session

### Отправка сообщения:
1. User A шифрует сообщение локально
2. Бот получает зашифрованный blob
3. Бот пересылает blob User B
4. User B дешифрует локально

## 🎯 UI/UX через Inline Keyboards

### Главное меню:
```
[ 💬 Новый чат ]  [ 📝 Мои чаты ]
[ ⚙️ Настройки ]  [ ℹ️ Помощь ]
```

### Чат интерфейс:
```
[ 📤 Отправить ]  [ 📄 История ]
[ 🔒 Проверить ключи ]
[ ⬅️ Назад ]
```

## 🔒 Безопасность

### Что НЕ хранится на сервере:
- ❌ Приватные ключи пользователей
- ❌ Расшифрованные сообщения
- ❌ Message keys или chain keys

### Что хранится:
- ✅ Публичные identity keys
- ✅ Prekey bundles  
- ✅ Зашифрованные message blobs (с TTL в Redis)
- ✅ Метаданные сессий (PostgreSQL)
- ✅ Rate limiting данные (Redis)

### Forward Secrecy:
- Ключи сообщений удаляются после использования
- Компрометация текущих ключей не раскрывает прошлые сообщения

### Post-Compromise Security:
- Double Ratchet обеспечивает восстановление безопасности
- Новые ключи генерируются при каждом обмене сообщениями

## 📚 Development Notes

### Модульность:
- Каждый компонент имеет четкую ответственность
- Crypto модуль полностью изолирован
- API слой абстрагирован от bot логики

### Тестирование:
```fish
pnpm test          # Все тесты
pnpm test:backend  # Только backend
pnpm test:frontend # Только frontend
```

### Линтинг и форматирование:
```fish
pnpm lint    # ESLint проверка
pnpm format  # Prettier форматирование
```

## 📝 README Format

Вот примерный README в "формально-разговорчивом" стиле:

```markdown
# SecureTG 🔐

> Ну что, надоели обычные мессенджеры? Хочется что-то посерьезнее? Держи E2E зашифрованный Telegram бот на базе Signal Protocol.

## Что это такое?

Обычный Telegram бот, но с twist'ом - все сообщения шифруются на клиенте и бот физически не может их прочитать. Да, прям как в Signal, только через Telegram API.

### Фичи:
- 🔒 **Signal Protocol** - тот самый, что в WhatsApp
- 🔄 **Double Ratchet** - forward secrecy из коробки  
- 🚫 **Zero Knowledge** - сервер ничего не знает о твоих сообщениях
- 📱 **Inline UI** - никаких веб-морд, только кнопочки
- ⚡ **TypeScript** - потому что типы это святое

## Быстрый старт

```fish
# Клонируем
gh repo clone username/secure-tg
cd secure-tg


# Ставим зависимости (PNPM ftw)
pnpm install

# Настраиваем .env файлы
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
# (не забудь BOT_TOKEN от @BotFather и DATABASE_URL)

# Применяем миграции БД
pnpm --filter backend db:migrate

# Запускаем в dev режиме
pnpm dev
```

## Как работает магия?

1. Регистрируешься в боте → генерируются ключи
2. Выбираешь собеседника → обмениваетесь публичными ключами  
3. Пишешь сообщение → оно шифруется у тебя на устройстве
4. Бот получает зашифрованную каку → пересылает как есть
5. Собеседник получает → дешифрует у себя

**Важно:** бот видит только зашифрованные блобы. Даже если его взломают или засекьюрят - твои сообщения останутся в безопасности.

## Структура проекта

- `backend/` - криптография и API (Fastify + Prisma)
- `frontend/` - сам Telegram бот (Grammy)
- `scripts/` - всякие хелперы для разработки

Каждый модуль живет своей жизнью, но дружат через HTTP API.

## FAQ

**Q: Это безопасно?**  
A: Используем проверенный Signal Protocol. Если его взломают - у всех проблемы, не только у нас.

**Q: А что если сервер скомпрометируют?**  
A: Максимум что получат - зашифрованные блобы и метаданные. Сами сообщения не расшифруют.

**Q: Можно ли доверять коду?**  
A: Код открытый, криптография стандартная. Проверяй сам или найми аудиторов.

**Q: Почему не P2P?**  
A: NAT traversal это pain, а Telegram API работает из коробки. Компромисс в пользу простоты.

## Разработка

```bash
pnpm dev          # Все сервисы  
pnpm dev:backend  # Только API
pnpm dev:frontend # Только бот
pnpm build        # Билд в продакшн
pnpm lint         # Проверка кода
pnpm format       # Prettier magic
```

## Лицензия

**GPL v3** - потому что свобода должна оставаться свободной.

Что это значает:
- ✅ Используй как хочешь
- ✅ Модифицируй код  
- ✅ Коммерческое использование
- ⚠️ **НО:** любые модификации тоже должны быть GPL
- ⚠️ **И:** если встраиваешь в свой проект - он тоже становится GPL

```

Это продуманная архитектура для enterprise-level Telegram бота с end-to-end шифрованием, которая обеспечивает максимальную безопасность при сохранении удобства разработки и масштабируемости.