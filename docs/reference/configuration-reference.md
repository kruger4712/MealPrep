# Configuration Reference

## Overview
Comprehensive reference for all configuration options available in the MealPrep application, covering backend API settings, frontend configuration, deployment parameters, and environment-specific variables.

## Configuration Hierarchy

### Configuration Sources (Priority Order)
1. **Environment Variables** (Highest Priority)
2. **User Secrets** (Development Only)
3. **appsettings.{Environment}.json**
4. **appsettings.json** (Lowest Priority)

---

## Backend Configuration (ASP.NET Core)

### Connection Strings

#### Database Connection
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MealPrepDB;Trusted_Connection=true;MultipleActiveResultSets=true;Connection Timeout=30;",
    "ReadOnlyConnection": "Server=replica-server;Database=MealPrepDB;Trusted_Connection=true;Application Intent=ReadOnly;"
  }
}
```

**Environment Variable Override:**
```bash
ConnectionStrings__DefaultConnection="Server=prod-server;Database=MealPrepDB;User Id=mealprep_user;Password=secure_password;Encrypt=true;"
```

#### Redis Cache Connection
```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379",
    "RedisCluster": "redis-cluster.cache.windows.net:6380,password=your_password,ssl=True,abortConnect=False"
  }
}
```

### Authentication & Security

#### JWT Configuration
```json
{
  "Authentication": {
    "JwtSettings": {
      "SecretKey": "your-super-secure-secret-key-here-minimum-32-characters",
      "Issuer": "MealPrepAPI",
      "Audience": "MealPrepApp",
      "ExpirationInHours": 24,
      "RefreshTokenExpirationInDays": 30,
      "RequireHttpsMetadata": true,
      "ValidateIssuerSigningKey": true,
      "ValidateIssuer": true,
      "ValidateAudience": true,
      "ValidateLifetime": true,
      "ClockSkew": "00:05:00"
    }
  }
}
```

**Environment Variables:**
```bash
Authentication__JwtSettings__SecretKey="production-secret-key"
Authentication__JwtSettings__ExpirationInHours=1
Authentication__JwtSettings__RequireHttpsMetadata=true
```

#### Password Requirements
```json
{
  "Authentication": {
    "PasswordSettings": {
      "RequiredLength": 8,
      "RequireNonAlphanumeric": true,
      "RequireLowercase": true,
      "RequireUppercase": true,
      "RequireDigit": true,
      "RequiredUniqueChars": 6,
      "MaxFailedAccessAttempts": 5,
      "DefaultLockoutTimeSpan": "00:15:00"
    }
  }
}
```

### Google Cloud AI Configuration

#### Gemini AI Settings
```json
{
  "GoogleCloud": {
    "ProjectId": "mealprep-ai-project",
    "ServiceAccountKeyPath": "/path/to/service-account.json",
    "Location": "us-central1",
    "ModelConfig": {
      "ModelName": "gemini-pro",
      "Temperature": 0.7,
      "TopP": 0.8,
      "TopK": 40,
      "MaxOutputTokens": 2048,
      "SafetySettings": [
        {
          "Category": "HARM_CATEGORY_HATE_SPEECH",
          "Threshold": "BLOCK_MEDIUM_AND_ABOVE"
        },
        {
          "Category": "HARM_CATEGORY_DANGEROUS_CONTENT", 
          "Threshold": "BLOCK_MEDIUM_AND_ABOVE"
        }
      ]
    },
    "RateLimiting": {
      "RequestsPerMinute": 60,
      "RequestsPerDay": 10000,
      "EnableRetry": true,
      "MaxRetryAttempts": 3,
      "RetryDelayMs": 1000
    }
  }
}
```

**Environment Variables:**
```bash
GoogleCloud__ProjectId="production-project-id"
GoogleCloud__ServiceAccountKeyPath="/secrets/gcp-service-account.json"
GoogleCloud__ModelConfig__Temperature=0.5
GoogleCloud__RateLimiting__RequestsPerMinute=100
```

### API Configuration

#### General API Settings
```json
{
  "Api": {
    "Version": "v1",
    "Title": "MealPrep API",
    "Description": "AI-powered family meal planning API",
    "ContactEmail": "api-support@mealprep.com",
    "EnableSwagger": true,
    "EnableCors": true,
    "CorsOrigins": ["https://app.mealprep.com", "https://staging.mealprep.com"],
    "DefaultPageSize": 20,
    "MaxPageSize": 100,
    "EnableCompression": true,
    "EnableResponseCaching": true,
    "CacheMaxAge": 300
  }
}
```

#### Rate Limiting
```json
{
  "RateLimiting": {
    "General": {
      "RequestsPerMinute": 100,
      "RequestsPerHour": 1000,
      "RequestsPerDay": 10000
    },
    "AI": {
      "RequestsPerMinute": 20,
      "RequestsPerHour": 200,
      "RequestsPerDay": 1000
    },
    "Authentication": {
      "RequestsPerMinute": 10,
      "RequestsPerHour": 50,
      "RequestsPerDay": 200
    },
    "EnableIpWhitelist": false,
    "WhitelistedIps": ["127.0.0.1", "::1"]
  }
}
```

### Logging Configuration

#### Serilog Settings
```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/mealprep-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "outputTemplate": "[{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "ApplicationInsights",
        "Args": {
          "restrictedToMinimumLevel": "Warning"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithProcessId", "WithThreadId"],
    "Properties": {
      "Application": "MealPrep.API",
      "Environment": "Development"
    }
  }
}
```

### Health Checks Configuration

```json
{
  "HealthChecks": {
    "Endpoints": {
      "Database": {
        "Enabled": true,
        "Timeout": "00:00:10",
        "Tags": ["database", "sql"]
      },
      "Redis": {
        "Enabled": true,
        "Timeout": "00:00:05",
        "Tags": ["cache", "redis"]
      },
      "GoogleCloudAI": {
        "Enabled": true,
        "Timeout": "00:00:15",
        "Tags": ["ai", "external"]
      }
    },
    "UI": {
      "Enabled": true,
      "Path": "/health-ui",
      "RequireAuthentication": true
    },
    "Monitoring": {
      "Enabled": true,
      "IntervalSeconds": 30,
      "FailureThreshold": 3,
      "AlertWebhookUrl": "https://your-monitoring-webhook.com/alerts"
    }
  }
}
```

### File Storage Configuration

#### Image Upload Settings
```json
{
  "FileStorage": {
    "Provider": "AzureBlob",
    "AzureBlob": {
      "ConnectionString": "DefaultEndpointsProtocol=https;AccountName=mealprepstorage;AccountKey=your_key;EndpointSuffix=core.windows.net",
      "ContainerName": "recipe-images",
      "BaseUrl": "https://mealprepstorage.blob.core.windows.net/recipe-images"
    },
    "Local": {
      "BasePath": "wwwroot/uploads",
      "BaseUrl": "https://localhost:7001/uploads"
    },
    "Upload": {
      "MaxFileSizeBytes": 5242880,
      "AllowedExtensions": [".jpg", ".jpeg", ".png", ".gif"],
      "AllowedMimeTypes": ["image/jpeg", "image/png", "image/gif"],
      "GenerateUniqueName": true,
      "EnableImageResizing": true,
      "ThumbnailSizes": [
        { "Name": "thumbnail", "Width": 150, "Height": 150 },
        { "Name": "medium", "Width": 400, "Height": 300 },
        { "Name": "large", "Width": 800, "Height": 600 }
      ]
    }
  }
}
```

### Email Configuration

#### SMTP Settings
```json
{
  "Email": {
    "Provider": "SMTP",
    "Smtp": {
      "Host": "smtp.sendgrid.net",
      "Port": 587,
      "EnableSsl": true,
      "Username": "apikey",
      "Password": "your_sendgrid_api_key",
      "FromEmail": "noreply@mealprep.com",
      "FromName": "MealPrep Support"
    },
    "SendGrid": {
      "ApiKey": "your_sendgrid_api_key",
      "FromEmail": "noreply@mealprep.com",
      "FromName": "MealPrep Support"
    },
    "Templates": {
      "EmailVerification": "d-1234567890abcdef1234567890abcdef",
      "PasswordReset": "d-abcdef1234567890abcdef1234567890",
      "WeeklyMenuReminder": "d-1111111111111111111111111111111"
    },
    "EnableEmailSending": true,
    "MaxRetryAttempts": 3,
    "RetryDelaySeconds": 5
  }
}
```

### Background Services Configuration

#### Hangfire Settings
```json
{
  "Hangfire": {
    "ConnectionString": "Server=localhost;Database=MealPrepHangfire;Trusted_Connection=true;",
    "DashboardPath": "/hangfire",
    "EnableDashboard": true,
    "DashboardAuthenticationRequired": true,
    "WorkerCount": 5,
    "Queues": ["default", "ai-processing", "email", "reports"],
    "RetryAttempts": 3,
    "JobExpirationHours": 24
  }
}
```

#### Scheduled Jobs
```json
{
  "ScheduledJobs": {
    "WeeklyMenuReminders": {
      "Enabled": true,
      "CronExpression": "0 9 * * SUN",
      "TimeZone": "America/New_York"
    },
    "DatabaseBackup": {
      "Enabled": true,
      "CronExpression": "0 2 * * *",
      "TimeZone": "UTC"
    },
    "CleanupOldAIRequests": {
      "Enabled": true,
      "CronExpression": "0 3 * * *",
      "RetentionDays": 30
    },
    "UpdateRecipeRatings": {
      "Enabled": true,
      "CronExpression": "*/15 * * * *"
    }
  }
}
```

---

## Frontend Configuration (React)

### Environment Configuration

#### Development (.env.development)
```env
# API Configuration
REACT_APP_API_BASE_URL=https://localhost:7001/api
REACT_APP_API_VERSION=v1
REACT_APP_ENABLE_API_MOCKING=false

