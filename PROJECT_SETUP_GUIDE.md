# Guía de Setup de Proyectos - Vibe Platform

**Versión:** 1.0
**Fecha:** 2025-10-21
**Propósito:** Instrucciones paso a paso para configurar los 3 proyectos principales

---

## ÍNDICE DE PROYECTOS

1. **MCP Dashboard** (FASE 0) - Prioridad CRÍTICA
2. **Copy Strategy System** (FASE 2) - Prioridad ALTA
3. **Kanban Board** - Proyecto de ejemplo/referencia

---

# PROYECTO 1: MCP DASHBOARD (FASE 0)

## Descripción
Dashboard centralizado para gestión de campañas Meta Ads con sistema de alertas automáticas.

## Stack Tecnológico
- Frontend: Next.js 14 + TypeScript + Tailwind + shadcn/ui
- Backend: Fastify + TypeScript + Prisma
- Database: PostgreSQL (Supabase)
- Cache/Queues: Redis (Upstash)
- Deploy: Vercel (frontend) + Railway (backend)

## Estructura de Directorios
```
vibe-mcp/
├── apps/
│   ├── web/                 # Next.js App
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   │   ├── login/
│   │   │   │   └── register/
│   │   │   ├── (dashboard)/
│   │   │   │   ├── accounts/
│   │   │   │   ├── campaigns/
│   │   │   │   ├── alerts/
│   │   │   │   └── settings/
│   │   │   └── layout.tsx
│   │   ├── components/
│   │   ├── lib/
│   │   └── package.json
│   │
│   └── api/                 # Fastify Backend
│       ├── src/
│       │   ├── routes/
│       │   │   ├── auth.ts
│       │   │   ├── meta-accounts.ts
│       │   │   ├── campaigns.ts
│       │   │   └── alerts.ts
│       │   ├── services/
│       │   │   ├── meta-api.service.ts
│       │   │   ├── alert.service.ts
│       │   │   └── auth.service.ts
│       │   ├── jobs/
│       │   │   └── alert-processor.ts
│       │   └── server.ts
│       └── package.json
│
├── packages/
│   ├── database/
│   │   ├── prisma/
│   │   │   └── schema.prisma
│   │   └── package.json
│   ├── ui/
│   │   └── components/
│   └── types/
│       └── index.ts
│
└── package.json (root)
```

## Setup Paso a Paso

### PASO 1: Inicializar Proyecto (5 minutos)

```bash
# Crear directorio del proyecto
mkdir vibe-mcp
cd vibe-mcp

# Inicializar pnpm workspace
cat > package.json << 'EOF'
{
  "name": "vibe-mcp",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "clean": "turbo run clean"
  },
  "devDependencies": {
    "turbo": "^1.11.0",
    "typescript": "^5.3.0"
  }
}
EOF

# Crear archivo turbo.json
cat > turbo.json << 'EOF'
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "outputs": []
    }
  }
}
EOF

pnpm install
```

### PASO 2: Setup Frontend (15 minutos)

```bash
# Crear app Next.js
mkdir -p apps/web
cd apps/web
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir --import-alias "@/*"

# Instalar dependencias adicionales
pnpm add zustand @tanstack/react-query axios
pnpm add lucide-react class-variance-authority clsx tailwind-merge
pnpm add @radix-ui/react-dialog @radix-ui/react-dropdown-menu
pnpm add -D @types/node

# Inicializar shadcn/ui
npx shadcn-ui@latest init

# Agregar componentes shadcn necesarios
npx shadcn-ui@latest add button
npx shadcn-ui@latest add card
npx shadcn-ui@latest add input
npx shadcn-ui@latest add label
npx shadcn-ui@latest add table
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add dropdown-menu
npx shadcn-ui@latest add badge
npx shadcn-ui@latest add alert
```

**Crear archivo de configuración de ambiente:**
```bash
cat > .env.local << 'EOF'
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_META_APP_ID=your-meta-app-id
EOF
```

