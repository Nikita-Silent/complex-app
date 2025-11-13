# Правила и Соглашения Написания Кода

> **Подход к разработке**: Делай частями → Проверяй работоспособность → Документируй → Продолжай

## 📋 Содержание

1. [Общие Принципы](#общие-принципы)
2. [Архитектура Микросервисов](#архитектура-микросервисов)
3. [Go Guidelines](#go-guidelines)
4. [API Gateway](#api-gateway)
5. [База Данных (GORM)](#база-данных-gorm)
6. [React/TypeScript Guidelines](#reacttypescript-guidelines)
7. [API Design & OpenAPI](#api-design--openapi)
8. [Testing](#testing)
9. [Docker & DevOps](#docker--devops)
10. [Git Workflow](#git-workflow)
11. [Security](#security)

---

## Общие Принципы

### Итеративная Разработка

**Каждая задача должна пройти цикл:**

```
1. Реализация → 2. Проверка → 3. Тестирование → 4. Документирование → 5. Коммит
```

- ✅ **Никогда не переходи** к следующей задаче, пока текущая не завершена полностью
- ✅ **Всегда тестируй** код вручную (curl, Postman, браузер)
- ✅ **Всегда пиши** unit и integration тесты
- ✅ **Всегда обновляй** OpenAPI документацию
- ✅ **Коммить** после каждой завершённой задачи

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

## Архитектура Микросервисов

### Паттерн API Gateway

Все запросы идут через API Gateway:

```
Client → API Gateway → Service Router → Target Microservice
                ↓
        Service Unavailable?
                ↓
        Error Handler → Client
```

### Независимость Сервисов

**Принципы:**
1. **Нет прямых вызовов** - сервисы не вызывают друг друга напрямую
2. **Общая база данных** - сервисы могут использовать общую БД, но отдельные таблицы
3. **Минимальные зависимости** - каждый сервис работает независимо
4. **Graceful degradation** - если сервис недоступен, Gateway возвращает 503

### Структура Сервиса (Layered Architecture)

```
┌─────────────────────────────────┐
│      Handler Layer              │  HTTP handlers, request/response
├─────────────────────────────────┤
│      Service Layer              │  Business logic
├─────────────────────────────────┤
│      Repository Layer           │  Data access (GORM)
├─────────────────────────────────┤
│      Database                   │  PostgreSQL
└─────────────────────────────────┘
```

**Зависимости:** Handler → Service → Repository → Database

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
│   └── config/              # Конфигурация
├── tests/                   # Интеграционные тесты
├── Dockerfile
└── go.mod
```

### Именование

#### Константы - SCREAMING_SNAKE_CASE

```go
// ✅ Правильно
const (
    API_ENDPOINT_TASKS   = "/api/tasks"
    API_ENDPOINT_NEWS    = "/api/news"
    MAX_RETRY_COUNT      = 3
    DEFAULT_TIMEOUT      = 30 * time.Second
)

// ✅ Правильно - map констант
const API_ENDPOINTS = map[string]string{
    "USERS":   "/api/users",
    "TASKS":   "/api/tasks",
    "NEWS":    "/api/news",
}
```

#### Переменные и Функции - camelCase

```go
// ✅ Правильно
var userName string
var isActive bool

func getUserByID(id string) (*User, error) {
    // Implementation
}

func calculateTotal(items []Item) float64 {
    // Implementation
}

// ❌ Неправильно
var user_name string
func get_user_by_id() {}
```

#### Типы и Структуры - PascalCase

```go
// ✅ Правильно - экспортируемые типы
type User struct {
    ID    string
    Name  string
    Email string
}

type UserService interface {
    GetUserByID(ctx context.Context, id string) (*User, error)
}

// ✅ Правильно - приватные типы
type userRepository struct {
    db *gorm.DB
}
```

#### Пакеты - camelCase

```go
// ✅ Правильно
package userService
package authHandler
package taskRepository

// ❌ Неправильно
package user_service
package UserService
```

### Error Handling

```go
// ✅ Правильно - всегда оборачивай ошибки с контекстом
func GetUser(ctx context.Context, id string) (*User, error) {
    user, err := db.FindUserByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get user %s: %w", id, err)
    }
    return user, nil
}

// ✅ Правильно - используй кастомные ошибки
var (
    ErrUserNotFound    = errors.New("user not found")
    ErrInvalidInput    = errors.New("invalid input")
    ErrUnauthorized    = errors.New("unauthorized")
)

// Проверка ошибок
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrUserNotFound
}
```

### Context Usage

```go
// ✅ Правильно - всегда принимай context первым параметром
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    // Проверка отмены контекста
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    return s.repo.FindByID(ctx, id)
}

// ✅ Правильно - передавай context через весь call chain
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    user, err := h.service.GetUser(ctx, userID)
    // Handle response
}
```

### Dependency Injection

```go
// ✅ Правильно - используй интерфейсы
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
}

type UserService struct {
    repo   UserRepository
    logger *slog.Logger
}

func NewUserService(repo UserRepository, logger *slog.Logger) *UserService {
    return &UserService{
        repo:   repo,
        logger: logger,
    }
}
```

### Структурированное Логирование

```go
// ✅ Правильно - используй slog или zap
import "log/slog"

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

## API Gateway

### Ответственность Gateway

1. **Request Routing** - маршрутизация к нужному микросервису
2. **Service Discovery** - знание о доступных сервисах
3. **Error Handling** - обработка недоступности сервисов
4. **Health Checks** - проверка здоровья сервисов
5. **Logging** - логирование всех запросов

### URI-Based Routing

```go
// ✅ Правильно - маппинг URI на сервисы
const (
    API_ENDPOINT_TASKS   = "/api/tasks"
    API_ENDPOINT_NEWS    = "/api/news"
    API_ENDPOINT_USERS   = "/api/users"
    API_ENDPOINT_AUTH    = "/api/auth"
    API_ENDPOINT_SCANNER = "/api/scanner"
)

var SERVICE_ROUTES = map[string]string{
    API_ENDPOINT_TASKS:   "http://tasks-service:8080",
    API_ENDPOINT_NEWS:    "http://news-service:8080",
    API_ENDPOINT_USERS:   "http://user-service:8080",
    API_ENDPOINT_AUTH:    "http://auth-service:8080",
    API_ENDPOINT_SCANNER: "http://scanner-service:8080",
}
```

### Обработка Недоступности Сервисов

```go
// ✅ Правильно - проверка доступности и graceful error
func (g *Gateway) handleServiceUnavailable(w http.ResponseWriter, r *http.Request, serviceURL string) {
    // Log failure
    g.logger.Error("Service unavailable",
        slog.String("service", serviceURL),
        slog.String("path", r.URL.Path),
    )
    
    // Optionally notify monitoring service
    go g.notifyServiceFailure(serviceURL)
    
    // Return user-friendly error
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusServiceUnavailable)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "success": false,
        "error": map[string]string{
            "code":    "SERVICE_UNAVAILABLE",
            "message": "Service temporarily unavailable. Please try again later.",
        },
    })
}
```

### Health Checks

```go
// ✅ Правильно - периодическая проверка здоровья сервисов
func (g *Gateway) startHealthChecks(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            g.checkAllServices()
        }
    }
}

func (g *Gateway) checkServiceHealth(serviceURL string) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    req, err := http.NewRequestWithContext(ctx, "GET", serviceURL+"/health", nil)
    if err != nil {
        return err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("service unhealthy: status %d", resp.StatusCode)
    }
    
    return nil
}
```

---

## База Данных (GORM)

### Модели с GORM

```go
// ✅ Правильно - модель с правильными GORM тегами
type User struct {
    ID          string    `gorm:"primaryKey;type:uuid;default:gen_random_uuid()" json:"id"`
    Name        string    `gorm:"type:varchar(255);not null" json:"name"`
    Email       string    `gorm:"type:varchar(255);uniqueIndex;not null" json:"email"`
    Phone       string    `gorm:"type:varchar(20)" json:"phone,omitempty"`
    TelegramID  *string   `gorm:"type:varchar(255);index" json:"telegram_id,omitempty"`
    AuthentikID *string   `gorm:"type:varchar(255);index" json:"authentik_id,omitempty"`
    CreatedAt   time.Time `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt   time.Time `gorm:"autoUpdateTime" json:"updated_at"`
    DeletedAt   gorm.DeletedAt `gorm:"index" json:"deleted_at,omitempty"`
}

// Указываем имя таблицы
func (User) TableName() string {
    return "users"
}
```

### AutoMigrate - Основной Способ Миграций

```go
// ✅ Правильно - AutoMigrate при старте приложения
func NewDatabase(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
        NowFunc: func() time.Time {
            return time.Now().UTC()
        },
    })
    if err != nil {
        return nil, fmt.Errorf("failed to connect to database: %w", err)
    }
    
    // Run AutoMigrate
    if err := db.AutoMigrate(
        &User{},
        &Task{},
        &News{},
        // ... все модели
    ); err != nil {
        return nil, fmt.Errorf("failed to run migrations: %w", err)
    }
    
    // Configure connection pool
    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }
    
    sqlDB.SetMaxOpenConns(25)
    sqlDB.SetMaxIdleConns(5)
    sqlDB.SetConnMaxLifetime(5 * time.Minute)
    
    return db, nil
}
```

### Repository Pattern

```go
// ✅ Правильно - Repository интерфейс и имплементация
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, limit, offset int) ([]*User, error)
}

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) GetByID(ctx context.Context, id string) (*User, error) {
    var user User
    if err := r.db.WithContext(ctx).Where("id = ?", id).First(&user).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("failed to get user: %w", err)
    }
    return &user, nil
}
```

### Транзакции

```go
// ✅ Правильно - используй транзакции для связанных операций
func (s *UserService) CreateUserWithProfile(ctx context.Context, user *User, profile *Profile) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
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

