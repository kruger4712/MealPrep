# Development Tools Guide

## Overview
Essential development tools, IDE configurations, and utilities for efficient MealPrep development across the full stack including AI integration.

## Required Software

### Core Development Environment

#### .NET Development
```bash
# .NET 8 SDK (Latest LTS)
# Download: https://dotnet.microsoft.com/download/dotnet/8.0
dotnet --version  # Should show 8.0.x

# Verify installation
dotnet --list-sdks
dotnet --list-runtimes
```

#### Node.js & npm
```bash
# Node.js 18+ (LTS recommended)
# Download: https://nodejs.org/
node --version   # Should show v18.x.x or higher
npm --version    # Should show 9.x.x or higher

# Alternative: Use Node Version Manager
nvm install 18
nvm use 18
```

#### Database Systems
```bash
# SQL Server 2022 (Windows/Linux)
# Download: https://www.microsoft.com/en-us/sql-server/sql-server-downloads

# OR MySQL 8.0+
# Download: https://dev.mysql.com/downloads/mysql/

# Connection test
sqlcmd -S localhost -E -Q "SELECT @@VERSION"  # SQL Server
mysql -u root -p -e "SELECT VERSION()"        # MySQL
```

#### Containerization
```bash
# Docker Desktop
# Download: https://www.docker.com/products/docker-desktop
docker --version
docker-compose --version

# Verify Docker is running
docker run hello-world
```

#### Version Control
```bash
# Git (Latest version)
# Download: https://git-scm.com/
git --version

# Configure Git for MealPrep
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

### AI Development Tools

#### Google Cloud SDK
```bash
# Google Cloud CLI
# Download: https://cloud.google.com/sdk/docs/install
gcloud --version

# Authenticate and configure project
gcloud auth login
gcloud config set project your-mealprep-project-id
gcloud auth application-default login
```

#### Python (for AI scripts and tools)
```bash
# Python 3.9+ for AI utilities
# Download: https://www.python.org/downloads/
python --version  # or python3 --version

# Install AI development packages
pip install google-cloud-aiplatform openai anthropic
```

## Primary IDEs

### Visual Studio 2022 (Recommended for .NET)

#### Installation
- **Workloads**: ASP.NET and web development, .NET desktop development
- **Individual Components**: 
  - .NET 8.0 Runtime
  - .NET Framework 4.8 targeting pack
  - Node.js development tools
  - Git for Windows

#### Essential Extensions
```bash
# Install via Extensions Manager or command line
# Visual Studio Marketplace extensions:

# Code Quality & Analysis
- SonarLint for Visual Studio 2022
- Roslynator 2022
- CodeMaid VS2022

# Productivity
- GitHub Copilot
- GitLens — Git supercharged
- Bracket Pair Colorizer
- Output enhancer

# AI & ML Development
- Python Tools for Visual Studio (PTVS)
- Azure AI Tools

# Database
- SQL Server Data Tools (SSDT)
- MySQL for Visual Studio
```

#### Configuration
```json
// Visual Studio settings (Tools > Options)
{
  "TextEditor": {
    "AllLanguages": {
      "TabSize": 4,
      "IndentSize": 4,
      "InsertTabs": false
    },
    "CSharp": {
      "CodeStyle": {
        "Formatting": {
          "NewLines": {
            "PlaceOpenBraceOnNewLineForMethods": true,
            "PlaceOpenBraceOnNewLineForTypes": true
          }
        }
      }
    }
  },
  "Debugging": {
    "General": {
      "EnableJustMyCode": false,
      "SuppressJITOptimizations": true
    }
  }
}
```

### Visual Studio Code (Recommended for Frontend)

#### Essential Extensions
```bash
# Install via VS Code Extensions or command line
code --install-extension ms-dotnettools.csharp
code --install-extension ms-dotnettools.vscode-dotnet-runtime
code --install-extension bradlc.vscode-tailwindcss
code --install-extension esbenp.prettier-vscode
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension eamodio.gitlens
code --install-extension github.copilot
code --install-extension ms-vscode.vscode-json
code --install-extension ms-python.python
code --install-extension googlecloudtools.cloudcode

# React & TypeScript
code --install-extension msjsdiag.vscode-react-native
code --install-extension bradlc.vscode-tailwindcss
code --install-extension steoates.autoimport-es6-ts
code --install-extension formulahendry.auto-rename-tag