**Crear store de Zustand (lib/store.ts):**
```typescript
import { create } from 'zustand';

interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
}

interface AuthStore {
  user: User | null;
  token: string | null;
  setUser: (user: User | null) => void;
  setToken: (token: string | null) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthStore>((set) => ({
  user: null,
  token: null,
  setUser: (user) => set({ user }),
  setToken: (token) => set({ token }),
  logout: () => set({ user: null, token: null }),
}));
```

**Crear cliente API (lib/api.ts):**
```typescript
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001',
  headers: {
    'Content-Type': 'application/json',
  },
});

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### PASO 3: Setup Backend (20 minutos)

```bash
# Volver a la raíz y crear estructura backend
cd ../..
mkdir -p apps/api/src/{routes,services,jobs,types}
cd apps/api

# Inicializar package.json
cat > package.json << 'EOF'
{
  "name": "@vibe/api",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "migrate": "prisma migrate dev"
  },
  "dependencies": {
    "fastify": "^4.24.0",
    "@fastify/cors": "^8.4.0",
    "@fastify/jwt": "^7.2.0",
    "@fastify/env": "^4.3.0",
    "@prisma/client": "^5.6.0",
    "bcrypt": "^5.1.1",
    "bullmq": "^4.15.0",
    "ioredis": "^5.3.2",
    "axios": "^1.6.0",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "@types/node": "^20.9.0",
    "@types/bcrypt": "^5.0.2",
    "tsx": "^4.6.0",
    "typescript": "^5.3.0",
    "prisma": "^5.6.0"
  }
}
EOF

pnpm install

# Crear tsconfig.json
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
EOF

# Crear archivo .env
cat > .env << 'EOF'
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/vibe_mcp

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-in-production

# Meta API
META_APP_ID=your-meta-app-id
META_APP_SECRET=your-meta-app-secret

# Redis
REDIS_URL=redis://localhost:6379

# Server
PORT=3001
NODE_ENV=development
EOF
```

**Crear servidor Fastify (src/server.ts):**
```typescript
import Fastify from 'fastify';
import cors from '@fastify/cors';
import jwt from '@fastify/jwt';
import env from '@fastify/env';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
const fastify = Fastify({ logger: true });

const schema = {
  type: 'object',
  required: ['DATABASE_URL', 'JWT_SECRET'],
  properties: {
    PORT: { type: 'string', default: '3001' },
    DATABASE_URL: { type: 'string' },
    JWT_SECRET: { type: 'string' },
    META_APP_ID: { type: 'string' },
    META_APP_SECRET: { type: 'string' },
  }
};

async function buildServer() {
  await fastify.register(env, { schema, dotenv: true });

  await fastify.register(cors, {
    origin: ['http://localhost:3000'],
    credentials: true,
  });

  await fastify.register(jwt, {
    secret: fastify.config.JWT_SECRET,
  });

  // Decorador para Prisma
  fastify.decorate('prisma', prisma);

  // Health check
  fastify.get('/health', async () => {
    return { status: 'ok', timestamp: new Date().toISOString() };
  });

  // Importar rutas
  // await fastify.register(import('./routes/auth.js'));
  // await fastify.register(import('./routes/meta-accounts.js'));
  // await fastify.register(import('./routes/campaigns.js'));
  // await fastify.register(import('./routes/alerts.js'));

  return fastify;
}

