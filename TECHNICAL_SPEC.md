# Technical Specification - Vibe Kanban Board

## Document Information
**Project:** Vibe Kanban
**Version:** 1.0
**Last Updated:** 2025-10-21
**Task ID:** BD54

---

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Client Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Web App    │  │  Mobile Web  │  │   PWA        │      │
│  │  (React)     │  │  (Responsive)│  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTPS/WSS
                            │
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway / Load Balancer              │
│                      (NGINX / AWS ALB)                      │
└─────────────────────────────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
┌───────────────▼────────┐  ┌──────────▼──────────┐
│   Application Server   │  │  WebSocket Server   │
│    (Node.js/NestJS)    │  │   (Socket.io)       │
│                        │  │                     │
│  ┌──────────────────┐  │  │  ┌──────────────┐  │
│  │  REST/GraphQL    │  │  │  │ Real-time    │  │
│  │  API Endpoints   │  │  │  │ Events       │  │
│  └──────────────────┘  │  │  └──────────────┘  │
└────────────┬───────────┘  └──────────┬──────────┘
             │                         │
             └──────────┬──────────────┘
                        │
        ┌───────────────┼───────────────┬─────────────┐
        │               │               │             │
┌───────▼──────┐ ┌─────▼─────┐ ┌───────▼──────┐ ┌───▼────┐
│  PostgreSQL  │ │   Redis   │ │   S3/Blob    │ │ Queue  │
│  (Primary)   │ │  (Cache)  │ │   Storage    │ │(RabbitMQ)│
└──────────────┘ └───────────┘ └──────────────┘ └────────┘
```

### 1.2 Technology Stack

#### Frontend
- **Framework:** React 18.x with TypeScript
- **State Management:** Zustand or Redux Toolkit
- **UI Components:** Custom components with Headless UI / Radix UI
- **Styling:** Tailwind CSS
- **Drag and Drop:** @dnd-kit/core
- **Routing:** React Router v6
- **Forms:** React Hook Form + Zod validation
- **HTTP Client:** Axios or React Query
- **WebSocket Client:** Socket.io-client
- **Build Tool:** Vite
- **Testing:** Vitest, React Testing Library, Playwright

#### Backend
- **Runtime:** Node.js 20.x LTS
- **Framework:** NestJS (TypeScript)
- **API:** RESTful + GraphQL (optional)
- **Authentication:** Passport.js with JWT
- **Validation:** class-validator, class-transformer
- **ORM:** Prisma or TypeORM
- **WebSocket:** Socket.io
- **Task Queue:** Bull (Redis-based)
- **Logging:** Winston or Pino
- **Testing:** Jest, Supertest

#### Database
- **Primary:** PostgreSQL 15.x
- **Cache:** Redis 7.x
- **File Storage:** AWS S3 / Azure Blob / Google Cloud Storage
- **Search:** PostgreSQL Full-Text Search or Elasticsearch (future)

#### DevOps
- **Containerization:** Docker + Docker Compose
- **Orchestration:** Kubernetes (production) or Docker Swarm
- **CI/CD:** GitHub Actions or GitLab CI
- **Monitoring:** Prometheus + Grafana
- **Error Tracking:** Sentry
- **Logging:** ELK Stack (Elasticsearch, Logstash, Kibana)

---

## 2. Database Schema

### 2.1 Entity Relationship Diagram

```
┌─────────────┐       ┌──────────────┐       ┌─────────────┐
│    User     │───┐   │    Board     │───┐   │   Column    │
└─────────────┘   │   └──────────────┘   │   └─────────────┘
                  │                      │
                  │   ┌──────────────┐   │   ┌─────────────┐
                  └───│ BoardMember  │   └───│    Card     │
                      └──────────────┘       └─────────────┘
                                                    │
                                    ┌───────────────┼──────────────┐
                                    │               │              │
                            ┌───────▼──────┐ ┌─────▼──────┐ ┌────▼─────┐
                            │   Comment    │ │ Attachment │ │CardLabel │
                            └──────────────┘ └────────────┘ └──────────┘
