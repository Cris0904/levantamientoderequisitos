# Documento Maestro - Vibe Kanban Platform (Contexto para LLM)

**Versión:** 1.0
**Fecha:** 2025-10-21
**Propósito:** Contexto completo para asistentes LLM que trabajarán en el proyecto

---

## RESUMEN EJECUTIVO DEL PROYECTO

### Visión General
Plataforma SaaS en 3 fases para automatización y optimización de estrategias de tráfico pago (Meta Ads), dirigida a traficers y empresarios que buscan gestionar campañas de forma centralizada.

### Problema que Resuelve
- Traficers gestionan múltiples cuentas de Meta Ads manualmente
- Falta de alertas automáticas sobre problemas en campañas
- No existe forma de monetizar estrategias de tráfico probadas
- Ausencia de workflows de automatización accesibles para no-técnicos

### Solución Propuesta (3 Fases)

#### FASE 0: MCP as a Service (Dashboard Base)
- Dashboard centralizado para gestión de múltiples cuentas Meta Ads
- Sistema de alertas automáticas (campañas rechazadas, presupuesto agotado, etc.)
- Integración con Meta Marketing API
- Acceso a workflows n8n embebidos
- **Precio:** $29/mes
- **Timeline:** Semana 1-2 (esta semana)

#### FASE 1: Marketplace de Workflows (Descartada temporalmente)
- Inicialmente se pensó en marketplace abierto donde usuarios publicaran workflows
- **DECISIÓN:** Se descartó porque traficers no tienen nivel técnico suficiente
- **ALTERNATIVA:** Workflows predefinidos por equipo interno

#### FASE 2: Copy Strategy (PRIORIDAD MÁXIMA)
- Sistema tipo "copy trading" de forex aplicado a tráfico pago
- Usuarios se suscriben a estrategias de expertos verificados
- Replicación automática de estrategias (Método Esfera, Método Cañón, etc.)
- Estrategas reciben 30-50% de suscripciones
- **Precio adicional:** $50-$150/mes por estrategia
- **Timeline:** Mes 2-3

---

## ARQUITECTURA TÉCNICA

### Stack Tecnológico

**Frontend:**
```
- Next.js 14+ (App Router)
- React 18 + TypeScript
- Tailwind CSS + shadcn/ui
- Zustand (state management)
- React Query (data fetching)
```

**Backend:**
```
- Node.js 20+ con TypeScript
- Fastify (framework)
- Prisma ORM
- Bull + Redis (job queues)
```

**Bases de Datos:**
```
- PostgreSQL (Supabase) - datos relacionales
- TimescaleDB extension - métricas temporales
- Redis - cache y queues
```

**Integraciones Core:**
```
- Meta Marketing API (acceso a campañas)
- n8n Cloud/Self-hosted (automatización)
- Stripe (pagos y suscripciones)
- SendGrid (emails transaccionales)
```

**Hosting:**
```
- Vercel (frontend + edge functions)
- Railway (backend + PostgreSQL + Redis)
```

### Estructura de Monorepo

```
vibe-platform/
├── apps/
│   ├── web/                 # Next.js frontend
│   ├── api/                 # Fastify backend
│   └── workflow-service/    # n8n integration service
├── packages/
│   ├── database/            # Prisma schemas
│   ├── ui/                  # Shared UI components
│   ├── config/              # Shared configs
│   └── types/               # Shared TypeScript types
├── tools/
│   └── scripts/             # Setup y deployment scripts
└── docs/
    ├── REQUIREMENTS.md
    ├── ARCHITECTURE.md
    └── API_DOCS.md
```

---

## MODELO DE DATOS (Prisma Schemas)

### User
```prisma
model User {
  id            String    @id @default(uuid())
  email         String    @unique
  passwordHash  String
  firstName     String
  lastName      String
  avatarUrl     String?
  role          UserRole  @default(TRAFFICER)

  // Relaciones
  metaAccounts  MetaAccount[]
  subscriptions Subscription[]
  strategies    Strategy[]

  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

enum UserRole {
  TRAFFICER
  STRATEGIST
  ADMIN
}
```

