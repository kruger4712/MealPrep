# MealMenu Deployment Guide

## Overview

This guide covers deployment strategies for the MealMenu family meal preparation application across different environments, from local development to production cloud deployments.

## Technology Stack Deployment

### Backend Deployment (ASP.NET Core 8)
- **Runtime**: .NET 8.0 
- **Database**: SQL Server / MySQL
- **Caching**: Redis
- **AI Service**: Google Gemini AI
- **Authentication**: JWT with ASP.NET Core Identity

### Frontend Deployment (React TypeScript)
- **Build Tool**: Vite
- **Node Version**: 18.x or higher
- **Package Manager**: npm/yarn
- **Static Hosting**: Azure Static Web Apps / AWS S3 + CloudFront

## Local Development Environment

### Prerequisites
```bash
# Required Software
- .NET 8.0 SDK
- Node.js 18.x or higher
- SQL Server LocalDB or MySQL 8.0
- Redis (optional for local development)
- Visual Studio 2022 or VS Code
- Docker Desktop (for containerized development)
```

### Backend Setup
```bash
# Clone repository
git clone https://github.com/your-org/mealmenu.git
cd mealmenu

# Navigate to backend
cd src/MealMenu.API

# Restore packages
dotnet restore

# Set up user secrets for local development
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=(localdb)\\mssqllocaldb;Database=MealMenuDB;Trusted_Connection=true;MultipleActiveResultSets=true"
dotnet user-secrets set "GoogleCloud:ProjectId" "your-project-id"
dotnet user-secrets set "GoogleCloud:ServiceAccountKeyPath" "path/to/service-account.json"
dotnet user-secrets set "JWT:SecretKey" "your-super-secret-jwt-key-here"

# Apply database migrations
dotnet ef database update

# Run application
dotnet run
```

### Frontend Setup
```bash
# Navigate to frontend
cd src/meal-prep-frontend

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env.local

# Edit .env.local with local API endpoint
VITE_API_BASE_URL=https://localhost:7001/api
VITE_ENVIRONMENT=development

# Start development server
npm run dev
```

### Docker Development Environment
```yaml
# docker-compose.yml
version: '3.8'

services:
  sql-server:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
      - MSSQL_PID=Developer
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  mealmenu-api:
    build:
      context: .
      dockerfile: src/MealMenu.API/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=sql-server,1433;Database=MealMenuDB;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true
      - Redis__ConnectionString=redis:6379
    ports:
      - "7001:80"
    depends_on:
      - sql-server
      - redis
    volumes:
      - ./secrets:/app/secrets:ro

  mealmenu-frontend:
    build:
      context: .
      dockerfile: src/meal-prep-frontend/Dockerfile
    environment:
      - VITE_API_BASE_URL=http://localhost:7001/api
    ports:
      - "3000:80"
    depends_on:
      - mealmenu-api

volumes:
  sqlserver_data:
  redis_data:
```

```bash
# Start development environment
docker-compose up -d

# View logs
docker-compose logs -f

# Stop environment
docker-compose down
```

## Production Deployment

### Azure Deployment

#### Azure Resources Setup
```bash
# Create resource group
az group create --name rg-mealmenu-prod --location eastus

# Create App Service Plan
az appservice plan create \
  --name asp-mealmenu-prod \
  --resource-group rg-mealmenu-prod \
  --sku P1V3 \
  --is-linux

# Create Web App
az webapp create \
  --name app-mealmenu-api-prod \
  --resource-group rg-mealmenu-prod \
  --plan asp-mealmenu-prod \
  --runtime "DOTNETCORE:8.0"

# Create SQL Database
az sql server create \
  --name sql-mealmenu-prod \
  --resource-group rg-mealmenu-prod \
  --location eastus \
  --admin-user mealmenudba \
  --admin-password 'YourComplexPassword123!'

az sql db create \
  --name MealMenuDB \
  --server sql-mealmenu-prod \
  --resource-group rg-mealmenu-prod \
  --service-objective S1

# Create Redis Cache
az redis create \
  --name redis-mealmenu-prod \
  --resource-group rg-mealmenu-prod \
  --location eastus \
  --sku Basic \
  --vm-size c0

# Create Storage Account for file uploads
az storage account create \
  --name stmealmenu \
  --resource-group rg-mealmenu-prod \
  --location eastus \
  --sku Standard_LRS
```