const start = async () => {
  try {
    const server = await buildServer();
    const port = Number(server.config.PORT) || 3001;
    await server.listen({ port, host: '0.0.0.0' });
    console.log(`Server listening on port ${port}`);
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
};

start();
```

**Crear servicio de Meta API (src/services/meta-api.service.ts):**
```typescript
import axios from 'axios';

export class MetaAPIService {
  private baseURL = 'https://graph.facebook.com/v18.0';

  async getCampaigns(accessToken: string, adAccountId: string) {
    try {
      const response = await axios.get(
        `${this.baseURL}/act_${adAccountId}/campaigns`,
        {
          params: {
            access_token: accessToken,
            fields: 'id,name,status,objective,daily_budget,lifetime_budget,created_time,updated_time',
          },
        }
      );
      return response.data;
    } catch (error) {
      console.error('Error fetching campaigns:', error);
      throw error;
    }
  }

  async getCampaignInsights(accessToken: string, campaignId: string) {
    try {
      const response = await axios.get(
        `${this.baseURL}/${campaignId}/insights`,
        {
          params: {
            access_token: accessToken,
            fields: 'impressions,clicks,spend,ctr,cpc,cpp,cpm,reach',
            time_range: JSON.stringify({ since: '2024-01-01', until: '2024-12-31' }),
          },
        }
      );
      return response.data;
    } catch (error) {
      console.error('Error fetching insights:', error);
      throw error;
    }
  }

  async pauseCampaign(accessToken: string, campaignId: string) {
    try {
      const response = await axios.post(
        `${this.baseURL}/${campaignId}`,
        {
          status: 'PAUSED',
          access_token: accessToken,
        }
      );
      return response.data;
    } catch (error) {
      console.error('Error pausing campaign:', error);
      throw error;
    }
  }
}
```

### PASO 4: Setup Database (10 minutos)

```bash
# Volver a la raíz y crear package de database
cd ../..
mkdir -p packages/database/prisma
cd packages/database

# Crear package.json
cat > package.json << 'EOF'
{
  "name": "@vibe/database",
  "version": "1.0.0",
  "main": "./index.ts",
  "types": "./index.ts",
  "scripts": {
    "db:generate": "prisma generate",
    "db:push": "prisma db push",
    "db:migrate": "prisma migrate dev",
    "db:studio": "prisma studio"
  },
  "dependencies": {
    "@prisma/client": "^5.6.0"
  },
  "devDependencies": {
    "prisma": "^5.6.0"
  }
}
EOF

pnpm install

# Crear schema.prisma
cat > prisma/schema.prisma << 'EOF'
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  passwordHash  String
  firstName     String
  lastName      String
  avatarUrl     String?
  role          UserRole  @default(TRAFFICER)

  metaAccounts  MetaAccount[]
  subscriptions Subscription[]

  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  @@map("users")
}

enum UserRole {
  TRAFFICER
  STRATEGIST
  ADMIN
}

model MetaAccount {
  id                String   @id @default(uuid())
  userId            String
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  accessToken       String
  accountId         String
  accountName       String
  adAccountId       String

  isActive          Boolean  @default(true)
  alertsEnabled     Boolean  @default(true)

  campaigns         Campaign[]
  alerts            Alert[]

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  @@unique([userId, accountId])
  @@map("meta_accounts")
}

model Campaign {
  id                String      @id @default(uuid())
  metaAccountId     String
  metaAccount       MetaAccount @relation(fields: [metaAccountId], references: [id], onDelete: Cascade)

  campaignId        String      @unique
  name              String
  status            String
  objective         String?
  dailyBudget       Decimal?    @db.Decimal(10, 2)
  lifetimeBudget    Decimal?    @db.Decimal(10, 2)

  // Cached insights
  impressions       Int?
  clicks            Int?
  spend             Decimal?    @db.Decimal(10, 2)
  ctr               Decimal?    @db.Decimal(5, 2)

  lastSyncedAt      DateTime?

  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt

  @@map("campaigns")
}

model Alert {
  id                String      @id @default(uuid())
  metaAccountId     String
  metaAccount       MetaAccount @relation(fields: [metaAccountId], references: [id], onDelete: Cascade)

  type              AlertType
  severity          AlertSeverity
  title             String
  message           String
  metadata          Json?

  isRead            Boolean     @default(false)
  isResolved        Boolean     @default(false)
  actionTaken       String?
  resolvedAt        DateTime?

  createdAt         DateTime    @default(now())

  @@map("alerts")
}

enum AlertType {
  CAMPAIGN_REJECTED
  BUDGET_DEPLETED
  LOW_PERFORMANCE
  ACCOUNT_ISSUE
  PAYMENT_FAILED
}

enum AlertSeverity {
  INFO
  WARNING
  CRITICAL
}

model Subscription {
  id                String   @id @default(uuid())
  userId            String
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  plan              SubscriptionPlan
  status            SubscriptionStatus

  stripeCustomerId      String?
  stripeSubscriptionId  String?

  currentPeriodStart    DateTime
  currentPeriodEnd      DateTime
  cancelAtPeriodEnd     Boolean @default(false)

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  @@map("subscriptions")
}

enum SubscriptionPlan {
  FREE
  BASE
  PRO
}

