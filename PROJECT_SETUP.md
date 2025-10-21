# Project Setup Guide - Vibe Kanban

## Overview
This document provides step-by-step instructions for setting up the Vibe Kanban development environment.

---

## Prerequisites

### Required Software
- **Node.js:** v20.x LTS or higher
- **npm:** v10.x or higher (comes with Node.js)
- **Docker:** v24.x or higher
- **Docker Compose:** v2.x or higher
- **Git:** v2.x or higher
- **PostgreSQL:** v15.x (or use Docker)
- **Redis:** v7.x (or use Docker)

### Optional Tools
- **VS Code** with extensions:
  - ESLint
  - Prettier
  - TypeScript
  - Prisma
  - Docker
  - GitLens
- **Postman** or **Insomnia** for API testing
- **TablePlus** or **pgAdmin** for database management

---

## Project Structure

```
vibe-kanban/
├── frontend/                  # React frontend application
│   ├── public/               # Static assets
│   ├── src/
│   │   ├── components/       # Reusable UI components
│   │   │   ├── Board/
│   │   │   ├── Card/
│   │   │   ├── Column/
│   │   │   └── common/
│   │   ├── pages/            # Page components
│   │   │   ├── Login/
│   │   │   ├── Dashboard/
│   │   │   ├── Board/
│   │   │   └── Analytics/
│   │   ├── hooks/            # Custom React hooks
│   │   ├── services/         # API service layer
│   │   ├── store/            # State management
│   │   ├── types/            # TypeScript types
│   │   ├── utils/            # Utility functions
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── tailwind.config.js
│
├── backend/                   # NestJS backend application
│   ├── src/
│   │   ├── modules/
│   │   │   ├── auth/         # Authentication module
│   │   │   ├── users/        # User management
│   │   │   ├── boards/       # Board management
│   │   │   ├── columns/      # Column operations
│   │   │   ├── cards/        # Card operations
│   │   │   ├── comments/     # Comments
│   │   │   ├── attachments/  # File uploads
│   │   │   └── websocket/    # Real-time features
│   │   ├── common/
│   │   │   ├── decorators/
│   │   │   ├── filters/
│   │   │   ├── guards/
│   │   │   ├── interceptors/
│   │   │   └── pipes/
│   │   ├── config/           # Configuration
│   │   ├── database/         # Database setup
│   │   │   ├── migrations/
│   │   │   └── seeds/
│   │   ├── main.ts
│   │   └── app.module.ts
│   ├── test/                 # E2E tests
│   ├── package.json
│   ├── tsconfig.json
│   ├── nest-cli.json
│   └── prisma/
│       └── schema.prisma
│
├── docker/                    # Docker configurations
│   ├── nginx/
│   │   └── nginx.conf
│   ├── postgres/
│   │   └── init.sql
│   └── redis/
│       └── redis.conf
│
├── docs/                      # Additional documentation
│   ├── api/
│   ├── architecture/
│   └── deployment/
│
├── .github/                   # GitHub workflows
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
├── docker-compose.yml         # Development environment
├── docker-compose.prod.yml    # Production environment
├── .gitignore
├── .env.example
└── README.md
```

---

## Setup Instructions

### 1. Clone Repository

```bash
git clone https://github.com/your-org/vibe-kanban.git
cd vibe-kanban
```

### 2. Environment Configuration

#### Backend Environment Variables

Create `.env` file in `backend/` directory:

```env
# Application
NODE_ENV=development
PORT=3000
API_PREFIX=api/v1

# Database
DATABASE_URL=postgresql://kanban_user:kanban_pass@localhost:5432/kanban_dev
DATABASE_POOL_MIN=2
DATABASE_POOL_MAX=10

# Redis
REDIS_URL=redis://localhost:6379
REDIS_TTL=3600

# Authentication
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_EXPIRY=1h
JWT_REFRESH_SECRET=your-refresh-secret-key
JWT_REFRESH_EXPIRY=30d

# File Upload
UPLOAD_MAX_SIZE=10485760
AWS_REGION=us-east-1
AWS_S3_BUCKET=vibe-kanban-uploads
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
EMAIL_FROM=noreply@vibekanban.com

# Frontend URL
FRONTEND_URL=http://localhost:5173

# CORS
CORS_ORIGIN=http://localhost:5173

# Rate Limiting
RATE_LIMIT_TTL=60
RATE_LIMIT_MAX=100

# Logging
LOG_LEVEL=debug
```

#### Frontend Environment Variables

Create `.env` file in `frontend/` directory:

