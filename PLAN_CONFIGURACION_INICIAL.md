# Plan de Configuración Inicial - Vibe Kanban

## Resumen Ejecutivo

Este documento define las **tareas paralelas e independientes** para configurar los 3 proyectos identificados. Cada proyecto tiene su propio setup que puede ejecutarse de forma concurrente para maximizar la velocidad de desarrollo.

---

## 📋 Índice de Proyectos

1. [Proyecto 1: MCP as a Service](#proyecto-1-mcp-as-a-service)
2. [Proyecto 2: Marketplace de Workflows](#proyecto-2-marketplace-de-workflows)
3. [Proyecto 3: Copy Strategy](#proyecto-3-copy-strategy)
4. [Infraestructura Compartida](#infraestructura-compartida)
5. [Timeline y Dependencias](#timeline-y-dependencias)

---

## ⚠️ IMPORTANTE: Preguntas Antes de Ejecutar

### Decisiones Técnicas Requeridas

#### 1. Stack de Frontend
- [ ] **¿Next.js o Create React App?**
  - **Recomendación:** Next.js 14+ (SSR, API Routes, mejor SEO)
  - **Alternativa:** Vite + React (más simple, más rápido en dev)

#### 2. Hosting
- [ ] **Frontend:**
  - **Recomendación:** Vercel (integración perfecta con Next.js)
  - **Alternativas:** Netlify, AWS Amplify, Railway

- [ ] **Backend:**
  - **Recomendación:** Railway (fácil, PostgreSQL incluido)
  - **Alternativas:** Render, Fly.io, AWS ECS

- [ ] **Base de Datos:**
  - **Recomendación:** Supabase (PostgreSQL + Auth + Storage)
  - **Alternativas:** Railway PostgreSQL, AWS RDS, Neon

#### 3. n8n Deployment
- [ ] **¿Cloud o Self-hosted?**
  - **Recomendación (FASE 0-1):** n8n Cloud ($20/mes, menos mantenimiento)
  - **Recomendación (FASE 2+):** Self-hosted (más control, webhooks privados)

#### 4. Autenticación
- [ ] **¿Qué método de auth?**
  - **Recomendación:** NextAuth.js con Meta OAuth
  - **Alternativas:** Supabase Auth, Clerk, Auth0

#### 5. Monorepo vs Multi-repo
- [ ] **¿Estructura de repositorios?**
  - **Recomendación:** Monorepo con Turborepo
    - `apps/web` (Frontend)
    - `apps/api` (Backend)
    - `packages/shared` (Tipos, utilidades)
  - **Alternativa:** Repositorios separados

#### 6. Presupuesto Inicial
¿Cuál es el presupuesto mensual para infraestructura?
- **Mínimo (desarrollo):** ~$50/mes
  - Vercel Hobby (gratis)
  - Railway Hobby ($5)
  - Supabase Free tier (gratis)
  - n8n Cloud Starter ($20/mes)
  - SendGrid Free (gratis)

- **Recomendado (producción):** ~$150-200/mes
  - Vercel Pro ($20)
  - Railway Pro ($20-50)
  - Supabase Pro ($25)
  - n8n Cloud Pro ($50)
  - Stripe fees (variable)
  - Sentry ($26)

---

## 🚀 PROYECTO 1: MCP as a Service

### Objetivo
Plataforma base para que traficers gestionen campañas de Meta Ads con dashboard centralizado y alertas.

### Tareas de Configuración (Pueden ejecutarse en paralelo)

#### TAREA P1-A: Setup de Frontend
**Responsable:** Dev Frontend
**Duración estimada:** 2-3 horas
**Dependencias:** Ninguna

```bash
# 1. Crear proyecto Next.js
npx create-next-app@latest vibe-kanban-web --typescript --tailwind --app --use-npm

cd vibe-kanban-web

# 2. Instalar dependencias core
npm install zustand @tanstack/react-query axios zod
npm install date-fns recharts lucide-react
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu
npm install @radix-ui/react-select @radix-ui/react-toast

# 3. Instalar shadcn/ui
npx shadcn-ui@latest init

# 4. Agregar componentes shadcn
npx shadcn-ui@latest add button card dialog dropdown-menu input label select toast

# 5. Configurar estructura de carpetas
mkdir -p src/components/dashboard
mkdir -p src/components/campaigns
mkdir -p src/components/alerts
mkdir -p src/lib/api
mkdir -p src/stores
mkdir -p src/types
mkdir -p src/hooks

# 6. Configurar variables de entorno
cat > .env.local << EOF
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_META_APP_ID=your_app_id
NEXTAUTH_SECRET=generate_random_secret
NEXTAUTH_URL=http://localhost:3000
EOF

# 7. Iniciar dev server
npm run dev
```

**Entregables:**
- [ ] Proyecto Next.js configurado
- [ ] shadcn/ui instalado y funcionando
- [ ] Estructura de carpetas creada
- [ ] Dev server corriendo en localhost:3000

---

#### TAREA P1-B: Setup de Backend (Meta Ads Service)
**Responsable:** Dev Backend
**Duración estimada:** 3-4 horas
**Dependencias:** Ninguna (hasta TAREA I-A para DB)

```bash
# 1. Crear proyecto
mkdir vibe-kanban-api && cd vibe-kanban-api
npm init -y

# 2. Instalar TypeScript y configuración
npm install -D typescript @types/node ts-node nodemon
npx tsc --init

# 3. Configurar tsconfig.json
cat > tsconfig.json << EOF
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
EOF

# 4. Instalar dependencias principales
npm install fastify @fastify/cors @fastify/jwt
npm install @prisma/client
npm install -D prisma
npm install zod
npm install facebook-nodejs-business-sdk
npm install bull ioredis
npm install dotenv

# 5. Inicializar Prisma
npx prisma init

# 6. Crear estructura de carpetas
mkdir -p src/controllers
mkdir -p src/services
mkdir -p src/models
mkdir -p src/middlewares
mkdir -p src/utils
mkdir -p src/types
mkdir -p src/clients

# 7. Crear archivo principal
cat > src/index.ts << 'EOF'
import Fastify from 'fastify';
import cors from '@fastify/cors';
import jwt from '@fastify/jwt';
import dotenv from 'dotenv';

dotenv.config();

const fastify = Fastify({
  logger: true,
});

// Plugins
fastify.register(cors, {
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
});

fastify.register(jwt, {
  secret: process.env.JWT_SECRET || 'supersecret',
});

// Health check
fastify.get('/health', async () => {
  return { status: 'ok', timestamp: new Date().toISOString() };
});

// Start server
const start = async () => {
  try {
    await fastify.listen({
      port: parseInt(process.env.PORT || '4000'),
      host: '0.0.0.0',
    });
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
};

start();
EOF

# 8. Configurar scripts en package.json
npm pkg set scripts.dev="nodemon --exec ts-node src/index.ts"
npm pkg set scripts.build="tsc"
npm pkg set scripts.start="node dist/index.js"

# 9. Variables de entorno
cat > .env << EOF
PORT=4000
FRONTEND_URL=http://localhost:3000
DATABASE_URL=postgresql://user:password@localhost:5432/vibekanban
JWT_SECRET=your_jwt_secret_here
META_APP_ID=your_meta_app_id
META_APP_SECRET=your_meta_app_secret
EOF

# 10. Iniciar servidor
npm run dev
```

**Entregables:**
- [ ] Proyecto TypeScript configurado
- [ ] Fastify server corriendo en localhost:4000
- [ ] Prisma inicializado
- [ ] Estructura de carpetas creada
- [ ] Endpoint /health funcionando

---

#### TAREA P1-C: Integración con Meta API
**Responsable:** Dev Backend
**Duración estimada:** 4-6 horas
**Dependencias:** TAREA P1-B

**Pasos:**

1. **Crear Meta App en Facebook Developers**
   - Ir a https://developers.facebook.com/apps
   - Crear nueva app → Tipo: Business
   - Agregar producto: Marketing API
   - Configurar OAuth redirect: `http://localhost:3000/api/auth/meta/callback`
   - Obtener App ID y App Secret

2. **Implementar OAuth Flow**

```typescript
// src/controllers/meta-auth.controller.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { prisma } from '../utils/prisma';
import { encrypt } from '../utils/encryption';

export class MetaAuthController {
  async connect(request: FastifyRequest, reply: FastifyReply) {
    const redirectUri = `https://www.facebook.com/v18.0/dialog/oauth?client_id=${process.env.META_APP_ID}&redirect_uri=${process.env.META_REDIRECT_URI}&scope=ads_management,ads_read,business_management`;

    return reply.redirect(redirectUri);
  }

  async callback(
    request: FastifyRequest<{ Querystring: { code: string } }>,
    reply: FastifyReply
  ) {
    const { code } = request.query;

    // Intercambiar code por access token
    const tokenResponse = await fetch(
      `https://graph.facebook.com/v18.0/oauth/access_token?client_id=${process.env.META_APP_ID}&client_secret=${process.env.META_APP_SECRET}&redirect_uri=${process.env.META_REDIRECT_URI}&code=${code}`
    );

    const { access_token } = await tokenResponse.json();

    // Obtener cuentas publicitarias
    const accountsResponse = await fetch(
      `https://graph.facebook.com/v18.0/me/adaccounts?access_token=${access_token}`
    );

    const { data: accounts } = await accountsResponse.json();

    // Guardar en DB (encriptado)
    const userId = request.user.userId; // Del JWT

    for (const account of accounts) {
      await prisma.metaAccount.upsert({
        where: {
          userId_metaAccountId: {
            userId,
            metaAccountId: account.id,
          },
        },
        create: {
          userId,
          metaAccountId: account.id,
          accessToken: encrypt(access_token),
        },
        update: {
          accessToken: encrypt(access_token),
        },
      });
    }

    return reply.redirect(`${process.env.FRONTEND_URL}/dashboard?connected=true`);
  }

  async disconnect(
    request: FastifyRequest<{ Params: { accountId: string } }>,
    reply: FastifyReply
  ) {
    const { accountId } = request.params;
    const userId = request.user.userId;

    await prisma.metaAccount.delete({
      where: {
        userId_metaAccountId: {
          userId,
          metaAccountId: accountId,
        },
      },
    });

    return { success: true };
  }
}
```

3. **Crear cliente de Meta API**

```typescript
// src/clients/meta-api.client.ts
import { FacebookAdsApi, AdAccount, Campaign } from 'facebook-nodejs-business-sdk';

export class MetaAPIClient {
  private api: FacebookAdsApi;

  constructor(accessToken: string) {
    FacebookAdsApi.init(accessToken);
    this.api = FacebookAdsApi.getInstance();
  }

  async getCampaigns(accountId: string) {
    const account = new AdAccount(`act_${accountId}`);
    const campaigns = await account.getCampaigns(
      ['id', 'name', 'status', 'objective', 'daily_budget'],
      { limit: 100 }
    );
    return campaigns;
  }

  async getCampaignInsights(campaignId: string) {
    const campaign = new Campaign(campaignId);
    const insights = await campaign.getInsights([
      'impressions',
      'clicks',
      'spend',
      'ctr',
      'cpc',
      'cpm',
    ]);
    return insights[0];
  }

  // Más métodos...
}
```

**Entregables:**
- [ ] Meta App creada y configurada
- [ ] OAuth flow implementado
- [ ] Cliente de Meta API funcional
- [ ] Tokens almacenados de forma segura (encriptados)

---

#### TAREA P1-D: Sistema de Alertas
**Responsable:** Dev Backend
**Duración estimada:** 4-5 horas
**Dependencias:** TAREA P1-B, TAREA I-A (Redis)

```typescript
// src/services/alerts-engine.service.ts
import { Queue, Worker } from 'bullmq';
import { prisma } from '../utils/prisma';
import { MetaAPIClient } from '../clients/meta-api.client';
import { sendEmail } from '../utils/email';

const alertsQueue = new Queue('alerts-check', {
  connection: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

export class AlertsEngineService {
  // Programar chequeo de alertas cada 15 minutos
  async scheduleAlertChecks() {
    await alertsQueue.add(
      'check-all-alerts',
      {},
      {
        repeat: {
          pattern: '*/15 * * * *', // Cada 15 min
        },
      }
    );
  }

  // Worker que procesa las alertas
  startWorker() {
    const worker = new Worker(
      'alerts-check',
      async (job) => {
        const alerts = await prisma.alert.findMany({
          where: { isActive: true },
          include: { user: { include: { metaAccounts: true } } },
        });

        for (const alert of alerts) {
          await this.checkAlert(alert);
        }
      },
      {
        connection: {
          host: process.env.REDIS_HOST,
          port: parseInt(process.env.REDIS_PORT || '6379'),
        },
      }
    );

    return worker;
  }

  private async checkAlert(alert: any) {
    const { user, campaignId, type, condition } = alert;

    // Obtener access token del usuario
    const metaAccount = user.metaAccounts[0];
    if (!metaAccount) return;

    const client = new MetaAPIClient(decrypt(metaAccount.accessToken));

    // Obtener insights de la campaña
    const insights = await client.getCampaignInsights(campaignId);

    // Evaluar condición
    const shouldFire = this.evaluateCondition(insights, condition);

    if (shouldFire) {
      // Enviar alerta
      await this.fireAlert(alert, insights);

      // Actualizar lastFired
      await prisma.alert.update({
        where: { id: alert.id },
        data: { lastFired: new Date() },
      });
    }
  }

  private evaluateCondition(insights: any, condition: any): boolean {
    const { metric, operator, value } = condition;
    const actualValue = insights[metric];

    switch (operator) {
      case '>':
        return actualValue > value;
      case '<':
        return actualValue < value;
      case '>=':
        return actualValue >= value;
      case '<=':
        return actualValue <= value;
      case '==':
        return actualValue == value;
      default:
        return false;
    }
  }

  private async fireAlert(alert: any, insights: any) {
    // Enviar email
    await sendEmail({
      to: alert.user.email,
      subject: `Alerta: ${alert.type}`,
      html: `
        <h2>Alerta de campaña</h2>
        <p>Tu campaña ${alert.campaignId} ha cumplido la condición de alerta.</p>
        <p>Métrica: ${alert.condition.metric} ${alert.condition.operator} ${alert.condition.value}</p>
        <p>Valor actual: ${insights[alert.condition.metric]}</p>
      `,
    });

    // TODO: Enviar push notification, webhook, etc.
  }
}
```

**Entregables:**
- [ ] Queue de alertas configurada con Bull
- [ ] Worker procesando alertas cada 15 min
- [ ] Sistema de evaluación de condiciones
- [ ] Envío de emails funcionando

---

#### TAREA P1-E: Dashboard Frontend
**Responsable:** Dev Frontend
**Duración estimada:** 8-12 horas
**Dependencias:** TAREA P1-A, TAREA P1-C

```typescript
// src/app/dashboard/page.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { Card } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { CampaignsList } from '@/components/dashboard/campaigns-list';
import { MetricsOverview } from '@/components/dashboard/metrics-overview';
import { AlertsPanel } from '@/components/dashboard/alerts-panel';
import { api } from '@/lib/api';

export default function DashboardPage() {
  const { data: campaigns, isLoading } = useQuery({
    queryKey: ['campaigns'],
    queryFn: api.getCampaigns,
  });

  const { data: metrics } = useQuery({
    queryKey: ['metrics-summary'],
    queryFn: api.getMetricsSummary,
  });

  const { data: alerts } = useQuery({
    queryKey: ['alerts'],
    queryFn: api.getAlerts,
  });

  if (isLoading) {
    return <div>Cargando...</div>;
  }

  return (
    <div className="p-6 space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Dashboard</h1>
        <Button onClick={() => window.location.href = '/api/meta/auth/connect'}>
          Conectar cuenta de Meta
        </Button>
      </div>

      <MetricsOverview metrics={metrics} />

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2">
          <CampaignsList campaigns={campaigns} />
        </div>

        <div>
          <AlertsPanel alerts={alerts} />
        </div>
      </div>
    </div>
  );
}
```

```typescript
// src/components/dashboard/metrics-overview.tsx
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { TrendingUp, DollarSign, Eye, MousePointer } from 'lucide-react';

interface MetricsOverviewProps {
  metrics: {
    totalSpend: number;
    totalImpressions: number;
    totalClicks: number;
    avgCTR: number;
  };
}

export function MetricsOverview({ metrics }: MetricsOverviewProps) {
  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
      <Card>
        <CardHeader className="flex flex-row items-center justify-between pb-2">
          <CardTitle className="text-sm font-medium">Inversión Total</CardTitle>
          <DollarSign className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold">
            ${metrics?.totalSpend?.toLocaleString() || 0}
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardHeader className="flex flex-row items-center justify-between pb-2">
          <CardTitle className="text-sm font-medium">Impresiones</CardTitle>
          <Eye className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold">
            {metrics?.totalImpressions?.toLocaleString() || 0}
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardHeader className="flex flex-row items-center justify-between pb-2">
          <CardTitle className="text-sm font-medium">Clics</CardTitle>
          <MousePointer className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold">
            {metrics?.totalClicks?.toLocaleString() || 0}
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardHeader className="flex flex-row items-center justify-between pb-2">
          <CardTitle className="text-sm font-medium">CTR Promedio</CardTitle>
          <TrendingUp className="h-4 w-4 text-muted-foreground" />
        </CardHeader>
        <CardContent>
          <div className="text-2xl font-bold">
            {metrics?.avgCTR?.toFixed(2) || 0}%
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

**Entregables:**
- [ ] Dashboard page implementada
- [ ] Componentes de métricas
- [ ] Lista de campañas
- [ ] Panel de alertas
- [ ] Integración con API

---

## 🛍️ PROYECTO 2: Marketplace de Workflows

### Tareas de Configuración

#### TAREA P2-A: Setup de Workflow Service
**Responsable:** Dev Backend
**Duración estimada:** 3-4 horas
**Dependencias:** TAREA P1-B (estructura base compartida)

```bash
# 1. Crear servicio dentro del monorepo (o repo separado)
mkdir -p services/workflow && cd services/workflow

# 2. Copiar configuración base de meta-ads service
# (TypeScript, Fastify, Prisma, etc.)

# 3. Agregar dependencias específicas
npm install stripe
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
npm install sharp # Para procesamiento de imágenes

# 4. Crear estructura específica
mkdir -p src/controllers/marketplace
mkdir -p src/controllers/workflows
mkdir -p src/services/payment
mkdir -p src/services/storage

# 5. Configurar Stripe
cat >> .env << EOF
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_CONNECT_CLIENT_ID=ca_...
EOF

# 6. Configurar AWS S3 (para videos/thumbnails)
cat >> .env << EOF
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_S3_BUCKET=vibe-kanban-workflows
AWS_REGION=us-east-1
EOF
```

**Schema Prisma adicional:**

```prisma
// Agregar al schema.prisma existente

model Workflow {
  id              String    @id @default(cuid())
  creatorId       String
  creator         User      @relation(fields: [creatorId], references: [id])
  title           String
  description     String    @db.Text
  n8nJson         Json
  category        WorkflowCategory
  price           Decimal   @db.Decimal(10, 2)
  videoUrl        String?
  thumbnailUrl    String?
  downloadsCount  Int       @default(0)
  ratingAvg       Decimal?  @db.Decimal(3, 2)
  ratingCount     Int       @default(0)
  isPublished     Boolean   @default(false)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  purchases       WorkflowPurchase[]
  reviews         WorkflowReview[]

  @@index([category, isPublished])
  @@index([creatorId])
  @@index([ratingAvg])
}

enum WorkflowCategory {
  META_ADS
  ANALYTICS
  AUTOMATION
  INTEGRATION
  OTHER
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
}
```

**Entregables:**
- [ ] Workflow service configurado
- [ ] Stripe integrado
- [ ] S3 configurado para uploads
- [ ] Schema Prisma actualizado

---

#### TAREA P2-B: Sistema de Publicación
**Responsable:** Dev Backend
**Duración estimada:** 4-6 horas
**Dependencias:** TAREA P2-A

```typescript
// src/controllers/workflows.controller.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { prisma } from '../utils/prisma';
import { uploadToS3 } from '../services/storage.service';
import { z } from 'zod';

const createWorkflowSchema = z.object({
  title: z.string().min(5).max(100),
  description: z.string().min(20).max(1000),
  n8nJson: z.object({}).passthrough(), // Validar estructura básica
  category: z.enum(['META_ADS', 'ANALYTICS', 'AUTOMATION', 'INTEGRATION', 'OTHER']),
  price: z.number().min(5).max(500),
  videoUrl: z.string().url().optional(),
});

export class WorkflowsController {
  async create(
    request: FastifyRequest<{ Body: z.infer<typeof createWorkflowSchema> }>,
    reply: FastifyReply
  ) {
    const userId = request.user.userId;
    const data = createWorkflowSchema.parse(request.body);

    // Validar JSON de n8n (estructura básica)
    if (!data.n8nJson.nodes || !data.n8nJson.connections) {
      return reply.code(400).send({
        error: 'Invalid n8n workflow JSON',
      });
    }

    const workflow = await prisma.workflow.create({
      data: {
        ...data,
        creatorId: userId,
        isPublished: false, // Requiere revisión/aprobación
      },
    });

    return workflow;
  }

  async uploadVideo(request: FastifyRequest, reply: FastifyReply) {
    const data = await request.file();

    if (!data) {
      return reply.code(400).send({ error: 'No file uploaded' });
    }

    // Validar que sea video
    if (!data.mimetype.startsWith('video/')) {
      return reply.code(400).send({ error: 'File must be a video' });
    }

    // Subir a S3
    const videoUrl = await uploadToS3(data, 'workflow-videos');

    return { videoUrl };
  }

  async publish(
    request: FastifyRequest<{ Params: { id: string } }>,
    reply: FastifyReply
  ) {
    const { id } = request.params;
    const userId = request.user.userId;

    // Verificar que el workflow es del usuario
    const workflow = await prisma.workflow.findUnique({
      where: { id },
    });

    if (!workflow || workflow.creatorId !== userId) {
      return reply.code(404).send({ error: 'Workflow not found' });
    }

    // Validar que tenga video
    if (!workflow.videoUrl) {
      return reply.code(400).send({
        error: 'Workflow must have a tutorial video before publishing',
      });
    }

    // Publicar
    const updated = await prisma.workflow.update({
      where: { id },
      data: { isPublished: true },
    });

    return updated;
  }
}
```

**Entregables:**
- [ ] Endpoint de creación de workflows
- [ ] Validación de JSON de n8n
- [ ] Upload de videos a S3
- [ ] Sistema de publicación

---

#### TAREA P2-C: Marketplace Frontend
**Responsable:** Dev Frontend
**Duración estimada:** 8-12 horas
**Dependencias:** TAREA P2-A, TAREA P1-A

```typescript
// src/app/marketplace/page.tsx
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { WorkflowCard } from '@/components/marketplace/workflow-card';
import { WorkflowFilters } from '@/components/marketplace/workflow-filters';
import { api } from '@/lib/api';

export default function MarketplacePage() {
  const [filters, setFilters] = useState({
    category: 'all',
    priceRange: [0, 500],
    sortBy: 'popular',
  });

  const { data: workflows, isLoading } = useQuery({
    queryKey: ['marketplace-workflows', filters],
    queryFn: () => api.getMarketplaceWorkflows(filters),
  });

  return (
    <div className="container mx-auto p-6">
      <div className="flex justify-between items-center mb-8">
        <h1 className="text-4xl font-bold">Marketplace de Workflows</h1>
        <Button href="/workflows/new">
          Publicar Workflow
        </Button>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-4 gap-6">
        <aside className="lg:col-span-1">
          <WorkflowFilters filters={filters} onChange={setFilters} />
        </aside>

        <main className="lg:col-span-3">
          {isLoading ? (
            <div>Cargando...</div>
          ) : (
            <div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6">
              {workflows?.map((workflow) => (
                <WorkflowCard key={workflow.id} workflow={workflow} />
              ))}
            </div>
          )}
        </main>
      </div>
    </div>
  );
}
```

```typescript
// src/components/marketplace/workflow-card.tsx
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Star, Download } from 'lucide-react';

interface WorkflowCardProps {
  workflow: {
    id: string;
    title: string;
    description: string;
    price: number;
    category: string;
    ratingAvg: number;
    downloadsCount: number;
    thumbnailUrl?: string;
    creator: {
      name: string;
    };
  };
}

export function WorkflowCard({ workflow }: WorkflowCardProps) {
  return (
    <Card className="hover:shadow-lg transition-shadow">
      {workflow.thumbnailUrl && (
        <img
          src={workflow.thumbnailUrl}
          alt={workflow.title}
          className="w-full h-48 object-cover rounded-t-lg"
        />
      )}

      <CardHeader>
        <div className="flex justify-between items-start">
          <CardTitle className="text-lg">{workflow.title}</CardTitle>
          <Badge>{workflow.category}</Badge>
        </div>
        <CardDescription className="line-clamp-2">
          {workflow.description}
        </CardDescription>
      </CardHeader>

      <CardContent>
        <div className="flex items-center gap-4 text-sm text-muted-foreground">
          <div className="flex items-center gap-1">
            <Star className="h-4 w-4 fill-yellow-400 text-yellow-400" />
            <span>{workflow.ratingAvg?.toFixed(1) || 'N/A'}</span>
          </div>
          <div className="flex items-center gap-1">
            <Download className="h-4 w-4" />
            <span>{workflow.downloadsCount}</span>
          </div>
        </div>
        <p className="text-sm text-muted-foreground mt-2">
          por {workflow.creator.name}
        </p>
      </CardContent>

      <CardFooter className="flex justify-between items-center">
        <span className="text-2xl font-bold">${workflow.price}</span>
        <Button onClick={() => handlePurchase(workflow.id)}>
          Comprar
        </Button>
      </CardFooter>
    </Card>
  );
}
```

**Entregables:**
- [ ] Página de marketplace
- [ ] Componente de workflow card
- [ ] Sistema de filtros
- [ ] Integración con API

---

#### TAREA P2-D: Integración de Pagos (Stripe)
**Responsable:** Dev Backend
**Duración estimada:** 6-8 horas
**Dependencias:** TAREA P2-A

```typescript
// src/services/payment.service.ts
import Stripe from 'stripe';
import { prisma } from '../utils/prisma';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
});

export class PaymentService {
  async createCheckoutSession(workflowId: string, buyerId: string) {
    const workflow = await prisma.workflow.findUnique({
      where: { id: workflowId },
      include: { creator: true },
    });

    if (!workflow) {
      throw new Error('Workflow not found');
    }

    // Verificar que el usuario no lo haya comprado ya
    const existingPurchase = await prisma.workflowPurchase.findUnique({
      where: {
        workflowId_buyerId: {
          workflowId,
          buyerId,
        },
      },
    });

    if (existingPurchase) {
      throw new Error('You already own this workflow');
    }

    const session = await stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      line_items: [
        {
          price_data: {
            currency: 'usd',
            product_data: {
              name: workflow.title,
              description: workflow.description,
              images: workflow.thumbnailUrl ? [workflow.thumbnailUrl] : [],
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

  async handleWebhook(rawBody: string, signature: string) {
    let event: Stripe.Event;

    try {
      event = stripe.webhooks.constructEvent(
        rawBody,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch (err) {
      throw new Error('Invalid signature');
    }

    switch (event.type) {
      case 'checkout.session.completed':
        await this.fulfillPurchase(event.data.object as Stripe.Checkout.Session);
        break;
      case 'charge.refunded':
        await this.handleRefund(event.data.object as Stripe.Charge);
        break;
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

    // Incrementar downloads
    await prisma.workflow.update({
      where: { id: workflowId },
      data: { downloadsCount: { increment: 1 } },
    });

    // Crear transferencia al creador (70%)
    const platformFee = Math.round(session.amount_total! * 0.3);
    const creatorAmount = session.amount_total! - platformFee;

    // TODO: Implementar Stripe Connect para transferencias automáticas
    // Por ahora, simplemente registramos el ingreso del creador
  }

  private async handleRefund(charge: Stripe.Charge) {
    // Eliminar compra y decrementar downloads
    const purchase = await prisma.workflowPurchase.findUnique({
      where: { stripeId: charge.id },
    });

    if (purchase) {
      await prisma.workflowPurchase.delete({
        where: { id: purchase.id },
      });

      await prisma.workflow.update({
        where: { id: purchase.workflowId },
        data: { downloadsCount: { decrement: 1 } },
      });
    }
  }
}
```

**Endpoints:**

```typescript
// src/controllers/payment.controller.ts
export class PaymentController {
  async createCheckoutSession(
    request: FastifyRequest<{ Body: { workflowId: string } }>,
    reply: FastifyReply
  ) {
    const { workflowId } = request.body;
    const buyerId = request.user.userId;

    const paymentService = new PaymentService();
    const session = await paymentService.createCheckoutSession(workflowId, buyerId);

    return { sessionId: session.id, url: session.url };
  }

  async webhook(request: FastifyRequest, reply: FastifyReply) {
    const signature = request.headers['stripe-signature'] as string;
    const rawBody = request.rawBody; // Requiere @fastify/raw-body plugin

    const paymentService = new PaymentService();
    await paymentService.handleWebhook(rawBody, signature);

    return { received: true };
  }
}
```

**Entregables:**
- [ ] Stripe checkout configurado
- [ ] Webhook handler implementado
- [ ] Registro de compras
- [ ] Sistema de revenue split (30/70)

---

## 🎯 PROYECTO 3: Copy Strategy

### Tareas de Configuración

#### TAREA P3-A: Setup de Strategy Service
**Responsable:** Dev Backend
**Duración estimada:** 3-4 horas
**Dependencias:** TAREA P1-B

Similar al setup de Workflow Service, crear servicio independiente con schema específico para estrategias.

```prisma
model Strategy {
  id                  String    @id @default(cuid())
  name                String
  slug                String    @unique
  description         String    @db.Text
  estrategaId         String
  estratega           User      @relation(fields: [estrategaId], references: [id])
  baseN8nTemplate     Json
  configurableParams  Json      // Schema: [{name, type, default, min, max}]
  monthlyPrice        Decimal   @db.Decimal(10, 2)
  setupFee            Decimal?  @db.Decimal(10, 2)
  isActive            Boolean   @default(true)
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt

  subscriptions       StrategySubscription[]

  @@index([estrategaId, isActive])
  @@index([slug])
}

model StrategySubscription {
  id              String    @id @default(cuid())
  strategyId      String
  strategy        Strategy  @relation(fields: [strategyId], references: [id])
  userId          String
  user            User      @relation(fields: [userId], references: [id])
  metaAccountId   String
  params          Json      // Valores específicos
  mcpToken        String    @unique
  n8nWorkflowId   String?
  status          SubscriptionStatus @default(PENDING)
  startedAt       DateTime  @default(now())
  nextBillingDate DateTime
  cancelledAt     DateTime?

  metrics         StrategyMetric[]

  @@index([userId, status])
  @@index([strategyId, status])
}

enum SubscriptionStatus {
  PENDING
  ACTIVE
  PAUSED
  CANCELLED
  TRIAL
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

  @@index([subscriptionId, timestamp(sort: Desc)])
}

model EstrategaProfile {
  id                    String    @id @default(cuid())
  userId                String    @unique
  user                  User      @relation(fields: [userId], references: [id])
  bio                   String?   @db.Text
  portfolioUrl          String?
  isVerified            Boolean   @default(false)
  verificationDate      DateTime?
  revenueSharePercent   Decimal   @db.Decimal(5, 2) @default(40.00)
  totalEarnings         Decimal   @db.Decimal(10, 2) @default(0)
  activeSubscribers     Int       @default(0)
  lifetimeSubscribers   Int       @default(0)

  @@index([isVerified])
}
```

**Entregables:**
- [ ] Strategy service configurado
- [ ] Schema Prisma para estrategias
- [ ] Estructura de carpetas
- [ ] Endpoints base

---

#### TAREA P3-B: Integración con n8n (API Client)
**Responsable:** Dev Backend
**Duración estimada:** 4-6 horas
**Dependencias:** TAREA P3-A

```typescript
// src/clients/n8n.client.ts
import axios, { AxiosInstance } from 'axios';

interface N8nWorkflow {
  id?: string;
  name: string;
  nodes: any[];
  connections: any;
  settings?: {
    executionOrder?: 'v0' | 'v1';
    saveDataErrorExecution?: 'all' | 'none';
    saveDataSuccessExecution?: 'all' | 'none';
    saveManualExecutions?: boolean;
    timezone?: string;
  };
  active: boolean;
}

export class N8nClient {
  private client: AxiosInstance;

  constructor() {
    this.client = axios.create({
      baseURL: process.env.N8N_API_URL,
      headers: {
        'X-N8N-API-KEY': process.env.N8N_API_KEY,
        'Content-Type': 'application/json',
      },
      timeout: 30000,
    });
  }

  async createWorkflow(workflow: N8nWorkflow) {
    try {
      const response = await this.client.post('/workflows', workflow);
      return response.data;
    } catch (error) {
      console.error('Error creating n8n workflow:', error);
      throw new Error('Failed to create workflow in n8n');
    }
  }

  async updateWorkflow(workflowId: string, updates: Partial<N8nWorkflow>) {
    const response = await this.client.patch(`/workflows/${workflowId}`, updates);
    return response.data;
  }

  async deleteWorkflow(workflowId: string) {
    await this.client.delete(`/workflows/${workflowId}`);
  }

  async activateWorkflow(workflowId: string) {
    return this.updateWorkflow(workflowId, { active: true });
  }

  async deactivateWorkflow(workflowId: string) {
    return this.updateWorkflow(workflowId, { active: false });
  }

  async getWorkflow(workflowId: string) {
    const response = await this.client.get(`/workflows/${workflowId}`);
    return response.data;
  }

  async getWorkflowExecutions(workflowId: string, limit = 10) {
    const response = await this.client.get('/executions', {
      params: {
        workflowId,
        limit,
      },
    });
    return response.data;
  }

  async testWorkflow(workflowId: string) {
    const response = await this.client.post(`/workflows/${workflowId}/test`);
    return response.data;
  }
}
```

**Configuración de n8n:**

1. **Si usas n8n Cloud:**
   - Ir a Settings → API
   - Generar API Key
   - Agregar a .env: `N8N_API_KEY=n8n_api_xxx`
   - `N8N_API_URL=https://yourinstance.app.n8n.cloud/api/v1`

2. **Si usas self-hosted:**
   ```bash
   docker run -d \
     --name n8n \
     -p 5678:5678 \
     -e N8N_BASIC_AUTH_ACTIVE=true \
     -e N8N_BASIC_AUTH_USER=admin \
     -e N8N_BASIC_AUTH_PASSWORD=password \
     -e N8N_HOST=n8n.yourdomain.com \
     -v ~/.n8n:/home/node/.n8n \
     n8nio/n8n
   ```

**Entregables:**
- [ ] Cliente de n8n implementado
- [ ] n8n cloud/self-hosted configurado
- [ ] API key generada
- [ ] Tests de conexión

---

#### TAREA P3-C: Sistema de Replicación de Estrategias
**Responsable:** Dev Backend
**Duración estimada:** 8-12 horas
**Dependencias:** TAREA P3-A, TAREA P3-B

Este es el corazón del Copy Strategy. Ver implementación completa en ARQUITECTURA_TECNICA.md sección 3.3.

**Pasos principales:**
1. Usuario se suscribe a estrategia
2. Sistema clona template de n8n
3. Inyecta parámetros del usuario
4. Genera token MCP único
5. Configura credenciales en n8n
6. Activa workflow
7. Registra suscripción

**Entregables:**
- [ ] Servicio de replicación implementado
- [ ] Sistema de clonación de templates
- [ ] Inyección de parámetros
- [ ] Generación de MCP tokens
- [ ] Activación automática

---

#### TAREA P3-D: MCP Personalizado
**Responsable:** Dev Backend
**Duración estimada:** 6-8 horas
**Dependencias:** TAREA P1-C (Meta API Client)

```typescript
// src/controllers/mcp.controller.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { prisma } from '../utils/prisma';
import { decrypt } from '../utils/encryption';
import { MetaAPIClient } from '../clients/meta-api.client';

export class MCPController {
  async handleRequest(
    request: FastifyRequest<{
      Headers: { 'x-mcp-token': string };
      Body: {
        action: string;
        params: any;
      };
    }>,
    reply: FastifyReply
  ) {
    // 1. Validar token MCP
    const mcpToken = request.headers['x-mcp-token'];

    if (!mcpToken) {
      return reply.code(401).send({ error: 'MCP token required' });
    }

    // 2. Buscar suscripción asociada
    const subscription = await prisma.strategySubscription.findUnique({
      where: { mcpToken },
      include: {
        user: {
          include: {
            metaAccounts: true,
          },
        },
      },
    });

    if (!subscription) {
      return reply.code(401).send({ error: 'Invalid MCP token' });
    }

    // 3. Obtener credenciales de Meta
    const metaAccount = subscription.user.metaAccounts.find(
      (acc) => acc.metaAccountId === subscription.metaAccountId
    );

    if (!metaAccount) {
      return reply.code(404).send({ error: 'Meta account not found' });
    }

    const accessToken = decrypt(metaAccount.accessToken);
    const metaClient = new MetaAPIClient(accessToken);

    // 4. Ejecutar acción solicitada
    const { action, params } = request.body;

    try {
      let result;

      switch (action) {
        case 'getCampaigns':
          result = await metaClient.getCampaigns(subscription.metaAccountId);
          break;

        case 'getCampaignMetrics':
          result = await metaClient.getCampaignInsights(params.campaignId);
          break;

        case 'createCampaign':
          result = await metaClient.createCampaign(subscription.metaAccountId, params);
          break;

        case 'updateCampaign':
          result = await metaClient.updateCampaign(params.campaignId, params.updates);
          break;

        case 'pauseCampaign':
          result = await metaClient.pauseCampaign(params.campaignId);
          break;

        case 'resumeCampaign':
          result = await metaClient.resumeCampaign(params.campaignId);
          break;

        default:
          return reply.code(400).send({ error: 'Unknown action' });
      }

      // 5. Guardar métricas si es consulta
      if (action === 'getCampaignMetrics') {
        await this.saveMetrics(subscription.id, result);
      }

      return result;
    } catch (error) {
      console.error('MCP Error:', error);
      return reply.code(500).send({ error: 'MCP request failed' });
    }
  }

  private async saveMetrics(subscriptionId: string, insights: any) {
    await prisma.strategyMetric.create({
      data: {
        subscriptionId,
        timestamp: new Date(),
        investment: insights.spend || 0,
        impressions: insights.impressions || 0,
        clicks: insights.clicks || 0,
        conversions: insights.conversions || 0,
        revenue: insights.revenue,
        roi: insights.roi,
        rawData: insights,
      },
    });
  }
}
```

**Endpoints:**

```typescript
// POST /api/mcp
// Headers: { 'X-MCP-Token': 'token_xxx' }
// Body: { action: 'getCampaigns', params: {} }
```

**Entregables:**
- [ ] Endpoint MCP implementado
- [ ] Validación de tokens
- [ ] Proxy a Meta API
- [ ] Almacenamiento de métricas

---

#### TAREA P3-E: Frontend de Estrategias
**Responsable:** Dev Frontend
**Duración estimada:** 10-14 horas
**Dependencias:** TAREA P3-A, TAREA P1-A

```typescript
// src/app/strategies/page.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { StrategyCard } from '@/components/strategies/strategy-card';
import { EstrategaProfile } from '@/components/strategies/estratega-profile';
import { api } from '@/lib/api';

export default function StrategiesPage() {
  const { data: strategies, isLoading } = useQuery({
    queryKey: ['strategies'],
    queryFn: api.getStrategies,
  });

  return (
    <div className="container mx-auto p-6">
      <h1 className="text-4xl font-bold mb-8">Estrategias Verificadas</h1>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {strategies?.map((strategy) => (
          <StrategyCard key={strategy.id} strategy={strategy} />
        ))}
      </div>
    </div>
  );
}
```

```typescript
// src/app/strategies/[slug]/page.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { useParams } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { Card } from '@/components/ui/card';
import { EstrategaProfile } from '@/components/strategies/estratega-profile';
import { PerformanceChart } from '@/components/strategies/performance-chart';
import { SubscribeDialog } from '@/components/strategies/subscribe-dialog';

export default function StrategyDetailPage() {
  const params = useParams();
  const slug = params.slug as string;

  const { data: strategy } = useQuery({
    queryKey: ['strategy', slug],
    queryFn: () => api.getStrategyBySlug(slug),
  });

  if (!strategy) return <div>Cargando...</div>;

  return (
    <div className="container mx-auto p-6">
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
        <div className="lg:col-span-2">
          <h1 className="text-4xl font-bold mb-4">{strategy.name}</h1>
          <p className="text-lg text-muted-foreground mb-6">
            {strategy.description}
          </p>

          <Card className="p-6 mb-6">
            <h2 className="text-2xl font-bold mb-4">Resultados Promedio</h2>
            <PerformanceChart strategyId={strategy.id} />
          </Card>

          <Card className="p-6">
            <h2 className="text-2xl font-bold mb-4">Parámetros Configurables</h2>
            <ul className="space-y-2">
              {strategy.configurableParams.map((param: any) => (
                <li key={param.name}>
                  <strong>{param.label}:</strong> {param.description}
                </li>
              ))}
            </ul>
          </Card>
        </div>

        <div>
          <Card className="p-6 mb-6 sticky top-6">
            <div className="text-3xl font-bold mb-4">
              ${strategy.monthlyPrice}/mes
            </div>
            {strategy.setupFee && (
              <p className="text-muted-foreground mb-4">
                + ${strategy.setupFee} fee único
              </p>
            )}

            <SubscribeDialog strategy={strategy}>
              <Button className="w-full" size="lg">
                Suscribirse
              </Button>
            </SubscribeDialog>
          </Card>

          <EstrategaProfile estratega={strategy.estratega} />
        </div>
      </div>
    </div>
  );
}
```

**Entregables:**
- [ ] Listado de estrategias
- [ ] Página de detalle de estrategia
- [ ] Diálogo de suscripción
- [ ] Perfil de estratega
- [ ] Gráficos de performance

---

## 🏗️ INFRAESTRUCTURA COMPARTIDA

### Tareas que afectan a todos los proyectos

#### TAREA I-A: Setup de Bases de Datos
**Responsable:** Dev Backend / DevOps
**Duración estimada:** 2-3 horas
**Dependencias:** Ninguna

**Opción 1: Supabase (Recomendada para MVP)**

```bash
# 1. Crear cuenta en https://supabase.com
# 2. Crear nuevo proyecto
# 3. Obtener connection string

# 4. Configurar en .env
DATABASE_URL="postgresql://postgres:[YOUR-PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres"

# 5. Instalar extensión TimescaleDB (para métricas)
# En Supabase Dashboard → Database → Extensions
# Habilitar "timescaledb"

# 6. Ejecutar migraciones
cd vibe-kanban-api
npx prisma migrate dev --name init
npx prisma generate

# 7. Seed de datos inicial (opcional)
npx prisma db seed
```

**Opción 2: Railway**

```bash
# 1. Instalar Railway CLI
npm install -g @railway/cli

# 2. Login
railway login

# 3. Crear proyecto
railway init

# 4. Provisionar PostgreSQL
railway add

# 5. Obtener connection string
railway variables

# 6. Configurar en .env y ejecutar migraciones
```

**Redis Setup:**

```bash
# Opción 1: Upstash (Serverless Redis)
# 1. Crear cuenta en https://upstash.com
# 2. Crear database
# 3. Obtener REDIS_URL

# Opción 2: Railway
railway add # Seleccionar Redis

# Opción 3: Local (desarrollo)
docker run -d -p 6379:6379 redis:7-alpine
```

**Entregables:**
- [ ] PostgreSQL configurado y accesible
- [ ] TimescaleDB extension habilitada
- [ ] Redis configurado
- [ ] Migraciones ejecutadas
- [ ] Connection strings en .env

---

#### TAREA I-B: Setup de Autenticación
**Responsable:** Dev Full-stack
**Duración estimada:** 4-6 horas
**Dependencias:** TAREA I-A, TAREA P1-A, TAREA P1-B

```bash
# Instalar NextAuth.js
cd vibe-kanban-web
npm install next-auth
```

```typescript
// src/app/api/auth/[...nextauth]/route.ts
import NextAuth, { NextAuthOptions } from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';

export const authOptions: NextAuthOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        // Validar usuario
        const user = await prisma.user.findUnique({
          where: { email: credentials.email },
        });

        if (!user) {
          return null;
        }

        // Validar password (usar bcrypt)
        const isValid = await bcrypt.compare(
          credentials.password,
          user.password
        );

        if (!isValid) {
          return null;
        }

        return {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role,
        };
      },
    }),
  ],
  session: {
    strategy: 'jwt',
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      if (session.user) {
        session.user.role = token.role as string;
      }
      return session;
    },
  },
  pages: {
    signIn: '/auth/signin',
  },
};

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

**Entregables:**
- [ ] NextAuth configurado
- [ ] Login/Registro funcionando
- [ ] Protección de rutas
- [ ] Middleware de autenticación

---

#### TAREA I-C: CI/CD Pipeline
**Responsable:** DevOps / Dev Backend
**Duración estimada:** 3-4 horas
**Dependencias:** Todas las tareas de setup

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Run linter
        run: npm run lint

      - name: Type check
        run: npm run type-check

  deploy-frontend:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'

  deploy-backend:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to Railway
        uses: berviantoleo/railway-deploy@main
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: api
```

**Entregables:**
- [ ] GitHub Actions configurado
- [ ] Tests automatizados
- [ ] Deploy automático a Vercel (frontend)
- [ ] Deploy automático a Railway (backend)

---

## 📅 TIMELINE Y DEPENDENCIAS

### Semana 1: Setup Inicial (Tareas Paralelas)

**Día 1-2:**
- ✅ TAREA I-A: Setup de Bases de Datos (2-3h)
- ✅ TAREA P1-A: Setup de Frontend (2-3h) [Paralelo]
- ✅ TAREA P1-B: Setup de Backend (3-4h) [Paralelo]

**Día 3-4:**
- ✅ TAREA I-B: Setup de Autenticación (4-6h)
- ✅ TAREA P1-C: Integración con Meta API (4-6h) [Puede empezar antes]

**Día 5:**
- ✅ TAREA P1-D: Sistema de Alertas (4-5h)
- ✅ TAREA I-C: CI/CD Pipeline (3-4h) [Paralelo]

### Semana 2: FASE 0 - MVP

**Día 1-3:**
- ✅ TAREA P1-E: Dashboard Frontend (8-12h)

**Día 4-5:**
- ✅ Testing interno
- ✅ Ajustes y bug fixes
- ✅ Documentación

**Resultado:** FASE 0 lista para equipo interno

### Semana 3-6: FASE 1 - Marketplace

**Semana 3:**
- ✅ TAREA P2-A: Setup de Workflow Service (3-4h)
- ✅ TAREA P2-B: Sistema de Publicación (4-6h)
- ✅ TAREA P2-D: Integración de Pagos (6-8h)

**Semana 4:**
- ✅ TAREA P2-C: Marketplace Frontend (8-12h)

**Semana 5-6:**
- ✅ Testing beta
- ✅ Refinamiento basado en feedback

**Resultado:** Marketplace lanzado a gremiación

### Semana 7-12: FASE 2 - Copy Strategy

**Semana 7-8:**
- ✅ TAREA P3-A: Setup de Strategy Service (3-4h)
- ✅ TAREA P3-B: Integración con n8n (4-6h)
- ✅ TAREA P3-D: MCP Personalizado (6-8h)

**Semana 9-10:**
- ✅ TAREA P3-C: Sistema de Replicación (8-12h)
- ✅ TAREA P3-E: Frontend de Estrategias (10-14h)

**Semana 11:**
- ✅ Levantamiento de Método Esfera (con hermano de Cristian)
- ✅ Implementación de 2-3 estrategias iniciales

**Semana 12:**
- ✅ Testing con estrategas
- ✅ Refinamiento final
- ✅ Preparación para lanzamiento público

---

## ✅ CHECKLIST PRE-EJECUCIÓN

Antes de ejecutar cualquier tarea, asegúrate de:

### Decisiones Técnicas
- [ ] Stack de frontend elegido (Next.js recomendado)
- [ ] Hosting de frontend elegido (Vercel recomendado)
- [ ] Hosting de backend elegido (Railway recomendado)
- [ ] Base de datos elegida (Supabase recomendado)
- [ ] n8n Cloud vs Self-hosted decidido
- [ ] Método de autenticación elegido (NextAuth recomendado)
- [ ] Monorepo vs Multi-repo decidido

### Cuentas y Credenciales
- [ ] Cuenta de GitHub creada
- [ ] Cuenta de Vercel creada (si aplica)
- [ ] Cuenta de Railway/Render creada (si aplica)
- [ ] Cuenta de Supabase creada (si aplica)
- [ ] Cuenta de n8n Cloud creada (si aplica)
- [ ] Meta App creada en Facebook Developers
- [ ] Cuenta de Stripe creada (modo test)
- [ ] Cuenta de SendGrid creada

### Presupuesto
- [ ] Presupuesto mensual definido
- [ ] Tarjeta de crédito lista para servicios de pago

### Equipo
- [ ] Dev Frontend asignado
- [ ] Dev Backend asignado
- [ ] DevOps asignado (puede ser mismo que backend)
- [ ] Comunicación establecida (Slack/Discord)

---

## 🚀 COMANDO PARA EJECUTAR

Una vez aprobado todo, puedes ejecutar los setups en paralelo:

```bash
# Terminal 1: Frontend
git clone <repo>
cd vibe-kanban-web
# Seguir TAREA P1-A

# Terminal 2: Backend
cd vibe-kanban-api
# Seguir TAREA P1-B

# Terminal 3: Infraestructura
# Seguir TAREA I-A
```

---

**¿LISTO PARA PROCEDER?**

Por favor confirma:
1. ✅ Has revisado todas las decisiones técnicas
2. ✅ Tienes todas las cuentas necesarias
3. ✅ El presupuesto está aprobado
4. ✅ El equipo está asignado

**Responde "SÍ, EJECUTAR CONFIGURACIÓN" para que proceda con los comandos específicos.**