enum SubscriptionStatus {
  ACTIVE
  CANCELLED
  PAST_DUE
  EXPIRED
}
EOF

# Generar cliente Prisma
pnpm db:generate
```

### PASO 5: Configurar Supabase (5 minutos)

```bash
# Ir a https://supabase.com/dashboard
# 1. Crear nuevo proyecto "vibe-mcp"
# 2. Copiar DATABASE_URL de Settings > Database
# 3. Actualizar .env en apps/api con la URL

# Ejecutar migraciones
cd packages/database
pnpm db:push
```

### PASO 6: Setup Redis y Jobs (10 minutos)

**Crear procesador de alertas (apps/api/src/jobs/alert-processor.ts):**
```typescript
import { Worker, Queue } from 'bullmq';
import { PrismaClient } from '@prisma/client';
import { MetaAPIService } from '../services/meta-api.service.js';

const prisma = new PrismaClient();
const metaAPI = new MetaAPIService();

const alertQueue = new Queue('alerts', {
  connection: {
    host: process.env.REDIS_HOST || 'localhost',
    port: Number(process.env.REDIS_PORT) || 6379,
  },
});

const worker = new Worker('alerts', async (job) => {
  console.log(`Processing alert job ${job.id}`);

  const metaAccounts = await prisma.metaAccount.findMany({
    where: { alertsEnabled: true },
  });

  for (const account of metaAccounts) {
    try {
      const campaigns = await metaAPI.getCampaigns(
        account.accessToken,
        account.adAccountId
      );

      // Verificar campañas rechazadas
      for (const campaign of campaigns.data) {
        if (campaign.status === 'DISAPPROVED') {
          await prisma.alert.create({
            data: {
              metaAccountId: account.id,
              type: 'CAMPAIGN_REJECTED',
              severity: 'CRITICAL',
              title: `Campaña rechazada: ${campaign.name}`,
              message: `La campaña ${campaign.name} ha sido rechazada por Meta.`,
              metadata: { campaignId: campaign.id },
            },
          });
        }

        // Verificar presupuesto bajo
        if (campaign.daily_budget && campaign.spend >= campaign.daily_budget * 0.9) {
          await prisma.alert.create({
            data: {
              metaAccountId: account.id,
              type: 'BUDGET_DEPLETED',
              severity: 'WARNING',
              title: `Presupuesto casi agotado: ${campaign.name}`,
              message: `La campaña ${campaign.name} ha gastado el 90% de su presupuesto diario.`,
              metadata: { campaignId: campaign.id },
            },
          });
        }
      }
    } catch (error) {
      console.error(`Error processing account ${account.id}:`, error);
    }
  }
}, {
  connection: {
    host: process.env.REDIS_HOST || 'localhost',
    port: Number(process.env.REDIS_PORT) || 6379,
  },
});

// Programar job cada 30 minutos
export async function scheduleAlertJobs() {
  await alertQueue.add('check-alerts', {}, {
    repeat: {
      pattern: '*/30 * * * *', // Cada 30 minutos
    },
  });
}
```

### PASO 7: Deploy (15 minutos)

**Vercel (Frontend):**
```bash
# Instalar Vercel CLI
pnpm add -g vercel

# Desde apps/web
cd apps/web
vercel login
vercel

# Configurar environment variables en Vercel Dashboard:
# NEXT_PUBLIC_API_URL=https://your-backend.railway.app
# NEXT_PUBLIC_META_APP_ID=your-meta-app-id
```

**Railway (Backend):**
```bash
# 1. Ir a https://railway.app
# 2. Crear nuevo proyecto desde GitHub
# 3. Agregar PostgreSQL service
# 4. Agregar Redis service
# 5. Configurar variables de entorno
# 6. Deploy automático

