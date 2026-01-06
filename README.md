# @ts-core/two-fa-totp

TOTP (Time-based One-Time Password) провайдер для двухфакторной аутентификации. Реализует алгоритм TOTP по RFC 6238, совместимый с Google Authenticator, Authy, Microsoft Authenticator и другими приложениями-аутентификаторами.

## Содержание

- [Установка](#установка)
- [Зависимости](#зависимости)
- [Основные возможности](#основные-возможности)
- [Быстрый старт](#быстрый-старт)
- [API Reference](#api-reference)
- [Интеграция с TwoFaService](#интеграция-с-twofaservice)
- [Генерация QR-кода](#генерация-qr-кода)
- [Полный пример](#полный-пример)
- [Интеграция с NestJS](#интеграция-с-nestjs)
- [Рекомендации по безопасности](#рекомендации-по-безопасности)
- [Связанные пакеты](#связанные-пакеты)

## Установка

```bash
npm install @ts-core/two-fa-totp
```

```bash
yarn add @ts-core/two-fa-totp
```

```bash
pnpm add @ts-core/two-fa-totp
```

## Зависимости

| Пакет | Описание |
|-------|----------|
| `@ts-core/common` | Базовые классы и интерфейсы |
| `@ts-core/two-fa` | Общие интерфейсы 2FA |
| `@ts-core/two-fa-backend` | Серверная реализация 2FA |
| `speakeasy` | Библиотека для генерации и проверки TOTP |

## Основные возможности

- Генерация криптографически стойких секретов в формате Base32
- Валидация одноразовых паролей с настраиваемым временным окном
- Совместимость со всеми популярными аутентификаторами (Google, Microsoft, Authy, 1Password и др.)
- Реализация интерфейса `ITwoFaProvider` для интеграции с `@ts-core/two-fa-backend`
- Поддержка RFC 6238 (TOTP) и RFC 4226 (HOTP)

## Быстрый старт

### Создание провайдера

```typescript
import { TotpProvider } from '@ts-core/two-fa-totp';
import { Logger } from '@ts-core/common';

const provider = new TotpProvider(new Logger(), {
    name: 'MyApp',      // Имя приложения в аутентификаторе
    length: 20,         // Длина секрета в байтах
    window: 1           // Допустимое отклонение временного окна
});
```

### Генерация секрета

```typescript
// Создание секрета для пользователя
const details = await provider.create(userId);
console.log(details.secret);  // Секрет в формате Base32, например: "JBSWY3DPEHPK3PXP"
```

### Проверка токена

```typescript
// Проверка токена из приложения-аутентификатора
const result = await provider.validate('123456', { secret: userSecret });

if (result.isValid) {
    console.log('Токен валиден!');
} else {
    console.log('Неверный токен');
}
```

## API Reference

### TotpProvider

Класс провайдера TOTP, реализующий интерфейс `ITwoFaProvider`:

```typescript
class TotpProvider extends LoggerWrapper implements ITwoFaProvider<ITotpCreateDetails> {
    constructor(logger: ILogger, options: ITotpOptions);

    // Генерация нового секрета
    create(ownerUid: TwoFaOwnerUid): Promise<ITotpCreateDetails>;

    // Проверка токена
    validate(token: string, details: ITotpCreateDetails): Promise<ITwoFaValidateDetails>;

    // Тип провайдера (всегда 'totp')
    get type(): string;
}
```

### ITotpOptions

Опции конфигурации провайдера:

```typescript
interface ITotpOptions {
    name: string;    // Имя приложения (отображается в аутентификаторе)
    length: number;  // Длина секрета в байтах (рекомендуется: 20)
    window: number;  // Временное окно валидации (рекомендуется: 1)
}
```

| Параметр | Описание | Рекомендуемое значение |
|----------|----------|------------------------|
| `name` | Имя вашего приложения | Название вашего сервиса |
| `length` | Длина секрета в байтах | 20 (160 бит) |
| `window` | Количество 30-секундных интервалов для допуска | 1 (±30 секунд) |

### ITotpCreateDetails

Результат создания TOTP:

```typescript
interface ITotpCreateDetails {
    secret: string;  // Секрет в формате Base32
}
```

### ITwoFaValidateDetails

Результат валидации:

```typescript
interface ITwoFaValidateDetails {
    isValid: boolean;  // true если токен валиден
}
```

## Интеграция с TwoFaService

Провайдер предназначен для использования совместно с `TwoFaService`:

```typescript
import { TwoFaService, TwoFaDatabaseService } from '@ts-core/two-fa-backend';
import { TotpProvider } from '@ts-core/two-fa-totp';
import { Logger } from '@ts-core/common';

// Создание провайдера
const totpProvider = new TotpProvider(logger, {
    name: 'MyApp',
    length: 20,
    window: 1
});

// Создание сервиса с TOTP провайдером
const twoFaService = new TwoFaService(logger, database, [totpProvider]);

// Провайдер автоматически используется для типа 'totp'
const secret = await twoFaService.create(userId, 'totp');
await twoFaService.save(userId, 'totp', '123456');
await twoFaService.validate(userId, 'totp', '654321');
```

## Генерация QR-кода

Для удобной настройки аутентификатора необходимо сгенерировать QR-код:

### Формат URL для аутентификатора

```
otpauth://totp/{issuer}:{account}?secret={secret}&issuer={issuer}
```

### Пример с библиотекой qrcode

```typescript
import * as QRCode from 'qrcode';
import { TotpProvider } from '@ts-core/two-fa-totp';

async function generateQrCode(
    provider: TotpProvider,
    userId: number,
    userEmail: string,
    appName: string
): Promise<string> {
    // Создание секрета
    const details = await provider.create(userId);

    // Формирование URL для аутентификатора
    const otpauthUrl = [
        'otpauth://totp/',
        encodeURIComponent(appName),
        ':',
        encodeURIComponent(userEmail),
        '?secret=',
        details.secret,
        '&issuer=',
        encodeURIComponent(appName)
    ].join('');

    // Генерация QR-кода в формате Data URL
    const qrCodeDataUrl = await QRCode.toDataURL(otpauthUrl);

    return qrCodeDataUrl;
}

// Использование
const qrCode = await generateQrCode(provider, 123, 'user@example.com', 'MyApp');
// qrCode = "data:image/png;base64,..."
```

### Пример HTML для отображения

```html
<div class="two-fa-setup">
    <h3>Настройка двухфакторной аутентификации</h3>
    <p>Отсканируйте QR-код в приложении Google Authenticator или Authy:</p>
    <img src="{{qrCodeDataUrl}}" alt="QR-код для 2FA" />
    <p>Или введите код вручную: <code>{{secret}}</code></p>
    <form>
        <input type="text" name="token" placeholder="Введите код из приложения" />
        <button type="submit">Подтвердить</button>
    </form>
</div>
```

## Полный пример

```typescript
import { TotpProvider, ITotpOptions } from '@ts-core/two-fa-totp';
import { TwoFaService, TwoFaDatabaseService } from '@ts-core/two-fa-backend';
import { Logger } from '@ts-core/common';
import * as QRCode from 'qrcode';

class TwoFaManager {
    private twoFaService: TwoFaService;
    private appName: string;

    constructor(
        logger: Logger,
        database: TwoFaDatabaseService,
        appName: string
    ) {
        this.appName = appName;

        const options: ITotpOptions = {
            name: appName,
            length: 20,     // 160 бит — рекомендуемая длина
            window: 1       // ±30 секунд допуска
        };

        const totpProvider = new TotpProvider(logger, options);
        this.twoFaService = new TwoFaService(logger, database, [totpProvider]);
    }

    /**
     * Инициализация настройки 2FA
     * @returns QR-код в формате Data URL и секрет для ручного ввода
     */
    async setup(userId: number, email: string): Promise<{ qrCode: string; secret: string }> {
        const details = await this.twoFaService.create(userId, 'totp');

        const otpauthUrl = `otpauth://totp/${this.appName}:${email}?secret=${details.secret}&issuer=${this.appName}`;
        const qrCode = await QRCode.toDataURL(otpauthUrl);

        return {
            qrCode,
            secret: details.secret
        };
    }

    /**
     * Активация 2FA после проверки токена
     */
    async enable(userId: number, token: string): Promise<void> {
        await this.twoFaService.save(userId, 'totp', token);
    }

    /**
     * Проверка токена при входе
     */
    async verify(userId: number, token: string): Promise<boolean> {
        try {
            await this.twoFaService.validate(userId, 'totp', token);
            return true;
        } catch {
            return false;
        }
    }

    /**
     * Проверка, включена ли 2FA у пользователя
     */
    async isEnabled(userId: number): Promise<boolean> {
        return this.twoFaService.database.has(userId, true);
    }

    /**
     * Отключение 2FA
     */
    async disable(userId: number): Promise<void> {
        const resetUid = await this.twoFaService.resetStart(userId, 'totp');
        await this.twoFaService.resetFinish(resetUid);
    }
}

// Использование
const manager = new TwoFaManager(logger, database, 'MyApp');

// Настройка 2FA
const { qrCode, secret } = await manager.setup(userId, 'user@example.com');
// Показать qrCode пользователю

// После сканирования QR-кода пользователь вводит код
await manager.enable(userId, '123456');

// При следующем входе
const isValid = await manager.verify(userId, '654321');
```

## Интеграция с NestJS

### Модуль TOTP

```typescript
import { Module } from '@nestjs/common';
import { TotpProvider, ITotpOptions } from '@ts-core/two-fa-totp';
import { Logger } from '@ts-core/common';

@Module({
    providers: [
        {
            provide: 'TOTP_OPTIONS',
            useValue: {
                name: process.env.APP_NAME || 'MyApp',
                length: 20,
                window: 1
            } as ITotpOptions
        },
        {
            provide: TotpProvider,
            useFactory: (logger: Logger, options: ITotpOptions) => {
                return new TotpProvider(logger, options);
            },
            inject: [Logger, 'TOTP_OPTIONS']
        }
    ],
    exports: [TotpProvider]
})
export class TotpModule {}
```

### Полный модуль 2FA

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { TwoFaEntity, TwoFaService, TwoFaDatabaseService } from '@ts-core/two-fa-backend';
import { TotpProvider, ITotpOptions } from '@ts-core/two-fa-totp';
import { Logger } from '@ts-core/common';

@Module({
    imports: [TypeOrmModule.forFeature([TwoFaEntity])],
    providers: [
        TwoFaDatabaseService,
        {
            provide: TwoFaService,
            useFactory: (logger: Logger, db: TwoFaDatabaseService) => {
                const totpProvider = new TotpProvider(logger, {
                    name: process.env.APP_NAME || 'MyApp',
                    length: 20,
                    window: 1
                });
                return new TwoFaService(logger, db, [totpProvider]);
            },
            inject: [Logger, TwoFaDatabaseService]
        }
    ],
    exports: [TwoFaService]
})
export class TwoFaModule {}
```

## Рекомендации по безопасности

### Хранение секретов

- Храните секреты в зашифрованном виде в базе данных
- Используйте симметричное шифрование (AES-256) для поля `details`
- Никогда не логируйте секреты в открытом виде

### Защита эндпоинтов

- Используйте HTTPS для всех запросов, связанных с 2FA
- Реализуйте rate limiting для эндпоинта валидации (например, 5 попыток в минуту)
- Блокируйте аккаунт после множественных неудачных попыток

### Временное окно

```typescript
// window: 1 означает допуск ±1 интервал (±30 секунд)
// Это компенсирует небольшое расхождение часов между сервером и устройством

// Более строгая настройка (только текущий интервал):
const strictProvider = new TotpProvider(logger, {
    name: 'SecureApp',
    length: 20,
    window: 0  // Только текущий 30-секундный интервал
});

// Более мягкая настройка (для устройств с плохой синхронизацией):
const lenientProvider = new TotpProvider(logger, {
    name: 'MyApp',
    length: 20,
    window: 2  // ±60 секунд допуска
});
```

### Резервные коды

Рекомендуется предоставлять пользователям резервные коды на случай потери доступа к аутентификатору:

```typescript
// Генерация резервных кодов
function generateBackupCodes(count: number = 10): string[] {
    const codes: string[] = [];
    for (let i = 0; i < count; i++) {
        // Формат: XXXX-XXXX (8 символов)
        const code = Math.random().toString(36).substring(2, 6).toUpperCase() +
                     '-' +
                     Math.random().toString(36).substring(2, 6).toUpperCase();
        codes.push(code);
    }
    return codes;
}
```

## Как работает TOTP

```
┌─────────────────────────────────────────────────────────────────┐
│                    АЛГОРИТМ TOTP (RFC 6238)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Секрет (Secret) — общий между сервером и аутентификатором  │
│     Формат: Base32, например "JBSWY3DPEHPK3PXP"                │
│                                                                 │
│  2. Время (Time) — текущее время Unix, делённое на 30          │
│     Counter = floor(Unix_time / 30)                             │
│                                                                 │
│  3. HMAC-SHA1(Secret, Counter) → 20 байт                       │
│                                                                 │
│  4. Динамическое усечение → 6-значный код                      │
│     Код меняется каждые 30 секунд                              │
│                                                                 │
│  Пример:                                                        │
│  Время: 1234567890 → Counter: 41152263                         │
│  Секрет: "JBSWY3DPEHPK3PXP"                                    │
│  Результат: 123456                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Связанные пакеты

| Пакет | Описание |
|-------|----------|
| `@ts-core/two-fa` | Общие интерфейсы и типы |
| `@ts-core/two-fa-backend` | Серверная реализация 2FA |

## Автор

**Renat Gubaev** — [renat.gubaev@gmail.com](mailto:renat.gubaev@gmail.com)

- GitHub: [ManhattanDoctor](https://github.com/ManhattanDoctor)
- Репозиторий: [ts-core-two-fa-totp](https://github.com/ManhattanDoctor/ts-core-two-fa-totp)

## Лицензия

ISC
