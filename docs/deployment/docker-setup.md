# Docker Setup Guide

## Overview
Comprehensive guide for containerizing the MealPrep application using Docker, including development environments, production builds, multi-stage builds, and orchestration with Docker Compose.

## Docker Architecture

### Container Strategy
```
???????????????????    ???????????????????    ???????????????????
?   Frontend      ?    ?   Backend API   ?    ?   Database      ?
?   (React/Nginx) ?    ?   (.NET Core)   ?    ?   (SQL Server)  ?
?   Port: 80      ?    ?   Port: 5000    ?    ?   Port: 1433    ?
???????????????????    ???????????????????    ???????????????????
         ?                       ?                       ?
         ?????????????????????????????????????????????????
                                 ?
                    ???????????????????
                    ?     Redis       ?
                    ?   (Caching)     ?
                    ?   Port: 6379    ?
                    ???????????????????
```

### Development vs Production
- **Development**: Hot reload, debug symbols, development certificates
- **Production**: Optimized builds, health checks, security hardening
- **Testing**: Isolated environments, test databases, mock services

## Backend API Dockerfile

### Multi-Stage Production Build
```dockerfile
# Backend/Dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build-env
WORKDIR /app

# Copy csproj files and restore dependencies
COPY *.csproj ./
RUN dotnet restore

# Copy source code and build
COPY . ./
RUN dotnet publish -c Release -o out

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

# Create non-root user for security
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install required packages
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy published app
COPY --from=build-env /app/out .

# Create directories and set permissions
RUN mkdir -p /app/logs /app/uploads && \
    chown -R appuser:appuser /app

# Configure environment
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:5000
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENV DOTNET_USE_POLLING_FILE_WATCHER=true

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Switch to non-root user
USER appuser

EXPOSE 5000

ENTRYPOINT ["dotnet", "MealPrep.API.dll"]
```

### Development Dockerfile
```dockerfile
# Backend/Dockerfile.dev
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app

# Install development tools
RUN dotnet tool install --global dotnet-ef --version 8.0.0
ENV PATH="$PATH:/root/.dotnet/tools"

# Copy project files
COPY *.csproj ./
RUN dotnet restore

# Copy source code
COPY . ./

# Set environment for development
ENV ASPNETCORE_ENVIRONMENT=Development
ENV ASPNETCORE_URLS=http://+:5000
ENV DOTNET_USE_POLLING_FILE_WATCHER=true

# Enable hot reload
ENV DOTNET_WATCH_RESTART_ON_RUDE_EDIT=true

EXPOSE 5000

# Use dotnet watch for hot reload
CMD ["dotnet", "watch", "run", "--urls", "http://0.0.0.0:5000"]
```

## Frontend Dockerfile