```env
# API
VITE_API_URL=http://localhost:3000/api/v1
VITE_WS_URL=ws://localhost:3000

# Application
VITE_APP_NAME=Vibe Kanban
VITE_APP_VERSION=1.0.0

# Features
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_ERROR_TRACKING=false

# Analytics (optional)
VITE_GA_TRACKING_ID=UA-XXXXXXXXX-X

# Sentry (optional)
VITE_SENTRY_DSN=https://xxx@sentry.io/xxx
```

### 3. Using Docker (Recommended for Quick Start)

#### Start Development Environment

```bash
# Start all services (PostgreSQL, Redis, Backend, Frontend)
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down

# Stop and remove volumes (clean slate)
docker-compose down -v
```

The application will be available at:
- Frontend: http://localhost:5173
- Backend API: http://localhost:3000
- API Docs: http://localhost:3000/api/docs

### 4. Manual Setup (Without Docker)

#### Backend Setup

```bash
cd backend

# Install dependencies
npm install

# Generate Prisma client
npx prisma generate

# Run database migrations
npx prisma migrate dev

# Seed database with sample data
npx prisma db seed

# Start development server
npm run start:dev
```

Backend will run on http://localhost:3000

#### Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Start development server
npm run dev
```

Frontend will run on http://localhost:5173

---

## Database Setup

### Option 1: Using Docker

```bash
# PostgreSQL and Redis are automatically started with docker-compose
docker-compose up -d postgres redis
```

### Option 2: Local Installation

#### PostgreSQL Setup

```bash
# macOS (using Homebrew)
brew install postgresql@15
brew services start postgresql@15

# Ubuntu/Debian
sudo apt update
sudo apt install postgresql-15
sudo systemctl start postgresql

# Create database and user
psql postgres
CREATE USER kanban_user WITH PASSWORD 'kanban_pass';
CREATE DATABASE kanban_dev OWNER kanban_user;
GRANT ALL PRIVILEGES ON DATABASE kanban_dev TO kanban_user;
\q
```

#### Redis Setup

```bash
# macOS
brew install redis
brew services start redis

# Ubuntu/Debian
sudo apt install redis-server
sudo systemctl start redis
```

### Database Migrations

```bash
cd backend

# Create new migration
npx prisma migrate dev --name migration_name

# Apply migrations
npx prisma migrate deploy

# Reset database (WARNING: destroys all data)
npx prisma migrate reset

# View database in Prisma Studio
npx prisma studio
```

---

## Running Tests

### Backend Tests

```bash
cd backend

# Run all tests
npm test

# Run unit tests
npm run test:unit

# Run integration tests
npm run test:integration

# Run with coverage
npm run test:cov

# Run in watch mode
npm run test:watch
```

### Frontend Tests

```bash
cd frontend

# Run unit tests
npm run test

# Run with coverage
npm run test:coverage

# Run in watch mode
npm run test:watch

# Run E2E tests
npm run test:e2e

# Run E2E tests with UI
npm run test:e2e:ui
```

---

## Development Workflow

### 1. Running Development Servers

```bash
# Terminal 1: Backend
cd backend
npm run start:dev

# Terminal 2: Frontend
cd frontend
npm run dev

# Terminal 3: Database (if using Prisma Studio)
cd backend
npx prisma studio
```

### 2. Code Quality

#### Linting

```bash
# Backend
cd backend
npm run lint
npm run lint:fix

# Frontend
cd frontend
npm run lint
npm run lint:fix
```

#### Formatting

```bash
# Backend
cd backend
npm run format

# Frontend
cd frontend
npm run format
```

#### Type Checking

```bash
# Backend
cd backend
npm run type-check

# Frontend
cd frontend
npm run type-check
```

### 3. Pre-commit Hooks

The project uses Husky for Git hooks:

```bash
# Install Husky
npm install -D husky

# Initialize hooks
npx husky install

# Hooks will automatically run on:
# - pre-commit: lint-staged, format
# - pre-push: tests
```

---

## API Documentation

### Swagger/OpenAPI

Once the backend is running, access interactive API documentation at:
- **URL:** http://localhost:3000/api/docs
- **JSON:** http://localhost:3000/api/docs-json

### Testing API with cURL

```bash
# Register user
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123!",
    "firstName": "John",
    "lastName": "Doe"
  }'

# Login
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123!"
  }'

# Create board (with auth token)
curl -X POST http://localhost:3000/api/v1/boards \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{
    "name": "My Project",
    "description": "Project board"
  }'
```

---

## Troubleshooting

### Common Issues

#### Port Already in Use

```bash
# Find process using port 3000
lsof -i :3000