#### Application Settings Configuration
```bash
# Configure App Service settings
az webapp config appsettings set \
  --name app-mealmenu-api-prod \
  --resource-group rg-mealmenu-prod \
  --settings \
    "ConnectionStrings__DefaultConnection=Server=sql-mealmenu-prod.database.windows.net;Database=MealMenuDB;User Id=mealmenudba;Password=YourComplexPassword123!;Encrypt=true" \
    "Redis__ConnectionString=redis-mealmenu-prod.redis.cache.windows.net:6380,password=your-redis-key,ssl=True,abortConnect=False" \
    "GoogleCloud__ProjectId=your-project-id" \
    "JWT__SecretKey=your-production-jwt-secret" \
    "ASPNETCORE_ENVIRONMENT=Production"
```

#### GitHub Actions CI/CD Pipeline
```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
        
    - name: Restore dependencies
      run: dotnet restore src/MealMenu.API
      
    - name: Build
      run: dotnet build src/MealMenu.API --no-restore
      
    - name: Test
      run: dotnet test src/MealMenu.Tests --no-build --verbosity normal
      
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        
    - name: Install frontend dependencies
      run: |
        cd src/meal-prep-frontend
        npm ci
        
    - name: Build frontend
      run: |
        cd src/meal-prep-frontend
        npm run build
        
    - name: Test frontend
      run: |
        cd src/meal-prep-frontend
        npm run test:ci

  deploy-to-azure:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x
        
    - name: Build and publish API
      run: |
        dotnet publish src/MealMenu.API -c Release -o ./api-publish
        
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'app-mealmenu-api-prod'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: './api-publish'
        
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        
    - name: Build frontend for production
      run: |
        cd src/meal-prep-frontend
        npm ci
        npm run build
        
    - name: Deploy to Azure Static Web Apps
      uses: Azure/static-web-apps-deploy@v1
      with:
        azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
        repo_token: ${{ secrets.GITHUB_TOKEN }}
        action: "upload"
        app_location: "src/meal-prep-frontend"
        output_location: "dist"
```

### AWS Deployment

#### Infrastructure as Code (CloudFormation)
```yaml
# infrastructure/cloudformation-template.yml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'MealMenu Application Infrastructure'

Parameters:
  Environment:
    Type: String
    Default: 'prod'
    AllowedValues: ['dev', 'staging', 'prod']

Resources:
  # VPC and Networking
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub 'mealmenu-vpc-${Environment}'

  # Application Load Balancer
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub 'mealmenu-alb-${Environment}'
      Scheme: internet-facing
      Type: application
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2

  # ECS Cluster
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub 'mealmenu-cluster-${Environment}'
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT

  # RDS Database
  RDSDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: !Sub 'mealmenu-db-${Environment}'
      DBInstanceClass: db.t3.micro
      Engine: mysql
      EngineVersion: 8.0.35
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      AllocatedStorage: 20
      VPCSecurityGroups:
        - !Ref DatabaseSecurityGroup
      DBSubnetGroupName: !Ref DBSubnetGroup

  # ElastiCache Redis
  RedisCluster:
    Type: AWS::ElastiCache::CacheCluster
    Properties:
      CacheNodeType: cache.t3.micro
      Engine: redis
      NumCacheNodes: 1
      VpcSecurityGroupIds:
        - !Ref RedisSecurityGroup
      CacheSubnetGroupName: !Ref RedisSubnetGroup

  # S3 Bucket for static assets
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'mealmenu-assets-${Environment}'
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

Parameters:
  DBPassword:
    Type: String
    NoEcho: true
    Description: Password for RDS database
```

#### ECS Task Definition
```yaml
# infrastructure/ecs-task-definition.json
{
  "family": "mealmenu-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "mealmenu-api",
      "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/mealmenu-api:latest",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ASPNETCORE_ENVIRONMENT",
          "value": "Production"
        }
      ],
      "secrets": [
        {
          "name": "ConnectionStrings__DefaultConnection",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:mealmenu/database-connection"
        },
        {
          "name": "JWT__SecretKey",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:mealmenu/jwt-secret"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/mealmenu-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

## Database Migration and Management

### Entity Framework Migrations
```bash
# Create new migration
dotnet ef migrations add InitialCreate --project src/MealMenu.Infrastructure --startup-project src/MealMenu.API