## React/TypeScript Guidelines

### Структура Frontend

```
frontend/
├── src/
│   ├── components/          # Переиспользуемые компоненты
│   │   ├── common/         # Общие UI компоненты
│   │   └── features/       # Фича-специфичные
│   ├── pages/              # Страницы
│   ├── hooks/              # Custom hooks
│   ├── services/           # API клиенты
│   ├── types/              # TypeScript типы
│   ├── contexts/           # React Contexts
│   └── utils/              # Утилиты
```

### Компоненты с TypeScript

```typescript
// ✅ Правильно - компонент с явными типами
interface UserCardProps {
  user: User;
  onEdit?: (id: string) => void;
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
      <p>{user.email}</p>
      {onEdit && <button onClick={handleEdit}>Edit</button>}
    </div>
  );
};
```

### Custom Hooks

```typescript
// ✅ Правильно - hook с правильными типами
interface UseUserReturn {
  user: User | null;
  loading: boolean;
  error: Error | null;
  refetch: () => Promise<void>;
}

export const useUser = (userId: string): UseUserReturn => {
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

### API Client

```typescript
// ✅ Правильно - типизированный API client
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080';

async function apiRequest<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const response = await fetch(`${API_BASE_URL}${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });
  
  if (!response.ok) {
    if (response.status === 503) {
      throw new Error('Service temporarily unavailable. Please try again later.');
    }
    const error = await response.json().catch(() => ({ message: 'Unknown error' }));
    throw new Error(error.message || `HTTP error! status: ${response.status}`);
  }
  
  return response.json();
}

// Типизированные функции для API
export const userService = {
  async getUser(id: string): Promise<User> {
    return apiRequest<User>(`/api/users/${id}`);
  },
  
  async createUser(user: CreateUserDTO): Promise<User> {
    return apiRequest<User>('/api/users', {
      method: 'POST',
      body: JSON.stringify(user),
    });
  },
};
```

---

## API Design & OpenAPI

### OpenAPI Specification

**Используем OpenAPI 3.0+ и Scalar для документации**

#### Базовая структура openapi.yaml

```yaml
openapi: 3.0.3
info:
  title: Microservices Application API
  description: API для микросервисного приложения с Gateway
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com

servers:
  - url: http://localhost:8080
    description: Local development
  - url: https://api.example.com
    description: Production

components:
  schemas:
    # Общие схемы
    SuccessResponse:
      type: object
      properties:
        success:
          type: boolean
          example: true
        data:
          type: object
    
    ErrorResponse:
      type: object
      properties:
        success:
          type: boolean
          example: false
        error:
          type: object
          properties:
            code:
              type: string
              example: "INVALID_INPUT"
            message:
              type: string
              example: "Invalid input provided"
    
    # Модель User
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        email:
          type: string
          format: email
        phone:
          type: string
        telegram_id:
          type: string
        authentik_id:
          type: string
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

paths:
  /api/auth/register:
    post:
      tags:
        - Authentication
      summary: Register new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
                email:
                  type: string
                  format: email
                password:
                  type: string
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SuccessResponse'
        '400':
          description: Invalid input
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
```

### Интеграция Scalar в Gateway

```go
// ✅ Правильно - добавить Scalar UI в Gateway
import (
    "embed"
    "net/http"
)

//go:embed openapi.yaml
var openapiSpec embed.FS

func setupRoutes(router *mux.Router) {
    // Serve OpenAPI spec
    router.HandleFunc("/openapi.yaml", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/yaml")
        http.FileServer(http.FS(openapiSpec)).ServeHTTP(w, r)
    })
    
    // Serve Scalar UI
    router.HandleFunc("/docs", func(w http.ResponseWriter, r *http.Request) {
        html := `
<!DOCTYPE html>
<html>
<head>
    <title>API Documentation</title>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
</head>
<body>
    <script id="api-reference" data-url="/openapi.yaml"></script>
    <script src="https://cdn.jsdelivr.net/npm/@scalar/api-reference"></script>
</body>
</html>
        `
        w.Header().Set("Content-Type", "text/html")
        w.Write([]byte(html))
    })
}
```

### REST API Standards

#### HTTP Methods
- `GET` - получение ресурсов
- `POST` - создание ресурсов
- `PUT` - полное обновление ресурса
- `PATCH` - частичное обновление
- `DELETE` - удаление ресурса

#### URL Structure
```
GET    /api/v1/users           # Список пользователей
GET    /api/v1/users/:id       # Получить пользователя
POST   /api/v1/users           # Создать пользователя
PUT    /api/v1/users/:id       # Обновить пользователя
PATCH  /api/v1/users/:id       # Частичное обновление
DELETE /api/v1/users/:id       # Удалить пользователя

# Вложенные ресурсы
GET    /api/v1/users/:id/tasks # Задачи пользователя
```

#### Response Format

```go
// ✅ Правильно - стандартизированный формат ответа
type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   *APIError   `json:"error,omitempty"`
}

type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

// Success response
c.JSON(http.StatusOK, Response{
    Success: true,
    Data:    user,
})

// Error response
c.JSON(http.StatusBadRequest, Response{
    Success: false,
    Error: &APIError{
        Code:    "INVALID_INPUT",
        Message: "Email is required",
    },
})
```

#### HTTP Status Codes

- `200 OK` - успешный GET/PUT/PATCH/DELETE
- `201 Created` - успешный POST
- `204 No Content` - успешный DELETE без тела
- `400 Bad Request` - ошибка валидации
- `401 Unauthorized` - не аутентифицирован
- `403 Forbidden` - не авторизован
- `404 Not Found` - ресурс не найден
- `409 Conflict` - конфликт (дубликат)
- `500 Internal Server Error` - внутренняя ошибка
- `503 Service Unavailable` - сервис недоступен

---

## Testing

### Философия Тестирования

**Обязательные требования:**
- ✅ **Весь код должен быть покрыт тестами**
- ✅ **Тесты доступности сервисов** - проверка работоспособности
- ✅ **Тесты недоступности** - проверка graceful handling
- ✅ **Используй моки** для внешних зависимостей

### Go Unit Tests

```go
// ✅ Правильно - table-driven tests
func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        user    *User
        wantErr bool
        errType error
    }{
        {
            name: "valid user",
            user: &User{Name: "John", Email: "john@example.com"},
            wantErr: false,
        },
        {
            name: "invalid email",
            user: &User{Name: "John", Email: "invalid"},
            wantErr: true,
            errType: ErrInvalidEmail,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockRepo := new(MockUserRepository)
            service := NewUserService(mockRepo, logger)
            
            if !tt.wantErr {
                mockRepo.On("Create", mock.Anything, tt.user).Return(nil)
            }
            
            err := service.CreateUser(context.Background(), tt.user)
            
            if tt.wantErr {
                assert.Error(t, err)
                if tt.errType != nil {
                    assert.ErrorIs(t, err, tt.errType)
                }
            } else {
                assert.NoError(t, err)
                mockRepo.AssertExpectations(t)
            }
        })
    }
}
```

### Mock Repository

```go
// ✅ Правильно - используй testify/mock
type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Create(ctx context.Context, user *User) error {
    args := m.Called(ctx, user)
    return args.Error(0)
}

func (m *MockUserRepository) GetByID(ctx context.Context, id string) (*User, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}
```

### Service Availability Tests

```go
// ✅ Правильно - тест доступности сервиса
func TestGateway_ServiceAvailable(t *testing.T) {
    // Start mock service
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    }))
    defer server.Close()
    
    gateway := NewGateway(DefaultConfig())
    gateway.serviceRoutes["/api/test"] = server.URL
    
    req := httptest.NewRequest("GET", "/api/test/data", nil)
    w := httptest.NewRecorder()
    
    gateway.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusOK, w.Code)
}

// ✅ Правильно - тест недоступности
func TestGateway_ServiceUnavailable(t *testing.T) {
    gateway := NewGateway(DefaultConfig())
    gateway.serviceRoutes["/api/test"] = "http://unavailable-service:8080"
    
    req := httptest.NewRequest("GET", "/api/test/data", nil)
    w := httptest.NewRecorder()
    
    gateway.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusServiceUnavailable, w.Code)
    
    var response ErrorResponse
    json.NewDecoder(w.Body).Decode(&response)
    assert.Contains(t, response.Error.Message, "unavailable")
}
```

### React Tests

```typescript
// ✅ Правильно - component test
import { render, screen, fireEvent } from '@testing-library/react';
import { UserCard } from './UserCard';

describe('UserCard', () => {
  const mockUser: User = {
    id: '1',
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
    
    expect(handleEdit).toHaveBeenCalledWith('1');
  });
});
```

---

## Docker & DevOps

### Multi-stage Dockerfile для Go

```dockerfile
# ✅ Правильно - multi-stage build
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

FROM alpine:latest

RUN apk --no-cache add ca-certificates curl

WORKDIR /root/

COPY --from=builder /app/server .

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["./server"]
```

### Docker Compose

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  gateway:
    build: ./gateway
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/appdb?sslmode=disable
      - TASKS_SERVICE_URL=http://tasks-service:8080
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
```

### Health Check Endpoint

```go
// ✅ Правильно - health check для каждого сервиса
func (h *Handler) HealthCheck(w http.ResponseWriter, r *http.Request) {
    // Check database connection
    if err := h.db.Exec("SELECT 1").Error; err != nil {
        http.Error(w, "Database unavailable", http.StatusServiceUnavailable)
        return
    }
    
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{
        "status":  "healthy",
        "service": "auth-service",
    })
}
```

---

## Git Workflow

### Branch Naming

```
main                       # Продакшн
develop                    # Разработка