# Kill process
kill -9 <PID>

# Or change port in .env
PORT=3001
```

#### Database Connection Error

```bash
# Check PostgreSQL is running
pg_isready

# Check connection string
psql "postgresql://kanban_user:kanban_pass@localhost:5432/kanban_dev"

# Reset database
cd backend
npx prisma migrate reset
```

#### Redis Connection Error

```bash
# Check Redis is running
redis-cli ping

# Should return: PONG

# Start Redis if not running
redis-server
```

#### Prisma Client Not Generated

```bash
cd backend
npx prisma generate
```

#### Node Modules Issues

```bash
# Clear node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Clear npm cache
npm cache clean --force
```

#### Docker Issues

```bash
# Remove all containers and volumes
docker-compose down -v

# Remove all images
docker-compose down --rmi all

# Rebuild from scratch
docker-compose build --no-cache
docker-compose up -d
```

---

## Development Best Practices

### Git Workflow

```bash
# Create feature branch
git checkout -b feature/add-card-labels

# Make changes and commit
git add .
git commit -m "feat: add label support to cards"

# Push to remote
git push origin feature/add-card-labels

# Create pull request on GitHub
```

### Commit Message Convention

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add new feature
fix: bug fix
docs: documentation changes
style: formatting changes
refactor: code refactoring
test: add tests
chore: build/tooling changes
```

### Branch Naming

```
feature/short-description
bugfix/issue-number-description
hotfix/critical-bug
refactor/component-name
docs/update-readme
```

---

## Performance Optimization

### Frontend

```bash
# Analyze bundle size
cd frontend
npm run build
npm run analyze

# Optimize images
npm install -D imagemin imagemin-webp
```

### Backend

```bash
# Enable production mode
NODE_ENV=production

# Use PM2 for process management
npm install -g pm2
pm2 start dist/main.js --name kanban-api -i max

# Monitor performance
pm2 monit
```

---

## Deployment

### Build for Production

#### Frontend

```bash
cd frontend
npm run build

# Output in dist/ directory
# Upload to CDN or static hosting
```

#### Backend

```bash
cd backend
npm run build

# Output in dist/ directory
# Deploy to Node.js server
```

### Using Docker for Production

```bash
# Build production images
docker-compose -f docker-compose.prod.yml build

# Start production services
docker-compose -f docker-compose.prod.yml up -d

# View logs
docker-compose -f docker-compose.prod.yml logs -f
```

---

## Security Checklist

Before deploying to production:

- [ ] Change all default passwords and secrets
- [ ] Enable HTTPS/SSL certificates
- [ ] Configure CORS properly
- [ ] Set up rate limiting
- [ ] Enable security headers (helmet.js)
- [ ] Implement input validation
- [ ] Set up monitoring and logging
- [ ] Configure backup strategy
- [ ] Review and update dependencies
- [ ] Enable database encryption at rest
- [ ] Set up firewall rules
- [ ] Configure environment variables properly
- [ ] Disable debug mode and verbose logging
- [ ] Enable audit logging
- [ ] Set up intrusion detection

---

## Monitoring Setup

### Application Monitoring

```bash
# Install PM2
npm install -g pm2

# Start with monitoring
pm2 start dist/main.js --name kanban-api
pm2 monit

# Setup PM2 monitoring service
pm2 install pm2-server-monit
```

### Log Aggregation

```bash
# Using Winston for structured logging
# Logs are automatically written to:
# - logs/error.log (errors only)
# - logs/combined.log (all logs)
```

### Database Monitoring

```bash
# Enable PostgreSQL logging
# Edit postgresql.conf:
log_statement = 'all'
log_duration = on

# View slow queries
SELECT * FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```

---

## Resources

### Official Documentation
- [React Documentation](https://react.dev)
- [NestJS Documentation](https://docs.nestjs.com)
- [Prisma Documentation](https://www.prisma.io/docs)
- [PostgreSQL Documentation](https://www.postgresql.org/docs)
- [Redis Documentation](https://redis.io/documentation)

### Learning Resources
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [Socket.io Guide](https://socket.io/docs/v4/)

### Tools
- [Postman](https://www.postman.com)
- [TablePlus](https://tableplus.com)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [VS Code](https://code.visualstudio.com)

---

## Support

For issues and questions:
- Create an issue on GitHub
- Check existing documentation
- Review API documentation at /api/docs
- Contact the development team

---

## License

This project is proprietary software. All rights reserved.

---

## Contributors

Maintained by the Vibe Kanban development team.

For contribution guidelines, see CONTRIBUTING.md