# Authentication
REACT_APP_AUTH_TOKEN_KEY=mealprep_auth_token
REACT_APP_REFRESH_TOKEN_KEY=mealprep_refresh_token
REACT_APP_TOKEN_REFRESH_THRESHOLD_MINUTES=5

# Application Settings
REACT_APP_APP_NAME="MealPrep Development"
REACT_APP_APP_VERSION=$npm_package_version
REACT_APP_ENABLE_DEVELOPMENT_TOOLS=true
REACT_APP_ENABLE_REDUX_DEVTOOLS=true
REACT_APP_LOG_LEVEL=debug

# Feature Flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_SOCIAL_FEATURES=false
REACT_APP_ENABLE_PREMIUM_FEATURES=false
REACT_APP_ENABLE_ANALYTICS=false

# External Services
REACT_APP_GOOGLE_ANALYTICS_ID=""
REACT_APP_SENTRY_DSN=""
REACT_APP_HOTJAR_ID=""

# UI Configuration
REACT_APP_DEFAULT_THEME=light
REACT_APP_ENABLE_DARK_MODE=true
REACT_APP_DEFAULT_PAGE_SIZE=20
REACT_APP_MAX_PAGE_SIZE=100
REACT_APP_ENABLE_INFINITE_SCROLL=true