```

### 2.2 Table Definitions

#### users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  avatar_url TEXT,
  email_verified BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_login_at TIMESTAMP,
  is_active BOOLEAN DEFAULT TRUE,

  CONSTRAINT email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

#### boards
```sql
CREATE TABLE boards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  background_url TEXT,
  background_color VARCHAR(7) DEFAULT '#ffffff',
  is_archived BOOLEAN DEFAULT FALSE,
  is_template BOOLEAN DEFAULT FALSE,
  visibility VARCHAR(20) DEFAULT 'private' CHECK (visibility IN ('private', 'team', 'public')),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT name_not_empty CHECK (LENGTH(TRIM(name)) > 0)
);

CREATE INDEX idx_boards_owner ON boards(owner_id);
CREATE INDEX idx_boards_created_at ON boards(created_at DESC);
CREATE INDEX idx_boards_archived ON boards(is_archived);
```

#### board_members
```sql
CREATE TABLE board_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  board_id UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(20) DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
  joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  invited_by UUID REFERENCES users(id),

  UNIQUE(board_id, user_id)
);

CREATE INDEX idx_board_members_board ON board_members(board_id);
CREATE INDEX idx_board_members_user ON board_members(user_id);
```

#### columns
```sql
CREATE TABLE columns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  board_id UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  position INTEGER NOT NULL,
  wip_limit INTEGER,
  color VARCHAR(7),
  is_collapsed BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT position_non_negative CHECK (position >= 0),
  CONSTRAINT wip_limit_positive CHECK (wip_limit IS NULL OR wip_limit > 0)
);

CREATE INDEX idx_columns_board ON columns(board_id, position);
CREATE UNIQUE INDEX idx_columns_board_position ON columns(board_id, position);
```

#### cards
```sql
CREATE TABLE cards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  column_id UUID NOT NULL REFERENCES columns(id) ON DELETE CASCADE,
  title VARCHAR(500) NOT NULL,
  description TEXT,
  position INTEGER NOT NULL,
  priority VARCHAR(20) DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high', 'critical')),
  due_date TIMESTAMP,
  estimated_hours DECIMAL(8, 2),
  actual_hours DECIMAL(8, 2),
  is_archived BOOLEAN DEFAULT FALSE,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP,

  CONSTRAINT position_non_negative CHECK (position >= 0),
  CONSTRAINT hours_non_negative CHECK (
    (estimated_hours IS NULL OR estimated_hours >= 0) AND
    (actual_hours IS NULL OR actual_hours >= 0)
  )
);

CREATE INDEX idx_cards_column ON cards(column_id, position);
CREATE INDEX idx_cards_created_by ON cards(created_by);
CREATE INDEX idx_cards_due_date ON cards(due_date) WHERE due_date IS NOT NULL;
CREATE INDEX idx_cards_archived ON cards(is_archived);
CREATE INDEX idx_cards_priority ON cards(priority);
```

#### card_assignees
```sql
CREATE TABLE card_assignees (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  assigned_by UUID REFERENCES users(id),

  UNIQUE(card_id, user_id)
);

CREATE INDEX idx_card_assignees_card ON card_assignees(card_id);
CREATE INDEX idx_card_assignees_user ON card_assignees(user_id);
```

#### labels
```sql
CREATE TABLE labels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  board_id UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  color VARCHAR(7) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(board_id, name)
);

CREATE INDEX idx_labels_board ON labels(board_id);
```

#### card_labels
```sql
CREATE TABLE card_labels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
  label_id UUID NOT NULL REFERENCES labels(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(card_id, label_id)
);

CREATE INDEX idx_card_labels_card ON card_labels(card_id);
CREATE INDEX idx_card_labels_label ON card_labels(label_id);
```

#### comments
```sql
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
  author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  is_edited BOOLEAN DEFAULT FALSE,

  CONSTRAINT content_not_empty CHECK (LENGTH(TRIM(content)) > 0)
);

