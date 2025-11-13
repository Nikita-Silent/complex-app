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

## Authentik OIDC Integration

### Настройка OIDC Provider

```go
// ✅ Правильно - конфигурация OIDC
type AuthentikConfig struct {
    Issuer       string // https://authentik.example.com/application/o/your-app/
    ClientID     string
    ClientSecret string
    RedirectURL  string // http://localhost:8080/api/auth/authentik/callback
    Scopes       []string // openid, profile, email
}

type OIDCProvider struct {
    config   *AuthentikConfig
    verifier *oidc.IDTokenVerifier
    oauth2   *oauth2.Config
}

func NewOIDCProvider(cfg *AuthentikConfig) (*OIDCProvider, error) {
    ctx := context.Background()
    
    provider, err := oidc.NewProvider(ctx, cfg.Issuer)
    if err != nil {
        return nil, fmt.Errorf("failed to create OIDC provider: %w", err)
    }
    
    // Configure OAuth2
    oauth2Config := &oauth2.Config{
        ClientID:     cfg.ClientID,
        ClientSecret: cfg.ClientSecret,
        RedirectURL:  cfg.RedirectURL,
        Endpoint:     provider.Endpoint(),
        Scopes:       cfg.Scopes,
    }
    
    // Create ID token verifier
    verifier := provider.Verifier(&oidc.Config{
        ClientID: cfg.ClientID,
    })
    
    return &OIDCProvider{
        config:   cfg,
        verifier: verifier,
        oauth2:   oauth2Config,
    }, nil
}
```

### OIDC Authorization Flow

```go
// ✅ Правильно - инициация OIDC flow
func (h *AuthHandler) HandleAuthentikAuthorize(w http.ResponseWriter, r *http.Request) {
    // Generate random state for CSRF protection
    state := generateRandomState()
    
    // Store state in session or cache
    h.stateStore.Set(state, time.Now().Add(10*time.Minute))
    
    // Generate authorization URL
    authURL := h.oidcProvider.oauth2.AuthCodeURL(state, oauth2.AccessTypeOffline)
    
    // Redirect to Authentik
    http.Redirect(w, r, authURL, http.StatusFound)
}

// ✅ Правильно - обработка callback
func (h *AuthHandler) HandleAuthentikCallback(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Verify state parameter
    state := r.URL.Query().Get("state")
    if !h.stateStore.Verify(state) {
        http.Error(w, "Invalid state parameter", http.StatusBadRequest)
        return
    }
    
    // Exchange authorization code for tokens
    code := r.URL.Query().Get("code")
    oauth2Token, err := h.oidcProvider.oauth2.Exchange(ctx, code)
    if err != nil {
        http.Error(w, "Failed to exchange token", http.StatusInternalServerError)
        return
    }
    
    // Extract ID token
    rawIDToken, ok := oauth2Token.Extra("id_token").(string)
    if !ok {
        http.Error(w, "No id_token in response", http.StatusInternalServerError)
        return
    }
    
    // Verify ID token
    idToken, err := h.oidcProvider.verifier.Verify(ctx, rawIDToken)
    if err != nil {
        http.Error(w, "Failed to verify ID token", http.StatusUnauthorized)
        return
    }
    
    // Extract user info from ID token
    var claims struct {
        Sub           string `json:"sub"`
        Email         string `json:"email"`
        EmailVerified bool   `json:"email_verified"`
        Name          string `json:"name"`
        PreferredUsername string `json:"preferred_username"`
    }
    
    if err := idToken.Claims(&claims); err != nil {
        http.Error(w, "Failed to parse claims", http.StatusInternalServerError)
        return
    }
    
    // Find or create user
    user, err := h.userService.FindOrCreateByAuthentikID(ctx, claims.Sub, claims.Email, claims.Name)
    if err != nil {
        http.Error(w, "Failed to process user", http.StatusInternalServerError)
        return
    }
    
    // Generate JWT token for our application
    jwtToken, err := h.generateJWT(user)
    if err != nil {
        http.Error(w, "Failed to generate token", http.StatusInternalServerError)
        return
    }
    
    // Redirect to frontend with token
    redirectURL := fmt.Sprintf("/dashboard?token=%s", jwtToken)
    http.Redirect(w, r, redirectURL, http.StatusFound)
}
```

