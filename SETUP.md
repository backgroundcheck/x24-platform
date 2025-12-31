# Setup Guide

## Prerequisites

- Node.js 18+ (for X24 Portal)
- Docker & Docker Compose (for OpenCTI)
- Git
- Google Gemini API key
- GitHub PAT (for pushing changes)

## Local Development Setup

### 1. Clone the Repository

```bash
git clone https://github.com/backgroundcheck/x24-platform.git
cd x24-platform
```

### 2. Initialize Submodules (if using submodule approach)

```bash
git submodule update --init --recursive
```

### 3. X24 Portal Setup

```bash
cd x24-portal

# Install dependencies
npm install

# Create .env file
cat > .env << EOF
VITE_GEMINI_API_KEY=your_gemini_api_key_here
VITE_API_KEY=your_fallback_api_key_here
X24_OPENCTI_URL=http://localhost:8080
X24_OPENCTI_KEY=your_opencti_token_here
VITE_TRACING_ENABLED=true
EOF

# Start development server
npm run dev       # Runs on http://localhost:3002

# Build for production
npm run build

# Run tests
npm run test
```

### 4. OpenCTI Setup

```bash
cd opencti

# Start OpenCTI Docker stack
docker-compose up -d

# Wait 2-5 minutes for services to initialize
# Check status: docker ps

# Access dashboard
# URL: http://localhost:8080
# Email: admin@opencti.io
# Password: changeme (CHANGE THIS IN PRODUCTION!)
```

### 5. Verify Integration

```bash
# In X24 Portal terminal:
npm run dev

# Verify Gemini connector works
# Check browser console for API calls
# Verify OpenCTI connection at http://localhost:8080
```

## Docker Services

OpenCTI provides:

```
x24-platform-monorepo/opencti/
├── Platform (port 8080)
├── GraphQL API (port 4000)
├── PostgreSQL (port 5432)
├── Elasticsearch (port 9200)
├── RabbitMQ (port 5672)
├── MinIO S3 (port 9000)
├── 20+ threat feed connectors
└── 3 worker instances
```

## Environment Variables Reference

### X24 Portal

| Variable | Required | Description |
|----------|----------|-------------|
| `VITE_GEMINI_API_KEY` | ✅ Yes | Google Gemini API key |
| `VITE_API_KEY` | ⚠️ Optional | Fallback API key |
| `X24_VIRUSTOTAL_KEY` | ⚠️ Optional | VirusTotal API key |
| `X24_SHODAN_KEY` | ⚠️ Optional | Shodan API key |
| `X24_OPENCTI_URL` | ✅ Yes | OpenCTI instance URL |
| `X24_OPENCTI_KEY` | ✅ Yes | OpenCTI API token |
| `VITE_TRACING_ENABLED` | ⚠️ Optional | Enable OpenTelemetry tracing |

### OpenCTI (.env in docker/)

| Variable | Default | Notes |
|----------|---------|-------|
| `OPENCTI_ADMIN_EMAIL` | `admin@opencti.io` | Change immediately |
| `OPENCTI_ADMIN_PASSWORD` | `changeme` | **CHANGE in production** |
| `PGSQL_PASSWORD` | `changeme` | **CHANGE in production** |
| `RABBITMQ_DEFAULT_USER` | `guest` | Change if needed |
| `RABBITMQ_DEFAULT_PASS` | `guest` | Change if needed |

## Troubleshooting

### X24 Portal won't start
```bash
# Clear node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
npm run dev
```

### OpenCTI containers fail to start
```bash
# Check Docker logs
docker-compose logs -f opencti

# Rebuild and restart
docker-compose down -v
docker-compose up -d --build
```

### Gemini API errors
```bash
# Verify API key is set
echo $VITE_GEMINI_API_KEY

# Check console for 401/403 errors
# Ensure key has appropriate permissions
```

### Port conflicts
```bash
# X24 Portal uses 3002
# OpenCTI uses 8080, 4000, 5432, 9200, 5672, 9000
# Verify ports are available:
netstat -ano | findstr :3002
netstat -ano | findstr :8080
```

## Git Workflow

### Making Changes

```bash
# Create a branch
git checkout -b feature/my-feature

# Make changes
# Commit regularly
git commit -m "feat: add new widget"

# Push to GitHub
git push origin feature/my-feature

# Create Pull Request on GitHub
```

### Keeping Up to Date

```bash
# Pull latest changes
git pull origin main

# Update submodules if using them
git submodule update --recursive
```

## Deployment

See deployment guides in individual README files:
- `x24-portal/README.md` — Frontend deployment
- `opencti/README.md` — Backend deployment

## Support

- **Issues**: GitHub Issues
- **Discussions**: GitHub Discussions
- **Documentation**: See `docs/` folder

## License

Proprietary — X24 Intelligence Platform
