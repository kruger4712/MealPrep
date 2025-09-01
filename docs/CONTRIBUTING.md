# Contributing to MealPrep

Welcome to the MealPrep project! We're excited that you're interested in contributing to our AI-powered family meal preparation application.

## ?? Quick Start

### Prerequisites
- .NET 8 SDK
- Node.js 18+
- Docker Desktop
- Visual Studio 2022 or VS Code

### Technology Stack
**Backend:** C# ASP.NET Core 8, Entity Framework Core, Google Gemini AI
**Frontend:** React 18 with TypeScript, Material-UI, React Query
**Database:** MySQL 8.0 or SQL Server 2022

## Development Setup

### 1. Fork and Clone
```bash
git clone https://github.com/yourusername/MealPrep.git
cd MealPrep
git remote add upstream https://github.com/originalowner/MealPrep.git
```

### 2. Backend Setup
```bash
cd MealPrep.API
dotnet restore
dotnet user-secrets set "GoogleCloud:ProjectId" "your-project-id"
dotnet ef database update
dotnet run
```

### 3. Frontend Setup
```bash
cd meal-prep-frontend
npm install
echo "REACT_APP_API_URL=https://localhost:7000" > .env.local
npm start
```

## Coding Standards

### C# Backend
- Use PascalCase for classes and methods
- Use camelCase with underscore prefix for fields (`_aiService`)
- Add XML documentation for public APIs
- Follow async/await patterns consistently

### TypeScript Frontend
- Components: PascalCase (`AiSuggestions`)
- Variables/Functions: camelCase (`generateSuggestions`)
- Use TypeScript strictly, avoid `any`
- Follow React functional component patterns

## Testing Requirements
- Minimum 80% code coverage for new code
- Unit tests for all business logic
- Integration tests for API endpoints
- Component tests for React components

## Pull Request Process

1. Create feature branch from `main`
2. Make changes following coding standards
3. Add/update tests
4. Update documentation
5. Submit PR with descriptive title and description

### PR Template
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] All tests pass
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No new warnings
```

## Issue Reporting

### Bug Reports
Include:
- Environment details (OS, .NET version, browser)
- Steps to reproduce
- Expected vs actual behavior
- Screenshots/error messages

### Feature Requests
Include:
- Problem description
- Proposed solution
- Implementation considerations
- Impact on existing features

## Community Guidelines

- Be respectful and inclusive
- Use welcoming language
- Focus on constructive feedback
- Help other contributors

## Getting Help

1. Check documentation in `docs/` folder
2. Search existing issues
3. Create GitHub Discussion
4. Join community channels

Thank you for contributing to MealPrep! ????