CREATE INDEX idx_comments_card ON comments(card_id, created_at DESC);
CREATE INDEX idx_comments_author ON comments(author_id);
```

#### attachments
```sql
CREATE TABLE attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
  file_name VARCHAR(255) NOT NULL,
  file_url TEXT NOT NULL,
  file_size BIGINT NOT NULL,
  mime_type VARCHAR(100) NOT NULL,
  uploaded_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT file_size_limit CHECK (file_size > 0 AND file_size <= 10485760) -- 10MB
);

CREATE INDEX idx_attachments_card ON attachments(card_id);
CREATE INDEX idx_attachments_uploader ON attachments(uploaded_by);
```

#### checklists
```sql
CREATE TABLE checklists (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  position INTEGER NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT position_non_negative CHECK (position >= 0)
);

CREATE INDEX idx_checklists_card ON checklists(card_id, position);
```

#### checklist_items
```sql
CREATE TABLE checklist_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  checklist_id UUID NOT NULL REFERENCES checklists(id) ON DELETE CASCADE,
  content VARCHAR(500) NOT NULL,
  position INTEGER NOT NULL,
  is_completed BOOLEAN DEFAULT FALSE,
  completed_at TIMESTAMP,
  completed_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT position_non_negative CHECK (position >= 0)
);

CREATE INDEX idx_checklist_items_checklist ON checklist_items(checklist_id, position);
```

#### activity_log
```sql
CREATE TABLE activity_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  board_id UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
  card_id UUID REFERENCES cards(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  action_type VARCHAR(50) NOT NULL,
  entity_type VARCHAR(50) NOT NULL,
  entity_id UUID,
  old_value JSONB,
  new_value JSONB,
  metadata JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_activity_board ON activity_log(board_id, created_at DESC);
CREATE INDEX idx_activity_card ON activity_log(card_id, created_at DESC);
CREATE INDEX idx_activity_user ON activity_log(user_id, created_at DESC);
CREATE INDEX idx_activity_type ON activity_log(action_type);
```

#### notifications
```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type VARCHAR(50) NOT NULL,
  title VARCHAR(255) NOT NULL,
  message TEXT,
  entity_type VARCHAR(50),
  entity_id UUID,
  is_read BOOLEAN DEFAULT FALSE,
  read_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT read_consistency CHECK (
    (is_read = FALSE AND read_at IS NULL) OR
    (is_read = TRUE AND read_at IS NOT NULL)
  )
);

CREATE INDEX idx_notifications_user ON notifications(user_id, created_at DESC);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read) WHERE is_read = FALSE;
```

---

## 3. API Specification

### 3.1 Authentication Endpoints

#### POST /api/v1/auth/register
**Description:** Register a new user account

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe"
}
```

**Response (201):**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe"
  },
  "tokens": {
    "accessToken": "jwt_token",
    "refreshToken": "refresh_token",
    "expiresIn": 3600
  }
}
```

#### POST /api/v1/auth/login
**Description:** Login with email and password

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response (200):**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "avatarUrl": "https://..."
  },
  "tokens": {
    "accessToken": "jwt_token",
    "refreshToken": "refresh_token",
    "expiresIn": 3600
  }
}
```

### 3.2 Board Endpoints

#### GET /api/v1/boards
**Description:** Get all boards for authenticated user

**Query Parameters:**
- `page` (number, default: 1)
- `limit` (number, default: 20)
- `archived` (boolean, default: false)