feature/user-auth         # Новая фича
bugfix/login-error        # Исправление бага
hotfix/security-patch     # Срочное исправление
refactor/user-service     # Рефакторинг
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
git commit -m "docs(openapi): add user endpoints documentation"
```

### Коммиты после завершения задачи

```bash
# После каждой завершённой задачи:
1. Проверь работоспособность
2. Запусти тесты
3. Обнови документацию
4. Закоммить:

git add .
git commit -m "feat(auth): implement user registration

- Add User model with GORM
- Implement POST /api/auth/register endpoint
- Add unit and integration tests
- Update OpenAPI documentation"
```

---

## Security

### Общие Правила

1. **Никогда не коммитьте секреты**
   ```bash
   # .gitignore
   .env
   .env.local
   *.pem
   *.key
   secrets/
   ```

2. **Input Validation**
   ```go
   // ✅ Правильно - всегда валидируй входные данные
   type CreateUserRequest struct {
       Email string `json:"email" binding:"required,email"`
       Name  string `json:"name" binding:"required,min=2,max=100"`
   }
   
   if err := c.ShouldBindJSON(&req); err != nil {
       return BadRequest(err)
   }
   ```

3. **SQL Injection Protection**
   ```go
   // ✅ Правильно - GORM защищает от SQL injection
   db.Where("email = ?", userEmail).First(&user)
   
   // ❌ ОПАСНО
   db.Raw(fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", userEmail))
   ```

4. **CORS Configuration**
   ```go
   // ✅ Правильно - настройка CORS
   config := cors.DefaultConfig()
   config.AllowOrigins = []string{"https://yourdomain.com"}
   config.AllowMethods = []string{"GET", "POST", "PUT", "DELETE"}
   config.AllowHeaders = []string{"Origin", "Content-Type", "Authorization"}
   router.Use(cors.New(config))
   ```

---

## Чек-лист Качества Кода

Перед тем как считать задачу завершённой, проверь:

- [ ] ✅ Код работает и протестирован вручную (curl/Postman)
- [ ] ✅ Unit тесты написаны и проходят (`go test ./...` или `npm test`)
- [ ] ✅ Integration тесты (где нужно) проходят
- [ ] ✅ OpenAPI документация обновлена
- [ ] ✅ README сервиса обновлён
- [ ] ✅ Нет linter ошибок (`golangci-lint run` или `npm run lint`)
- [ ] ✅ Код следует соглашениям именования
- [ ] ✅ Логирование добавлено для важных операций
- [ ] ✅ Обработка ошибок реализована правильно
- [ ] ✅ Health check endpoint работает
- [ ] ✅ Self code review проведён

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
- [OpenAPI Specification](https://swagger.io/specification/)
- [Scalar Documentation](https://github.com/scalar/scalar)
- [REST API Best Practices](https://restfulapi.net/)

### Testing
- [Testify (Go)](https://github.com/stretchr/testify)
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/)

---

**Помните:** Эти правила созданы для улучшения качества кода и ускорения разработки. Следуйте итеративному подходу: делайте по одной задаче, проверяйте, тестируйте, документируйте, и только потом переходите к следующей.
