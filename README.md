# Vibe Kanban

A modern, real-time Kanban board application built with React, NestJS, and PostgreSQL.

## Overview

Vibe Kanban is a collaborative task management tool that helps teams visualize their workflow, track progress, and improve productivity. Built with cutting-edge technologies, it provides a seamless experience for managing projects of any size.

## Key Features

- **Real-time Collaboration** - See updates instantly as your team makes changes
- **Drag & Drop Interface** - Intuitive card movement between columns
- **Rich Card Details** - Descriptions, checklists, attachments, comments, and more
- **Team Management** - Invite members, assign tasks, and manage permissions
- **Labels & Filters** - Organize and find cards quickly
- **Analytics Dashboard** - Track team performance with cycle time, burndown charts, and more
- **Mobile Responsive** - Works seamlessly on desktop, tablet, and mobile
- **Keyboard Shortcuts** - Power user features for efficient navigation
- **Accessible** - WCAG 2.1 Level AA compliant

## Technology Stack

### Frontend
- React 18 with TypeScript
- Tailwind CSS for styling
- Zustand for state management
- @dnd-kit for drag and drop
- Socket.io for real-time updates
- Vite for blazing-fast builds

### Backend
- NestJS framework
- Prisma ORM
- PostgreSQL database
- Redis for caching
- Socket.io for WebSocket connections
- JWT authentication

### DevOps
- Docker & Docker Compose
- GitHub Actions for CI/CD
- Prometheus & Grafana for monitoring
- Sentry for error tracking

## Quick Start

### Prerequisites
- Node.js 20.x or higher
- Docker and Docker Compose
- Git

### Installation

1. Clone the repository:
```bash
git clone https://github.com/your-org/vibe-kanban.git
cd vibe-kanban
```

2. Copy environment variables:
```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

3. Start with Docker:
```bash
docker-compose up -d
```

4. Access the application:
- Frontend: http://localhost:5173
- Backend API: http://localhost:3000
- API Documentation: http://localhost:3000/api/docs

## Documentation

- [Requirements Document](./REQUIREMENTS.md) - Complete feature requirements and user stories
- [Technical Specification](./TECHNICAL_SPEC.md) - Architecture, database schema, and API documentation
- [Project Setup Guide](./PROJECT_SETUP.md) - Detailed setup instructions and troubleshooting

## Development

### Running Locally

#### Backend
```bash
cd backend
npm install
npx prisma migrate dev
npm run start:dev
```

#### Frontend
```bash
cd frontend
npm install
npm run dev
```

### Testing

```bash
# Backend tests
cd backend
npm test
npm run test:e2e

# Frontend tests
cd frontend
npm test
npm run test:e2e
```

### Code Quality

```bash
# Linting
npm run lint
npm run lint:fix

# Formatting
npm run format

# Type checking
npm run type-check
```

## Project Structure

```
vibe-kanban/
├── frontend/          # React application
├── backend/           # NestJS API
├── docker/            # Docker configurations
├── docs/              # Additional documentation
├── .github/           # CI/CD workflows
└── README.md
```

## API Documentation

Interactive API documentation is available at `/api/docs` when the backend server is running.

Key endpoints:
- `POST /api/v1/auth/register` - User registration
- `POST /api/v1/auth/login` - User login
- `GET /api/v1/boards` - List boards
- `POST /api/v1/boards` - Create board
- `GET /api/v1/boards/:id` - Get board with columns and cards
- `POST /api/v1/cards` - Create card
- `PUT /api/v1/cards/:id/move` - Move card

## Contributing

We welcome contributions! Please follow these steps:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/amazing-feature`
3. Commit your changes: `git commit -m 'feat: add amazing feature'`
4. Push to the branch: `git push origin feature/amazing-feature`
5. Open a Pull Request

### Commit Convention

We follow [Conventional Commits](https://www.conventionalcommits.org/):
- `feat:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation changes
- `style:` - Code style changes
- `refactor:` - Code refactoring
- `test:` - Test additions or changes
- `chore:` - Build process or tooling changes

## Deployment

### Docker Production Build

```bash
docker-compose -f docker-compose.prod.yml up -d
```

### Manual Deployment

1. Build frontend:
```bash
cd frontend
npm run build
```

2. Build backend:
```bash
cd backend
npm run build
```

3. Start with PM2:
```bash
pm2 start dist/main.js --name kanban-api
```

See [Project Setup Guide](./PROJECT_SETUP.md) for detailed deployment instructions.

## Environment Variables

### Backend (.env)
```env
DATABASE_URL=postgresql://user:password@localhost:5432/kanban
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-secret-key
AWS_S3_BUCKET=your-bucket-name
```

### Frontend (.env)
```env
VITE_API_URL=http://localhost:3000/api/v1
VITE_WS_URL=ws://localhost:3000
```

See `.env.example` files for complete configuration options.

## Security

- All API requests require JWT authentication
- Passwords are hashed with bcrypt
- HTTPS enforced in production
- CORS configured for allowed origins
- Rate limiting enabled
- Input validation on all endpoints
- SQL injection prevention via Prisma ORM
- XSS protection with sanitization

Report security vulnerabilities to security@vibekanban.com

## Performance

- Average API response time: < 200ms
- Real-time update latency: < 500ms
- Supports 1000+ concurrent users per board
- Horizontal scaling via Docker/Kubernetes
- Redis caching for frequently accessed data
- Database query optimization with indexes
- CDN integration for static assets

## Browser Support

- Chrome (latest 2 versions)
- Firefox (latest 2 versions)
- Safari (latest 2 versions)
- Edge (latest 2 versions)
- Mobile browsers (iOS Safari, Chrome Android)

## Accessibility

- WCAG 2.1 Level AA compliant
- Keyboard navigation support
- Screen reader compatible
- High contrast mode
- Configurable text size
- Focus indicators

## License

This project is proprietary software. All rights reserved.

## Support

- Documentation: [docs/](./docs/)
- Issues: [GitHub Issues](https://github.com/your-org/vibe-kanban/issues)
- Email: support@vibekanban.com

## Roadmap

### Version 1.0 (Current)
- Core Kanban functionality
- Real-time collaboration
- User management
- Basic analytics

### Version 1.1 (Planned)
- Mobile native apps
- Advanced automation
- Custom fields
- API webhooks

### Version 2.0 (Future)
- AI-powered suggestions
- Multi-board views
- Advanced reporting
- Integrations marketplace

## Acknowledgments

Built with love by the Vibe Kanban team.

Special thanks to:
- React team for the amazing framework
- NestJS team for the robust backend framework
- Prisma team for the excellent ORM
- All open-source contributors

## Status

![Build Status](https://img.shields.io/github/actions/workflow/status/your-org/vibe-kanban/ci.yml)
![Coverage](https://img.shields.io/codecov/c/github/your-org/vibe-kanban)
![License](https://img.shields.io/badge/license-Proprietary-red)
![Version](https://img.shields.io/badge/version-1.0.0-blue)

---

Made with ❤️ by the Vibe Kanban Team