**Response (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Project Alpha",
      "description": "Main project board",
      "ownerId": "uuid",
      "backgroundUrl": "https://...",
      "isArchived": false,
      "memberCount": 5,
      "cardCount": 23,
      "createdAt": "2025-01-15T10:00:00Z",
      "updatedAt": "2025-01-20T14:30:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

#### POST /api/v1/boards
**Description:** Create a new board

**Request Body:**
```json
{
  "name": "New Project",
  "description": "Project description",
  "backgroundColor": "#f0f0f0",
  "visibility": "private"
}
```

**Response (201):**
```json
{
  "id": "uuid",
  "name": "New Project",
  "description": "Project description",
  "ownerId": "uuid",
  "backgroundColor": "#f0f0f0",
  "visibility": "private",
  "isArchived": false,
  "createdAt": "2025-01-21T10:00:00Z",
  "updatedAt": "2025-01-21T10:00:00Z"
}
```

#### GET /api/v1/boards/:id
**Description:** Get board details with columns and cards

**Response (200):**
```json
{
  "id": "uuid",
  "name": "Project Alpha",
  "description": "Main project board",
  "ownerId": "uuid",
  "backgroundColor": "#f0f0f0",
  "members": [
    {
      "id": "uuid",
      "userId": "uuid",
      "role": "owner",
      "user": {
        "id": "uuid",
        "email": "user@example.com",
        "firstName": "John",
        "lastName": "Doe",
        "avatarUrl": "https://..."
      }
    }
  ],
  "columns": [
    {
      "id": "uuid",
      "name": "To Do",
      "position": 0,
      "wipLimit": 5,
      "cards": [
        {
          "id": "uuid",
          "title": "Task 1",
          "description": "Description",
          "position": 0,
          "priority": "high",
          "dueDate": "2025-01-25T00:00:00Z",
          "assignees": [],
          "labels": [],
          "commentCount": 3,
          "attachmentCount": 1,
          "checklistProgress": { "completed": 2, "total": 5 }
        }
      ]
    }
  ],
  "labels": [
    {
      "id": "uuid",
      "name": "Bug",
      "color": "#ff0000"
    }
  ]
}
```

### 3.3 Card Endpoints

#### POST /api/v1/columns/:columnId/cards
**Description:** Create a new card in a column

**Request Body:**
```json
{
  "title": "New Task",
  "description": "Task description",
  "priority": "medium",
  "dueDate": "2025-01-30T00:00:00Z",
  "estimatedHours": 8,
  "assigneeIds": ["uuid1", "uuid2"],
  "labelIds": ["uuid3"]
}
```

**Response (201):**
```json
{
  "id": "uuid",
  "columnId": "uuid",
  "title": "New Task",
  "description": "Task description",
  "position": 0,
  "priority": "medium",
  "dueDate": "2025-01-30T00:00:00Z",
  "estimatedHours": 8,
  "assignees": [...],
  "labels": [...],
  "createdBy": "uuid",
  "createdAt": "2025-01-21T10:00:00Z"
}
```

#### PUT /api/v1/cards/:id/move
**Description:** Move card to different column/position

**Request Body:**
```json
{
  "columnId": "uuid",
  "position": 2
}
```

**Response (200):**
```json
{
  "id": "uuid",
  "columnId": "uuid",
  "position": 2,
  "updatedAt": "2025-01-21T10:30:00Z"
}
```

### 3.4 WebSocket Events

#### Client → Server Events

**join_board**
```json
{
  "boardId": "uuid"
}
```

**card_moved**
```json
{
  "cardId": "uuid",
  "sourceColumnId": "uuid",
  "targetColumnId": "uuid",
  "position": 2
}
```

**card_updated**
```json
{
  "cardId": "uuid",
  "updates": {
    "title": "Updated Title",
    "priority": "high"
  }
}
```

#### Server → Client Events

**board_updated**
```json
{
  "boardId": "uuid",
  "action": "card_created",
  "data": {
    "card": {...}
  },
  "userId": "uuid",
  "timestamp": "2025-01-21T10:00:00Z"
}
```

**card_moved**
```json
{
  "cardId": "uuid",
  "columnId": "uuid",
  "position": 2,
  "movedBy": "uuid",
  "timestamp": "2025-01-21T10:00:00Z"
}
```

**user_joined**
```json
{
  "userId": "uuid",
  "user": {
    "id": "uuid",
    "firstName": "John",
    "lastName": "Doe",
    "avatarUrl": "https://..."
  }
}
```

---

## 4. Security Implementation

### 4.1 Authentication Flow

```
1. User submits credentials
2. Server validates credentials
3. Server generates JWT with payload:
   {
     "sub": "user_id",
     "email": "user@example.com",
     "iat": timestamp,
     "exp": timestamp + 1h
   }
4. Server returns access token (1h) + refresh token (30d)
5. Client stores tokens (memory for access, httpOnly cookie for refresh)
6. Client includes access token in Authorization header
7. Server validates token on each request
8. Client refreshes token before expiry using refresh endpoint
```

### 4.2 Authorization Rules

**Board Access:**
- User must be board owner or member to view
- Only owner can delete board
- Admin and owner can invite members
- Members can create/edit cards

**Card Access:**
- Any board member can create cards
- Card creator, assignees, and board admins can edit
- Only creator and board admins can delete

### 4.3 Rate Limiting

```typescript
// Global rate limit: 100 requests per minute per IP
app.use(rateLimit({
  windowMs: 60 * 1000,
  max: 100
}));

// Auth endpoints: 5 requests per 15 minutes per IP
authRouter.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5
}));

// File upload: 10 uploads per hour per user
uploadRouter.use(rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 10,
  keyGenerator: (req) => req.user.id
}));
```

### 4.4 Input Validation

```typescript
// Example using class-validator
export class CreateCardDto {
  @IsString()
  @MinLength(1)
  @MaxLength(500)
  title: string;

  @IsString()
  @IsOptional()
  @MaxLength(5000)
  description?: string;

  @IsEnum(Priority)
  @IsOptional()
  priority?: Priority;

  @IsDateString()
  @IsOptional()
  dueDate?: string;

  @IsArray()
  @IsUUID('4', { each: true })
  @IsOptional()
  assigneeIds?: string[];
}
```

---

## 5. Performance Optimization

### 5.1 Database Optimization

**Indexing Strategy:**
- Primary keys: All id columns (UUID)
- Foreign keys: All reference columns
- Query optimization: Common filter/sort columns
- Composite indexes: Multi-column queries

**Query Optimization:**
```sql
-- Use JOIN instead of N+1 queries
SELECT c.*,
       json_agg(DISTINCT a.*) as assignees,
       json_agg(DISTINCT l.*) as labels
FROM cards c
LEFT JOIN card_assignees ca ON ca.card_id = c.id
LEFT JOIN users a ON a.id = ca.user_id
LEFT JOIN card_labels cl ON cl.card_id = c.id
LEFT JOIN labels l ON l.id = cl.label_id
WHERE c.column_id = $1
GROUP BY c.id;
```

**Connection Pooling:**
```typescript
// Prisma configuration
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")

  // Connection pool
  connection_limit = 20
  pool_timeout = 10
}
```

### 5.2 Caching Strategy

**Redis Cache Layers:**

1. **Session Cache** (TTL: 1 hour)
```typescript
key: `session:${userId}`
value: { user data, permissions }
```

2. **Board Cache** (TTL: 5 minutes)
```typescript
key: `board:${boardId}`
value: { board data, columns, cards }
invalidate: on any board update
```

3. **User Cache** (TTL: 15 minutes)
```typescript
key: `user:${userId}`
value: { profile data }
invalidate: on profile update
```

**Cache Invalidation:**
```typescript
// Invalidate on write operations
async updateCard(cardId, data) {
  const card = await prisma.card.update({ where: { id: cardId }, data });

  // Invalidate board cache
  await redis.del(`board:${card.columnId}`);

  // Broadcast update via WebSocket
  this.socketService.emit('card_updated', card);

  return card;
}
```

### 5.3 Frontend Optimization

**Code Splitting:**
```typescript
// Route-based splitting
const BoardPage = lazy(() => import('./pages/Board'));
const AnalyticsPage = lazy(() => import('./pages/Analytics'));

// Component-based splitting
const CardModal = lazy(() => import('./components/CardModal'));
```

**Lazy Loading:**
```typescript
// Virtual scrolling for large lists
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={cards.length}
  itemSize={100}
>
  {({ index, style }) => (
    <Card key={cards[index].id} data={cards[index]} style={style} />
  )}
</FixedSizeList>
```

**Optimistic Updates:**
```typescript
// Update UI immediately, rollback on error
const moveCard = useMutation({
  mutationFn: (data) => api.moveCard(data),
  onMutate: async (newData) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries(['board', boardId]);

    // Snapshot previous value
    const previousBoard = queryClient.getQueryData(['board', boardId]);

    // Optimistically update
    queryClient.setQueryData(['board', boardId], (old) => ({
      ...old,
      columns: updateColumnPositions(old.columns, newData)
    }));

    return { previousBoard };
  },
  onError: (err, newData, context) => {
    // Rollback on error
    queryClient.setQueryData(['board', boardId], context.previousBoard);
  }
});
```

---

## 6. Testing Strategy

### 6.1 Unit Tests

**Backend (Jest):**
```typescript
describe('CardService', () => {
  describe('createCard', () => {
    it('should create a card with valid data', async () => {
      const cardData = {
        columnId: 'uuid',
        title: 'Test Card',
        priority: 'high'
      };

      const card = await cardService.create(cardData);

      expect(card).toMatchObject(cardData);
      expect(card.id).toBeDefined();
    });

    it('should throw error with invalid column', async () => {
      const cardData = {
        columnId: 'invalid',
        title: 'Test Card'
      };

      await expect(cardService.create(cardData)).rejects.toThrow();
    });
  });
});
```

**Frontend (Vitest + React Testing Library):**
```typescript
describe('CardComponent', () => {
  it('renders card with title and description', () => {
    const card = {
      id: '1',
      title: 'Test Card',
      description: 'Description'
    };

    render(<Card data={card} />);

    expect(screen.getByText('Test Card')).toBeInTheDocument();
    expect(screen.getByText('Description')).toBeInTheDocument();
  });

  it('calls onMove when dragged to new column', async () => {
    const onMove = vi.fn();
    render(<Card data={card} onMove={onMove} />);

    // Simulate drag and drop
    const cardElement = screen.getByTestId('card-1');
    fireEvent.dragStart(cardElement);
    fireEvent.drop(screen.getByTestId('column-2'));

    expect(onMove).toHaveBeenCalledWith(card.id, 'column-2');
  });
});
```

### 6.2 Integration Tests

```typescript
describe('Board API Integration', () => {
  let authToken: string;
  let boardId: string;

  beforeAll(async () => {
    // Setup test database
    await setupTestDatabase();

    // Create test user and get auth token
    const response = await request(app)
      .post('/api/v1/auth/register')
      .send({
        email: 'test@example.com',
        password: 'TestPass123!'
      });

    authToken = response.body.tokens.accessToken;
  });

  it('should create board and add cards', async () => {
    // Create board
    const boardResponse = await request(app)
      .post('/api/v1/boards')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ name: 'Test Board' });

    expect(boardResponse.status).toBe(201);
    boardId = boardResponse.body.id;

    // Get board details
    const getResponse = await request(app)
      .get(`/api/v1/boards/${boardId}`)
      .set('Authorization', `Bearer ${authToken}`);

    expect(getResponse.status).toBe(200);
    expect(getResponse.body.columns).toHaveLength(3); // Default columns
  });
});
```

### 6.3 E2E Tests (Playwright)

```typescript
test('complete kanban workflow', async ({ page }) => {
  // Login
  await page.goto('/login');
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'TestPass123!');
  await page.click('button[type="submit"]');

  // Create board
  await page.click('text=Create Board');
  await page.fill('[name="name"]', 'My Board');
  await page.click('button:text("Create")');

  // Verify redirect to board
  await expect(page).toHaveURL(/\/boards\/[a-f0-9-]+/);

  // Create card
  await page.click('text=Add Card');
  await page.fill('[name="title"]', 'Test Task');
  await page.fill('[name="description"]', 'Task description');
  await page.click('button:text("Create Card")');

  // Verify card appears
  await expect(page.locator('text=Test Task')).toBeVisible();

  // Move card via drag and drop
  const card = page.locator('[data-card-id]').first();
  const targetColumn = page.locator('[data-column-name="In Progress"]');
  await card.dragTo(targetColumn);

  // Verify card moved
  const movedCard = targetColumn.locator('text=Test Task');
  await expect(movedCard).toBeVisible();
});
```

---

## 7. Deployment Architecture

### 7.1 Production Environment

```yaml
# Docker Compose for production
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - api
      - frontend

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
    environment:
      - NODE_ENV=production
    restart: unless-stopped

  api:
    build:
      context: ./backend
      dockerfile: Dockerfile.prod
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    deploy:
      replicas: 3

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=kanban
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 7.2 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:unit

      - name: Run integration tests
        run: npm run test:integration

      - name: Run E2E tests
        run: npm run test:e2e

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker images
        run: |
          docker build -t kanban-frontend:${{ github.sha }} ./frontend
          docker build -t kanban-backend:${{ github.sha }} ./backend

      - name: Push to registry
        run: |
          docker push kanban-frontend:${{ github.sha }}
          docker push kanban-backend:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          kubectl set image deployment/frontend frontend=kanban-frontend:${{ github.sha }}
          kubectl set image deployment/backend backend=kanban-backend:${{ github.sha }}
          kubectl rollout status deployment/frontend
          kubectl rollout status deployment/backend
```

---

## 8. Monitoring and Observability

### 8.1 Logging

```typescript
// Structured logging with Winston
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'kanban-api' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Request logging middleware
app.use((req, res, next) => {
  logger.info('HTTP Request', {
    method: req.method,
    url: req.url,
    userId: req.user?.id,
    ip: req.ip
  });
  next();
});
```

### 8.2 Metrics

```typescript
// Prometheus metrics
import { Counter, Histogram, register } from 'prom-client';

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
});

const cardCreationCounter = new Counter({
  name: 'cards_created_total',
  help: 'Total number of cards created'
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### 8.3 Health Checks

```typescript
// Health check endpoint
app.get('/health', async (req, res) => {
  const health = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    status: 'ok',
    checks: {
      database: await checkDatabase(),
      redis: await checkRedis(),
      storage: await checkStorage()
    }
  };

  const isHealthy = Object.values(health.checks).every(check => check.status === 'ok');
  res.status(isHealthy ? 200 : 503).json(health);
});
```

---

## 9. Error Handling

### 9.1 Error Response Format

```typescript
// Standard error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "title",
        "message": "Title is required"
      }
    ],
    "timestamp": "2025-01-21T10:00:00Z",
    "path": "/api/v1/cards"
  }
}
```

### 9.2 Error Codes

```typescript
enum ErrorCode {
  // Authentication (401)
  UNAUTHORIZED = 'UNAUTHORIZED',
  INVALID_TOKEN = 'INVALID_TOKEN',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',