# Database
code --install-extension ms-mssql.mssql
code --install-extension cweijan.vscode-mysql-client2

# Testing & Quality
code --install-extension ms-vscode.test-adapter-converter
code --install-extension orta.vscode-jest
code --install-extension ms-playwright.playwright

# Docker & DevOps
code --install-extension ms-vscode-remote.remote-containers
code --install-extension ms-azuretools.vscode-docker
```

#### VS Code Configuration
```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.organizeImports": true
  },
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.wordWrap": "wordWrapColumn",
  "editor.wordWrapColumn": 100,
  "files.exclude": {
    "**/node_modules": true,
    "**/bin": true,
    "**/obj": true
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "emmet.includeLanguages": {
    "typescript": "html",
    "typescriptreact": "html"
  },
  "csharp.format.enable": true,
  "omnisharp.enableRoslynAnalyzers": true,
  "dotnet.completion.showCompletionItemsFromUnimportedNamespaces": true
}
```

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": ".NET Core Launch (web)",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build",
      "program": "${workspaceFolder}/MealPrep.API/bin/Debug/net8.0/MealPrep.API.dll",
      "args": [],
      "cwd": "${workspaceFolder}/MealPrep.API",
      "stopAtEntry": false,
      "serverReadyAction": {
        "action": "openExternally",
        "pattern": "\\bNow listening on:\\s+(https?://\\S+)"
      },
      "env": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      },
      "sourceFileMap": {
        "/Views": "${workspaceFolder}/Views"
      }
    },
    {
      "name": "Launch React App",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceFolder}/meal-prep-frontend",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["start"]
    }
  ]
}
```

## Database Tools

### SQL Server Management Studio (SSMS)
```bash
# Download: https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms
# Features needed:
- Query execution and optimization
- Database design and diagramming
- Index management and analysis
- Backup and restore operations
```

### MySQL Workbench
```bash
# Download: https://www.mysql.com/products/workbench/
# Configuration for MealPrep:
- Connection: localhost:3306
- Schema: mealprep_dev, mealprep_test, mealprep_prod
```

### Database GUI Alternatives
```bash
# Azure Data Studio (Cross-platform)
# Download: https://docs.microsoft.com/en-us/sql/azure-data-studio/

# DBeaver (Universal database tool)
# Download: https://dbeaver.io/

# DataGrip (JetBrains - Paid)
# Download: https://www.jetbrains.com/datagrip/
```

## API Development Tools

### Postman
```bash
# Download: https://www.postman.com/downloads/
# MealPrep API Collection features:
- Environment variables for dev/staging/prod
- Authentication token management
- Automated testing scripts
- API documentation generation
```

#### Postman Configuration
```javascript
// Pre-request Script for Authentication
pm.sendRequest({
    url: pm.environment.get("base_url") + "/api/auth/login",
    method: 'POST',
    header: {
        'Content-Type': 'application/json',
    },
    body: {
        mode: 'raw',
        raw: JSON.stringify({
            email: pm.environment.get("test_email"),
            password: pm.environment.get("test_password")
        })
    }
}, function (err, response) {
    if (response.json().token) {
        pm.environment.set("auth_token", response.json().token);
    }
});
```

### Swagger/OpenAPI Tools
```bash
# Swagger Editor (Online)
# URL: https://editor.swagger.io/

# Swagger Codegen
npm install -g swagger-codegen-cli

# Generate client SDK
swagger-codegen generate -i http://localhost:7000/swagger/v1/swagger.json -l typescript-fetch -o ./generated-client
```

### REST Client (VS Code Extension)
```http
### MealPrep API Testing
@baseUrl = https://localhost:7001/api
@authToken = your-jwt-token-here

### Login
POST {{baseUrl}}/auth/login
Content-Type: application/json

{
  "email": "test@example.com",
  "password": "TestPassword123!"
}

### Get Recipes
GET {{baseUrl}}/recipes
Authorization: Bearer {{authToken}}

### Generate AI Meal Suggestions
POST {{baseUrl}}/ai/meal-suggestions
Authorization: Bearer {{authToken}}
Content-Type: application/json

{
  "familyMembers": [1, 2],
  "mealType": "dinner",
  "budget": 25.00
}
```

## Frontend Development Tools

### Package Managers
```bash
# npm (comes with Node.js)
npm --version

# Yarn (alternative package manager)
npm install -g yarn
yarn --version

# Package management commands
npm install / yarn install
npm run build / yarn build
npm test / yarn test
```