### Production Build with Nginx
```dockerfile
# Frontend/Dockerfile
# Build stage
FROM node:18-alpine AS build-stage
WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# Copy source and build
COPY . .
RUN npm run build

# Production stage with Nginx
FROM nginx:alpine AS production-stage

# Install security updates
RUN apk update && apk upgrade

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf /etc/nginx/conf.d/default.conf

# Copy built application
COPY --from=build-stage /app/dist /usr/share/nginx/html

# Create non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -S appuser -G appuser

# Set permissions
RUN chown -R appuser:appuser /usr/share/nginx/html && \
    chown -R appuser:appuser /var/cache/nginx && \
    chown -R appuser:appuser /var/log/nginx && \
    chown -R appuser:appuser /etc/nginx/conf.d

# Create nginx PID directory
RUN mkdir -p /var/run && \
    chown -R appuser:appuser /var/run

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:80/health || exit 1

# Switch to non-root user
USER appuser

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Nginx Configuration
```nginx
# Frontend/nginx.conf
user appuser;
worker_processes auto;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Performance optimizations
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# Frontend/default.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Security
    server_tokens off;

    # Main application
    location / {
        try_files $uri $uri/ /index.html;
        
        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            add_header X-Content-Type-Options nosniff;
        }
    }

    # API proxy
    location /api/ {
        proxy_pass http://mealprep-api:5000/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Security
    location ~ /\. {
        deny all;
    }
}
```

## Docker Compose Configuration

### Development Environment
```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  # SQL Server Database
  mealprep-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mealprep-db-dev
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Dev@Password123!
      - MSSQL_PID=Developer
    ports:
      - "1433:1433"
    volumes:
      - mealprep-db-data:/var/opt/mssql
      - ./database/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P Dev@Password123! -Q 'SELECT 1'"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # Redis Cache
  mealprep-redis:
    image: redis:7-alpine
    container_name: mealprep-redis-dev
    ports:
      - "6379:6379"
    volumes:
      - mealprep-redis-data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Backend API
  mealprep-api:
    build:
      context: ./Backend
      dockerfile: Dockerfile.dev
    container_name: mealprep-api-dev
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=mealprep-db;Database=MealPrepDev;User Id=sa;Password=Dev@Password123!;TrustServerCertificate=True;
      - ConnectionStrings__Redis=mealprep-redis:6379
      - Jwt__Secret=development-secret-key-change-in-production
      - Gemini__ApiKey=${GEMINI_API_KEY}
    ports:
      - "5000:5000"
    volumes:
      - ./Backend:/app
      - /app/bin
      - /app/obj
    depends_on:
      mealprep-db:
        condition: service_healthy
      mealprep-redis:
        condition: service_healthy
    command: dotnet watch run --urls http://0.0.0.0:5000

  # Frontend Application
  mealprep-frontend:
    build:
      context: ./Frontend
      dockerfile: Dockerfile.dev
    container_name: mealprep-frontend-dev
    environment:
      - VITE_API_URL=http://localhost:5000/api
      - VITE_APP_ENV=development
    ports:
      - "3000:3000"
    volumes:
      - ./Frontend:/app
      - /app/node_modules
    depends_on:
      - mealprep-api

volumes:
  mealprep-db-data:
  mealprep-redis-data:

networks:
  default:
    name: mealprep-dev-network
```

### Production Environment
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  # Database (external in production)
  mealprep-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mealprep-db-prod
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=${DB_SA_PASSWORD}
      - MSSQL_PID=Standard
    volumes:
      - mealprep-db-data:/var/opt/mssql
      - mealprep-db-backups:/var/backups
    ports:
      - "1433:1433"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P ${DB_SA_PASSWORD} -Q 'SELECT 1'"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # Redis Cache
  mealprep-redis:
    image: redis:7-alpine
    container_name: mealprep-redis-prod
    ports:
      - "6379:6379"
    volumes:
      - mealprep-redis-data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Backend API
  mealprep-api:
    build:
      context: ./Backend
      dockerfile: Dockerfile
    container_name: mealprep-api-prod
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=${DB_CONNECTION_STRING}
      - ConnectionStrings__Redis=mealprep-redis:6379
      - Jwt__Secret=${JWT_SECRET}
      - Jwt__Issuer=${JWT_ISSUER}
      - Jwt__Audience=${JWT_AUDIENCE}
      - Gemini__ApiKey=${GEMINI_API_KEY}
    ports:
      - "5000:5000"
    depends_on:
      mealprep-db:
        condition: service_healthy
      mealprep-redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # Frontend Application
  mealprep-frontend:
    build:
      context: ./Frontend
      dockerfile: Dockerfile
    container_name: mealprep-frontend-prod
    environment:
      - VITE_API_URL=${API_URL}
      - VITE_APP_ENV=production
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./ssl:/etc/ssl/certs
    depends_on:
      mealprep-api:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  mealprep-db-data:
  mealprep-db-backups:
  mealprep-redis-data:

networks:
  default:
    name: mealprep-prod-network
```

## Development Workflow

### Development Setup
```bash
#!/bin/bash
# scripts/dev-setup.sh

echo "Setting up MealPrep development environment..."

# Create environment files
cp .env.example .env.development
echo "Please update .env.development with your API keys"

# Build and start development containers
docker-compose -f docker-compose.dev.yml up --build -d

# Wait for database to be ready
echo "Waiting for database to be ready..."
sleep 30

# Run database migrations
docker-compose -f docker-compose.dev.yml exec mealprep-api dotnet ef database update

# Seed development data
docker-compose -f docker-compose.dev.yml exec mealprep-api dotnet run --seed-data

echo "Development environment is ready!"
echo "Frontend: http://localhost:3000"
echo "API: http://localhost:5000"
echo "API Docs: http://localhost:5000/swagger"
```