### MetaAccount
```prisma
model MetaAccount {
  id                String   @id @default(uuid())
  userId            String
  user              User     @relation(fields: [userId], references: [id])

  // Meta API Credentials
  accessToken       String   @encrypted
  accountId         String
  accountName       String
  adAccountId       String

  // Configuración
  isActive          Boolean  @default(true)
  alertsEnabled     Boolean  @default(true)

  // Relaciones
  campaigns         Campaign[]
  alerts            Alert[]

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  @@unique([userId, accountId])
}
```

### Strategy
```prisma
model Strategy {
  id              String   @id @default(uuid())
  name            String
  description     String

  // Información del estratega
  strategistId    String
  strategist      User     @relation(fields: [strategistId], references: [id])

  // Método (esfera, cañón, etc.)
  methodType      String

  // Configuración de la estrategia
  configSchema    Json     // JSON Schema para parámetros configurables
  n8nWorkflowId   String   // ID del workflow en n8n

  // Pricing
  monthlyPrice    Decimal  @db.Decimal(10, 2)
  revenueShare    Decimal  @db.Decimal(5, 2) // % para estratega

  // Estado
  isActive        Boolean  @default(false)
  isVerified      Boolean  @default(false)

  // Métricas
  totalSubscribers Int     @default(0)
  avgROAS          Decimal? @db.Decimal(10, 2)

  // Relaciones
  subscriptions   StrategySubscription[]

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}
```

### StrategySubscription
```prisma
model StrategySubscription {
  id              String   @id @default(uuid())

  userId          String
  user            User     @relation(fields: [userId], references: [id])

  strategyId      String
  strategy        Strategy @relation(fields: [strategyId], references: [id])

  // Configuración personalizada
  config          Json     // Parámetros configurados por el usuario

  // Estado
  status          SubscriptionStatus @default(ACTIVE)

  // Stripe
  stripeSubscriptionId String?

  // Fechas
  startedAt       DateTime @default(now())
  endedAt         DateTime?

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@unique([userId, strategyId])
}

enum SubscriptionStatus {
  ACTIVE
  PAUSED
  CANCELLED
  EXPIRED
}
```

### Alert
```prisma
model Alert {
  id              String      @id @default(uuid())

  metaAccountId   String
  metaAccount     MetaAccount @relation(fields: [metaAccountId], references: [id])

  type            AlertType
  severity        AlertSeverity
  title           String
  message         String

  // Metadata específica del tipo de alerta
  metadata        Json

  // Estado
  isRead          Boolean     @default(false)
  isResolved      Boolean     @default(false)

  // Acciones tomadas
  actionTaken     String?
  resolvedAt      DateTime?

  createdAt       DateTime    @default(now())
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
```

---

## INTEGRACIONES CLAVE

### Meta Marketing API

**Endpoints Principales:**
```
GET /{ad-account-id}/campaigns
GET /{campaign-id}/insights
POST /{ad-account-id}/campaigns
PUT /{campaign-id}
```

**Permisos Requeridos:**
```
- ads_management
- ads_read
- business_management
```

**Webhook Events:**
```
- CAMPAIGN_STATUS_CHANGED
- AD_ACCOUNT_UPDATE
- BILLING_EVENT
```

### n8n Integration

**Workflow Types:**

1. **Alert Workflows:**
   - Monitoreo cada 30 minutos
   - Verificación de métricas
   - Envío de notificaciones

2. **Strategy Workflows:**
   - Replicación de campañas
   - Ajuste de presupuestos
   - Optimización basada en reglas

