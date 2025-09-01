# Getting Started Guide

## Overview
Quick start guide for new developers joining the MealPrep project.

## Prerequisites

### Required Software
- **.NET 8 SDK** - [Download here](https://dotnet.microsoft.com/download/dotnet/8.0)
- **Node.js 18+** - [Download here](https://nodejs.org/)
- **Docker Desktop** - [Download here](https://www.docker.com/products/docker-desktop)
- **Visual Studio 2022** or **VS Code** - [Visual Studio](https://visualstudio.microsoft.com/) | [VS Code](https://code.visualstudio.com/)
- **Git** - [Download here](https://git-scm.com/)

### Optional Tools
- **SQL Server Management Studio** (for SQL Server)
- **MySQL Workbench** (for MySQL)
- **Postman** (for API testing)
- **Azure CLI** (for cloud deployment)

## Technology Stack Overview

### Backend Technologies
- **ASP.NET Core 8** - Web API framework
- **Entity Framework Core 8** - ORM for database operations
- **ASP.NET Core Identity** - Authentication and authorization
- **SignalR** - Real-time communication
- **Google.Cloud.AiPlatform** - Gemini AI integration
- **AutoMapper** - Object mapping
- **FluentValidation** - Input validation
- **Serilog** - Structured logging

### Frontend Technologies
- **React 18** - UI framework
- **TypeScript** - Type-safe JavaScript
- **React Query/TanStack Query** - Server state management
- **React Hook Form** - Form handling
- **Material-UI** - Component library
- **React Beautiful DnD** - Drag and drop
- **Axios** - HTTP client

### Database & Infrastructure
- **MySQL 8.0** or **SQL Server 2022** - Primary database
- **Redis** - Caching and session storage
- **Docker** - Containerization
- **Azure/AWS** - Cloud hosting

## Setup Steps

### 1. Repository Setup
```bash
# Clone the repository
git clone https://github.com/your-org/MealPrep.git
cd MealPrep

# Add upstream remote for staying in sync
git remote add upstream https://github.com/original-org/MealPrep.git

# Verify remotes
git remote -v
```

### 2. Backend Setup

#### Environment Configuration
```bash
# Navigate to API project
cd MealPrep.API

# Restore NuGet packages
dotnet restore

# Initialize user secrets for secure configuration
dotnet user-secrets init

# Set up user secrets (replace with your values)
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=MealPrepDB;Trusted_Connection=true;"
dotnet user-secrets set "GoogleCloud:ProjectId" "your-google-cloud-project-id"
dotnet user-secrets set "GoogleCloud:ServiceAccountKeyPath" "path/to/your/service-account.json"
dotnet user-secrets set "JWT:Secret" "your-jwt-secret-key-minimum-32-characters"
dotnet user-secrets set "Redis:ConnectionString" "localhost:6379"
```

#### Database Setup
```bash
# Install EF Core tools (if not already installed)
dotnet tool install --global dotnet-ef

# Create initial migration
dotnet ef migrations add InitialCreate

# Update database
dotnet ef database update

# Seed initial data (if seeding is configured)
dotnet run --seed-data
```

#### Run Backend
```bash
# Run the API
dotnet run

# Or with hot reload for development
dotnet watch run

# API should be available at:
# - HTTPS: https://localhost:7001
# - HTTP: http://localhost:5000
# - Swagger UI: https://localhost:7001/swagger
```

### 3. Frontend Setup

#### Environment Configuration
```bash
# Navigate to frontend project
cd meal-prep-frontend

# Install dependencies
npm install

# Create environment file
cp .env.example .env.local

# Edit .env.local with your configuration
# REACT_APP_API_URL=https://localhost:7001/api
# REACT_APP_ENVIRONMENT=development
```

#### Run Frontend
```bash
# Start development server
npm start

# Or using Yarn
yarn start

# Frontend should be available at:
# http://localhost:3000
```

### 4. Docker Development Environment (Alternative)

#### Using Docker Compose
```bash
# Start all services (database, API, frontend)
docker-compose -f docker-compose.dev.yml up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down

# Services will be available at:
# - Frontend: http://localhost:3000
# - API: https://localhost:7001
# - Database: localhost:1433 (SQL Server) or localhost:3306 (MySQL)
# - Redis: localhost:6379
```

## Verification Steps

### 1. Backend Verification
```bash
# Check API health
curl https://localhost:7001/health

# Access Swagger documentation
# Open browser: https://localhost:7001/swagger

# Test authentication endpoint
curl -X POST https://localhost:7001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "TestPassword123!",
    "firstName": "Test",
    "lastName": "User"
  }'
```

### 2. Frontend Verification
- Open browser to http://localhost:3000
- Verify login page loads
- Check browser console for any errors
- Test responsive design on different screen sizes

### 3. Database Verification
```bash
# Connect to database and verify tables exist
# For SQL Server (using sqlcmd)
sqlcmd -S localhost -d MealPrepDB -Q "SELECT COUNT(*) FROM INFORMATION_SCHEMA.TABLES"

# For MySQL
mysql -h localhost -u root -p MealPrepDB -e "SHOW TABLES;"
```

### 4. AI Integration Verification
```bash
# Test AI service endpoint (after authentication)
curl -X POST https://localhost:7001/api/ai/meal-suggestions \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "familyMembers": [...],
    "mealType": "dinner",
    "budget": 25.00
  }'
```

## Development Workflow

### 1. Daily Development Routine
```bash
# Start of day - sync with upstream
git checkout main
git pull upstream main

# Create feature branch
git checkout -b feature/your-feature-name

# Start development environment
docker-compose up -d db redis  # Start dependencies
dotnet watch run  # In MealPrep.API folder
npm start  # In meal-prep-frontend folder
```

### 2. Code Quality Checks
```bash
# Backend - Run tests
dotnet test

# Backend - Check code formatting
dotnet format

# Frontend - Run tests
npm test

# Frontend - Check linting
npm run lint

# Frontend - Type checking
npm run type-check
```

## Common Issues and Solutions

### Backend Issues

**Issue**: Entity Framework migrations fail
```bash
# Solution: Reset migrations
rm -rf Migrations/
dotnet ef migrations add InitialCreate
dotnet ef database update
```

**Issue**: JWT authentication not working
```bash
# Solution: Verify user secrets
dotnet user-secrets list
# Ensure JWT:Secret is set and at least 32 characters
```

### Frontend Issues

**Issue**: npm install fails with permission errors
```bash
# Solution: Clear npm cache
npm cache clean --force
# Use npm ci for clean install
npm ci
```

**Issue**: TypeScript compilation errors
```bash
# Solution: Restart TypeScript language server in VS Code
# Ctrl+Shift+P -> "TypeScript: Restart TS Server"
```

### Docker Issues

**Issue**: Port already in use
```bash
# Solution: Check what's using the port
netstat -an | findstr :3000
# Kill the process or change port in docker-compose.yml
```

**Issue**: Database connection fails in Docker
```bash
# Solution: Ensure connection string uses service name
# "Server=db;Database=MealPrepDB;..." (not localhost)
```

## IDE Configuration

### Visual Studio 2022 Setup
1. Install ASP.NET and web development workload
2. Install Node.js development tools
3. Configure formatting settings (Tools ? Options ? Text Editor)
4. Install recommended extensions:
   - Roslynator
   - SonarLint
   - GitLens

### VS Code Setup
```bash
# Install recommended extensions
code --install-extension ms-dotnettools.csharp
code --install-extension bradlc.vscode-tailwindcss
code --install-extension esbenp.prettier-vscode
code --install-extension ms-vscode.vscode-typescript-next
```

**Recommended VS Code Extensions:**
- C# Dev Kit
- React Extension Pack
- Prettier
- ESLint
- GitLens
- Docker
- Thunder Client (API testing)

## Next Steps

1. **Review Architecture** - Read [System Architecture](../ARCHITECTURE.md)
2. **Coding Standards** - Review [Coding Standards](coding-standards.md)
3. **Testing Strategy** - Understand [Testing Approach](testing-strategy.md)
4. **Git Workflow** - Learn [Git Workflow](git-workflow.md)
5. **Code Reviews** - Check [Review Guidelines](code-review-guidelines.md)

## Getting Help

### Documentation Resources
- [API Documentation](../API.md)
- [Database Design](../DATABASE.md)
- [AI Integration Guide](../AI_INTEGRATION.md)
- [Deployment Guide](../DEPLOYMENT.md)

### Community Support
- **GitHub Issues** - Report bugs and request features
- **GitHub Discussions** - Ask questions and share ideas
- **Team Chat** - Join the development team chat
- **Code Reviews** - Get feedback on your code

### Troubleshooting
- [Common Issues](../TROUBLESHOOTING.md)
- [Development Tools](development-tools.md)
- [Performance Tips](../frontend/performance-optimization.md)

## Contributing Guidelines

Before making your first contribution:
1. Read the [Contributing Guide](../CONTRIBUTING.md)
2. Understand the [Code Review Process](code-review-guidelines.md)
3. Set up your development environment
4. Run the test suite to ensure everything works
5. Make a small test change to verify your setup

Welcome to the MealPrep development team! ????

---

*This guide is updated regularly. If you find any issues or have suggestions for improvement, please create an issue or submit a pull request.*