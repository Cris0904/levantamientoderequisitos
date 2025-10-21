# Arquitectura Técnica Detallada - Plataforma Vibe Kanban

## Índice
1. [Visión General](#1-visión-general)
2. [Arquitectura de Microservicios](#2-arquitectura-de-microservicios)
3. [Stack Tecnológico por Proyecto](#3-stack-tecnológico-por-proyecto)
4. [Infraestructura y DevOps](#4-infraestructura-y-devops)
5. [Seguridad](#5-seguridad)
6. [Integraciones](#6-integraciones)
7. [Flujos de Datos Detallados](#7-flujos-de-datos-detallados)

---

## 1. Visión General

### 1.1 Principios Arquitectónicos
- **Modularidad:** Componentes independientes y reutilizables
- **Escalabilidad:** Diseño preparado para crecer de 100 a 10,000+ usuarios
- **Seguridad:** Security-first approach con encriptación end-to-end
- **Observabilidad:** Monitoreo completo de métricas y logs
- **Developer Experience:** Tooling moderno y automatización

### 1.2 Diagrama de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USUARIOS FINALES                            │
│   [Traficers]    [Estrategas]    [Empresarios]    [Administradores] │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         CAPA DE FRONTEND                             │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │  Next.js App     │  │  React Dashboard │  │  Mobile (Future) │ │
│  │  (SSR + SSG)     │  │  (SPA)           │  │  React Native    │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│                                                                      │
│  Estado Global: Zustand + React Query                               │
│  UI Components: shadcn/ui + Tailwind CSS                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      CAPA DE API GATEWAY                             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Next.js API Routes / tRPC                                    │  │
│  │  - Rate Limiting                                              │  │
│  │  - Authentication Middleware                                  │  │
│  │  - Request Validation (Zod)                                   │  │
│  │  - CORS Configuration                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
┌──────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
│   META ADS SERVICE   │ │ WORKFLOW SERVICE │ │  STRATEGY SERVICE    │
│                      │ │                  │ │                      │
│ - Campaigns CRUD     │ │ - Marketplace    │ │ - Subscriptions      │
│ - Metrics Fetching   │ │ - Publishing     │ │ - Estrategas Mgmt    │
│ - Alerts Engine      │ │ - Payments       │ │ - Performance Track  │
│ - MCP Integration    │ │ - Reviews        │ │ - Revenue Share      │
└──────────────────────┘ └──────────────────┘ └──────────────────────┘
         │                        │                      │
         └────────────────────────┼──────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CAPA DE INTEGRACIONES                             │
│                                                                      │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌────────────┐      │
│  │  Meta API │  │  n8n API  │  │  Stripe   │  │ SendGrid   │      │
│  │  Client   │  │  Client   │  │  Client   │  │  (Email)   │      │
│  └───────────┘  └───────────┘  └───────────┘  └────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
┌──────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
│    PostgreSQL        │ │      Redis       │ │    TimescaleDB       │
│  (Relational Data)   │ │  (Cache/Queue)   │ │  (Time Series)       │
│                      │ │                  │ │                      │
│ - Users              │ │ - Sessions       │ │ - Campaign Metrics   │
│ - Workflows          │ │ - Jobs (Bull)    │ │ - Performance Data   │
│ - Strategies         │ │ - Rate Limits    │ │ - Analytics          │
│ - Subscriptions      │ │ - Pub/Sub        │ │                      │
└──────────────────────┘ └──────────────────┘ └──────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CAPA DE OBSERVABILIDAD                            │
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │  Sentry    │  │ LogRocket  │  │ Prometheus │  │  Grafana   │   │
│  │  (Errors)  │  │ (Sessions) │  │ (Metrics)  │  │ (Dashboard)│   │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Arquitectura de Microservicios

### 2.1 Meta Ads Service

**Responsabilidades:**
- Gestión de conexiones con cuentas de Meta
- CRUD de campañas, ad sets, ads
- Fetching de métricas en tiempo real
- Sistema de alertas basadas en umbrales
- Integración con MCP personalizado

**Endpoints Principales:**
```typescript
// Autenticación
POST   /api/meta/auth/connect
GET    /api/meta/auth/accounts
DELETE /api/meta/auth/disconnect/:accountId

// Campañas
GET    /api/meta/campaigns
POST   /api/meta/campaigns
PATCH  /api/meta/campaigns/:id
DELETE /api/meta/campaigns/:id
POST   /api/meta/campaigns/:id/pause
POST   /api/meta/campaigns/:id/resume

// Métricas
GET    /api/meta/metrics/:campaignId
GET    /api/meta/metrics/:campaignId/realtime
GET    /api/meta/metrics/summary

// Alertas
GET    /api/meta/alerts
POST   /api/meta/alerts
PATCH  /api/meta/alerts/:id
DELETE /api/meta/alerts/:id
```

**Stack:**
- Runtime: Node.js 20+ con TypeScript
- Framework: Fastify (alta performance)
- Validación: Zod
- ORM: Prisma
- Meta SDK: @facebook/business-sdk

**Estructura de Carpetas:**
```
services/meta-ads/
├── src/
│   ├── controllers/
│   │   ├── auth.controller.ts
│   │   ├── campaigns.controller.ts
│   │   ├── metrics.controller.ts
│   │   └── alerts.controller.ts
│   ├── services/
│   │   ├── meta-api.service.ts
│   │   ├── alerts-engine.service.ts
│   │   └── metrics-fetcher.service.ts
│   ├── models/
│   │   └── prisma schema
│   ├── utils/
│   │   ├── meta-client.ts
│   │   └── encryption.ts
│   ├── types/
│   │   └── meta.types.ts
│   └── index.ts
├── tests/
├── prisma/
│   └── schema.prisma
├── package.json
└── tsconfig.json
```

### 2.2 Workflow Service

**Responsabilidades:**
- Marketplace de workflows
- Sistema de publicación y moderación
- Gestión de compras y pagos
- Reviews y ratings
- Integración con n8n para importación

**Endpoints Principales:**
```typescript
// Workflows
GET    /api/workflows
GET    /api/workflows/:id
POST   /api/workflows
PATCH  /api/workflows/:id
DELETE /api/workflows/:id
POST   /api/workflows/:id/publish

// Marketplace
GET    /api/marketplace/workflows
GET    /api/marketplace/workflows/:id
POST   /api/marketplace/purchase/:workflowId

// Reviews
GET    /api/workflows/:id/reviews
POST   /api/workflows/:id/reviews
PATCH  /api/reviews/:id
DELETE /api/reviews/:id

// Ideas
GET    /api/ideas
POST   /api/ideas
POST   /api/ideas/:id/vote
POST   /api/ideas/:id/comment
```

**Stack:**
- Runtime: Node.js 20+ con TypeScript
- Framework: Fastify
- Validación: Zod
- ORM: Prisma
- Pagos: Stripe SDK
- Storage: AWS S3 (videos/thumbnails)

### 2.3 Strategy Service

**Responsabilidades:**
- Gestión de estrategias predefinidas
- Sistema de suscripciones
- Perfiles de estrategas
- Programa de postulación
- Revenue share y pagos
- Replicación automática de estrategias

**Endpoints Principales:**
```typescript
// Estrategias
GET    /api/strategies
GET    /api/strategies/:id
POST   /api/strategies (admin/estratega)
PATCH  /api/strategies/:id
DELETE /api/strategies/:id

// Suscripciones
GET    /api/subscriptions
POST   /api/subscriptions/subscribe/:strategyId
DELETE /api/subscriptions/:id
PATCH  /api/subscriptions/:id/pause
PATCH  /api/subscriptions/:id/resume

// Estrategas
GET    /api/estrategas
GET    /api/estrategas/:id
POST   /api/estrategas/apply
PATCH  /api/estrategas/:id/verify (admin)

// Performance
GET    /api/strategies/:id/performance
GET    /api/subscriptions/:id/metrics
```

**Stack:**
- Runtime: Node.js 20+ con TypeScript
- Framework: Fastify
- Validación: Zod
- ORM: Prisma
- n8n: API Client personalizado
- Jobs: Bull + Redis (replicación async)

---

## 3. Stack Tecnológico por Proyecto

### 3.1 PROYECTO 1: MCP as a Service

#### Frontend
```json
{
  "dependencies": {
    "next": "^14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "zustand": "^4.4.0",
    "@tanstack/react-query": "^5.0.0",
    "axios": "^1.6.0",
    "socket.io-client": "^4.6.0",
    "recharts": "^2.10.0",
    "date-fns": "^2.30.0",
    "tailwindcss": "^3.4.0",
    "@radix-ui/react-*": "latest",
    "lucide-react": "^0.294.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "^18.2.0",
    "eslint": "^8.55.0",
    "prettier": "^3.1.0"
  }
}
```

#### Backend (Meta Ads Service)
```json
{
  "dependencies": {
    "fastify": "^4.25.0",
    "@fastify/cors": "^8.4.0",
    "@fastify/jwt": "^7.2.0",
    "prisma": "^5.7.0",
    "@prisma/client": "^5.7.0",
    "zod": "^3.22.0",
    "@facebook/business-sdk": "^18.0.0",
    "bull": "^4.12.0",
    "ioredis": "^5.3.0",
    "socket.io": "^4.6.0"
  }
}
```

#### Base de Datos
```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String
  role          UserRole  @default(TRAFICER)
  isVerified    Boolean   @default(false)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  metaAccounts  MetaAccount[]
  alerts        Alert[]
  subscriptions StrategySubscription[]
}

enum UserRole {
  TRAFICER
  ESTRATEGA
  EMPRESARIO
  ADMIN
}

model MetaAccount {
  id              String   @id @default(cuid())
  userId          String
  user            User     @relation(fields: [userId], references: [id])
  metaAccountId   String
  accessToken     String   // encrypted
  refreshToken    String?  // encrypted
  expiresAt       DateTime?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@unique([userId, metaAccountId])
}

model Alert {
  id          String      @id @default(cuid())
  userId      String
  user        User        @relation(fields: [userId], references: [id])
  campaignId  String
  type        AlertType
  condition   Json        // { metric: "spend", operator: ">", value: 100 }
  isActive    Boolean     @default(true)
  lastFired   DateTime?
  createdAt   DateTime    @default(now())

  @@index([userId, isActive])
}

enum AlertType {
  BUDGET_THRESHOLD
  PERFORMANCE_DROP
  CAMPAIGN_ERROR
  DAILY_SUMMARY
}
```

### 3.2 PROYECTO 2: Marketplace de Workflows

#### Schema Prisma
```prisma
model Workflow {
  id              String    @id @default(cuid())
  creatorId       String
  creator         User      @relation(fields: [creatorId], references: [id])
  title           String
  description     String    @db.Text
  n8nJson         Json
  category        String
  price           Decimal   @db.Decimal(10, 2)
  videoUrl        String?
  thumbnailUrl    String?
  downloadsCount  Int       @default(0)
  ratingAvg       Decimal?  @db.Decimal(3, 2)
  isPublished     Boolean   @default(false)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  purchases       WorkflowPurchase[]
  reviews         WorkflowReview[]

  @@index([category, isPublished])
  @@index([creatorId])
}

model WorkflowPurchase {
  id          String    @id @default(cuid())
  workflowId  String
  workflow    Workflow  @relation(fields: [workflowId], references: [id])
  buyerId     String
  buyer       User      @relation(fields: [buyerId], references: [id])
  pricePaid   Decimal   @db.Decimal(10, 2)
  stripeId    String?   @unique
  purchasedAt DateTime  @default(now())

  @@unique([workflowId, buyerId])
  @@index([buyerId])
}

model WorkflowReview {
  id          String    @id @default(cuid())
  workflowId  String
  workflow    Workflow  @relation(fields: [workflowId], references: [id])
  reviewerId  String
  reviewer    User      @relation(fields: [reviewerId], references: [id])
  rating      Int       // 1-5
  comment     String?   @db.Text
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@unique([workflowId, reviewerId])
  @@index([workflowId])
}

model WorkflowIdea {
  id          String    @id @default(cuid())
  authorId    String
  author      User      @relation(fields: [authorId], references: [id])
  title       String
  description String    @db.Text
  category    String?
  votes       Int       @default(0)
  status      IdeaStatus @default(PENDING)
  createdAt   DateTime  @default(now())

  comments    IdeaComment[]
  voters      IdeaVote[]

  @@index([status])
  @@index([authorId])
}

enum IdeaStatus {
  PENDING
  APPROVED
  IMPLEMENTED
  REJECTED
}

model IdeaVote {
  id      String        @id @default(cuid())
  ideaId  String
  idea    WorkflowIdea  @relation(fields: [ideaId], references: [id])
  userId  String
  user    User          @relation(fields: [userId], references: [id])

  @@unique([ideaId, userId])
}

model IdeaComment {
  id        String        @id @default(cuid())
  ideaId    String
  idea      WorkflowIdea  @relation(fields: [ideaId], references: [id])
  authorId  String
  author    User          @relation(fields: [authorId], references: [id])
  content   String        @db.Text
  createdAt DateTime      @default(now())

  @@index([ideaId])
}
```

#### Integración con Stripe
```typescript
// services/workflow/src/services/payment.service.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
});

export class PaymentService {
  async createCheckoutSession(workflowId: string, buyerId: string) {
    const workflow = await prisma.workflow.findUnique({
      where: { id: workflowId },
      include: { creator: true },
    });

    const session = await stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      line_items: [
        {
          price_data: {
            currency: 'usd',
            product_data: {
              name: workflow.title,
              description: workflow.description,
            },
            unit_amount: Math.round(workflow.price.toNumber() * 100),
          },
          quantity: 1,
        },
      ],
      mode: 'payment',
      success_url: `${process.env.FRONTEND_URL}/marketplace/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.FRONTEND_URL}/marketplace/workflows/${workflowId}`,
      metadata: {
        workflowId,
        buyerId,
        creatorId: workflow.creatorId,
      },
    });

    return session;
  }

  async handleWebhook(event: Stripe.Event) {
    switch (event.type) {
      case 'checkout.session.completed':
        await this.fulfillPurchase(event.data.object as Stripe.Checkout.Session);
        break;
      // otros eventos...
    }
  }

  private async fulfillPurchase(session: Stripe.Checkout.Session) {
    const { workflowId, buyerId, creatorId } = session.metadata!;

    // Crear registro de compra
    await prisma.workflowPurchase.create({
      data: {
        workflowId,
        buyerId,
        pricePaid: session.amount_total! / 100,
        stripeId: session.id,
      },
    });

    // Incrementar contador de descargas
    await prisma.workflow.update({
      where: { id: workflowId },
      data: { downloadsCount: { increment: 1 } },
    });

    // Crear transferencia al creador (70%)
    const creatorAmount = Math.round(session.amount_total! * 0.7);
    // Implementar lógica de Stripe Connect...
  }
}
```

### 3.3 PROYECTO 3: Copy Strategy

#### Schema Prisma
```prisma
model Strategy {
  id                  String    @id @default(cuid())
  name                String
  description         String    @db.Text
  estrategaId         String
  estratega           User      @relation(fields: [estrategaId], references: [id])
  baseN8nTemplate     Json
  configurableParams  Json      // Schema de parámetros
  monthlyPrice        Decimal   @db.Decimal(10, 2)
  isActive            Boolean   @default(true)
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt

  subscriptions       StrategySubscription[]

  @@index([estrategaId, isActive])
}

model StrategySubscription {
  id              String    @id @default(cuid())
  strategyId      String
  strategy        Strategy  @relation(fields: [strategyId], references: [id])
  userId          String
  user            User      @relation(fields: [userId], references: [id])
  metaAccountId   String
  params          Json      // Valores específicos del usuario
  mcpToken        String    @unique
  n8nWorkflowId   String?   // ID del workflow en n8n
  status          SubscriptionStatus @default(ACTIVE)
  startedAt       DateTime  @default(now())
  cancelledAt     DateTime?

  metrics         StrategyMetric[]

  @@index([userId, status])
  @@index([strategyId, status])
}

enum SubscriptionStatus {
  ACTIVE
  PAUSED
  CANCELLED
  PENDING
}

model StrategyMetric {
  id              String                @id @default(cuid())
  subscriptionId  String
  subscription    StrategySubscription  @relation(fields: [subscriptionId], references: [id])
  timestamp       DateTime              @default(now())
  investment      Decimal               @db.Decimal(10, 2)
  impressions     BigInt
  clicks          BigInt
  conversions     Int
  revenue         Decimal?              @db.Decimal(10, 2)
  roi             Decimal?              @db.Decimal(5, 2)
  rawData         Json

  @@index([subscriptionId, timestamp])
}

model EstrategaProfile {
  id                    String    @id @default(cuid())
  userId                String    @unique
  user                  User      @relation(fields: [userId], references: [id])
  bio                   String?   @db.Text
  portfolioUrl          String?
  linkedinUrl           String?
  isVerified            Boolean   @default(false)
  verificationDate      DateTime?
  revenueSharePercent   Decimal   @db.Decimal(5, 2) @default(40.00)
  totalEarnings         Decimal   @db.Decimal(10, 2) @default(0)
  activeSubscribers     Int       @default(0)

  applications          EstrategaApplication[]

  @@index([isVerified])
}

model EstrategaApplication {
  id                    String    @id @default(cuid())
  userId                String
  user                  User      @relation(fields: [userId], references: [id])
  strategyDescription   String    @db.Text
  expectedResults       String?   @db.Text
  portfolioUrls         Json?
  status                ApplicationStatus @default(PENDING)
  submittedAt           DateTime  @default(now())
  reviewedAt            DateTime?
  reviewerNotes         String?   @db.Text

  @@index([userId])
  @@index([status])
}

enum ApplicationStatus {
  PENDING
  UNDER_REVIEW
  APPROVED
  REJECTED
}
```

#### Servicio de Replicación de Estrategias
```typescript
// services/strategy/src/services/strategy-replication.service.ts
import { PrismaClient } from '@prisma/client';
import { N8nClient } from '../clients/n8n.client';
import { generateMCPToken } from '../utils/token';

export class StrategyReplicationService {
  constructor(
    private prisma: PrismaClient,
    private n8nClient: N8nClient
  ) {}

  async subscribeToStrategy(
    userId: string,
    strategyId: string,
    metaAccountId: string,
    params: Record<string, any>
  ) {
    // 1. Validar parámetros contra schema
    const strategy = await this.prisma.strategy.findUnique({
      where: { id: strategyId },
      include: { estratega: true },
    });

    if (!strategy) throw new Error('Strategy not found');

    this.validateParams(params, strategy.configurableParams);

    // 2. Generar token MCP único
    const mcpToken = generateMCPToken(userId, strategyId);

    // 3. Crear workflow en n8n
    const workflowJson = this.injectParams(
      strategy.baseN8nTemplate,
      params,
      mcpToken
    );

    const n8nWorkflow = await this.n8nClient.createWorkflow({
      name: `${strategy.name} - User ${userId}`,
      nodes: workflowJson.nodes,
      connections: workflowJson.connections,
      settings: {
        executionOrder: 'v1',
      },
      active: true,
    });

    // 4. Crear suscripción
    const subscription = await this.prisma.strategySubscription.create({
      data: {
        strategyId,
        userId,
        metaAccountId,
        params,
        mcpToken,
        n8nWorkflowId: n8nWorkflow.id,
        status: 'ACTIVE',
      },
    });

    // 5. Incrementar contador de estratega
    await this.prisma.estrategaProfile.update({
      where: { userId: strategy.estrategaId },
      data: { activeSubscribers: { increment: 1 } },
    });

    return subscription;
  }

  private validateParams(
    params: Record<string, any>,
    schema: any
  ): void {
    // Validación con Zod o similar
    // Implementar según schema de configurableParams
  }

  private injectParams(
    template: any,
    params: Record<string, any>,
    mcpToken: string
  ): any {
    // Clonar template
    const workflow = JSON.parse(JSON.stringify(template));

    // Inyectar parámetros en nodos específicos
    workflow.nodes = workflow.nodes.map((node: any) => {
      if (node.type === 'mcp-custom') {
        return {
          ...node,
          credentials: {
            mcpApi: {
              token: mcpToken,
            },
          },
        };
      }

      // Reemplazar variables de parámetros
      if (node.parameters) {
        node.parameters = this.replaceVariables(
          node.parameters,
          params
        );
      }

      return node;
    });

    return workflow;
  }

  private replaceVariables(
    obj: any,
    params: Record<string, any>
  ): any {
    if (typeof obj === 'string') {
      // Reemplazar {{variable}} con valor
      return obj.replace(/\{\{(\w+)\}\}/g, (_, key) => {
        return params[key] ?? `{{${key}}}`;
      });
    }

    if (Array.isArray(obj)) {
      return obj.map(item => this.replaceVariables(item, params));
    }

    if (typeof obj === 'object' && obj !== null) {
      const result: any = {};
      for (const [key, value] of Object.entries(obj)) {
        result[key] = this.replaceVariables(value, params);
      }
      return result;
    }

    return obj;
  }
}
```

---

## 4. Infraestructura y DevOps

### 4.1 Entornos

```
Development (local)
├── Frontend: localhost:3000
├── Backend: localhost:4000
├── PostgreSQL: localhost:5432
├── Redis: localhost:6379
└── n8n: localhost:5678

Staging
├── Frontend: staging.vibekanban.com
├── Backend: api-staging.vibekanban.com
├── PostgreSQL: RDS (AWS) / Supabase
├── Redis: ElastiCache / Upstash
└── n8n: n8n.staging.vibekanban.com

Production
├── Frontend: app.vibekanban.com
├── Backend: api.vibekanban.com
├── PostgreSQL: RDS (AWS) / Supabase
├── Redis: ElastiCache / Upstash
└── n8n: n8n.vibekanban.com
```

### 4.2 CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run test
      - run: npm run lint

  deploy-frontend:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: vercel/action@v2
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'

  deploy-backend:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: superfly/flyctl-actions@v1
        with:
          args: 'deploy --remote-only'
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

### 4.3 Variables de Entorno

```bash
# .env.example

# Database
DATABASE_URL="postgresql://user:password@localhost:5432/vibekanban"
TIMESCALEDB_URL="postgresql://user:password@localhost:5432/vibekanban_metrics"

# Redis
REDIS_URL="redis://localhost:6379"

# Meta API
META_APP_ID="your_app_id"
META_APP_SECRET="your_app_secret"
META_REDIRECT_URI="http://localhost:3000/api/auth/meta/callback"

# n8n
N8N_API_URL="https://n8n.vibekanban.com/api/v1"
N8N_API_KEY="your_n8n_api_key"

# Stripe
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
STRIPE_CONNECT_CLIENT_ID="ca_..."

# Auth
JWT_SECRET="your_jwt_secret"
JWT_EXPIRES_IN="7d"

# Email
SENDGRID_API_KEY="SG...."
FROM_EMAIL="noreply@vibekanban.com"

# Frontend
NEXT_PUBLIC_API_URL="http://localhost:4000"
NEXT_PUBLIC_SOCKET_URL="ws://localhost:4000"

# Monitoring
SENTRY_DSN="https://..."
LOGR OCKET_APP_ID="..."

# Feature Flags
ENABLE_MARKETPLACE="true"
ENABLE_COPY_STRATEGY="false"
```

---

## 5. Seguridad

### 5.1 Autenticación y Autorización

```typescript
// middleware/auth.middleware.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import jwt from 'jsonwebtoken';

export async function authMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
) {
  try {
    const token = request.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      return reply.code(401).send({ error: 'Unauthorized' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as {
      userId: string;
      role: string;
    };

    request.user = decoded;
  } catch (error) {
    return reply.code(401).send({ error: 'Invalid token' });
  }
}

export function requireRole(...roles: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    if (!request.user) {
      return reply.code(401).send({ error: 'Unauthorized' });
    }

    if (!roles.includes(request.user.role)) {
      return reply.code(403).send({ error: 'Forbidden' });
    }
  };
}
```

### 5.2 Encriptación de Datos Sensibles

```typescript
// utils/encryption.ts
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex');

export function encrypt(text: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);

  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  const authTag = cipher.getAuthTag();

  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
}

export function decrypt(encryptedData: string): string {
  const [ivHex, authTagHex, encrypted] = encryptedData.split(':');

  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');

  const decipher = crypto.createDecipheriv(ALGORITHM, KEY, iv);
  decipher.setAuthTag(authTag);

  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}
```

### 5.3 Rate Limiting

```typescript
// plugins/rate-limit.plugin.ts
import rateLimit from '@fastify/rate-limit';

export default async function rateLimitPlugin(fastify: FastifyInstance) {
  await fastify.register(rateLimit, {
    global: true,
    max: 100,
    timeWindow: '15 minutes',
    redis: fastify.redis, // Usa Redis para distributed rate limiting
    errorResponseBuilder: (req, context) => {
      return {
        error: 'Too many requests',
        retryAfter: context.after,
      };
    },
  });
}
```

---

## 6. Integraciones

### 6.1 Cliente de Meta API

```typescript
// clients/meta-api.client.ts
import { FacebookAdsApi, Campaign, AdAccount } from 'facebook-nodejs-business-sdk';

export class MetaAPIClient {
  private api: FacebookAdsApi;

  constructor(accessToken: string) {
    this.api = FacebookAdsApi.init(accessToken);
  }

  async getCampaigns(accountId: string) {
    const account = new AdAccount(`act_${accountId}`);
    const campaigns = await account.getCampaigns([
      'id',
      'name',
      'status',
      'objective',
      'daily_budget',
      'lifetime_budget',
    ]);

    return campaigns;
  }

  async getCampaignMetrics(campaignId: string, dateRange: { since: string; until: string }) {
    const campaign = new Campaign(campaignId);
    const insights = await campaign.getInsights([
      'impressions',
      'clicks',
      'spend',
      'reach',
      'ctr',
      'cpc',
      'cpp',
      'cpm',
      'frequency',
    ], {
      time_range: dateRange,
    });

    return insights[0];
  }

  async createCampaign(accountId: string, params: any) {
    const account = new AdAccount(`act_${accountId}`);
    const campaign = await account.createCampaign([], params);
    return campaign;
  }

  async pauseCampaign(campaignId: string) {
    const campaign = new Campaign(campaignId);
    await campaign.update([], { status: 'PAUSED' });
  }

  async resumeCampaign(campaignId: string) {
    const campaign = new Campaign(campaignId);
    await campaign.update([], { status: 'ACTIVE' });
  }
}
```

### 6.2 Cliente de n8n

```typescript
// clients/n8n.client.ts
import axios, { AxiosInstance } from 'axios';

export class N8nClient {
  private client: AxiosInstance;

  constructor() {
    this.client = axios.create({
      baseURL: process.env.N8N_API_URL,
      headers: {
        'X-N8N-API-KEY': process.env.N8N_API_KEY,
      },
    });
  }

  async createWorkflow(workflow: {
    name: string;
    nodes: any[];
    connections: any;
    settings?: any;
    active?: boolean;
  }) {
    const response = await this.client.post('/workflows', workflow);
    return response.data;
  }

  async updateWorkflow(workflowId: string, updates: any) {
    const response = await this.client.patch(`/workflows/${workflowId}`, updates);
    return response.data;
  }

  async deleteWorkflow(workflowId: string) {
    await this.client.delete(`/workflows/${workflowId}`);
  }

  async activateWorkflow(workflowId: string) {
    const response = await this.client.patch(`/workflows/${workflowId}`, {
      active: true,
    });
    return response.data;
  }

  async deactivateWorkflow(workflowId: string) {
    const response = await this.client.patch(`/workflows/${workflowId}`, {
      active: false,
    });
    return response.data;
  }

  async getWorkflowExecutions(workflowId: string) {
    const response = await this.client.get(`/executions`, {
      params: { workflowId },
    });
    return response.data;
  }
}
```

---

## 7. Flujos de Datos Detallados

### 7.1 Flujo de Suscripción a Estrategia

```
┌─────────────┐
│   Usuario   │
│  Frontend   │
└──────┬──────┘
       │ 1. POST /api/subscriptions/subscribe/:strategyId
       │    Body: { metaAccountId, params: {...} }
       ▼
┌──────────────────────┐
│  Strategy Service    │
│                      │
│  2. Validar params   │
│  3. Generar MCP token│
└──────┬───────────────┘
       │ 4. Clonar template
       │    + inyectar params
       ▼
┌──────────────────────┐
│    n8n Client        │
│                      │
│  5. Create workflow  │
│     en n8n cloud     │
└──────┬───────────────┘
       │ 6. Retorna workflow ID
       ▼
┌──────────────────────┐
│  Strategy Service    │
│                      │
│  7. Crear registro   │
│     StrategySubscription │
│  8. Incrementar      │
│     activeSubscribers│
└──────┬───────────────┘
       │ 9. Retornar subscription
       ▼
┌─────────────┐
│   Usuario   │
│  Redirect   │
│  Dashboard  │
└─────────────┘
```

### 7.2 Flujo de Ejecución de Workflow

```
┌──────────────────────┐
│   n8n Workflow       │
│   (corriendo cada X) │
└──────┬───────────────┘
       │ 1. Trigger (schedule/webhook)
       ▼
┌──────────────────────┐
│   MCP Node           │
│                      │
│  2. Llama a          │
│     MCP API con token│
└──────┬───────────────┘
       │ 3. Request con token
       ▼
┌──────────────────────┐
│  Meta Ads Service    │
│                      │
│  4. Validar token    │
│  5. Obtener userId   │
│     y metaAccountId  │
│  6. Decrypt credentials │
└──────┬───────────────┘
       │ 7. Llamar Meta API
       ▼
┌──────────────────────┐
│   Meta Graph API     │
│                      │
│  8. Retornar datos   │
│     o ejecutar acción│
└──────┬───────────────┘
       │ 9. Response
       ▼
┌──────────────────────┐
│   MCP Node (n8n)     │
│                      │
│  10. Procesar data   │
│  11. Tomar decisión  │
│      (lógica workflow)│
└──────┬───────────────┘
       │ 12. Si hay cambios
       ▼
┌──────────────────────┐
│   MCP Node           │
│                      │
│  13. Actualizar      │
│      campaña         │
└──────┬───────────────┘
       │ 14. Guardar métricas
       ▼
┌──────────────────────┐
│  TimescaleDB         │
│                      │
│  15. INSERT INTO     │
│      strategy_metrics│
└─────────────────────┘
```

---

## 8. Monitoreo y Observabilidad

### 8.1 Métricas Clave (Prometheus)

```typescript
// utils/metrics.ts
import promClient from 'prom-client';

export const register = new promClient.Registry();

// Métricas de negocio
export const activeSubscriptions = new promClient.Gauge({
  name: 'active_subscriptions_total',
  help: 'Total active strategy subscriptions',
  labelNames: ['strategy_id'],
  registers: [register],
});

export const workflowPurchases = new promClient.Counter({
  name: 'workflow_purchases_total',
  help: 'Total workflow purchases',
  labelNames: ['workflow_id'],
  registers: [register],
});

export const revenueGenerated = new promClient.Counter({
  name: 'revenue_generated_dollars',
  help: 'Total revenue generated in dollars',
  labelNames: ['source'], // 'workflow' | 'subscription'
  registers: [register],
});

// Métricas técnicas
export const apiRequestDuration = new promClient.Histogram({
  name: 'api_request_duration_seconds',
  help: 'API request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  registers: [register],
});

export const metaApiErrors = new promClient.Counter({
  name: 'meta_api_errors_total',
  help: 'Total errors from Meta API',
  labelNames: ['error_code'],
  registers: [register],
});
```

### 8.2 Logging Estructurado

```typescript
// utils/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport:
    process.env.NODE_ENV !== 'production'
      ? {
          target: 'pino-pretty',
          options: {
            colorize: true,
          },
        }
      : undefined,
  formatters: {
    level: (label) => {
      return { level: label };
    },
  },
  base: {
    env: process.env.NODE_ENV,
    service: process.env.SERVICE_NAME,
  },
});
```

---

**Documento creado:** 21 de Octubre, 2025
**Versión:** 1.0
**Próxima revisión:** Después de aprobación de stakeholders