### Build Tools & Bundlers
```bash
# Vite (React build tool)
npm create vite@latest meal-prep-frontend -- --template react-ts

# Webpack (if needed for advanced configuration)
npm install -g webpack webpack-cli

# ESBuild (fast bundler)
npm install -g esbuild
```

### Browser Developer Tools

#### Chrome DevTools Extensions
- **React Developer Tools**: Debug React components and hooks
- **Redux DevTools**: Monitor Redux state changes
- **Lighthouse**: Performance, accessibility, and SEO auditing
- **Web Vitals**: Core Web Vitals monitoring

#### Firefox Developer Tools
- **React DevTools**: Available as Firefox extension
- **Accessibility Inspector**: Built-in accessibility tools
- **Network Monitor**: Advanced network request analysis

## Testing Tools

### Backend Testing
```bash
# .NET Testing Tools
dotnet test
dotnet test --collect:"XPlat Code Coverage"

# Test Runners
- MSTest (built-in)
- xUnit (alternative)
- NUnit (alternative)

# Mocking Frameworks
- Moq (recommended)
- FakeItEasy (alternative)

# Code Coverage
- coverlet (cross-platform)
- dotCover (JetBrains)
```

### Frontend Testing
```bash
# Jest (JavaScript testing framework)
npm install --save-dev jest @types/jest

# React Testing Library
npm install --save-dev @testing-library/react @testing-library/jest-dom

# Playwright (E2E testing)
npm install --save-dev @playwright/test
npx playwright install

# Test commands
npm test           # Run unit tests
npm run test:e2e   # Run E2E tests
npm run test:coverage  # Run with coverage
```

### Load Testing
```bash
# k6 (load testing tool)
# Download: https://k6.io/docs/getting-started/installation/
k6 run load-test.js

# Artillery (alternative)
npm install -g artillery
artillery quick --count 100 --num 10 http://localhost:7000/api/recipes
```

## AI Development Tools

### Google Cloud AI Platform
```bash
# Google Cloud CLI setup for AI
gcloud auth application-default login
gcloud config set project your-mealprep-project

# Vertex AI SDK
pip install google-cloud-aiplatform

# Test AI connection
python -c "from google.cloud import aiplatform; print('AI Platform SDK ready')"
```

### AI Model Testing Tools
```bash
# Python environment for AI testing
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

# Install AI testing dependencies
pip install jupyter notebook pandas matplotlib requests
```

#### Jupyter Notebook for AI Testing
```python
# ai-testing.ipynb
import requests
import json
import pandas as pd

# Test meal suggestion API
def test_meal_suggestions():
    url = "http://localhost:7000/api/ai/meal-suggestions"
    headers = {"Authorization": f"Bearer {jwt_token}"}
    
    test_families = [
        {"familyMembers": [1, 2], "budget": 25.00},
        {"familyMembers": [1, 2, 3], "budget": 40.00}
    ]
    
    results = []
    for family in test_families:
        response = requests.post(url, headers=headers, json=family)
        results.append(response.json())
    
    return results

# Analyze suggestion quality
def analyze_suggestions(results):
    df = pd.DataFrame(results)
    print(f"Average family fit score: {df['familyFitScore'].mean()}")
    print(f"Average cost: ${df['estimatedCost'].mean():.2f}")
    return df
```

## Performance Monitoring Tools

### Application Performance Monitoring
```bash
# Application Insights (Azure)
# Install NuGet package: Microsoft.ApplicationInsights.AspNetCore

# Serilog for structured logging
# Install NuGet packages:
# - Serilog.AspNetCore
# - Serilog.Sinks.Console
# - Serilog.Sinks.File
```

### Database Performance
```bash
# SQL Server Profiler
# Built into SSMS for SQL Server performance analysis

# MySQL Performance Schema
# Built into MySQL for performance monitoring

# Entity Framework Core Logging
# Add to appsettings.json:
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

### Frontend Performance
```bash
# Web Vitals monitoring
npm install web-vitals

# Bundle analyzer
npm install --save-dev webpack-bundle-analyzer

# Lighthouse CI
npm install -g @lhci/cli
```

## Code Quality Tools

### Backend Code Quality
```bash
# SonarQube (self-hosted)
# Download: https://www.sonarqube.org/downloads/
# Configure for .NET projects

# StyleCop analyzers
# Install NuGet package: StyleCop.Analyzers