### Hot Reload Configuration
```dockerfile
# Frontend/Dockerfile.dev
FROM node:18-alpine
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install

# Copy source code
COPY . .

# Set environment
ENV VITE_API_URL=http://localhost:5000/api

# Enable hot reload
EXPOSE 3000
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--port", "3000"]
```

## Production Deployment

### Build Script
```bash
#!/bin/bash
# scripts/build-production.sh

set -e

echo "Building MealPrep for production..."

# Set version from git tag or commit
VERSION=${1:-$(git rev-parse --short HEAD)}
echo "Building version: $VERSION"

# Build backend
echo "Building backend API..."
docker build -t mealprep-api:$VERSION ./Backend

# Build frontend
echo "Building frontend..."
docker build -t mealprep-frontend:$VERSION ./Frontend

# Tag latest
docker tag mealprep-api:$VERSION mealprep-api:latest
docker tag mealprep-frontend:$VERSION mealprep-frontend:latest

echo "Build completed successfully!"
echo "Images built:"
echo "  - mealprep-api:$VERSION"
echo "  - mealprep-frontend:$VERSION"
```

### Deployment Script
```bash
#!/bin/bash
# scripts/deploy-production.sh

set -e

echo "Deploying MealPrep to production..."

# Check required environment variables
required_vars=("DB_CONNECTION_STRING" "JWT_SECRET" "GEMINI_API_KEY")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "Error: $var environment variable is required"
        exit 1
    fi
done

# Create backup before deployment
echo "Creating database backup..."
docker-compose -f docker-compose.prod.yml exec mealprep-db \
    /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P ${DB_SA_PASSWORD} \
    -Q "BACKUP DATABASE [MealPrepProd] TO DISK = N'/var/backups/pre_deploy_$(date +%Y%m%d_%H%M%S).bak'"

# Pull latest images
docker-compose -f docker-compose.prod.yml pull

# Deploy with zero downtime (if using load balancer)
echo "Deploying new version..."
docker-compose -f docker-compose.prod.yml up -d --no-deps mealprep-api
docker-compose -f docker-compose.prod.yml up -d --no-deps mealprep-frontend

# Run health checks
echo "Running health checks..."
sleep 30

# Check API health
if ! curl -f http://localhost:5000/health; then
    echo "API health check failed!"
    exit 1
fi

# Check frontend health
if ! curl -f http://localhost:80/health; then
    echo "Frontend health check failed!"
    exit 1
fi

echo "Deployment completed successfully!"
```

## Monitoring and Logging

### Logging Configuration
```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  # ELK Stack for logging
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.6.0
    container_name: logstash
    volumes:
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.6.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  # Update application services to send logs
  mealprep-api:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "service=mealprep-api"

  mealprep-frontend:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "service=mealprep-frontend"

volumes:
  elasticsearch-data:
```

### Monitoring with Prometheus
```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

volumes:
  prometheus-data:
  grafana-data:
```

## Security Considerations

### Security Hardening
```dockerfile
# Security-hardened Dockerfile example
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

# Security updates
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set security limits
RUN echo "appuser soft nofile 1024" >> /etc/security/limits.conf && \
    echo "appuser hard nofile 2048" >> /etc/security/limits.conf

WORKDIR /app
COPY --from=build-env /app/out .

# File permissions
RUN chown -R appuser:appuser /app && \
    chmod -R 755 /app

# Remove unnecessary packages
RUN apt-get autoremove -y && \
    apt-get autoclean

USER appuser

# Security labels
LABEL security.policy="restricted"
LABEL security.scan="enabled"
```

This comprehensive Docker setup provides a robust containerization strategy for the MealPrep application with development, testing, and production configurations, along with monitoring and security best practices.

*This Docker guide should be updated as container orchestration needs evolve and new Docker features become available.*