# Правила и соглашения написания кода

## Общие принципы

### Читаемость кода
- Код должен быть понятным и самодокументируемым
- Используйте осмысленные имена переменных, функций и структур
- Избегайте магических чисел и строк - используйте константы
- Комментируйте сложную бизнес-логику, но не очевидные вещи

### DRY (Don't Repeat Yourself)
- Избегайте дублирования кода
- Выносите повторяющуюся логику в отдельные функции/методы
- Используйте переиспользуемые компоненты и утилиты

### KISS (Keep It Simple, Stupid)
- Предпочитайте простое решение сложному
- Не добавляйте преждевременную оптимизацию
- Избегайте излишней абстракции

### SOLID принципы
- **S**ingle Responsibility - один класс/функция должна делать одну вещь
- **O**pen/Closed - открыт для расширения, закрыт для модификации
- **L**iskov Substitution - подклассы должны заменять базовые классы
- **I**nterface Segregation - много маленьких интерфейсов лучше одного большого
- **D**ependency Inversion - зависеть от абстракций, а не от конкретных реализаций

## Go (Backend)

### Именование
- **Пакеты**: короткие, строчные, без подчеркиваний и смешанного регистра (`auth`, `gateway`, не `authService` или `Auth_Service`)
- **Экспортируемые функции/типы**: PascalCase (`GetUser`, `UserService`)
- **Неэкспортируемые функции/типы**: camelCase (`getUser`, `userService`)
- **Константы**: PascalCase для экспортируемых, camelCase для неэкспортируемых
- **Интерфейсы**: имена должны заканчиваться на `-er` если возможно (`Reader`, `Writer`, `Handler`)

### Структура кода
```go
// Порядок в файле:
// 1. Пакет
package auth

// 2. Импорты (стандартная библиотека, внешние, внутренние)
import (
    "context"
    "fmt"
    
    "github.com/gin-gonic/gin"
    
    "project/internal/models"
)

// 3. Константы
const (
    DefaultTimeout = 30 * time.Second
)

// 4. Переменные
var (
    logger = log.New()
)

// 5. Типы и структуры
type User struct {
    ID    int64  `json:"id" gorm:"primaryKey"`
    Email string `json:"email" gorm:"uniqueIndex"`
}

// 6. Методы и функции
func (u *User) Validate() error {
    // ...
}
```

### Обработка ошибок
- Всегда проверяйте ошибки, не игнорируйте их
- Используйте `fmt.Errorf` с `%w` для оборачивания ошибок
- Возвращайте ошибки как последний параметр
- Логируйте ошибки на соответствующем уровне

```go
// ✅ Правильно
user, err := s.repo.GetUser(ctx, id)
if err != nil {
    return nil, fmt.Errorf("failed to get user: %w", err)
}

// ❌ Неправильно
user, _ := s.repo.GetUser(ctx, id)
```

### Контексты
- Всегда передавайте `context.Context` как первый параметр в функции, которые делают I/O операции
- Используйте контексты для отмены операций и таймаутов
- Не храните контексты в структурах, передавайте их явно

```go
// ✅ Правильно
func (s *Service) GetUser(ctx context.Context, id int64) (*User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    // ...
}

// ❌ Неправильно
type Service struct {
    ctx context.Context // Не храните контекст в структуре
}
```

### GORM
- Используйте теги для валидации и индексов
- Всегда используйте транзакции для операций, изменяющих несколько записей
- Используйте `Select` и `Omit` для контроля полей при создании/обновлении
- Избегайте N+1 проблем с помощью `Preload`

```go
// ✅ Правильно
type User struct {
    ID       int64  `gorm:"primaryKey"`
    Email    string `gorm:"uniqueIndex;not null"`
    Password string `gorm:"not null"`
}

// Использование транзакций
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    if err := tx.Create(&profile).Error; err != nil {
        return err
    }
    return nil
})
```

### Тестирование
- Имена тестов должны быть описательными: `TestGetUser_NotFound`, `TestCreateUser_InvalidEmail`
- Используйте табличные тесты для множественных сценариев
- Используйте моки для внешних зависимостей
- Тесты должны быть независимыми и идемпотентными

```go
func TestGetUser(t *testing.T) {
    tests := []struct {
        name    string
        userID  int64
        wantErr bool
    }{
        {
            name:    "valid user",
            userID:  1,
            wantErr: false,
        },
        {
            name:    "user not found",
            userID:  999,
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // тест
        })
    }
}
```

## React + TypeScript (Frontend)

### Именование
- **Компоненты**: PascalCase (`UserProfile`, `TaskList`)
- **Файлы компонентов**: PascalCase (`UserProfile.tsx`)
- **Хуки**: начинаются с `use` (`useAuth`, `useTasks`)
- **Утилиты/константы**: camelCase (`formatDate`, `API_BASE_URL`)
- **Типы/Интерфейсы**: PascalCase (`User`, `TaskResponse`)