**Implementación:**
```typescript
// n8n API Client
class N8NClient {
  async executeWorkflow(workflowId: string, data: any) {
    return await fetch(`${N8N_URL}/webhook/${workflowId}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
  }

  async deployStrategy(strategyConfig: StrategyConfig) {
    // Crear workflow en n8n basado en template
    const workflow = this.buildWorkflowFromTemplate(strategyConfig);
    return await this.createWorkflow(workflow);
  }
}
```

### Stripe Integration

**Productos:**
1. Base Subscription ($29/mes)
2. Strategy Subscriptions ($50-$150/mes)

**Webhook Events:**
```
- customer.subscription.created
- customer.subscription.updated
- customer.subscription.deleted
- invoice.payment_succeeded
- invoice.payment_failed
```

---

## FLUJOS DE USUARIO PRINCIPALES

### 1. Onboarding de Traficer

```
1. Registro (email + password)
2. Verificación de email
3. Conectar cuenta de Meta Ads (OAuth)
4. Importar primeras campañas
5. Configurar alertas deseadas
6. Tour del dashboard
```

### 2. Suscripción a Estrategia

```
1. Explorar catálogo de estrategias
2. Ver detalles + métricas (ROAS, suscriptores, etc.)
3. Seleccionar estrategia
4. Configurar parámetros (producto, presupuesto, días)
5. Pago con Stripe
6. Workflow se despliega automáticamente
7. Monitoreo en tiempo real
```

### 3. Publicación de Estrategia (Estratega)

```
1. Aplicar para ser estratega verificado
2. Subir documentación de resultados
3. Equipo interno verifica (2-3 días)
4. Estratega configura workflow en n8n (asistido)
5. Define parámetros configurables
6. Establece precio
7. Publica estrategia
8. Empieza a recibir suscriptores
```

---

## CONSIDERACIONES LEGALES CRÍTICAS

### ⚠️ ACCIÓN INMEDIATA REQUERIDA

**Consultoría Legal sobre:**

1. **Almacenamiento de Datos de Meta:**
   - ¿Podemos almacenar métricas de campañas?
   - ¿Necesitamos anonimizar datos?
   - ¿Cuánto tiempo podemos retener data?

2. **Políticas de Meta:**
   - Cumplimiento con términos de uso de API
   - Límites de rate limiting
   - Uso permitido de datos

3. **Acuerdos con Estrategas:**
   - Contratos de revenue share
   - Responsabilidad por resultados
   - Propiedad intelectual de estrategias

4. **Privacidad de Usuarios:**
   - GDPR compliance (si hay usuarios EU)
   - Términos de servicio
   - Política de privacidad

**Recomendación:** NO lanzar FASE 2 sin asesoría legal completa.

---

## MÉTRICAS DE ÉXITO (KPIs)

### FASE 0 (Semana 2-4)
- ✅ 5-10 usuarios internos activos
- ✅ 20+ cuentas Meta Ads conectadas
- ✅ 100+ alertas generadas correctamente
- ✅ 0 downtime crítico

### FASE 1 (Mes 1-2)
- 100+ usuarios beta (gremiación)
- 70% retention después de 30 días
- < 2 segundos tiempo de carga
- NPS > 50

### FASE 2 (Mes 3-6)
- 500 usuarios base activos
- 50+ suscripciones a estrategias
- 3-5 estrategias verificadas disponibles
- $25,000+ MRR

### FASE 3 (Mes 12-18)
- 5,000-10,000 usuarios
- 1,000+ suscripciones a estrategias
- 20+ estrategas verificados
- $500,000+ MRR

---

## ROADMAP DETALLADO

### ✅ Semana 1 (21-27 Oct): FASE 0 Development
**Objetivo:** MVP funcional para equipo interno

**Backend:**
- Setup de Supabase + Prisma
- Auth con JWT
- Meta API integration básica
- Sistema de alerts (polling cada 30min)

**Frontend:**
- Dashboard con listado de cuentas
- Vista de campañas por cuenta
- Centro de alertas
- Perfil de usuario

**Deploy:**
- Vercel (frontend)
- Railway (backend)
- CI/CD básico

### ✅ Semana 2 (28 Oct - 3 Nov): Testing Interno
**Objetivo:** Validar con equipo de agencia (5-10 usuarios)

- Onboarding de equipo interno
- Conectar cuentas reales
- Recolectar feedback
- Fix de bugs críticos
- Preparar presentación para gremiación

### 📅 Semana 3-4 (4-17 Nov): Beta Pública
**Objetivo:** Lanzar a gremiación (100 usuarios)

- Presentación en gremiación
- Onboarding masivo
- Soporte intensivo
- Iteración basada en feedback
- Análisis de métricas de uso

### 📅 Mes 2 (18 Nov - 17 Dic): Preparación FASE 2
**Objetivo:** Diseñar sistema de Copy Strategy

- Reunión con hermano de Cristian (Método Esfera)
- Documentar 2-3 estrategias predefinidas
- Diseñar UX de suscripción a estrategias
- Prototipo de replicación de campañas
- Consultoría legal (CRÍTICO)

### 📅 Mes 3-4 (Dic-Ene): Desarrollo FASE 2
**Objetivo:** Implementar Copy Strategy

- Sistema de suscripciones (Stripe)
- Workflows n8n para cada estrategia
- Dashboard de estrategias
- Sistema de configuración de parámetros
- Testing con beta testers

### 📅 Mes 5-6 (Feb-Mar): Launch FASE 2
**Objetivo:** Abrir estrategias al público

- Lanzar con 2-3 estrategias verificadas
- Marketing hacia traficers
- Onboarding de primeros estrategas
- Proceso de verificación
- Sistema de reviews y ratings

---

## RIESGOS Y MITIGACIONES

### 🔴 RIESGO CRÍTICO: Cumplimiento Legal
**Probabilidad:** Alta
**Impacto:** Crítico (cierre de plataforma)

**Mitigación:**
- Consultoría legal ANTES de FASE 2
- Implementar anonimización de datos
- Políticas de uso claras
- Auditoría de seguridad

### 🟠 RIESGO ALTO: Dependencia de Meta API
**Probabilidad:** Media
**Impacto:** Alto

**Mitigación:**
- Monitoreo de políticas de Meta
- Plan de contingencia (otras plataformas: Google Ads, TikTok Ads)
- Diversificación de revenue streams
- Caching agresivo de datos

### 🟡 RIESGO MEDIO: Calidad de Estrategias
**Probabilidad:** Media
**Impacto:** Medio (reputación)

**Mitigación:**
- Proceso de verificación estricto
- Período de prueba de 30 días
- Sistema de reviews transparente
- Money-back guarantee

### 🟢 RIESGO BAJO: Escalabilidad de n8n
**Probabilidad:** Baja (corto plazo)
**Impacto:** Medio

**Mitigación:**
- Plan de migración a código nativo en FASE 3
- Uso de n8n Cloud (escalable)
- Monitoreo de performance

---

## COMANDOS ÚTILES PARA SETUP

### Inicializar Monorepo
```bash
npx create-turbo@latest vibe-platform
cd vibe-platform
pnpm install
```

### Setup Frontend (Next.js)
```bash
cd apps/web
npx create-next-app@latest . --typescript --tailwind --app
pnpm add zustand @tanstack/react-query
pnpm add -D @types/node
```

### Setup Backend (Fastify)
```bash
cd apps/api
pnpm init
pnpm add fastify @fastify/cors @fastify/jwt
pnpm add prisma @prisma/client
pnpm add -D tsx typescript @types/node
npx prisma init
```

### Setup Database (Prisma)
```bash
cd packages/database
npx prisma init
# Editar schema.prisma
npx prisma generate
npx prisma migrate dev --name init
```

### Variables de Entorno
```bash
# .env.local (frontend)
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_STRIPE_KEY=pk_test_...

