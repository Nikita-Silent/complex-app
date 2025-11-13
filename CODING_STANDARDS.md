# Правила и Соглашения Написания Кода

## 📋 Содержание

1. [Общие Принципы](#общие-принципы)
2. [Go Guidelines](#go-guidelines)
3. [React/TypeScript Guidelines](#reacttypescript-guidelines)
4. [База Данных](#база-данных)
5. [API Design](#api-design)
6. [Git Workflow](#git-workflow)
7. [Testing](#testing)
8. [Security](#security)

---

## Общие Принципы

### DRY (Don't Repeat Yourself)
- Избегайте дублирования кода
- Создавайте переиспользуемые функции и компоненты
- Используйте константы вместо magic numbers

### KISS (Keep It Simple, Stupid)
- Пишите простой и понятный код
- Избегайте излишней сложности
- Предпочитайте явность неявности

### SOLID Принципы
- **S**ingle Responsibility - один класс/функция = одна ответственность
- **O**pen/Closed - открыт для расширения, закрыт для модификации
- **L**iskov Substitution - подтипы должны быть взаимозаменяемы
- **I**nterface Segregation - много специфичных интерфейсов лучше одного общего
- **D**ependency Inversion - зависимость от абстракций, а не конкретики

---

## Go Guidelines

### Структура Проекта

```
service-name/
├── cmd/
│   └── server/
│       └── main.go           # Точка входа
├── internal/
│   ├── handler/             # HTTP handlers
│   ├── service/             # Бизнес-логика
│   ├── repository/          # Работа с БД
│   ├── model/               # Модели данных
│   └── middleware/          # Middleware
├── pkg/                     # Публичные пакеты
├── config/                  # Конфигурация
├── migrations/              # SQL миграции
└── tests/                   # Тесты
```

### Именование

```go
// ✅ Правильно
type UserService interface {
    GetUserByID(ctx context.Context, id int64) (*User, error)
    CreateUser(ctx context.Context, user *User) error
}

// ❌ Неправильно
type userservice interface {
    get_user_by_id(ctx context.Context, id int64) (*User, error)
    create_user(ctx context.Context, user *User) error
}
```

**Правила:**
- PascalCase для экспортируемых имён
- camelCase для приватных
- Используйте осмысленные имена
- Избегайте сокращений (кроме общепринятых: ctx, err, req, resp)

### Error Handling

```go
// ✅ Правильно
func GetUser(id int64) (*User, error) {
    user, err := db.FindUserByID(id)
    if err != nil {
        return nil, fmt.Errorf("failed to get user %d: %w", id, err)
    }
    return user, nil
}

// ❌ Неправильно
func GetUser(id int64) (*User, error) {
    user, err := db.FindUserByID(id)
    if err != nil {
        return nil, err // Потеря контекста
    }
    return user, nil
}
```

**Правила:**
- Всегда проверяйте ошибки
- Используйте `%w` для wrap ошибок
- Добавляйте контекст к ошибкам
- Не игнорируйте ошибки (`_ = err` только с комментарием)

### Context Usage

```go
// ✅ Правильно
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
    // Проверка отмены контекста
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    return s.repo.FindByID(ctx, id)
}

// Всегда передавайте context первым параметром
```

### Dependency Injection

```go
// ✅ Правильно - используем интерфейсы
type UserRepository interface {
    FindByID(ctx context.Context, id int64) (*User, error)
    Create(ctx context.Context, user *User) error
}

type UserService struct {
    repo UserRepository
    logger *slog.Logger
}

func NewUserService(repo UserRepository, logger *slog.Logger) *UserService {
    return &UserService{
        repo: repo,
        logger: logger,
    }
}
```

### Logging

```go
// Используйте структурированное логирование
logger.Info("user created",
    slog.Int64("user_id", user.ID),
    slog.String("email", user.Email),
)

logger.Error("failed to create user",
    slog.String("error", err.Error()),
    slog.Any("user", user),
)
```

---

## React/TypeScript Guidelines

### Структура Проекта

```
frontend/
├── src/
│   ├── components/          # Переиспользуемые компоненты
│   │   ├── common/         # Общие UI компоненты
│   │   └── features/       # Фича-специфичные компоненты
│   ├── pages/              # Страницы приложения
│   ├── hooks/              # Custom hooks
│   ├── services/           # API сервисы
│   ├── store/              # State management (Redux/Zustand)
│   ├── types/              # TypeScript типы
│   ├── utils/              # Утилиты
│   └── App.tsx
├── public/
└── tests/
```

### Компоненты

```typescript
// ✅ Правильно - функциональный компонент с TypeScript
interface UserCardProps {
  user: User;
  onEdit?: (id: number) => void;
  className?: string;
}

export const UserCard: React.FC<UserCardProps> = ({ 
  user, 
  onEdit,
  className 
}) => {
  const handleEdit = () => {
    onEdit?.(user.id);
  };

  return (
    <div className={className}>
      <h3>{user.name}</h3>
      <button onClick={handleEdit}>Edit</button>
    </div>
  );
};

// ❌ Неправильно - без типов
export const UserCard = ({ user, onEdit }) => {
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={() => onEdit(user.id)}>Edit</button>
    </div>
  );
};
```

### Hooks

```typescript
// ✅ Правильно - custom hook с типами
interface UseUserReturn {
  user: User | null;
  loading: boolean;
  error: Error | null;
  refetch: () => Promise<void>;
}

export const useUser = (userId: number): UseUserReturn => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchUser = async () => {
    try {
      setLoading(true);
      const data = await userService.getUser(userId);
      setUser(data);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchUser();
  }, [userId]);

  return { user, loading, error, refetch: fetchUser };
};
```

### API Сервисы

```typescript
// ✅ Правильно - сервис с обработкой ошибок
class UserService {
  private readonly baseURL = '/api/users';

  async getUser(id: number): Promise<User> {
    try {
      const response = await fetch(`${this.baseURL}/${id}`);
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error('Failed to fetch user:', error);
      throw error;
    }
  }

  async createUser(user: CreateUserDTO): Promise<User> {
    const response = await fetch(this.baseURL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user),
    });
    
    if (!response.ok) {
      throw new Error('Failed to create user');
    }
    
    return await response.json();
  }
}

export const userService = new UserService();
```

### Именование

- **Компоненты**: PascalCase (`UserCard`, `LoginForm`)
- **Функции/переменные**: camelCase (`handleSubmit`, `userData`)
- **Константы**: UPPER_SNAKE_CASE (`API_BASE_URL`, `MAX_RETRY_COUNT`)
- **Типы/Интерфейсы**: PascalCase с `I` префиксом опционально (`User`, `IUserProps`)

---

## База Данных

### GORM Модели

```go
// ✅ Правильно
type User struct {
    ID        int64          `gorm:"primaryKey;autoIncrement" json:"id"`
    Email     string         `gorm:"uniqueIndex;not null;size:255" json:"email"`
    Name      string         `gorm:"not null;size:100" json:"name"`
    CreatedAt time.Time      `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime" json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

func (User) TableName() string {
    return "users"
}
```

### Миграции

```go
// AutoMigrate в main.go
func initDB() (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }
    
    // Автоматические миграции
    if err := db.AutoMigrate(
        &User{},
        &Task{},
        &News{},
    ); err != nil {
        return nil, err
    }
    
    return db, nil
}
```

### Транзакции

```go
// ✅ Правильно - используйте транзакции для связанных операций
func (s *UserService) CreateUserWithProfile(ctx context.Context, user *User, profile *Profile) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Create(user).Error; err != nil {
            return err
        }
        
        profile.UserID = user.ID
        if err := tx.Create(profile).Error; err != nil {
            return err
        }
        
        return nil
    })
}
```

---

## API Design

### REST Endpoints

```
# Стандартная структура
GET    /api/v1/users           # Список пользователей
GET    /api/v1/users/:id       # Получить пользователя
POST   /api/v1/users           # Создать пользователя
PUT    /api/v1/users/:id       # Обновить пользователя (полностью)
PATCH  /api/v1/users/:id       # Обновить пользователя (частично)
DELETE /api/v1/users/:id       # Удалить пользователя

# Вложенные ресурсы
GET    /api/v1/users/:id/tasks # Задачи пользователя
POST   /api/v1/users/:id/tasks # Создать задачу для пользователя
```

### Request/Response Format

```go
// Request
type CreateUserRequest struct {
    Email string `json:"email" binding:"required,email"`
    Name  string `json:"name" binding:"required,min=2,max=100"`
}

// Response - всегда структурированный
type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   *Error      `json:"error,omitempty"`
}

type Error struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

// Пример использования
func (h *UserHandler) CreateUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, Response{
            Success: false,
            Error: &Error{
                Code:    "INVALID_INPUT",
                Message: err.Error(),
            },
        })
        return
    }
    
    user, err := h.service.CreateUser(c.Request.Context(), &req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, Response{
            Success: false,
            Error: &Error{
                Code:    "INTERNAL_ERROR",
                Message: "Failed to create user",
            },
        })
        return
    }
    
    c.JSON(http.StatusCreated, Response{
        Success: true,
        Data:    user,
    })
}
```

### HTTP Status Codes

- `200 OK` - успешный GET/PUT/PATCH/DELETE
- `201 Created` - успешный POST
- `204 No Content` - успешный DELETE без тела ответа
- `400 Bad Request` - ошибка валидации
- `401 Unauthorized` - не аутентифицирован
- `403 Forbidden` - не авторизован
- `404 Not Found` - ресурс не найден
- `409 Conflict` - конфликт (например, email уже существует)
- `500 Internal Server Error` - внутренняя ошибка сервера

---

## Git Workflow

### Branch Naming

```
main                    # Продакшн код
develop                # Разработка

