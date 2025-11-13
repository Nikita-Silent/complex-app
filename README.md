# Microservices Application

Backend application with React frontend that integrates with external APIs. Built using microservices architecture with API Gateway pattern.

## Architecture

- **Backend**: Go microservices
- **Frontend**: React + TypeScript + Vite
- **Database**: PostgreSQL with GORM
- **API Gateway**: Custom Go implementation
- **Containerization**: Docker Compose

## Development Rules

This project uses Cursor Rules (MDC format) to maintain consistent coding standards and best practices. Rules are automatically applied based on file patterns.

See [`.cursorrules/README.mdc`](.cursorrules/README.mdc) for complete documentation.

### Quick Links

- **[Architecture](.cursorrules/architecture.mdc)** - Microservices architecture and system design
- **[Go Guidelines](.cursorrules/go-guidelines.mdc)** - Go coding standards and conventions
- **[Gateway Guidelines](.cursorrules/gateway-guidelines.mdc)** - API Gateway implementation
- **[Database Guidelines](.cursorrules/database-guidelines.mdc)** - GORM and migrations
- **[React Guidelines](.cursorrules/react-guidelines.mdc)** - React + TypeScript patterns
- **[Testing Guidelines](.cursorrules/testing-guidelines.mdc)** - Testing strategies
- **[Project Structure](.cursorrules/project-structure.mdc)** - Monorepo organization
- **[Docker Guidelines](.cursorrules/docker-guidelines.mdc)** - Docker Compose setup

## Getting Started

### Prerequisites
- Go 1.21+
- Node.js 18+
- Docker and Docker Compose
- PostgreSQL (or use Docker)

### Running with Docker Compose

```bash
# Start all services
docker-compose up --build

# Start in detached mode
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down
```

### Local Development

#### Backend (Go)
```bash
# Run gateway (migrations run automatically via GORM AutoMigrate)
go run gateway/cmd/server/main.go

# Run service (migrations run automatically via GORM AutoMigrate)
go run services/auth-service/cmd/server/main.go

# Run tests
go test ./...
```

#### Frontend (React)
```bash
cd frontend
npm install
npm run dev
```

## Services

- **Gateway** - API Gateway (port 8080)
- **Auth Service** - Authentication (Telegram, Authentik)
- **Tasks Service** - Task management
- **News Service** - News management with webhooks
- **User Service** - User profile management
- **Scanner Service** - QR/barcode scanning

## Contributing

Please refer to the development rules in `.cursorrules/` directory for coding standards and best practices.