### Структура компонента
```typescript
// 1. Импорты (React, библиотеки, типы, компоненты, утилиты)
import { useState, useEffect } from 'react';
import { useQuery } from '@tanstack/react-query';
import { User } from '../types';
import { Button } from './Button';
import { formatDate } from '../utils';

// 2. Типы и интерфейсы
interface UserProfileProps {
    userId: number;
    onUpdate?: (user: User) => void;
}

// 3. Компонент
export const UserProfile: React.FC<UserProfileProps> = ({ userId, onUpdate }) => {
    // 4. Хуки состояния
    const [isEditing, setIsEditing] = useState(false);
    
    // 5. Хуки эффектов и данных
    const { data: user, isLoading } = useQuery({
        queryKey: ['user', userId],
        queryFn: () => fetchUser(userId),
    });
    
    // 6. Обработчики событий
    const handleSave = async () => {
        // ...
    };
    
    // 7. Ранний возврат для загрузки/ошибок
    if (isLoading) return <div>Loading...</div>;
    
    // 8. JSX
    return (
        <div>
            {/* ... */}
        </div>
    );
};
```

### TypeScript
- Всегда используйте типы, избегайте `any`
- Используйте `interface` для объектов, `type` для union/intersection типов
- Используйте строгий режим TypeScript
- Определяйте типы для пропсов компонентов

```typescript
// ✅ Правильно
interface User {
    id: number;
    email: string;
    name: string;
}

function getUser(id: number): Promise<User> {
    // ...
}

// ❌ Неправильно
function getUser(id: any): Promise<any> {
    // ...
}
```

### React хуки
- Используйте функциональные компоненты и хуки
- Соблюдайте правила хуков (не вызывайте условно)
- Используйте `useCallback` и `useMemo` для оптимизации, но не преждевременно
- Используйте кастомные хуки для переиспользуемой логики

```typescript
// ✅ Правильно
const handleClick = useCallback(() => {
    // ...
}, [dependencies]);

// ❌ Неправильно
if (condition) {
    const [state, setState] = useState(); // Нарушение правил хуков
}
```

### Обработка состояний
- Используйте React Query для серверных состояний
- Используйте `useState` для локальных состояний компонента
- Используйте Context API для глобальных состояний (осторожно)
- Рассмотрите Zustand или Redux для сложных состояний

### Стилизация
- Используйте CSS Modules или styled-components
- Избегайте inline стилей для сложных стилей
- Используйте переменные CSS для цветов, размеров и т.д.
- Следуйте методологии BEM для именования классов (если используете CSS)

## Git и коммиты

### Сообщения коммитов
Используйте формат Conventional Commits:
```
<type>(<scope>): <subject>

<body>

<footer>
```

Типы:
- `feat`: новая функциональность
- `fix`: исправление бага
- `docs`: изменения в документации
- `style`: форматирование, отсутствующие точки с запятой и т.д.
- `refactor`: рефакторинг кода
- `test`: добавление тестов
- `chore`: обновление задач сборки, настройки и т.д.

Примеры:
```
feat(auth): add Telegram authentication
fix(gateway): resolve CORS issue
docs(readme): update installation instructions
refactor(tasks): simplify task creation logic
```

### Ветки
- `main`/`master` - стабильная версия
- `develop` - ветка разработки
- `feature/*` - новые функции
- `fix/*` - исправления багов
- `hotfix/*` - критические исправления

## Форматирование и линтеры

### Go
- Используйте `gofmt` для форматирования
- Используйте `golangci-lint` для линтинга
- Настройте pre-commit хуки для автоматического форматирования

### TypeScript/React
- Используйте ESLint с правилами для React и TypeScript
- Используйте Prettier для форматирования
- Настройте автоматическое форматирование при сохранении

## Безопасность

### Backend
- Всегда валидируйте входные данные
- Используйте параметризованные запросы (GORM делает это автоматически)
- Храните секреты в переменных окружения, не в коде
- Используйте HTTPS в продакшене
- Реализуйте rate limiting
- Используйте JWT токены с коротким временем жизни

### Frontend
- Не храните секреты в клиентском коде
- Валидируйте данные на клиенте и сервере
- Используйте HTTPS
- Защищайте от XSS атак (React делает это автоматически)
- Используйте Content Security Policy

## Производительность

### Backend
- Используйте пулы соединений для БД
- Кэшируйте часто запрашиваемые данные
- Используйте индексы в БД
- Оптимизируйте запросы к БД (избегайте N+1)
- Используйте горутины для параллельной обработки

### Frontend
- Используйте code splitting
- Ленивая загрузка компонентов (`React.lazy`)
- Оптимизируйте изображения
- Используйте мемоизацию для тяжелых вычислений
- Минимизируйте количество ре-рендеров

## Логирование

- Используйте структурированное логирование
- Логируйте на соответствующем уровне (DEBUG, INFO, WARN, ERROR)
- Не логируйте чувствительную информацию (пароли, токены)
- Используйте контекст в логах для трейсинга

```go
// ✅ Правильно
logger.Info("user created",
    "user_id", user.ID,
    "email", user.Email,
)

// ❌ Неправильно
logger.Info(fmt.Sprintf("user created: %+v", user)) // может содержать пароль
```

## Документация

- Пишите комментарии для экспортируемых функций и типов
- Используйте godoc для Go документации
- Документируйте API endpoints (Swagger/OpenAPI)
- Обновляйте README при изменении проекта
- Документируйте сложную бизнес-логику