### User Service для OIDC

```go
// ✅ Правильно - создание/обновление пользователя из OIDC
func (s *UserService) FindOrCreateByAuthentikID(
    ctx context.Context,
    authentikID, email, name string,
) (*User, error) {
    // Try to find existing user by Authentik ID
    user, err := s.repo.FindByAuthentikID(ctx, authentikID)
    if err == nil {
        // User exists, update info if needed
        if user.Name != name || user.Email != email {
            user.Name = name
            user.Email = email
            if err := s.repo.Update(ctx, user); err != nil {
                return nil, fmt.Errorf("failed to update user: %w", err)
            }
        }
        return user, nil
    }
    
    // User doesn't exist by Authentik ID, check by email
    user, err = s.repo.FindByEmail(ctx, email)
    if err == nil {
        // User exists with this email, link Authentik ID
        user.AuthentikID = &authentikID
        if err := s.repo.Update(ctx, user); err != nil {
            return nil, fmt.Errorf("failed to link authentik: %w", err)
        }
        return user, nil
    }
    
    // Create new user
    newUser := &User{
        Name:        name,
        Email:       email,
        AuthentikID: &authentikID,
    }
    
    if err := s.repo.Create(ctx, newUser); err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }
    
    return newUser, nil
}
```

### Environment Variables для Authentik

```bash
# .env.example
# Authentik OIDC Configuration
AUTHENTIK_ISSUER=https://authentik.example.com/application/o/your-app/
AUTHENTIK_CLIENT_ID=your_client_id_here
AUTHENTIK_CLIENT_SECRET=your_client_secret_here
AUTHENTIK_REDIRECT_URL=http://localhost:8080/api/auth/authentik/callback
AUTHENTIK_SCOPES=openid,profile,email
```

### Тестирование OIDC Flow

```go
// ✅ Правильно - mock OIDC provider для тестов
func TestAuthentikCallback(t *testing.T) {
    // Create mock OIDC server
    mockServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/.well-known/openid-configuration" {
            json.NewEncoder(w).Encode(map[string]interface{}{
                "issuer":                 mockServer.URL,
                "authorization_endpoint": mockServer.URL + "/authorize",
                "token_endpoint":         mockServer.URL + "/token",
                "jwks_uri":              mockServer.URL + "/jwks",
            })
        } else if r.URL.Path == "/token" {
            json.NewEncoder(w).Encode(map[string]interface{}{
                "access_token":  "mock_access_token",
                "id_token":      "mock_id_token",
                "refresh_token": "mock_refresh_token",
                "token_type":    "Bearer",
            })
        }
    }))
    defer mockServer.Close()
    
    // Test callback with mock code
    req := httptest.NewRequest("GET", "/api/auth/authentik/callback?code=test_code&state=valid_state", nil)
    w := httptest.NewRecorder()
    
    handler.HandleAuthentikCallback(w, req)
    
    assert.Equal(t, http.StatusFound, w.Code)
}
```

---

## Account Linking (Привязка Аккаунтов)

### Концепция

Пользователь может авторизоваться тремя способами:
1. Email + Password
2. Telegram OAuth
3. Authentik OIDC

**Привязка аккаунтов позволяет:**
- Авторизоваться любым из привязанных способов под одним аккаунтом
- Добавить дополнительные способы входа к существующему аккаунту
- Отвязать способ входа (если есть хотя бы один другой)

### Модель User с поддержкой всех методов