feature/user-auth      # Новая функциональность
bugfix/login-error     # Исправление бага
hotfix/security-patch  # Срочное исправление
refactor/user-service  # Рефакторинг
```

### Commit Messages

```bash
# Формат: <type>(<scope>): <subject>

feat(auth): add telegram authentication
fix(users): resolve email validation issue
refactor(gateway): improve routing logic
docs(readme): update installation instructions
test(users): add unit tests for user service
chore(deps): update dependencies

# Примеры
git commit -m "feat(auth): add JWT token validation"
git commit -m "fix(database): resolve connection pool leak"
git commit -m "refactor(api): extract common middleware"
```

### Pull Request Template

```markdown
## Описание
Краткое описание изменений

## Тип изменения
- [ ] Новая функциональность
- [ ] Исправление бага
- [ ] Рефакторинг
- [ ] Документация

## Чеклист
- [ ] Код следует стандартам проекта
- [ ] Добавлены/обновлены тесты
- [ ] Все тесты проходят
- [ ] Обновлена документация
- [ ] Нет конфликтов с main веткой

## Тестирование
Как протестировать изменения
```

---

## Testing

### Go Tests

```go
// ✅ Правильно - тесты с table-driven approach
func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        user    *User
        want    error
        wantErr bool
    }{
        {
            name: "valid user",
            user: &User{Email: "test@example.com", Name: "Test"},
            want: nil,
            wantErr: false,
        },
        {
            name: "invalid email",
            user: &User{Email: "invalid", Name: "Test"},
            want: ErrInvalidEmail,
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            service := NewUserService(mockRepo, mockLogger)
            err := service.CreateUser(context.Background(), tt.user)
            
            if (err != nil) != tt.wantErr {
                t.Errorf("CreateUser() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### React Tests

```typescript
// ✅ Правильно - тест компонента
import { render, screen, fireEvent } from '@testing-library/react';
import { UserCard } from './UserCard';

describe('UserCard', () => {
  const mockUser = {
    id: 1,
    name: 'John Doe',
    email: 'john@example.com',
  };

  it('renders user information', () => {
    render(<UserCard user={mockUser} />);
    
    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('calls onEdit when edit button is clicked', () => {
    const handleEdit = jest.fn();
    render(<UserCard user={mockUser} onEdit={handleEdit} />);
    
    fireEvent.click(screen.getByText('Edit'));
    
    expect(handleEdit).toHaveBeenCalledWith(1);
  });
});
```

---

## Security

### Общие Правила

1. **Никогда не коммитьте секреты**
   - Используйте `.env` файлы
   - Добавьте `.env` в `.gitignore`
   - Используйте переменные окружения

2. **Input Validation**
   ```go
   // Всегда валидируйте входные данные
   if err := c.ShouldBindJSON(&req); err != nil {
       return BadRequest(err)
   }
   ```

3. **SQL Injection Protection**
   ```go
   // ✅ Правильно - GORM защищает от SQL injection
   db.Where("email = ?", userEmail).First(&user)
   
   // ❌ ОПАСНО - никогда так не делайте
   db.Raw(fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", userEmail))
   ```

4. **XSS Protection**
   ```typescript
   // React автоматически экранирует значения
   // Но будьте осторожны с dangerouslySetInnerHTML
   
   // ❌ ОПАСНО
   <div dangerouslySetInnerHTML={{ __html: userInput }} />
   
   // ✅ Правильно
   <div>{userInput}</div>
   ```

5. **CORS Configuration**
   ```go
   // Настройте CORS правильно
   config := cors.DefaultConfig()
   config.AllowOrigins = []string{"https://yourdomain.com"}
   config.AllowMethods = []string{"GET", "POST", "PUT", "DELETE"}
   config.AllowHeaders = []string{"Origin", "Content-Type", "Authorization"}
   ```

---

## Дополнительные Ресурсы

### Go
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)

### React/TypeScript
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
- [Airbnb React Style Guide](https://github.com/airbnb/javascript/tree/master/react)

### API Design
- [REST API Best Practices](https://restfulapi.net/)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)

---

**Помните:** Эти правила созданы для улучшения качества кода и совместной работы. Следуйте им, но при необходимости адаптируйте под нужды проекта.
