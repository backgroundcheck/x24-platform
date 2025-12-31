# Architecture Guide

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      X24 Platform Monorepo                      │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────────┐      ┌──────────────────────────────┐
│     X24 Portal (Frontend)    │      │    OpenCTI (Backend)         │
├──────────────────────────────┤      ├──────────────────────────────┤
│ React 19 + TypeScript        │      │ GraphQL API + PostgreSQL     │
│ Vite build (7.5s)            │      │ Elasticsearch + RabbitMQ     │
│                              │      │ MinIO S3 + Workers           │
│ 26+ Intelligence Widgets     │      │ 20+ Automated Connectors     │
│ Real-time Gemini AI          │      │ Multi-tenant CTI             │
│ 2,300+ LOC + 400+ KB docs    │      │ Threat feeds + Ingestion     │
└──────────────────────────────┘      └──────────────────────────────┘
         ↓ HTTP/GraphQL                        ↓ REST/GraphQL
    Port 3002 (dev)                      Port 8080, 4000 (prod)
         │                                      │
         └──────────────────┬───────────────────┘
                            │
                  Shared Intelligence
                  (Risk Scoring, Matching)
```

## X24 Portal Architecture

### Directory Structure

```
x24-portal/
├── components/                 # React components
│   ├── Dashboard.tsx           # Main grid layout
│   ├── *Widget.tsx             # 26+ widget implementations
│   ├── InvestigationView.tsx   # Entity deep-dive modal
│   └── src/
│       └── lib/
│           ├── ai/
│           │   ├── gemini.ts           # Google Gemini wrappers
│           │   └── conversation.ts     # Multi-turn chat
│           ├── schemas.ts              # Zod schemas (single source of truth)
│           ├── connectors.ts           # 20+ threat intel APIs
│           ├── connectors/
│           │   ├── opencti.ts          # OpenCTI with circuit breaker
│           │   ├── sanctions-matcher.ts # Fuzzy matching engine (450+ LOC)
│           │   └── [other-connectors]
│           ├── errorHandling.ts        # Classify + retry + recovery
│           ├── utils.ts                # Common utilities
│           └── tracing.ts              # OpenTelemetry instrumentation
├── App.tsx                     # Root component
├── index.tsx                   # Entry point
├── index.html                  # HTML shell
├── vite.config.ts             # Vite configuration
├── tsconfig.json              # TypeScript config
├── package.json               # Dependencies + scripts
└── [other config files]
```

### Data Flow

```
User Input (Dashboard)
    ↓
handleNewInvestigation()
    ↓
InvestigationEntity created
    ↓
InvestigationView opens modal
    ↓
Entity type → determine connectors
    ↓
Parallel API queries (connectors)
    ↓
Gemini enrichment (runGroundedAnalysis)
    ↓
Risk aggregation (aggregateEntityRisk)
    ↓
Display results (header + tabs + graph)
    ↓
Export to OpenCTI (optional)
```

### Key Patterns

#### 1. Zod Schemas First
```typescript
// All data structures defined in schemas.ts
export const SanctionsMatchSchema = z.object({
  id: z.string(),
  matchScore: z.number().min(0).max(1),
  // ...
});
type SanctionsMatch = z.infer<typeof SanctionsMatchSchema>;
```

#### 2. Gemini API Wrappers
```typescript
// Never call @google/genai directly
// Use wrappers from gemini.ts
const result = await runJson({
  prompt,
  schema: MySchema,
  responseSchema: geminiSchema
});
```

#### 3. Widget Registration
```typescript
// Step 1: Create MyWidget.tsx implementing BaseWidgetProps
// Step 2: Add to AVAILABLE_WIDGETS array in Dashboard.tsx
// Step 3: Add to widgetComponents render switch
```

#### 4. Error Handling
```typescript
// Use withRetry for external APIs
const data = await withRetry(() => fetchData(), 3, 1000, 2);

// Use retryWithBackoff for critical operations
const result = await retryWithBackoff(() => apiCall());

// Classify errors for recovery
const classified = classifyError(error);
const recovery = getRecoveryStrategy(classified);
```

### Build Pipeline

```
npm run dev
  └─ Vite dev server (port 3002)
     └─ HMR (hot module reload)
     └─ TypeScript checking
     └─ Source maps for debugging

npm run build
  └─ Vite production build
     └─ Minification + tree-shaking
     └─ Code splitting (5 chunks)
     └─ ~380KB gzip (includes Gemini, all connectors)
     └─ Build time: 7.5s

npm run test
  └─ Vitest watch mode
     └─ jsdom environment
     └─ Coverage reports
```

## OpenCTI Architecture

### Microservices Stack

```
OpenCTI Platform
├── Frontend UI (React)
├── GraphQL API
├── Platform backend (Node.js)
├── Elasticsearch (search + analytics)
├── PostgreSQL (relational data)
├── RabbitMQ (async processing)
├── Redis (caching + sessions)
├── MinIO S3 (object storage)
└── 3 Worker instances (parallel processing)
```

### Data Model

```
Platform Layer
  └─ Objects (entities)
     ├─ Threat Actors
     ├─ Malware
     ├─ Tools
     ├─ Campaigns
     ├─ Incidents
     └─ Indicators
          ├─ IPv4, IPv6 (network)
          ├─ Domain-Name
          ├─ Url
          ├─ File (hashes)
          └─ Process
  └─ Relations (relationships)
     ├─ uses (threat actor uses malware)
     ├─ targets (campaign targets sector)
     ├─ exploits (tool exploits vulnerability)
     └─ [50+ relation types]