# Update database
dotnet ef database update --project src/MealMenu.Infrastructure --startup-project src/MealMenu.API

# Generate SQL script for production
dotnet ef migrations script --project src/MealMenu.Infrastructure --startup-project src/MealMenu.API --output migration.sql
```

### Production Database Deployment
```bash
# Pre-deployment backup
sqlcmd -S server-name -d database-name -Q "BACKUP DATABASE [MealMenuDB] TO DISK = 'C:\backup\pre-deployment-backup.bak'"

# Apply migrations
sqlcmd -S server-name -d database-name -i migration.sql

# Verify deployment
sqlcmd -S server-name -d database-name -Q "SELECT * FROM __EFMigrationsHistory ORDER BY MigrationId DESC"
```

## Monitoring and Logging

### Application Insights (Azure)
```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry(builder.Configuration["ApplicationInsights:ConnectionString"]);

// Custom telemetry
public class CustomTelemetryService
{
    private readonly TelemetryClient _telemetryClient;

    public void TrackMealSuggestionRequest(MealSuggestionRequest request, TimeSpan duration, bool success)
    {
        var telemetry = new EventTelemetry("MealSuggestionRequested");
        telemetry.Properties["FamilySize"] = request.FamilyMembers.Count.ToString();
        telemetry.Properties["MealType"] = request.MealType.ToString();
        telemetry.Metrics["Duration"] = duration.TotalMilliseconds;
        telemetry.Metrics["Success"] = success ? 1 : 0;
        
        _telemetryClient.TrackEvent(telemetry);
    }
}
```

### CloudWatch (AWS)
```csharp
// AWS CloudWatch logging
builder.Logging.AddAWSProvider(builder.Configuration.GetAWSLoggingConfigSection());

// Custom metrics
public class CloudWatchMetricsService
{
    private readonly IAmazonCloudWatch _cloudWatch;

    public async Task RecordMealSuggestionMetric(int familySize, TimeSpan responseTime)
    {
        await _cloudWatch.PutMetricDataAsync(new PutMetricDataRequest
        {
            Namespace = "MealMenu/API",
            MetricData = new List<MetricDatum>
            {
                new MetricDatum
                {
                    MetricName = "MealSuggestionResponseTime",
                    Value = responseTime.TotalMilliseconds,
                    Unit = StandardUnit.Milliseconds,
                    Dimensions = new List<Dimension>
                    {
                        new Dimension { Name = "FamilySize", Value = familySize.ToString() }
                    }
                }
            }
        });
    }
}
```

## Security Considerations

### Production Security Checklist
- [ ] HTTPS enforced across all endpoints
- [ ] JWT secrets stored in secure key vault
- [ ] Database connection strings encrypted
- [ ] API rate limiting configured
- [ ] CORS properly configured for frontend domains
- [ ] SQL injection protection through parameterized queries
- [ ] Input validation on all endpoints
- [ ] Secrets rotation strategy implemented
- [ ] Security headers configured
- [ ] Database firewall rules configured
- [ ] Regular security updates scheduled

### Configuration Management
```csharp
// Secure configuration
public class SecureConfigurationExtensions
{
    public static void AddSecureConfiguration(this IServiceCollection services, IConfiguration configuration)
    {
        // Use Azure Key Vault or AWS Secrets Manager
        services.Configure<JwtOptions>(options =>
        {
            options.SecretKey = GetSecretFromVault("jwt-secret-key");
            options.Issuer = configuration["JWT:Issuer"];
            options.Audience = configuration["JWT:Audience"];
            options.ExpirationMinutes = 60;
        });

        services.Configure<GoogleCloudOptions>(options =>
        {
            options.ProjectId = configuration["GoogleCloud:ProjectId"];
            options.ServiceAccountKey = GetSecretFromVault("google-service-account-key");
        });
    }

    private static string GetSecretFromVault(string secretName)
    {
        // Implementation depends on chosen secret management service
        // Azure Key Vault, AWS Secrets Manager, etc.
        return secretValue;
    }
}
```

This deployment guide provides comprehensive coverage for deploying the MealMenu application across different environments and cloud platforms, with emphasis on security, monitoring, and best practices.