# Variables a configurar:
# DATABASE_URL (de PostgreSQL service)
# REDIS_URL (de Redis service)
# JWT_SECRET
# META_APP_ID
# META_APP_SECRET
# PORT=3001
```

---

# PROYECTO 2: COPY STRATEGY SYSTEM (FASE 2)

## Descripción
Sistema de replicación de estrategias de tráfico donde usuarios se suscriben a estrategias de expertos verificados.

## Stack Tecnológico
- Frontend: Next.js 14 + TypeScript + Tailwind
- Backend: Fastify + TypeScript + Prisma
- Database: PostgreSQL + TimescaleDB
- Automation: n8n (Cloud o Self-hosted)
- Payments: Stripe
- Deploy: Vercel + Railway

## Extensiones al Proyecto MCP

### PASO 1: Extender Base de Datos (5 minutos)

**Agregar a packages/database/prisma/schema.prisma:**
```prisma
model Strategy {
  id              String   @id @default(uuid())
  name            String
  description     String
  methodType      String

  strategistId    String
  strategist      User     @relation("StrategistStrategies", fields: [strategistId], references: [id])

  configSchema    Json
  n8nWorkflowId   String

  monthlyPrice    Decimal  @db.Decimal(10, 2)
  revenueShare    Decimal  @db.Decimal(5, 2)

  isActive        Boolean  @default(false)
  isVerified      Boolean  @default(false)

  totalSubscribers Int     @default(0)
  avgROAS          Decimal? @db.Decimal(10, 2)

  subscriptions   StrategySubscription[]

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@map("strategies")
}

model StrategySubscription {
  id              String   @id @default(uuid())

  userId          String
  user            User     @relation("UserStrategySubscriptions", fields: [userId], references: [id])

  strategyId      String
  strategy        Strategy @relation(fields: [strategyId], references: [id])

  config          Json
  status          StrategySubscriptionStatus

  stripeSubscriptionId String?

  startedAt       DateTime @default(now())
  endedAt         DateTime?

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@unique([userId, strategyId])
  @@map("strategy_subscriptions")
}

enum StrategySubscriptionStatus {
  ACTIVE
  PAUSED
  CANCELLED
  EXPIRED
}

// Actualizar modelo User para agregar relaciones
model User {
  // ... campos existentes ...

  strategies            Strategy[] @relation("StrategistStrategies")
  strategySubscriptions StrategySubscription[] @relation("UserStrategySubscriptions")
}
```

### PASO 2: Setup n8n Integration (20 minutos)

**Opción A: n8n Cloud (Recomendado para MVP)**
```bash
# 1. Crear cuenta en https://n8n.io
# 2. Crear workspace
# 3. Obtener API Key de Settings > API

# Agregar a apps/api/.env:
N8N_API_URL=https://your-workspace.app.n8n.cloud
N8N_API_KEY=your-api-key
```

**Opción B: n8n Self-hosted**
```bash
# Usando Docker
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD=yourpassword \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Acceder a http://localhost:5678
```

**Crear servicio de n8n (apps/api/src/services/n8n.service.ts):**
```typescript
import axios from 'axios';

export class N8NService {
  private apiUrl = process.env.N8N_API_URL;
  private apiKey = process.env.N8N_API_KEY;

  async createWorkflow(workflow: any) {
    const response = await axios.post(
      `${this.apiUrl}/api/v1/workflows`,
      workflow,
      {
        headers: {
          'X-N8N-API-KEY': this.apiKey,
        },
      }
    );
    return response.data;
  }

  async executeWorkflow(workflowId: string, data: any) {
    const response = await axios.post(
      `${this.apiUrl}/webhook/${workflowId}`,
      data
    );
    return response.data;
  }

  async deployStrategyWorkflow(strategyConfig: any) {
    const workflow = this.buildStrategyWorkflow(strategyConfig);
    return await this.createWorkflow(workflow);
  }

  private buildStrategyWorkflow(config: any) {
    // Template base de workflow para estrategias
    return {
      name: `Strategy: ${config.name}`,
      nodes: [
        {
          name: 'Webhook',
          type: 'n8n-nodes-base.webhook',
          position: [250, 300],
          webhookId: config.webhookId,
        },
        {
          name: 'Meta API',
          type: 'n8n-nodes-base.httpRequest',
          position: [450, 300],
          parameters: {
            url: 'https://graph.facebook.com/v18.0/act_{{$json.adAccountId}}/campaigns',
            method: 'POST',
            // ... más configuración
          },
        },
      ],
      connections: {},
    };
  }
}
```

### PASO 3: Integración con Stripe (15 minutos)

```bash
# Instalar Stripe SDK
cd apps/api
pnpm add stripe

