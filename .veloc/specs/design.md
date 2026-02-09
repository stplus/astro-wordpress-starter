# Design Document: Kino Project Management System

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Components and Interfaces](#components-and-interfaces)
4. [Data Models](#data-models)
5. [Error Handling](#error-handling)
6. [Testing Strategy](#testing-strategy)
7. [Security Considerations](#security-considerations)
8. [Environment Setup](#environment-setup)

---

## Overview

### System Description

Kino is a modern project management system designed for the AI-augmented development paradigm where engineering managers oversee small human teams (1-3 developers) working alongside AI agent fleets. The system provides real-time visibility into both human and AI activities, automates status collection, integrates with development tools, and generates comprehensive stakeholder reports.

### Key Design Principles

1. **Event-Driven Architecture**: Real-time updates from AI agents and external systems drive state changes through a centralized event bus
2. **Modular Component Design**: Loosely coupled components with well-defined interfaces enable independent scaling and maintenance
3. **Data-Driven Intelligence**: Accumulated metrics and patterns inform automated insights and recommendations
4. **Progressive Enhancement**: Core functionality works without external integrations, with enhanced features when connected
5. **Audit-First Approach**: Every state change is logged for transparency, debugging, and compliance

### Architectural Approach

The system adopts a **three-tier architecture** with clear separation of concerns:

- **Presentation Layer**: Web-based UI with responsive design for desktop and mobile access
- **Application Layer**: RESTful API services handling business logic, orchestration, and external integrations
- **Data Layer**: Relational database for structured data with time-series optimization for metrics

Key architectural decisions:

- **Webhook-based integration** over polling for real-time AI agent updates (reduced latency, lower resource consumption)
- **Scheduled task workers** for email digests and report generation (decoupled from web requests, scalable)
- **OAuth delegation** for GitHub integration (secure, standard, user-controlled permissions)
- **Template-driven report generation** (flexible, maintainable, user-customizable)

---

## Architecture

### System Architecture Diagram

```mermaid
graph TD
    subgraph "Client Layer"
        WEB[Web Application]
        MOBILE[Mobile Browser]
    end
    
    subgraph "API Gateway"
        GATEWAY[API Gateway/Load Balancer]
    end
    
    subgraph "Application Layer"
        API[REST API Service]
        WEBHOOK[Webhook Receiver Service]
        SCHEDULER[Task Scheduler Service]
        REPORTER[Report Generator Service]
    end
    
    subgraph "Integration Layer"
        EMAIL[Email Service]
        GITHUB[GitHub Integration]
    end
    
    subgraph "Data Layer"
        DB[(PostgreSQL Database)]
        CACHE[(Redis Cache)]
        STORAGE[Object Storage]
    end
    
    subgraph "External Systems"
        AI_AGENTS[AI Agent Fleet]
        GITHUB_API[GitHub API]
        SMTP[SMTP Server]
    end
    
    WEB --> GATEWAY
    MOBILE --> GATEWAY
    GATEWAY --> API
    GATEWAY --> WEBHOOK
    
    API --> DB
    API --> CACHE
    API --> EMAIL
    API --> GITHUB
    
    WEBHOOK --> DB
    WEBHOOK --> CACHE
    
    SCHEDULER --> API
    SCHEDULER --> EMAIL
    SCHEDULER --> REPORTER
    
    REPORTER --> DB
    REPORTER --> STORAGE
    
    EMAIL --> SMTP
    GITHUB --> GITHUB_API
    
    AI_AGENTS -.webhook.-> WEBHOOK
    GITHUB_API -.webhook.-> WEBHOOK
```

### Data Flow: Real-Time AI Agent Update

```mermaid
sequenceDiagram
    participant Agent as AI Agent
    participant Webhook as Webhook Service
    participant EventBus as Event Bus
    participant API as API Service
    participant DB as Database
    participant Cache as Redis Cache
    participant UI as Web UI
    
    Agent->>Webhook: POST /webhooks/agent-update
    Webhook->>Webhook: Authenticate token
    Webhook->>Webhook: Validate payload
    Webhook->>EventBus: Publish AgentUpdateEvent
    EventBus->>API: Handle event
    API->>DB: Store agent activity
    API->>Cache: Invalidate project cache
    API->>EventBus: Publish ProjectUpdatedEvent
    EventBus->>UI: WebSocket notification
    UI->>UI: Refresh dashboard
    Webhook-->>Agent: 200 OK
```

### Component Interaction Model

The architecture follows an **event-driven pattern** where:

1. **External events** (webhooks, scheduled tasks) trigger state changes
2. **Core services** process events and update persistent state
3. **Cache layer** optimizes read-heavy operations (dashboards, reports)
4. **Notification system** pushes real-time updates to connected clients
5. **Background workers** handle asynchronous operations (emails, reports)

---

## Components and Interfaces

### Core Components

#### 1. API Service

**Responsibilities:**
- Handle HTTP requests from web clients
- Enforce authentication and authorization
- Orchestrate business logic operations
- Coordinate with external integrations
- Emit domain events for state changes

**Key Interfaces:**

```typescript
interface ProjectService {
  createProject(data: ProjectCreateRequest): Promise<Project>
  updateProject(id: string, data: ProjectUpdateRequest): Promise<Project>
  getProject(id: string): Promise<Project>
  listProjects(filters: ProjectFilters): Promise<ProjectList>
  deleteProject(id: string): Promise<void>
}

interface ScopeService {
  addScopeItem(projectId: string, item: ScopeItemRequest): Promise<ScopeItem>
  updateScopeItem(id: string, item: ScopeItemRequest): Promise<ScopeItem>
  reorderScopeItems(projectId: string, order: string[]): Promise<void>
  getScopeItems(projectId: string): Promise<ScopeItem[]>
}

interface MetricsService {
  getProjectMetrics(projectId: string, period: DateRange): Promise<ProjectMetrics>
  getDeveloperMetrics(userId: string, period: DateRange): Promise<DeveloperMetrics>
  getAIAgentMetrics(agentId: string, period: DateRange): Promise<AgentMetrics>
}
```

#### 2. Webhook Receiver Service

**Responsibilities:**
- Accept webhook payloads from AI agents and GitHub
- Authenticate and validate incoming requests
- Parse and normalize event data
- Publish events to internal event bus
- Handle webhook retry logic

**Key Interfaces:**

```typescript
interface WebhookHandler {
  authenticate(request: WebhookRequest): Promise<boolean>
  validate(payload: unknown): ValidationResult
  process(event: WebhookEvent): Promise<void>
}

interface AgentWebhookPayload {
  agent_id: string
  event_type: 'task_started' | 'code_commit' | 'test_execution' | 
               'task_completed' | 'blocker_identified'
  timestamp: string
  task_id: string
  data: Record<string, unknown>
}

interface GitHubWebhookPayload {
  action: string
  pull_request?: PullRequest
  repository: Repository
  sender: User
}
```

#### 3. Task Scheduler Service

**Responsibilities:**
- Execute scheduled jobs (email digests, auto-reports)
- Manage job queues and priorities
- Handle job failures and retries
- Track execution history

**Key Interfaces:**

```typescript
interface SchedulerService {
  scheduleJob(job: JobDefinition): Promise<string>
  cancelJob(jobId: string): Promise<void>
  getJobStatus(jobId: string): Promise<JobStatus>
}

interface JobDefinition {
  name: string
  schedule: CronExpression
  handler: string
  parameters: Record<string, unknown>
  retryPolicy: RetryPolicy
}
```

#### 4. Email Service

**Responsibilities:**
- Send transactional and digest emails
- Render email templates with dynamic content
- Track delivery status and bounces
- Handle retry logic for failed sends

**Key Interfaces:**

```typescript
interface EmailService {
  sendDigestEmail(recipients: string[], project: Project): Promise<EmailResult>
  sendReminderEmail(recipient: string, project: Project): Promise<EmailResult>
  sendReportEmail(recipients: EmailRecipients, report: Report): Promise<EmailResult>
  trackDelivery(emailId: string): Promise<DeliveryStatus>
}

interface EmailTemplate {
  subject: string
  htmlBody: string
  textBody: string
  variables: Record<string, string>
}
```

#### 5. Report Generator Service

**Responsibilities:**
- Compile data for newsletters and QBRs
- Render reports using templates
- Generate visualizations (charts, graphs)
- Export to multiple formats (HTML, PDF, PPTX)
- Store generated reports

**Key Interfaces:**

```typescript
interface ReportGenerator {
  generateNewsletter(config: NewsletterConfig): Promise<Report>
  generateQBR(config: QBRConfig): Promise<Report>
  generateCustomReport(template: ReportTemplate, data: ReportData): Promise<Report>
  exportReport(reportId: string, format: ExportFormat): Promise<File>
}

interface ReportTemplate {
  name: string
  sections: ReportSection[]
  styling: StyleConfig
  dataRequirements: DataRequirement[]
}
```

#### 6. GitHub Integration Service

**Responsibilities:**
- Manage OAuth authentication flow
- Subscribe to repository webhooks
- Fetch PR and commit data
- Extract metrics from code analysis tools
- Handle API rate limiting

**Key Interfaces:**

```typescript
interface GitHubService {
  authenticate(userId: string): Promise<OAuthToken>
  connectRepository(projectId: string, repoUrl: string): Promise<void>
  fetchPullRequest(repoId: string, prNumber: number): Promise<PullRequestData>
  fetchMetrics(repoId: string, period: DateRange): Promise<RepositoryMetrics>
  handleRateLimit(): Promise<void>
}

interface RepositoryMetrics {
  averageCoverage: number
  securityIssues: SecurityIssue[]
  staticAnalysisIssues: AnalysisIssue[]
  averageMergeTime: number
  totalCommits: number
  totalPRs: number
}
```

### Component Communication Patterns

**Synchronous Communication:**
- Client ↔ API Service: RESTful HTTP/HTTPS
- API Service ↔ GitHub Integration: HTTP with OAuth

**Asynchronous Communication:**
- Event Bus: Publish/Subscribe for domain events
- WebSocket: Real-time client updates
- Message Queue: Background job processing

**Data Access:**
- Direct database access for write operations
- Cache-aside pattern for frequently accessed data
- Read replicas for reporting queries

---

## Data Models

### Core Entity Relationships

```mermaid
erDiagram
    INITIATIVE ||--o{ PROJECT : contains
    PROJECT ||--o{ SCOPE_ITEM : has
    PROJECT ||--o{ PROJECT_UPDATE : receives
    PROJECT ||--o{ PROJECT_MEMBER : assigned_to
    PROJECT ||--o{ AGENT_ACTIVITY : tracked_in
    PROJECT ||--o{ GITHUB_REPOSITORY : connected_to
    
    USER ||--o{ PROJECT_MEMBER : participates_in
    USER ||--o{ PROJECT_UPDATE : submits
    
    GITHUB_REPOSITORY ||--o{ PULL_REQUEST : contains
    PULL_REQUEST ||--o{ PR_METRIC : has
    
    AGENT_ACTIVITY ||--o{ CODE_COMMIT : includes
    AGENT_ACTIVITY ||--o{ TEST_EXECUTION : includes
    
    PROJECT ||--o{ REPORT : generates

    INITIATIVE {
        uuid id PK
        string name
        string description
        timestamp created_at
    }
    
    PROJECT {
        uuid id PK
        uuid initiative_id FK
        string name
        string description
        date start_date
        date end_date
        enum priority
        enum status
        timestamp created_at
        timestamp updated_at
        uuid created_by FK
    }
    
    SCOPE_ITEM {
        uuid id PK
        uuid project_id FK
        string title
        string description
        enum priority
        date start_date
        date end_date
        string assignee_type
        string assignee_id
        int order_index
        enum status
    }
    
    PROJECT_MEMBER {
        uuid id PK
        uuid project_id FK
        uuid user_id FK
        enum role
        timestamp joined_at
    }
    
    AGENT_ACTIVITY {
        uuid id PK
        uuid project_id FK
        string agent_id
        string agent_type
        enum event_type
        uuid task_id FK
        jsonb event_data
        timestamp occurred_at
    }
    
    PROJECT_UPDATE {
        uuid id PK
        uuid project_id FK
        uuid submitted_by FK
        enum overall_status
        text achievements
        text challenges
        text planned_work
        text notes
        date period_start
        date period_end
        timestamp submitted_at
    }
    
    GITHUB_REPOSITORY {
        uuid id PK
        uuid project_id FK
        string repo_url
        string repo_full_name
        jsonb config
        timestamp connected_at
    }
    
    PULL_REQUEST {
        uuid id PK
        uuid repository_id FK
        int pr_number
        string title
        string author
        timestamp created_at
        timestamp merged_at
        int files_changed
        int lines_added
        int lines_deleted
    }
    
    PR_METRIC {
        uuid id PK
        uuid pull_request_id FK
        decimal coverage_percentage
        int security_issues_count
        int static_analysis_count
        jsonb detailed_metrics
    }
```

### Key Data Structures

#### Project Aggregate

```typescript
interface Project {
  id: string
  initiativeId: string
  name: string
  description: string
  startDate: Date
  endDate: Date
  priority: 'HIGH' | 'MEDIUM' | 'LOW'
  status: 'ON_TRACK' | 'AT_RISK' | 'BLOCKED' | 'COMPLETED'
  members: ProjectMember[]
  scopeItems: ScopeItem[]
  metadata: ProjectMetadata
  createdAt: Date
  updatedAt: Date
  createdBy: string
}

interface ProjectMetadata {
  progressPercentage: number
  lastActivityAt: Date
  lastUpdateAt: Date
  totalAIAgentActivities: number
  connectedRepositories: number
}
```

#### Agent Activity Event

```typescript
interface AgentActivity {
  id: string
  projectId: string
  agentId: string
  agentType: 'CLAUDE' | 'CODEX' | 'GEMINI' | 'OTHER'
  eventType: AgentEventType
  taskId: string
  eventData: AgentEventData
  occurredAt: Date
}

type AgentEventType = 
  | 'TASK_STARTED'
  | 'CODE_COMMIT'
  | 'TEST_EXECUTION'
  | 'TASK_COMPLETED'
  | 'BLOCKER_IDENTIFIED'

interface AgentEventData {
  // Flexible structure based on event type
  [key: string]: unknown
}
```

#### GitHub Metrics

```typescript
interface RepositoryMetrics {
  repositoryId: string
  periodStart: Date
  periodEnd: Date
  
  pullRequests: {
    total: number
    merged: number
    averageMergeTimeHours: number
  }
  
  codeQuality: {
    averageCoverage: number
    coverageTrend: TrendIndicator
  }
  
  security: {
    criticalIssues: number
    highIssues: number
    mediumIssues: number
    lowIssues: number
  }
  
  staticAnalysis: {
    criticalIssues: number
    highIssues: number
    mediumIssues: number
    lowIssues: number
  }
  
  developerMetrics: DeveloperMetric[]
}

interface DeveloperMetric {
  userId: string
  prsCreated: number
  prsMerged: number
  averageCoverage: number
  issuesIntroduced: number
  issuesFixed: number
  reviewsProvided: number
  trend: TrendIndicator
}
```

### Data Storage Strategy

**Relational Database (PostgreSQL):**
- Primary storage for all entities
- ACID compliance for project state changes
- Foreign key constraints for referential integrity
- Indexes on frequently queried fields (project_id, user_id, dates)
- Partitioning for agent_activity table by date (monthly partitions)

**Cache Layer (Redis):**
- Project dashboard data (TTL: 5 minutes)
- User session data
- Rate limiting counters
- Webhook processing locks (distributed locking)

**Object Storage (S3-compatible):**
- Generated report files (PDF, PPTX)
- Email attachments
- Uploaded project artifacts
- 90-day retention with lifecycle policies

### Audit Trail

All state-changing operations create audit log entries:

```typescript
interface AuditLog {
  id: string
  entityType: string
  entityId: string
  action: 'CREATE' | 'UPDATE' | 'DELETE'
  userId: string
  timestamp: Date
  previousState: Record<string, unknown>
  newState: Record<string, unknown>
  metadata: Record<string, unknown>
}
```

---

## Error Handling

### Error Handling Strategy

The system implements a **layered error handling approach** with clear boundaries:

1. **Input Validation Layer**: Catch and reject malformed requests at API gateway
2. **Business Logic Layer**: Handle domain-specific errors with context
3. **Integration Layer**: Manage external service failures with retry logic
4. **Data Layer**: Handle database constraints and transaction failures

### Error Classification

```typescript
enum ErrorCategory {
  VALIDATION = 'VALIDATION',           // 400 Bad Request
  AUTHENTICATION = 'AUTHENTICATION',   // 401 Unauthorized
  AUTHORIZATION = 'AUTHORIZATION',     // 403 Forbidden
  NOT_FOUND = 'NOT_FOUND',            // 404 Not Found
  CONFLICT = 'CONFLICT',              // 409 Conflict
  RATE_LIMIT = 'RATE_LIMIT',          // 429 Too Many Requests
  INTEGRATION = 'INTEGRATION',         // 502 Bad Gateway
  SERVER = 'SERVER'                    // 500 Internal Server Error
}

interface ApplicationError {
  category: ErrorCategory
  code: string
  message: string
  details?: Record<string, unknown>
  timestamp: Date
  requestId: string
  retryable: boolean
}
```

### Error Handling Patterns

**Validation Errors:**
- Validate at API boundary before processing
- Return structured error messages with field-level details
- Example: `{ field: 'endDate', message: 'End date must be after start date' }`

**Webhook Processing:**
- Return immediate HTTP response (200/400/401/500)
- Process payload asynchronously after acknowledgment
- Store failed webhooks in dead-letter queue for retry
- Alert on repeated failures from same source

**External Integration Failures:**
- Implement circuit breaker pattern (fail fast after N failures)
- Exponential backoff for retries (1s, 2s, 4s, 8s, 16s)
- Degrade gracefully (e.g., show cached GitHub data if API unavailable)
- Log integration failures with full context for debugging

**Database Failures:**
- Wrap transactions with retry logic for deadlock scenarios
- Implement connection pooling with health checks
- Fall back to read replicas for query operations
- Maintain database connection circuit breaker

**Report Generation Failures:**
- Generate partial reports with missing data indicators
- Queue failed jobs for retry (up to 3 attempts)
- Notify users of failures with diagnostic information
- Store error context for troubleshooting

### Logging Strategy

```typescript
interface LogEntry {
  level: 'DEBUG' | 'INFO' | 'WARN' | 'ERROR' | 'FATAL'
  timestamp: Date
  requestId: string
  userId?: string
  component: string
  message: string
  error?: Error
  context: Record<string, unknown>
}
```

**Log Levels:**
- **DEBUG**: Development-time diagnostics (disabled in production)
- **INFO**: Normal operations (webhook received, report generated)
- **WARN**: Recoverable issues (rate limit approaching, slow query)
- **ERROR**: Operation failures (webhook processing failed, email bounced)
- **FATAL**: System-critical failures (database unavailable, service crash)

**Structured Logging:**
- All logs in JSON format for parsing
- Include request ID for tracing
- Add correlation IDs for multi-service operations
- Sensitive data (tokens, passwords) never logged

### User-Facing Error Messages

**Principles:**
- Clear, actionable language (avoid technical jargon)
- Provide guidance on resolution when possible
- Never expose internal implementation details
- Include support contact for unresolvable errors

**Examples:**

```typescript
const USER_MESSAGES = {
  PROJECT_NOT_FOUND: 'The requested project could not be found. It may have been deleted.',
  INVALID_DATE_RANGE: 'Please ensure the end date is after the start date.',
  GITHUB_AUTH_EXPIRED: 'Your GitHub connection has expired. Please reconnect in Settings.',
  WEBHOOK_RATE_LIMITED: 'Too many updates received. Please try again in a few minutes.',
  REPORT_GENERATION_FAILED: 'Report generation failed. Our team has been notified. Please try again or contact support.'
}
```

---

## Testing Strategy

### Testing Pyramid

```mermaid
graph TB
    subgraph "Testing Layers"
        E2E[End-to-End Tests<br/>10%]
        INTEGRATION[Integration Tests<br/>30%]
        UNIT[Unit Tests<br/>60%]
    end
    
    E2E --> INTEGRATION
    INTEGRATION --> UNIT
    
    style E2E fill:#ff6b6b
    style INTEGRATION fill:#4ecdc4
    style UNIT fill:#95e1d3
```

### Unit Testing

**Scope:** Individual functions, classes, and modules in isolation

**Key Test Scenarios:**

1. **Project Service:**
   - Project creation with valid data
   - Validation failures (missing required fields, invalid dates)
   - Update operations with audit trail generation
   - Concurrent update conflict handling
   - Scope item reordering logic

2. **Webhook Handler:**
   - Token authentication success/failure
   - Payload validation for each event type
   - Malformed JSON handling
   - Event parsing and normalization
   - Rate limiting enforcement

3. **Metrics Calculator:**
   - Coverage trend calculations
   - Aggregate statistics computation
   - Date range filtering
   - Missing data handling
   - Percentile calculations

4. **Report Generator:**
   - Template rendering with valid data
   - Missing data handling (partial reports)
   - Chart generation from datasets
   - Export format conversions
   - Template variable substitution

**Testing Tools:**
- Jest/Vitest for JavaScript/TypeScript
- pytest for Python components
- Mock external dependencies (database, APIs, time)
- Snapshot testing for rendered templates

### Integration Testing

**Scope:** Component interactions, database operations, external service calls

**Key Test Scenarios:**

1. **API to Database:**
   - Complete project CRUD operations
   - Transaction rollback on error
   - Audit log creation
   - Query performance with realistic data volumes
   - Concurrent access patterns

2. **Webhook Processing Pipeline:**
   - Webhook receipt → Event bus → Database → Cache invalidation
   - Duplicate webhook detection
   - Out-of-order event handling
   - Dead-letter queue processing

3. **GitHub Integration:**
   - OAuth flow completion
   - Webhook subscription setup
   - PR data extraction
   - Metric calculation from raw data
   - Rate limit handling with retry

4. **Email Service:**
   - Template rendering with project data
   - Multi-recipient delivery
   - Bounce handling
   - Retry logic for failed sends

5. **Scheduler to Report Generator:**
   - Scheduled job execution
   - Report data compilation
   - Multi-format export
   - Storage and archival

**Testing Approach:**
- Use test database (Docker container)
- Mock external HTTP APIs with tools like WireMock
- Test environment variables and configuration
- Seed database with representative test data
- Test both success and failure paths

### End-to-End Testing

**Scope:** Complete user workflows through the UI

**Critical User Journeys:**

1. **Project Lifecycle:**
   - Create project with full metadata
   - Add team members and scope items
   - Receive AI agent updates via webhook
   - View real-time dashboard updates
   - Submit manual update via email link
   - Generate and download report

2. **GitHub Integration:**
   - Connect GitHub account
   - Link repository to project
   - Receive PR webhook
   - View metrics in dashboard
   - Identify developer needing support

3. **Report Generation:**
   - Configure monthly newsletter
   - Trigger newsletter generation
   - Preview report content
   - Email to stakeholders
   - Download archived report

**Testing Tools:**
- Playwright or Cypress for browser automation
- Test against staging environment
- Run on multiple browsers (Chrome, Firefox, Safari)
- Mobile viewport testing

### Performance Testing

**Key Scenarios:**

1. **Webhook Load:**
   - Simulate 100 concurrent webhook requests
   - Verify <2s processing time
   - Verify <5s dashboard update propagation
   - No data loss under load

2. **Dashboard Query Performance:**
   - Load project list with 1000+ projects
   - Gantt view rendering with overlapping projects
   - Metrics aggregation queries
   - Target: <1s page load time

3. **Report Generation:**
   - QBR generation with 50+ projects
   - Chart rendering performance
   - PDF export time
   - Target: <30s for complex reports

**Tools:**
- k6 or JMeter for load testing
- Database query profiling (EXPLAIN ANALYZE)
- Application Performance Monitoring (APM) tools

### Test Data Management

**Strategy:**
- Factories for generating test entities
- Seed data scripts for integration tests
- Anonymized production data for staging
- Separate test databases per environment
- Automated cleanup after test runs

**Example Test Factory:**

```typescript
class ProjectFactory {
  static create(overrides?: Partial<Project>): Project {
    return {
      id: uuid(),
      name: faker.commerce.productName(),
      initiativeId: uuid(),
      startDate: new Date(),
      endDate: addMonths(new Date(), 3),
      priority: 'MEDIUM',
      status: 'ON_TRACK',
      ...overrides
    }
  }
}
```

### Continuous Testing

**CI/CD Integration:**
- Run unit tests on every commit
- Run integration tests on pull requests
- Run E2E tests on staging deployment
- Performance tests on scheduled basis (weekly)
- Block merges if tests fail

**Code Coverage Targets:**
- Unit test coverage: >80%
- Integration test coverage: >60%
- Overall coverage: >75%
- Track coverage trends over time

---

## Security Considerations

### Authentication and Authorization

**Authentication Strategy:**

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant Gateway
    participant Auth
    participant API
    
    User->>Client: Login with credentials
    Client->>Auth: POST /auth/login
    Auth->>Auth: Verify credentials
    Auth->>Auth: Generate JWT
    Auth-->>Client: Return JWT + refresh token
    Client->>Client: Store tokens securely
    
    Client->>Gateway: API request + JWT
    Gateway->>Gateway: Verify JWT signature
    Gateway->>Gateway: Check expiration
    Gateway->>API: Forward with user context
    API->>API: Check authorization
    API-->>Client: Response
```

**Implementation Details:**

1. **User Authentication:**
   - JWT-based authentication (access + refresh tokens)
   - Access token lifetime: 15 minutes
   - Refresh token lifetime: 7 days
   - Tokens signed with RS256 (asymmetric keys)
   - Secure token storage (httpOnly cookies or secure localStorage)

2. **API Token Authentication:**
   - API tokens for webhook endpoints
   - Format: `kino_<environment>_<random_32_chars>`
   - Stored hashed (SHA-256) in database
   - Rotation capability per project
   - Rate limiting per token

3. **OAuth Integration:**
   - GitHub OAuth 2.0 for repository access
   - Minimal scope requests (read:repo, write:webhook)
   - Token refresh handling
   - Token revocation on disconnect

**Authorization Model:**

Role-Based Access Control (RBAC) with hierarchical permissions:

```typescript
enum Role {
  ADMIN = 'ADMIN',               // Full system access
  MANAGER = 'MANAGER',           // Manage all projects and users
  PROJECT_LEADER = 'PROJECT_LEADER', // Manage assigned projects
  MEMBER = 'MEMBER'              // View projects, submit updates
}

interface Permission {
  resource: string  // e.g., 'project', 'report', 'user'
  action: string    // e.g., 'create', 'read', 'update', 'delete'
  scope: string     // e.g., 'own', 'team', 'all'
}

const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  ADMIN: [
    { resource: '*', action: '*', scope: 'all' }
  ],
  MANAGER: [
    { resource: 'project', action: '*', scope: 'all' },
    { resource: 'report', action: '*', scope: 'all' },
    { resource: 'user', action: 'read', scope: 'all' }
  ],
  PROJECT_LEADER: [
    { resource: 'project', action: '*', scope: 'own' },
    { resource: 'report', action: 'create', scope: 'own' },
    { resource: 'user', action: 'read', scope: 'team' }
  ],
  MEMBER: [
    { resource: 'project', action: 'read', scope: 'own' },
    { resource: 'update', action: 'create', scope: 'own' }
  ]
}
```

### Input Validation

**Validation Layers:**

1. **Client-Side Validation:**
   - Immediate feedback for user experience
   - Format validation (dates, emails, URLs)
   - Required field checks
   - Never rely on client validation alone

2. **API Gateway Validation:**
   - Schema validation for all incoming requests
   - Content-Type verification
   - Request size limits (10MB max)
   - Rate limiting enforcement

3. **Business Logic Validation:**
   - Domain rule enforcement
   - Cross-field validation
   - Database constraint checks
   - Authorization checks

**Input Sanitization:**

```typescript
interface ValidationRules {
  // Project name: alphanumeric, spaces, hyphens, 3-100 chars
  projectName: /^[a-zA-Z0-9\s\-]{3,100}$/
  
  // Email: RFC 5322 compliant
  email: /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  
  // URLs: HTTPS only for production
  url: /^https:\/\/.+/
  
  // SQL injection prevention: parameterized queries only
  // XSS prevention: escape HTML in user-generated content
  // Command injection: never pass user input to shell
}
```

**Webhook Payload Validation:**

- Verify Content-Type header (application/json)
- Validate JSON structure against schema
- Check signature/HMAC if provided
- Reject payloads >1MB
- Log malformed requests for security analysis

### Data Protection

**Encryption:**

1. **Data in Transit:**
   - TLS 1.3 for all HTTPS connections
   - Certificate pinning for mobile apps
   - HSTS headers for web application

2. **Data at Rest:**
   - Database encryption (AES-256)
   - Encrypted backups
   - API tokens hashed before storage
   - OAuth tokens encrypted in database

3. **Sensitive Data Handling:**
   - Never log passwords, tokens, or API keys
   - Mask sensitive data in error messages
   - Redact personal information in audit logs
   - PII encryption where applicable

**Data Privacy:**

- Audit log retention: 1 year
- User data deletion on account removal
- GDPR compliance for user data access/export
- Data minimization (collect only necessary data)

### Security Headers

```typescript
const SECURITY_HEADERS = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Content-Security-Policy': "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'",
  'Referrer-Policy': 'strict-origin-when-cross-origin'
}
```

### Rate Limiting

**Tiered Rate Limits:**

```typescript
const RATE_LIMITS = {
  // User API requests
  api: {
    perMinute: 100,
    perHour: 1000,
    perDay: 10000
  },
  
  // Webhook endpoints
  webhook: {
    perMinute: 200,
    perHour: 5000
  },
  
  // Authentication attempts
  auth: {
    perMinute: 5,
    perHour: 20,
    lockoutDuration: 900 // 15 minutes
  },
  
  // Email sending
  email: {
    perMinute: 10,
    perHour: 100
  }
}
```

**Implementation:**
- Redis-based rate limiting with sliding window
- Return 429 status with Retry-After header
- Exponential backoff for repeated violations
- Alert on suspicious patterns (potential DDoS)

### Dependency Security

**Practices:**
- Automated dependency scanning (npm audit, Snyk)
- Regular updates for security patches
- Pin major versions, allow patch updates
- Review licenses for compliance
- Minimize dependency count

### Incident Response

**Security Monitoring:**
- Log all authentication failures
- Alert on repeated failed login attempts
- Monitor for SQL injection patterns
- Track unusual API usage patterns
- Real-time security event dashboard

**Response Procedures:**
1. Detect: Automated alerts on security events
2. Analyze: Review logs and determine severity
3. Contain: Disable compromised accounts/tokens
4. Remediate: Apply patches, rotate credentials
5. Document: Post-mortem and lessons learned

---

## Environment Setup

### Technology Stack

**Backend:**
- **Runtime**: Node.js 20.x LTS (JavaScript/TypeScript)
- **Framework**: Express.js or Fastify
- **Language**: TypeScript 5.x
- **Database**: PostgreSQL 15.x
- **Cache**: Redis 7.x
- **Object Storage**: AWS S3 or MinIO (S3-compatible)

**Frontend:**
- **Framework**: React 18.x or Vue 3.x
- **Language**: TypeScript 5.x
- **Build Tool**: Vite or Next.js
- **UI Components**: Tailwind CSS + shadcn/ui or Vuetify
- **Charts**: Recharts or Chart.js
- **Real-time**: Socket.io-client

**Infrastructure:**
- **Container Runtime**: Docker 24.x
- **Orchestration**: Docker Compose (dev), Kubernetes (prod)
- **Reverse Proxy**: Nginx or Traefik
- **Process Manager**: PM2 (if not containerized)

**DevOps:**
- **Version Control**: Git
- **CI/CD**: GitHub Actions or GitLab CI
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack or Loki
- **Error Tracking**: Sentry

### Project Structure

```
kino/
├── backend/
│   ├── src/
│   │   ├── api/                  # API route handlers
│   │   │   ├── projects/
│   │   │   ├── webhooks/
│   │   │   ├── reports/
│   │   │   └── auth/
│   │   ├── services/             # Business logic
│   │   │   ├── project.service.ts
│   │   │   ├── metrics.service.ts
│   │   │   ├── email.service.ts
│   │   │   └── github.service.ts
│   │   ├── models/               # Data models and schemas
│   │   ├── middleware/           # Express middleware
│   │   ├── utils/                # Utility functions
│   │   ├── config/               # Configuration management
│   │   ├── workers/              # Background job workers
│   │   └── app.ts                # Application entry point
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   └── fixtures/
│   ├── migrations/               # Database migrations
│   ├── package.json
│   ├── tsconfig.json
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── components/           # Reusable UI components
│   │   ├── pages/                # Page components
│   │   ├── hooks/                # Custom React hooks
│   │   ├── services/             # API client services
│   │   ├── store/                # State management
│   │   ├── utils/                # Utility functions
│   │   └── main.tsx              # Application entry point
│   ├── public/
│   ├── tests/
│   ├── package.json
│   ├── tsconfig.json
│   └── Dockerfile
│
├── shared/                       # Shared types and utilities
│   ├── types/
│   └── constants/
│
├── docs/                         # Documentation
│   ├── api/                      # API documentation
│   ├── architecture/             # Architecture diagrams
│   └── deployment/               # Deployment guides
│
├── infrastructure/               # Infrastructure as code
│   ├── docker-compose.yml
│   ├── kubernetes/
│   └── terraform/
│
├── scripts/                      # Utility scripts
│   ├── seed-db.ts
│   ├── migrate.ts
│   └── generate-types.ts
│
├── .github/
│   └── workflows/                # CI/CD workflows
│
├── .env.example                  # Environment variable template
├── README.md
└── package.json                  # Root workspace configuration
```

### Configuration Management

**Environment Variables:**

```bash
# Application
NODE_ENV=development|staging|production
PORT=3000
API_BASE_URL=http://localhost:3000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/kino
DATABASE_POOL_MIN=2
DATABASE_POOL_MAX=10

# Redis
REDIS_URL=redis://localhost:6379
REDIS_TTL=300

# Authentication
JWT_SECRET=<generated-secret>
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# GitHub Integration
GITHUB_CLIENT_ID=<github-oauth-app-id>
GITHUB_CLIENT_SECRET=<github-oauth-secret>
GITHUB_WEBHOOK_SECRET=<webhook-secret>

# Email
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=<smtp-username>
SMTP_PASSWORD=<smtp-password>
EMAIL_FROM=noreply@kino.example.com

# Object Storage
S3_ENDPOINT=https://s3.amazonaws.com
S3_BUCKET=kino-reports
S3_ACCESS_KEY=<access-key>
S3_SECRET_KEY=<secret-key>

# Monitoring
SENTRY_DSN=<sentry-dsn>
LOG_LEVEL=info

# Feature Flags
ENABLE_GITHUB_INTEGRATION=true
ENABLE_EMAIL_REMINDERS=true
```

**Configuration Files:**

```typescript
// config/index.ts
export const config = {
  app: {
    env: process.env.NODE_ENV || 'development',
    port: parseInt(process.env.PORT || '3000', 10),
    apiBaseUrl: process.env.API_BASE_URL
  },
  
  database: {
    url: process.env.DATABASE_URL,
    pool: {
      min: parseInt(process.env.DATABASE_POOL_MIN || '2', 10),
      max: parseInt(process.env.DATABASE_POOL_MAX || '10', 10)
    }
  },
  
  auth: {
    jwtSecret: process.env.JWT_SECRET,
    accessTokenExpiry: process.env.JWT_ACCESS_EXPIRY || '15m',
    refreshTokenExpiry: process.env.JWT_REFRESH_EXPIRY || '7d'
  },
  
  github: {
    clientId: process.env.GITHUB_CLIENT_ID,
    clientSecret: process.env.GITHUB_CLIENT_SECRET,
    webhookSecret: process.env.GITHUB_WEBHOOK_SECRET
  },
  
  email: {
    smtp: {
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT || '587', 10),
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASSWORD
      }
    },
    from: process.env.EMAIL_FROM
  },
  
  storage: {
    endpoint: process.env.S3_ENDPOINT,
    bucket: process.env.S3_BUCKET,
    credentials: {
      accessKeyId: process.env.S3_ACCESS_KEY,
      secretAccessKey: process.env.S3_SECRET_KEY
    }
  }
}
```

### Development Setup

**Prerequisites:**
- Node.js 20.x LTS
- Docker Desktop or Docker Engine + Docker Compose
- Git
- PostgreSQL client (psql) for debugging
- Redis client (redis-cli) for debugging

**Initial Setup:**

```bash
# Clone repository
git clone https://github.com/your-org/kino.git
cd kino

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env
# Edit .env with your local configuration

# Start infrastructure services
docker-compose up -d postgres redis

# Run database migrations
npm run migrate

# Seed development data
npm run seed

# Start development servers
npm run dev
# Backend: http://localhost:3000
# Frontend: http://localhost:5173
```

**Database Migrations:**

```bash
# Create new migration
npm run migration:create -- --name add_github_metrics

# Run pending migrations
npm run migrate

# Rollback last migration
npm run migrate:rollback

# Reset database (drop all + migrate)
npm run migrate:reset
```

### Docker Compose Setup

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: kino
      POSTGRES_USER: kino
      POSTGRES_PASSWORD: kino_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kino"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://kino:kino_dev@postgres:5432/kino
      REDIS_URL: redis://redis:6379
    volumes:
      - ./backend:/app
      - /app/node_modules
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: npm run dev

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "5173:5173"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      VITE_API_URL: http://localhost:3000
    command: npm run dev

volumes:
  postgres_data:
  redis_data:
```

### Deployment Considerations

**Production Deployment Checklist:**

- [ ] Environment variables configured (secrets management)
- [ ] Database migrations applied
- [ ] SSL/TLS certificates provisioned
- [ ] Object storage bucket created
- [ ] Email service configured and tested
- [ ] GitHub OAuth app created (production credentials)
- [ ] Monitoring and alerting configured
- [ ] Backup strategy implemented
- [ ] Load balancer/reverse proxy configured
- [ ] CDN configured for static assets
- [ ] Rate limiting rules active
- [ ] Security headers configured
- [ ] CORS policies set
- [ ] Log aggregation configured

**Scaling Strategy:**

- **Horizontal Scaling**: Multiple API server instances behind load balancer
- **Database**: Read replicas for reporting queries, connection pooling
- **Cache**: Redis cluster for high availability
- **Background Jobs**: Multiple worker instances processing job queue
- **Static Assets**: CDN for frontend and uploaded files
- **WebSocket**: Sticky sessions or shared state (Redis adapter)

**Monitoring Metrics:**

- Application: Request rate, error rate, response time (p50, p95, p99)
- Database: Query performance, connection pool usage, replication lag
- Cache: Hit rate, eviction rate, memory usage
- Workers: Job processing rate, queue depth, failed jobs
- Infrastructure: CPU, memory, disk, network usage
- Business: Active users, projects created, reports generated, webhook events

---

**End of Design Document**