```go
// ✅ Правильно - модель поддерживает все способы аутентификации
type User struct {
    ID              string    `gorm:"primaryKey;type:uuid;default:gen_random_uuid()" json:"id"`
    Name            string    `gorm:"type:varchar(255);not null" json:"name"`
    Email           string    `gorm:"type:varchar(255);uniqueIndex;not null" json:"email"`
    Phone           string    `gorm:"type:varchar(20)" json:"phone,omitempty"`
    
    // Email/Password authentication
    PasswordHash    *string   `gorm:"type:varchar(255)" json:"-"`
    
    // Telegram authentication
    TelegramID      *string   `gorm:"type:varchar(255);index" json:"telegram_id,omitempty"`
    TelegramUsername *string  `gorm:"type:varchar(255)" json:"telegram_username,omitempty"`
    
    // Authentik OIDC authentication
    AuthentikID     *string   `gorm:"type:varchar(255);index" json:"authentik_id,omitempty"`
    AuthentikEmail  *string   `gorm:"type:varchar(255)" json:"authentik_email,omitempty"`
    
    CreatedAt       time.Time `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt       time.Time `gorm:"autoUpdateTime" json:"updated_at"`
    DeletedAt       gorm.DeletedAt `gorm:"index" json:"deleted_at,omitempty"`
}

// Проверка доступных методов аутентификации
func (u *User) HasPasswordAuth() bool {
    return u.PasswordHash != nil
}

func (u *User) HasTelegramAuth() bool {
    return u.TelegramID != nil
}

func (u *User) HasAuthentikAuth() bool {
    return u.AuthentikID != nil
}

func (u *User) AuthMethodsCount() int {
    count := 0
    if u.HasPasswordAuth() {
        count++
    }
    if u.HasTelegramAuth() {
        count++
    }
    if u.HasAuthentikAuth() {
        count++
    }
    return count
}