# Recipe Configuration
REACT_APP_MAX_RECIPE_IMAGE_SIZE_MB=5
REACT_APP_SUPPORTED_IMAGE_FORMATS=jpg,jpeg,png,gif
REACT_APP_ENABLE_RECIPE_SHARING=true
REACT_APP_ENABLE_RECIPE_IMPORT=true

# AI Configuration
REACT_APP_AI_SUGGESTION_TIMEOUT_MS=30000
REACT_APP_MAX_AI_REQUESTS_PER_HOUR=50
REACT_APP_ENABLE_AI_FEEDBACK=true

# Performance
REACT_APP_ENABLE_SERVICE_WORKER=true
REACT_APP_ENABLE_CODE_SPLITTING=true
REACT_APP_BUNDLE_ANALYZER=false
```

#### Production (.env.production)
```env
# API Configuration
REACT_APP_API_BASE_URL=https://api.mealprep.com/api
REACT_APP_API_VERSION=v1
REACT_APP_ENABLE_API_MOCKING=false

# Authentication
REACT_APP_AUTH_TOKEN_KEY=mealprep_auth_token
REACT_APP_REFRESH_TOKEN_KEY=mealprep_refresh_token
REACT_APP_TOKEN_REFRESH_THRESHOLD_MINUTES=5

# Application Settings
REACT_APP_APP_NAME="MealPrep"
REACT_APP_APP_VERSION=$npm_package_version
REACT_APP_ENABLE_DEVELOPMENT_TOOLS=false
REACT_APP_ENABLE_REDUX_DEVTOOLS=false
REACT_APP_LOG_LEVEL=error

# Feature Flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_SOCIAL_FEATURES=true
REACT_APP_ENABLE_PREMIUM_FEATURES=true
REACT_APP_ENABLE_ANALYTICS=true

# External Services
REACT_APP_GOOGLE_ANALYTICS_ID=GA_MEASUREMENT_ID
REACT_APP_SENTRY_DSN=https://your-sentry-dsn@sentry.io/project-id
REACT_APP_HOTJAR_ID=your_hotjar_id

# Performance
REACT_APP_ENABLE_SERVICE_WORKER=true
REACT_APP_ENABLE_CODE_SPLITTING=true
REACT_APP_BUNDLE_ANALYZER=false
```

### Build Configuration

#### Webpack Customization (craco.config.js)
```javascript
const path = require('path');