# .env (backend)
DATABASE_URL=postgresql://...
JWT_SECRET=your-secret-key
META_APP_ID=...
META_APP_SECRET=...
STRIPE_SECRET_KEY=sk_test_...
N8N_API_URL=https://...
```

---

## ENDPOINTS API PRINCIPALES

### Authentication
```
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/refresh
GET    /api/auth/me
```

### Meta Accounts
```
GET    /api/meta-accounts
POST   /api/meta-accounts/connect
GET    /api/meta-accounts/:id
DELETE /api/meta-accounts/:id
```

### Campaigns
```
GET    /api/meta-accounts/:id/campaigns
GET    /api/campaigns/:id/insights
POST   /api/campaigns/:id/pause
POST   /api/campaigns/:id/resume
```

### Alerts
```
GET    /api/alerts
PATCH  /api/alerts/:id/read
PATCH  /api/alerts/:id/resolve
```

### Strategies (FASE 2)
```
GET    /api/strategies
GET    /api/strategies/:id
POST   /api/strategies/:id/subscribe
GET    /api/my-subscriptions
PATCH  /api/subscriptions/:id/pause
```

---

## PREGUNTAS FRECUENTES PARA LLMs

### ¿Cuál es el orden de prioridad de las fases?
1. FASE 0 (esta semana) - Dashboard + Alertas
2. FASE 2 (mes 2-3) - Copy Strategy (SE SALTÓ FASE 1)
3. FASE 1 fue descartada en su forma original

### ¿Cómo se integra n8n exactamente?
- n8n se ejecuta como servicio separado
- Backend se comunica vía webhooks y API REST
- Cada estrategia = 1 workflow en n8n
- Workflows se triggean vía webhook desde backend

### ¿Cómo funciona la replicación de estrategias?
1. Usuario se suscribe a estrategia
2. Configura parámetros (producto, presupuesto, días)
3. Backend llama a n8n API para ejecutar workflow
4. Workflow usa Meta API para crear/modificar campañas
5. Resultados se almacenan en DB
6. Dashboard muestra métricas en tiempo real

### ¿Qué es el "Método Esfera"?
Estrategia de tráfico creada por hermano de Cristian. Detalles se levantarán en reunión próxima semana. Por ahora, diseñar sistema genérico que soporte cualquier tipo de estrategia.

### ¿Cómo se verifica a un estratega?
1. Aplicación con portfolio de resultados
2. Revisión manual por equipo (screenshots, acceso a cuentas)
3. Validación de ROI/ROAS declarado
4. Entrevista técnica
5. Período de prueba de 30 días
6. Aprobación final

---

## RECURSOS EXTERNOS

### Documentación Oficial
- [Meta Marketing API](https://developers.facebook.com/docs/marketing-apis)
- [n8n Documentation](https://docs.n8n.io/)
- [Stripe Subscriptions](https://stripe.com/docs/billing/subscriptions/overview)
- [Prisma Docs](https://www.prisma.io/docs)

### Herramientas de Desarrollo
- [Meta Graph API Explorer](https://developers.facebook.com/tools/explorer)
- [Stripe Test Mode](https://dashboard.stripe.com/test)
- [Supabase Dashboard](https://app.supabase.com/)

### Inspiración
- Copy trading platforms: eToro, ZuluTrade
- Workflow marketplaces: Zapier Templates, Make Templates
- Dashboard references: Triple Whale, Northbeam

---

## GLOSARIO

**Traficer:** Profesional que gestiona campañas de tráfico pago (Meta Ads, Google Ads, etc.)

**MCP:** Model Context Protocol - En este proyecto se refiere al dashboard base

**Copy Strategy:** Sistema donde usuarios replican estrategias de expertos

**Método Esfera/Cañón:** Estrategias específicas de tráfico (detalles por definir)

**ROAS:** Return on Ad Spend (ROI de publicidad)

**WIP Limit:** Work In Progress - Límite de trabajo en progreso (Kanban)

**n8n:** Plataforma de automatización de workflows (alternativa open-source a Zapier)

**Gremiación:** Comunidad/grupo de traficers donde se lanzará beta

---

## CONTACTOS CLAVE

**Stakeholder Principal:** Cristian Muñoz
**Developer n8n:** Simón
**Estratega (Método Esfera):** Hermano de Cristian (reunión próxima semana)

---

## ESTADO ACTUAL (21 Oct 2025)

✅ **Completado:**
- Documentación completa (Requirements, Architecture, Setup Plan)
- Análisis de transcripción de reunión
- Definición de 3 fases
- Stack tecnológico decidido

⏳ **En Progreso:**
- Setup de n8n embebido (Simón)
- Preparación de infraestructura

📅 **Próximos Pasos:**
1. Aprobar stack y decisiones técnicas
2. Iniciar consultoría legal
3. Ejecutar setup inicial (3 tareas paralelas)
4. Testing interno (esta semana)
5. Presentación en gremiación (próxima semana)

---

**FIN DEL DOCUMENTO MAESTRO**

Este documento debe ser actualizado cada vez que haya cambios significativos en:
- Decisiones de producto
- Arquitectura técnica
- Roadmap
- Riesgos identificados
- KPIs

**Última actualización:** 2025-10-21
**Próxima revisión:** 2025-10-28 (después de testing interno)