// Можно ли отвязать метод аутентификации
func (u *User) CanUnlinkAuthMethod() bool {
    return u.AuthMethodsCount() > 1
}
```

### Привязка Telegram к существующему аккаунту

```go
// ✅ Правильно - привязка Telegram
func (h *UserHandler) LinkTelegram(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Получить текущего пользователя из JWT токена
    userID := ctx.Value("user_id").(string)
    user, err := h.userService.GetByID(ctx, userID)
    if err != nil {
        h.sendError(w, http.StatusNotFound, "User not found")
        return
    }
    
    // Проверить, что Telegram еще не привязан
    if user.HasTelegramAuth() {
        h.sendError(w, http.StatusBadRequest, "Telegram already linked to this account")
        return
    }
    
    // Получить данные Telegram из запроса
    var req struct {
        TelegramData map[string]interface{} `json:"telegram_data"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.sendError(w, http.StatusBadRequest, "Invalid request")
        return
    }
    
    // Верифицировать данные от Telegram
    telegramID, username, err := h.telegramService.VerifyData(req.TelegramData)
    if err != nil {
        h.sendError(w, http.StatusUnauthorized, "Invalid Telegram data")
        return
    }
    
    // Проверить, что этот Telegram ID не привязан к другому пользователю
    existingUser, err := h.userService.FindByTelegramID(ctx, telegramID)
    if err == nil && existingUser.ID != user.ID {
        h.sendError(w, http.StatusConflict, "Telegram account already linked to another user")
        return
    }
    
    // Привязать Telegram к текущему пользователю
    user.TelegramID = &telegramID
    user.TelegramUsername = &username
    
    if err := h.userService.Update(ctx, user); err != nil {
        h.sendError(w, http.StatusInternalServerError, "Failed to link Telegram")
        return
    }
    
    h.sendSuccess(w, map[string]interface{}{
        "message":     "Telegram account linked successfully",
        "telegram_id": telegramID,
        "username":    username,
    })
}
```

### Привязка Authentik к существующему аккаунту

```go
// ✅ Правильно - инициация привязки Authentik
func (h *AuthHandler) InitiateAuthentikLinking(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Получить текущего пользователя
    userID := ctx.Value("user_id").(string)
    user, err := h.userService.GetByID(ctx, userID)
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    // Проверить, что Authentik еще не привязан
    if user.HasAuthentikAuth() {
        http.Error(w, "Authentik already linked", http.StatusBadRequest)
        return
    }
    
    // Generate state with link prefix and user ID
    state := fmt.Sprintf("link_%s_%s", userID, generateRandomState())
    h.stateStore.Set(state, userID, time.Now().Add(10*time.Minute))
    
    // Redirect to Authentik
    authURL := h.oidcProvider.oauth2.AuthCodeURL(state, oauth2.AccessTypeOffline)
    http.Redirect(w, r, authURL, http.StatusFound)
}

// ✅ Правильно - callback для привязки Authentik
func (h *AuthHandler) HandleAuthentikLinkingCallback(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Verify state and extract user ID
    state := r.URL.Query().Get("state")
    if !strings.HasPrefix(state, "link_") {
        http.Error(w, "Invalid state", http.StatusBadRequest)
        return
    }
    
    userID, ok := h.stateStore.Get(state)
    if !ok {
        http.Error(w, "Invalid or expired state", http.StatusBadRequest)
        return
    }
    
    // Get current user
    user, err := h.userService.GetByID(ctx, userID.(string))
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    // Exchange code for tokens
    code := r.URL.Query().Get("code")
    oauth2Token, err := h.oidcProvider.oauth2.Exchange(ctx, code)
    if err != nil {
        http.Error(w, "Failed to exchange token", http.StatusInternalServerError)
        return
    }
    
    // Verify ID token and extract claims
    rawIDToken, _ := oauth2Token.Extra("id_token").(string)
    idToken, err := h.oidcProvider.verifier.Verify(ctx, rawIDToken)
    if err != nil {
        http.Error(w, "Failed to verify token", http.StatusUnauthorized)
        return
    }
    
    var claims struct {
        Sub   string `json:"sub"`
        Email string `json:"email"`
        Name  string `json:"name"`
    }
    if err := idToken.Claims(&claims); err != nil {
        http.Error(w, "Failed to parse claims", http.StatusInternalServerError)
        return
    }
    
    // Check if this Authentik ID is already linked to another user
    existingUser, err := h.userService.FindByAuthentikID(ctx, claims.Sub)
    if err == nil && existingUser.ID != user.ID {
        http.Error(w, "Authentik account already linked to another user", http.StatusConflict)
        return
    }
    
    // Link Authentik to current user
    user.AuthentikID = &claims.Sub
    user.AuthentikEmail = &claims.Email
    
    if err := h.userService.Update(ctx, user); err != nil {
        http.Error(w, "Failed to link Authentik", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "success": true,
        "data": map[string]interface{}{
            "message":      "Authentik account linked successfully",
            "authentik_id": claims.Sub,
        },
    })
}
```

### Отвязка метода аутентификации

```go
// ✅ Правильно - отвязка Telegram
func (h *UserHandler) UnlinkTelegram(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    userID := ctx.Value("user_id").(string)
    user, err := h.userService.GetByID(ctx, userID)
    if err != nil {
        h.sendError(w, http.StatusNotFound, "User not found")
        return
    }
    
    // Проверить, что можно отвязать (есть другие методы)
    if !user.CanUnlinkAuthMethod() {
        h.sendError(w, http.StatusBadRequest, "Cannot unlink - at least one authentication method must remain")
        return
    }
    
    // Проверить, что Telegram привязан
    if !user.HasTelegramAuth() {
        h.sendError(w, http.StatusBadRequest, "Telegram is not linked to this account")
        return
    }
    
    // Отвязать Telegram
    user.TelegramID = nil
    user.TelegramUsername = nil
    
    if err := h.userService.Update(ctx, user); err != nil {
        h.sendError(w, http.StatusInternalServerError, "Failed to unlink Telegram")
        return
    }
    
    h.sendSuccess(w, map[string]string{
        "message": "Telegram account unlinked successfully",
    })
}

// ✅ Правильно - отвязка Authentik
func (h *UserHandler) UnlinkAuthentik(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    userID := ctx.Value("user_id").(string)
    user, err := h.userService.GetByID(ctx, userID)
    if err != nil {
        h.sendError(w, http.StatusNotFound, "User not found")
        return
    }
    
    if !user.CanUnlinkAuthMethod() {
        h.sendError(w, http.StatusBadRequest, "Cannot unlink - at least one authentication method must remain")
        return
    }
    
    if !user.HasAuthentikAuth() {
        h.sendError(w, http.StatusBadRequest, "Authentik is not linked to this account")
        return
    }
    
    user.AuthentikID = nil
    user.AuthentikEmail = nil
    
    if err := h.userService.Update(ctx, user); err != nil {
        h.sendError(w, http.StatusInternalServerError, "Failed to unlink Authentik")
        return
    }
    
    h.sendSuccess(w, map[string]string{
        "message": "Authentik account unlinked successfully",
    })
}
```

### Получение статуса привязанных аккаунтов

```go
// ✅ Правильно - получить статус всех методов аутентификации
func (h *UserHandler) GetLinkedAccounts(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    userID := ctx.Value("user_id").(string)
    user, err := h.userService.GetByID(ctx, userID)
    if err != nil {
        h.sendError(w, http.StatusNotFound, "User not found")
        return
    }
    
    response := map[string]interface{}{
        "email_password": user.HasPasswordAuth(),
        "telegram": map[string]interface{}{
            "linked":   user.HasTelegramAuth(),
            "telegram_id": user.TelegramID,
            "username": user.TelegramUsername,
        },
        "authentik": map[string]interface{}{
            "linked":       user.HasAuthentikAuth(),
            "authentik_id": user.AuthentikID,
            "email":        user.AuthentikEmail,
        },
    }
    
    h.sendSuccess(w, response)
}
```

### Логика входа с проверкой всех методов

```go
// ✅ Правильно - вход с любым привязанным методом
func (s *AuthService) AuthenticateUser(ctx context.Context, method string, credentials interface{}) (*User, error) {
    var user *User
    var err error
    
    switch method {
    case "email":
        // Email + Password
        creds := credentials.(EmailPasswordCredentials)
        user, err = s.userRepo.FindByEmail(ctx, creds.Email)
        if err != nil {
            return nil, ErrInvalidCredentials
        }
        if !user.HasPasswordAuth() {
            return nil, ErrEmailAuthNotEnabled
        }
        if !verifyPassword(creds.Password, *user.PasswordHash) {
            return nil, ErrInvalidCredentials
        }
        
    case "telegram":
        // Telegram OAuth
        telegramID := credentials.(string)
        user, err = s.userRepo.FindByTelegramID(ctx, telegramID)
        if err != nil {
            return nil, ErrUserNotFound
        }
        
    case "authentik":
        // Authentik OIDC
        authentikID := credentials.(string)
        user, err = s.userRepo.FindByAuthentikID(ctx, authentikID)
        if err != nil {
            return nil, ErrUserNotFound
        }
    
    default:
        return nil, ErrInvalidAuthMethod
    }
    
    return user, nil
}
```

### Frontend - Страница профиля с управлением аккаунтами

```typescript
// ✅ Правильно - React компонент для управления привязанными аккаунтами
interface LinkedAccountsStatus {
  email_password: boolean;
  telegram: {
    linked: boolean;
    telegram_id?: string;
    username?: string;
  };
  authentik: {
    linked: boolean;
    authentik_id?: string;
    email?: string;
  };
}

export const AccountLinksSection: React.FC = () => {
  const [accounts, setAccounts] = useState<LinkedAccountsStatus | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchLinkedAccounts().then(setAccounts).finally(() => setLoading(false));
  }, []);

  const handleLinkTelegram = async () => {
    // Инициировать Telegram OAuth для привязки
    window.location.href = '/api/users/me/link/telegram';
  };

  const handleLinkAuthentik = async () => {
    // Инициировать Authentik OIDC для привязки
    window.location.href = '/api/users/me/link/authentik';
  };

  const handleUnlink = async (method: 'telegram' | 'authentik') => {
    if (!confirm(`Are you sure you want to unlink ${method}?`)) return;
    
    await fetch(`/api/users/me/unlink/${method}`, { method: 'DELETE' });
    // Refresh status
    const updated = await fetchLinkedAccounts();
    setAccounts(updated);
  };

  if (loading) return <div>Loading...</div>;
  if (!accounts) return <div>Error loading accounts</div>;

  return (
    <div className="account-links">
      <h2>Authentication Methods</h2>
      
      <div className="auth-method">
        <h3>Email & Password</h3>
        <span className={accounts.email_password ? 'linked' : 'not-linked'}>
          {accounts.email_password ? '✓ Configured' : '✗ Not set'}
        </span>
      </div>

      <div className="auth-method">
        <h3>Telegram</h3>
        {accounts.telegram.linked ? (
          <>
            <span className="linked">✓ Linked: @{accounts.telegram.username}</span>
            <button onClick={() => handleUnlink('telegram')}>Unlink</button>
          </>
        ) : (
          <button onClick={handleLinkTelegram}>Link Telegram</button>
        )}
      </div>

      <div className="auth-method">
        <h3>Authentik (SSO)</h3>
        {accounts.authentik.linked ? (
          <>
            <span className="linked">✓ Linked: {accounts.authentik.email}</span>
            <button onClick={() => handleUnlink('authentik')}>Unlink</button>
          </>
        ) : (
          <button onClick={handleLinkAuthentik}>Link Authentik</button>
        )}
      </div>
    </div>
  );
};
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
openapi: 3.1.1
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

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    
    oidcAuth:
      type: openIdConnect
      openIdConnectUrl: https://authentik.example.com/.well-known/openid-configuration

paths:
  /api/auth/register:
    post:
      tags:
        - Authentication
      summary: Register new user
      description: Register a new user with email/password
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - name
                - email
                - password
              properties:
                name:
                  type: string
                  minLength: 2
                  maxLength: 100
                email:
                  type: string
                  format: email
                password:
                  type: string
                  minLength: 8
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - type: object
                    properties:
                      data:
                        $ref: '#/components/schemas/User'
        '400':
          description: Invalid input
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: User already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/auth/login:
    post:
      tags:
        - Authentication
      summary: Login with credentials
      description: Login with email and password
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - email
                - password
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          token:
                            type: string
                            description: JWT access token
                          refresh_token:
                            type: string
                            description: Refresh token
                          user:
                            $ref: '#/components/schemas/User'
        '401':
          description: Invalid credentials
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/auth/telegram/login:
    post:
      tags:
        - Authentication
      summary: Login with Telegram
      description: Authenticate user via Telegram OAuth
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - telegram_data
              properties:
                telegram_data:
                  type: object
                  description: Data from Telegram OAuth callback
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          token:
                            type: string
                          user:
                            $ref: '#/components/schemas/User'
        '401':
          description: Authentication failed
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/auth/authentik/authorize:
    get:
      tags:
        - Authentication
      summary: Initiate Authentik OIDC flow
      description: Redirect user to Authentik for authentication
      responses:
        '302':
          description: Redirect to Authentik login page
          headers:
            Location:
              schema:
                type: string
                example: https://authentik.example.com/application/o/authorize/
  
  /api/auth/authentik/callback:
    get:
      tags:
        - Authentication
      summary: Authentik OIDC callback
      description: Handle callback from Authentik after authentication
      parameters:
        - in: query
          name: code
          required: true
          schema:
            type: string
          description: Authorization code from Authentik
        - in: query
          name: state
          required: true
          schema:
            type: string
          description: State parameter for CSRF protection
      responses:
        '302':
          description: Redirect to application with token
          headers:
            Location:
              schema:
                type: string
                example: /dashboard?token=eyJhbGc...
        '401':
          description: Authentication failed
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/auth/refresh:
    post:
      tags:
        - Authentication
      summary: Refresh access token
      description: Get new access token using refresh token
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - refresh_token
              properties:
                refresh_token:
                  type: string
      responses:
        '200':
          description: Token refreshed successfully
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          token:
                            type: string
                          refresh_token:
                            type: string
        '401':
          description: Invalid refresh token
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  # Account Linking Endpoints
  /api/users/me/linked-accounts:
    get:
      tags:
        - Account Linking
      summary: Get linked accounts status
      description: Get information about which authentication methods are linked to current user
      security:
        - bearerAuth: []
      responses:
        '200':
          description: Linked accounts status
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          email_password:
                            type: boolean
                            description: Has email/password authentication
                          telegram:
                            type: object
                            properties:
                              linked:
                                type: boolean
                              telegram_id:
                                type: string
                                nullable: true
                              username:
                                type: string
                                nullable: true
                          authentik:
                            type: object
                            properties:
                              linked:
                                type: boolean
                              authentik_id:
                                type: string
                                nullable: true
                              email:
                                type: string
                                nullable: true
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/users/me/link/telegram:
    post:
      tags:
        - Account Linking
      summary: Link Telegram account
      description: Link Telegram account to current authenticated user
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - telegram_data
              properties:
                telegram_data:
                  type: object
                  description: Data from Telegram OAuth
      responses:
        '200':
          description: Telegram account linked successfully
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          message:
                            type: string
                            example: Telegram account linked successfully
                          telegram_id:
                            type: string
        '400':
          description: Invalid data or already linked to another account
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Telegram account already linked to another user
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/users/me/link/authentik:
    get:
      tags:
        - Account Linking
      summary: Initiate Authentik account linking
      description: Start OIDC flow to link Authentik account to current user
      security:
        - bearerAuth: []
      responses:
        '302':
          description: Redirect to Authentik for linking
          headers:
            Location:
              schema:
                type: string
                example: https://authentik.example.com/application/o/authorize/?state=link_abc123
  
  /api/users/me/link/authentik/callback:
    get:
      tags:
        - Account Linking
      summary: Authentik linking callback
      description: Handle callback from Authentik for account linking
      security:
        - bearerAuth: []
      parameters:
        - in: query
          name: code
          required: true
          schema:
            type: string
        - in: query
          name: state
          required: true
          schema:
            type: string
          description: State with link_ prefix
      responses:
        '200':
          description: Authentik account linked successfully
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          message:
                            type: string
                            example: Authentik account linked successfully
                          authentik_id:
                            type: string
        '400':
          description: Invalid state or data
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Authentik account already linked to another user
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/users/me/unlink/telegram:
    delete:
      tags:
        - Account Linking
      summary: Unlink Telegram account
      description: Remove Telegram authentication from current user (requires at least one other auth method)
      security:
        - bearerAuth: []
      responses:
        '200':
          description: Telegram account unlinked successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SuccessResponse'
        '400':
          description: Cannot unlink - no other authentication methods available
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
  
  /api/users/me/unlink/authentik:
    delete:
      tags:
        - Account Linking
      summary: Unlink Authentik account
      description: Remove Authentik authentication from current user (requires at least one other auth method)
      security:
        - bearerAuth: []
      responses:
        '200':
          description: Authentik account unlinked successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SuccessResponse'
        '400':
          description: Cannot unlink - no other authentication methods available
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: Unauthorized
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
    image: postgres:17-alpine
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