# Agregar a .env:
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

**Crear servicio de Stripe (apps/api/src/services/stripe.service.ts):**
```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
});

export class StripeService {
  async createCustomer(email: string, userId: string) {
    return await stripe.customers.create({
      email,
      metadata: { userId },
    });
  }

  async createSubscription(
    customerId: string,
    priceId: string,
    strategyId: string
  ) {
    return await stripe.subscriptions.create({
      customer: customerId,
      items: [{ price: priceId }],
      metadata: { strategyId },
      payment_behavior: 'default_incomplete',
      expand: ['latest_invoice.payment_intent'],
    });
  }

  async cancelSubscription(subscriptionId: string) {
    return await stripe.subscriptions.cancel(subscriptionId);
  }

  async constructWebhookEvent(body: string, signature: string) {
    return stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  }
}
```

**Crear ruta de webhooks (apps/api/src/routes/webhooks.ts):**
```typescript
import { FastifyPluginAsync } from 'fastify';
import { StripeService } from '../services/stripe.service.js';

const stripeService = new StripeService();

const webhooks: FastifyPluginAsync = async (fastify) => {
  fastify.post('/stripe', async (request, reply) => {
    const signature = request.headers['stripe-signature'] as string;

    try {
      const event = await stripeService.constructWebhookEvent(
        request.body as string,
        signature
      );

      switch (event.type) {
        case 'customer.subscription.created':
          // Activar suscripción en DB
          break;
        case 'customer.subscription.deleted':
          // Cancelar suscripción en DB
          break;
        case 'invoice.payment_succeeded':
          // Confirmar pago
          break;
        case 'invoice.payment_failed':
          // Manejar fallo de pago
          break;
      }

      return { received: true };
    } catch (err) {
      return reply.code(400).send({ error: 'Webhook error' });
    }
  });
};

export default webhooks;
```

### PASO 4: Frontend de Estrategias (30 minutos)

**Crear página de catálogo (apps/web/app/(dashboard)/strategies/page.tsx):**
```typescript
import { StrategyCard } from '@/components/strategy-card';
import { apiClient } from '@/lib/api';

async function getStrategies() {
  const response = await apiClient.get('/strategies');
  return response.data;
}

export default async function StrategiesPage() {
  const strategies = await getStrategies();

  return (
    <div className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-8">Estrategias Disponibles</h1>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {strategies.map((strategy: any) => (
          <StrategyCard key={strategy.id} strategy={strategy} />
        ))}
      </div>
    </div>
  );
}
```

**Crear componente de tarjeta (apps/web/components/strategy-card.tsx):**
```typescript
'use client';

import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { TrendingUp } from 'lucide-react';

interface StrategyCardProps {
  strategy: {
    id: string;
    name: string;
    description: string;
    methodType: string;
    monthlyPrice: number;
    avgROAS: number | null;
    totalSubscribers: number;
  };
}

export function StrategyCard({ strategy }: StrategyCardProps) {
  return (
    <Card>
      <CardHeader>
        <div className="flex justify-between items-start">
          <CardTitle>{strategy.name}</CardTitle>
          <Badge>{strategy.methodType}</Badge>
        </div>
        <CardDescription>{strategy.description}</CardDescription>
      </CardHeader>

      <CardContent>
        <div className="space-y-2">
          {strategy.avgROAS && (
            <div className="flex items-center gap-2">
              <TrendingUp className="h-4 w-4" />
              <span className="text-sm">ROAS Promedio: {strategy.avgROAS}x</span>
            </div>
          )}
          <p className="text-sm text-muted-foreground">
            {strategy.totalSubscribers} suscriptores activos
          </p>
        </div>
      </CardContent>

      <CardFooter className="flex justify-between items-center">
        <span className="text-2xl font-bold">${strategy.monthlyPrice}/mes</span>
        <Button>Suscribirse</Button>
      </CardFooter>
    </Card>
  );
}
```

---

# PROYECTO 3: KANBAN BOARD (Referencia)

Este es el proyecto de referencia que ya está documentado en REQUIREMENTS.md.