module.exports = {
  webpack: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components'),
      '@pages': path.resolve(__dirname, 'src/pages'),
      '@services': path.resolve(__dirname, 'src/services'),
      '@utils': path.resolve(__dirname, 'src/utils'),
      '@types': path.resolve(__dirname, 'src/types'),
      '@hooks': path.resolve(__dirname, 'src/hooks'),
      '@contexts': path.resolve(__dirname, 'src/contexts')
    },
    configure: (webpackConfig, { env, paths }) => {
      // Enable source maps in production for better debugging
      if (env === 'production') {
        webpackConfig.devtool = 'source-map';
      }

      // Bundle splitting configuration
      webpackConfig.optimization.splitChunks = {
        chunks: 'all',
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/]/,
            name: 'vendors',
            chunks: 'all',
          },
          common: {
            name: 'common',
            minChunks: 2,
            chunks: 'all',
            enforce: true,
          },
        },
      };

      return webpackConfig;
    },
  },
  devServer: {
    port: 3000,
    open: true,
    historyApiFallback: true,
    proxy: {
      '/api': {
        target: 'https://localhost:7001',
        changeOrigin: true,
        secure: false,
      },
    },
  },
  typescript: {
    enableTypeChecking: true,
  },
  eslint: {
    enable: true,
    mode: 'extends',
  },
};
```

### TypeScript Configuration (tsconfig.json)
```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": [
      "dom",
      "dom.iterable",
      "es6"
    ],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "baseUrl": "src",
    "paths": {
      "@/*": ["*"],
      "@components/*": ["components/*"],
      "@pages/*": ["pages/*"],
      "@services/*": ["services/*"],
      "@utils/*": ["utils/*"],
      "@types/*": ["types/*"],
      "@hooks/*": ["hooks/*"],
      "@contexts/*": ["contexts/*"]
    }
  },
  "include": [
    "src"
  ]
}
```

---

## Docker Configuration

### Development Docker Compose
```yaml
version: '3.8'

services:
  mealprep-api:
    build:
      context: .
      dockerfile: MealPrep.API/Dockerfile
      target: development
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Password=password
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=MealPrepDB;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;
      - ConnectionStrings__Redis=redis:6379
      - GoogleCloud__ProjectId=mealprep-dev
    ports:
      - "7001:443"
      - "5000:80"
    volumes:
      - ~/.aspnet/https:/https:ro
      - ./logs:/app/logs
    depends_on:
      - sqlserver
      - redis

  mealprep-frontend:
    build:
      context: .
      dockerfile: frontend/Dockerfile
      target: development
    environment:
      - REACT_APP_API_BASE_URL=https://localhost:7001/api
      - REACT_APP_ENABLE_DEVELOPMENT_TOOLS=true
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    command: npm start

  sqlserver:
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

volumes:
  sqlserver_data:
  redis_data:
```

### Production Docker Compose
```yaml
version: '3.8'

services:
  mealprep-api:
    build:
      context: .
      dockerfile: MealPrep.API/Dockerfile
      target: production
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=https://+:443;http://+:80
    env_file:
      - .env.production
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./certs:/https:ro
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  mealprep-frontend:
    build:
      context: .
      dockerfile: frontend/Dockerfile
      target: production
    ports:
      - "8080:80"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - mealprep-api
      - mealprep-frontend
    restart: unless-stopped
```

---

## Environment-Specific Settings

### Development Environment
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "AllowedHosts": "*",
  "DetailedErrors": true,
  "EnableSensitiveDataLogging": true,
  "EnableDeveloperExceptionPage": true,
  "Cors": {
    "AllowAnyOrigin": true,
    "AllowAnyMethod": true,
    "AllowAnyHeader": true
  },
  "SwaggerUI": {
    "Enabled": true,
    "RoutePrefix": "swagger"
  }
}
```

### Staging Environment
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  },
  "AllowedHosts": "staging.mealprep.com",
  "DetailedErrors": false,
  "EnableSensitiveDataLogging": false,
  "EnableDeveloperExceptionPage": false,
  "Cors": {
    "AllowedOrigins": ["https://staging-app.mealprep.com"]
  },
  "SwaggerUI": {
    "Enabled": true,
    "RoutePrefix": "api-docs"
  }
}
```

### Production Environment
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Error",
      "Microsoft.EntityFrameworkCore": "Error"
    }
  },
  "AllowedHosts": "api.mealprep.com",
  "DetailedErrors": false,
  "EnableSensitiveDataLogging": false,
  "EnableDeveloperExceptionPage": false,
  "Cors": {
    "AllowedOrigins": ["https://app.mealprep.com"]
  },
  "SwaggerUI": {
    "Enabled": false
  },
  "ForwardedHeaders": {
    "ForwardedProtoHeaderName": "X-Forwarded-Proto",
    "ForwardedForHeaderName": "X-Forwarded-For"
  }
}
```