```

### Connectors

**Ingest Connectors** (pull threat data):
- MITRE ATT&CK
- CVE (NVD)
- MalwareBazaar
- URLhaus
- ThreatFox
- Malpedia
- CISA Known Exploited Vulnerabilities
- Alienvault OTX
- And 12+ more

**Export Connectors** (push to external systems):
- CSV export
- STIX export
- Text export

## Integration Points

### X24 Portal ↔ OpenCTI

```
X24 Portal                    OpenCTI
    │                            │
    ├─ Query threat actors       │
    │  └─────────────────────────→ GraphQL /objects/threat-actors
    │
    ├─ Query malware families    │
    │  └─────────────────────────→ GraphQL /objects/malware
    │
    ├─ Create relationships       │
    │  └─────────────────────────→ GraphQL /relationships
    │
    └─ Export case intelligence  │
       └─────────────────────────→ GraphQL /reports
```

### Risk Aggregation Pipeline

```
Input Data (from connectors)
    ├─ Sanctions matches
    ├─ PEP records
    ├─ Criminal records
    ├─ FTM ownership chains
    └─ Threat actor links
         ↓
Risk Scoring (risk-aggregation.ts)
    ├─ Score each domain (0–1)
    ├─ Apply weights (0.35 sanctions, 0.20 alias, ...)
    ├─ Calculate composite score (0–100)
    ├─ Evaluate cross-domain triggers
    └─ Apply overrides (OFAC → critical)
         ↓
Header Display
    ├─ Risk badge (color + level)
    ├─ Risk score (0–100)
    ├─ Key triggers
    └─ Recommendation
         ↓
Export (SharedEntityRiskPayload)
    └─ To Compliant.one + BackCheck
```

## Scaling Considerations

### X24 Portal
- **Frontend**: Static hosting (Vercel, AWS S3)
- **API calls**: Mostly client-side (Gemini, OpenCTI, threat intel APIs)
- **Caching**: Browser localStorage, IndexedDB for graphs
- **Load**: Scales with number of concurrent users

### OpenCTI
- **Horizontal scaling**: Add more worker instances
- **Database**: PostgreSQL replication for HA
- **Search**: Elasticsearch sharding for large datasets
- **Message queue**: RabbitMQ clusters for high throughput

## Security Considerations

### X24 Portal
- ✅ API keys in environment variables (never in code)
- ✅ CORS configuration for API access
- ✅ CSP headers for XSS protection
- ✅ Input validation via Zod
- ✅ Audit logging for compliance

### OpenCTI
- ✅ API token authentication (change default password!)
- ✅ PostgreSQL encryption for data at rest
- ✅ TLS for data in transit
- ✅ RabbitMQ authentication
- ✅ S3 bucket policies for MinIO

## Monitoring & Observability

### X24 Portal
- **Browser console logs** (development)
- **OpenTelemetry traces** (via tracing.ts)
- **Error tracking** (via errorHandling.ts)
- **Performance metrics** (Vite build time, API response times)

### OpenCTI
- **Docker logs** (`docker-compose logs`)
- **Elasticsearch monitoring** (port 9200)
- **RabbitMQ management UI** (port 15672)
- **Health checks** (docker-compose health_status)

## Disaster Recovery

### Local Development
```bash
# Backup local data
cp -r x24-portal/node_modules node_modules.backup
docker volume ls | grep opencti-* > volumes.txt

# Full reset
rm -rf x24-portal/node_modules
docker-compose down -v
npm install
docker-compose up -d
```

### Production
- Regular PostgreSQL backups
- S3 bucket versioning (MinIO)
- Elasticsearch snapshot/restore
- RabbitMQ persistent queues

## Performance Benchmarks

| Metric | Target | Actual |
|--------|--------|--------|
| Vite dev startup | <10s | 1-2s |
| Vite build | <15s | 7.5s |
| Gemini API call | <5s | 2-3s (avg) |
| OpenCTI GraphQL query | <1s | 200-500ms |
| Entity profile load | <3s | 1.5-2s |
| Sanctions matching | <100ms | 50-80ms |
| Graph visualization (50 nodes) | <1s | 800ms |

## Testing Strategy

### Unit Tests
- Individual dimension scorers (sanctions-matcher.test.ts)
- Utility functions
- Error handling logic

### Integration Tests
- Real-world scenarios (multi-domain risk)
- API connector chains
- OpenCTI GraphQL queries

### E2E Tests
- Dashboard search → entity profile
- Risk aggregation pipeline
- Export to Compliant.one

### Performance Tests
- Batch matching (100+ entities)
- Large graph rendering (500+ nodes)
- Parallel connector queries

## Documentation Structure

```
docs/
├── README.md                    # Overview
├── SETUP.md                     # This file
├── ARCHITECTURE.md              # Architecture deep-dive
├── API.md                       # API reference
├── SCHEMAS.md                   # Data schemas
├── CONNECTORS.md                # Threat intel API guide
├── DEPLOYMENT.md                # Production guide
├── TROUBLESHOOTING.md           # Common issues
└── GLOSSARY.md                  # Terminology
```

---

**Last Updated**: December 31, 2025  
**Version**: 1.0.0  
**Status**: Production Ready