  // Authorization (403)
  FORBIDDEN = 'FORBIDDEN',
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',

  // Validation (400)
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INVALID_INPUT = 'INVALID_INPUT',

  // Not Found (404)
  RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND',

  // Conflict (409)
  DUPLICATE_RESOURCE = 'DUPLICATE_RESOURCE',

  // Server Error (500)
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  DATABASE_ERROR = 'DATABASE_ERROR'
}
```

---

## 10. Migration Strategy

### 10.1 Database Migrations

```typescript
// Prisma migration example
// migrations/001_initial_schema.sql

-- Up
CREATE TABLE users (...);
CREATE TABLE boards (...);
-- ... all tables

-- Down
DROP TABLE IF EXISTS users CASCADE;
DROP TABLE IF EXISTS boards CASCADE;
```

### 10.2 Data Migration

```typescript
// Seed data for development
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function seed() {
  // Create demo user
  const user = await prisma.user.create({
    data: {
      email: 'demo@example.com',
      passwordHash: await hash('demo123'),
      firstName: 'Demo',
      lastName: 'User'
    }
  });

  // Create demo board
  const board = await prisma.board.create({
    data: {
      name: 'Welcome Board',
      ownerId: user.id,
      columns: {
        create: [
          { name: 'To Do', position: 0 },
          { name: 'In Progress', position: 1 },
          { name: 'Done', position: 2 }
        ]
      }
    }
  });
}
```

---

## Document Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-21 | Technical Team | Initial technical specification |