## Setup Rápido

```bash
# Usar la misma estructura de monorepo
mkdir vibe-kanban
cd vibe-kanban

# Seguir los mismos pasos de MCP Dashboard
# pero con los schemas de Prisma de REQUIREMENTS.md
```

**Referencia:** Ver REQUIREMENTS.md para detalles completos.

---

# SCRIPTS DE AUTOMATIZACIÓN

## Script de Setup Completo

**Crear archivo setup.sh en la raíz:**
```bash
#!/bin/bash

echo "🚀 Iniciando setup de Vibe Platform..."

# Colores
GREEN='\033[0;32m'
BLUE='\033[0;34m'
NC='\033[0m'

# Función para imprimir con color
print_step() {
  echo -e "${BLUE}==>${NC} $1"
}

print_success() {
  echo -e "${GREEN}✓${NC} $1"
}

# 1. Setup Monorepo
print_step "Configurando monorepo..."
pnpm install
print_success "Monorepo configurado"

# 2. Setup Database
print_step "Configurando base de datos..."
cd packages/database
pnpm db:generate
pnpm db:push
cd ../..
print_success "Base de datos configurada"

# 3. Setup Frontend
print_step "Configurando frontend..."
cd apps/web
pnpm install
npx shadcn-ui@latest init -y
cd ../..
print_success "Frontend configurado"

# 4. Setup Backend
print_step "Configurando backend..."
cd apps/api
pnpm install
cd ../..
print_success "Backend configurado"

# 5. Verificar servicios externos
print_step "Verificando configuración..."
if [ -z "$DATABASE_URL" ]; then
  echo "⚠️  DATABASE_URL no configurada en .env"
fi
if [ -z "$JWT_SECRET" ]; then
  echo "⚠️  JWT_SECRET no configurada en .env"
fi

print_success "Setup completado!"
echo ""
echo "Próximos pasos:"
echo "1. Configurar variables de entorno (.env files)"
echo "2. Ejecutar: pnpm dev"
echo "3. Acceder a http://localhost:3000"
```

## Script de Desarrollo

**Crear archivo dev.sh:**
```bash
#!/bin/bash

# Ejecutar todos los servicios en paralelo
echo "🚀 Iniciando servicios de desarrollo..."

# Usar tmux o screen para múltiples terminales
tmux new-session -d -s vibe
tmux send-keys -t vibe 'cd apps/web && pnpm dev' C-m
tmux split-window -t vibe -h
tmux send-keys -t vibe 'cd apps/api && pnpm dev' C-m
tmux split-window -t vibe -v
tmux send-keys -t vibe 'cd packages/database && pnpm db:studio' C-m
tmux attach -t vibe
```

---

# CHECKLIST DE VERIFICACIÓN

## Pre-Deploy
- [ ] Variables de entorno configuradas
- [ ] Base de datos migrada
- [ ] Tests pasando (cuando estén implementados)
- [ ] Meta API credentials válidas
- [ ] Stripe webhooks configurados

## Post-Deploy
- [ ] Health check endpoint respondiendo
- [ ] Frontend carga correctamente
- [ ] Login/Register funcionando
- [ ] Meta OAuth funcionando
- [ ] Alertas generándose correctamente

## Security
- [ ] JWT secrets no expuestos
- [ ] CORS configurado correctamente
- [ ] Rate limiting implementado
- [ ] Inputs validados
- [ ] Passwords hasheados con bcrypt

---

# TROUBLESHOOTING

## Frontend no se conecta al Backend
```bash
# Verificar NEXT_PUBLIC_API_URL
cat apps/web/.env.local

# Verificar CORS en backend
# src/server.ts debe tener origen correcto
```

## Prisma errores
```bash
# Regenerar cliente
cd packages/database
pnpm db:generate

# Reset database (⚠️ borra datos)
pnpm db:push --force-reset
```

## Meta API errores
```bash
# Verificar token en Graph API Explorer
# https://developers.facebook.com/tools/explorer

# Verificar permisos de app
# ads_management, ads_read, business_management
```

---

**FIN DE GUÍA DE SETUP**

Última actualización: 2025-10-21
Próxima revisión: Después de primer deploy exitoso