# .editorconfig for consistent formatting
# Already configured in project root
```

### Frontend Code Quality
```bash
# ESLint for code linting
npm install --save-dev eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin

# Prettier for code formatting
npm install --save-dev prettier

# Husky for git hooks
npm install --save-dev husky lint-staged

# Configure package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,css,md}": ["prettier --write"]
  }
}
```

## DevOps & CI/CD Tools

### Container Development
```bash
# Docker Compose for local development
# File: docker-compose.dev.yml in project root

# Build and run development environment
docker-compose -f docker-compose.dev.yml up -d

# View logs
docker-compose logs -f api
docker-compose logs -f frontend
```

### Cloud Development
```bash
# Azure CLI (if using Azure)
# Download: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
az --version
az login

# AWS CLI (if using AWS)
# Download: https://aws.amazon.com/cli/
aws --version
aws configure
```

## Utility Tools

### Text Editors & Utilities
```bash
# Notepad++ (Windows)
# Download: https://notepad-plus-plus.org/

# Sublime Text (Cross-platform)
# Download: https://www.sublimetext.com/

# JSON formatters and validators
# Online: https://jsonformatter.curiousconcept.com/
# VS Code extension: JSON Tools
```

### Network & API Tools
```bash
# cURL for command-line API testing
curl --version

# HTTPie (user-friendly HTTP client)
pip install httpie

# Fiddler (HTTP debugging proxy)
# Download: https://www.telerik.com/fiddler
```

### File Comparison & Merging
```bash
# Beyond Compare (Paid)
# Download: https://www.scootersoftware.com/

# WinMerge (Windows, Free)
# Download: https://winmerge.org/

# Meld (Cross-platform, Free)
# Download: https://meldmerge.org/
```

## Team Collaboration Tools

### Communication
- **Slack/Microsoft Teams**: Team communication
- **Discord**: Informal developer chat
- **Zoom/Google Meet**: Video conferences and pair programming

### Project Management
- **Jira**: Issue tracking and sprint planning
- **Trello**: Simple kanban boards
- **Azure DevOps**: Integrated DevOps platform
- **GitHub Projects**: Git-integrated project management

### Documentation
- **Confluence**: Team documentation wiki
- **Notion**: All-in-one workspace
- **GitBook**: Developer documentation
- **Markdown editors**: Typora, Mark Text

## Quick Setup Script

### Windows PowerShell Setup
```powershell
# install-mealprep-tools.ps1
# Run as Administrator

# Chocolatey package manager
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Install development tools
choco install -y git nodejs dotnet-8.0-sdk docker-desktop vscode visualstudio2022professional
choco install -y sql-server-management-studio postman googlechrome firefox

# Install optional tools
choco install -y notepadplusplus 7zip curl

Write-Host "MealPrep development tools installed successfully!"
Write-Host "Please restart your computer and configure the tools according to the documentation."
```

### macOS Setup Script
```bash
#!/bin/bash
# install-mealprep-tools.sh

# Homebrew package manager
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install development tools
brew install git node dotnet docker
brew install --cask visual-studio-code postman google-chrome firefox
brew install --cask docker

# Install optional tools
brew install curl wget jq
brew install --cask sublime-text

echo "MealPrep development tools installed successfully!"
echo "Please configure the tools according to the documentation."
```

## Troubleshooting

### Common Installation Issues

#### .NET SDK Issues
```bash
# Clear NuGet cache
dotnet nuget locals all --clear

# Repair .NET installation
# Re-download and reinstall from official site

# Verify installation
dotnet --info
```

#### Node.js Issues
```bash
# Clear npm cache
npm cache clean --force

# Reset npm registry
npm config set registry https://registry.npmjs.org/

# Reinstall Node.js from official site
```

#### Docker Issues
```bash
# Reset Docker Desktop
# Docker Desktop > Troubleshoot > Reset to factory defaults

# Check Docker daemon
docker info
docker version
```

### Performance Optimization

#### IDE Performance
- Increase IDE memory allocation
- Disable unnecessary extensions
- Exclude large folders from indexing
- Use SSD for faster file access

#### Development Environment
- Use Docker for consistent environments
- Implement hot reload for faster development
- Use incremental compilation
- Enable parallel builds

This comprehensive development tools guide ensures all team members have consistent, efficient development environments for building the MealPrep application.

*This guide should be updated as new tools are adopted and configurations evolve.*