---

## Configuration Validation

### Startup Configuration Validation
```csharp
public class ConfigurationValidator
{
    public static void ValidateConfiguration(IConfiguration configuration)
    {
        ValidateRequiredSettings(configuration);
        ValidateConnectionStrings(configuration);
        ValidateGoogleCloudSettings(configuration);
        ValidateAuthenticationSettings(configuration);
    }

    private static void ValidateRequiredSettings(IConfiguration configuration)
    {
        var requiredSettings = new[]
        {
            "ConnectionStrings:DefaultConnection",
            "Authentication:JwtSettings:SecretKey",
            "GoogleCloud:ProjectId"
        };

        foreach (var setting in requiredSettings)
        {
            if (string.IsNullOrEmpty(configuration[setting]))
            {
                throw new InvalidOperationException($"Required configuration setting '{setting}' is missing or empty.");
            }
        }
    }
}
```

### Configuration Model Classes
```csharp
public class JwtSettings
{
    public string SecretKey { get; set; }
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public int ExpirationInHours { get; set; }
    public int RefreshTokenExpirationInDays { get; set; }
    public bool RequireHttpsMetadata { get; set; }
    public bool ValidateIssuerSigningKey { get; set; }
    public bool ValidateIssuer { get; set; }
    public bool ValidateAudience { get; set; }
    public bool ValidateLifetime { get; set; }
    public TimeSpan ClockSkew { get; set; }
}

public class GoogleCloudSettings
{
    public string ProjectId { get; set; }
    public string ServiceAccountKeyPath { get; set; }
    public string Location { get; set; }
    public ModelConfig ModelConfig { get; set; }
    public RateLimitingConfig RateLimiting { get; set; }
}

public class ModelConfig
{
    public string ModelName { get; set; }
    public float Temperature { get; set; }
    public float TopP { get; set; }
    public int TopK { get; set; }
    public int MaxOutputTokens { get; set; }
    public SafetySetting[] SafetySettings { get; set; }
}
```

---

## Security Configuration

### Secrets Management

#### Azure Key Vault Integration
```json
{
  "KeyVault": {
    "Enabled": true,
    "VaultUri": "https://mealprep-keyvault.vault.azure.net/",
    "ClientId": "your-client-id",
    "ClientSecret": "your-client-secret",
    "TenantId": "your-tenant-id",
    "Secrets": [
      "database-connection-string",
      "jwt-secret-key",
      "google-cloud-service-account",
      "sendgrid-api-key"
    ]
  }
}
```

#### Development User Secrets
```bash
# Initialize user secrets
dotnet user-secrets init

# Set sensitive configuration
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=MealPrepDB;Trusted_Connection=true;"
dotnet user-secrets set "Authentication:JwtSettings:SecretKey" "your-development-secret-key"
dotnet user-secrets set "GoogleCloud:ServiceAccountKeyPath" "/path/to/dev-service-account.json"
```

### HTTPS Configuration

#### Certificate Settings
```json
{
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://0.0.0.0:5000"
      },
      "Https": {
        "Url": "https://0.0.0.0:5001",
        "Certificate": {
          "Path": "/path/to/certificate.pfx",
          "Password": "certificate-password"
        }
      }
    }
  }
}
```

---

## Monitoring and Observability Configuration

### Application Insights
```json
{
  "ApplicationInsights": {
    "InstrumentationKey": "your-instrumentation-key",
    "ConnectionString": "InstrumentationKey=your-key;IngestionEndpoint=https://your-region.in.applicationinsights.azure.com/",
    "EnableAdaptiveSampling": true,
    "SamplingPercentage": 100,
    "EnableHeartbeat": true,
    "EnableDependencyTracking": true,
    "EnablePerformanceCounterCollectionModule": true,
    "EnableEventCounterCollectionModule": true,
    "EnableSqlCommandTextInstrumentation": false
  }
}
```

### Metrics Configuration
```json
{
  "Metrics": {
    "Enabled": true,
    "Prometheus": {
      "Enabled": true,
      "Port": 9090,
      "Path": "/metrics"
    },
    "CustomMetrics": {
      "RecipeCreationRate": true,
      "AIRequestLatency": true,
      "UserEngagementMetrics": true,
      "ErrorRates": true
    }
  }
}
```

---

*Last Updated: December 2024*  
*Configuration reference is continuously updated with new settings and options*