# Git Workflow Guide

## Overview
Git branching strategy, workflow processes, and best practices for the MealPrep project to ensure clean code history, effective collaboration, and streamlined release management.

## Branching Strategy

### GitFlow Model
We use a modified GitFlow strategy optimized for continuous deployment and AI-powered meal planning features:

```
main (production)
??? develop (integration)
?   ??? feature/ai-meal-suggestions
?   ??? feature/family-persona-modeling
?   ??? feature/recipe-personalization
?   ??? feature/weekly-menu-generation
??? release/v1.2.0
??? hotfix/ai-service-timeout-fix
??? docs/update-api-documentation
```

### Branch Types

#### Main Branches

**main**
- **Purpose**: Production-ready code that's deployed to users
- **Protection**: Protected branch, requires PR approval and status checks
- **Deployment**: Auto-deploys to production environment
- **Merge Sources**: Only from `release` and `hotfix` branches
- **Naming**: Always `main`

**develop**
- **Purpose**: Integration branch where features come together
- **Protection**: Protected branch, requires PR approval
- **Deployment**: Auto-deploys to staging environment for testing
- **Merge Sources**: From `feature`, `docs`, and `bugfix` branches
- **Naming**: Always `develop`

#### Supporting Branches

**feature/**
- **Purpose**: New features and enhancements
- **Naming Convention**: `feature/short-description` or `feature/JIRA-123-description`
- **Examples**:
  - `feature/ai-meal-suggestions`
  - `feature/family-member-management`
  - `feature/recipe-rating-system`
- **Base Branch**: Created from `develop`
- **Merge Target**: `develop` via Pull Request
- **Lifetime**: Deleted after successful merge

**release/**
- **Purpose**: Prepare new production release
- **Naming Convention**: `release/vMAJOR.MINOR.PATCH`
- **Examples**: `release/v1.2.0`, `release/v2.0.0`
- **Base Branch**: Created from `develop`
- **Merge Targets**: Both `main` and `develop`
- **Lifetime**: Deleted after release is deployed

**hotfix/**
- **Purpose**: Critical production fixes that can't wait for next release
- **Naming Convention**: `hotfix/critical-issue-description`
- **Examples**: `hotfix/ai-service-crash`, `hotfix/login-security-fix`
- **Base Branch**: Created from `main`
- **Merge Targets**: Both `main` and `develop`
- **Lifetime**: Deleted after fix is deployed

**docs/**
- **Purpose**: Documentation updates and improvements
- **Naming Convention**: `docs/update-description`
- **Examples**: `docs/api-documentation`, `docs/deployment-guide`
- **Base Branch**: Created from `develop`
- **Merge Target**: `develop` via Pull Request
- **Lifetime**: Deleted after merge

**bugfix/**
- **Purpose**: Non-critical bug fixes for next release
- **Naming Convention**: `bugfix/issue-description`
- **Examples**: `bugfix/recipe-search-sorting`, `bugfix/ui-responsive-mobile`
- **Base Branch**: Created from `develop`
- **Merge Target**: `develop` via Pull Request
- **Lifetime**: Deleted after merge

## Workflow Processes

### Feature Development Workflow

#### 1. Start Feature Development
```bash
# Ensure you're on the latest develop
git checkout develop
git pull origin develop

# Create and switch to feature branch
git checkout -b feature/ai-meal-suggestions

# Verify you're on the correct branch
git branch
# * feature/ai-meal-suggestions
#   develop
#   main

# Set upstream for first push
git push -u origin feature/ai-meal-suggestions
```

#### 2. Development Cycle
```bash
# Make your changes and stage them
git add .

# Commit with descriptive message following conventional commits
git commit -m "feat(ai): implement family persona modeling for meal suggestions

- Add FamilyPersona model with dietary restrictions and preferences
- Implement persona-based prompt engineering for Gemini AI
- Add unit tests for persona creation and validation
- Update AI service to use family context in suggestions

Closes #123"

# Push changes to remote feature branch
git push origin feature/ai-meal-suggestions
```

#### 3. Keep Feature Branch Updated
```bash
# Regularly sync with develop to avoid merge conflicts
git checkout develop
git pull origin develop

git checkout feature/ai-meal-suggestions
git merge develop

# Alternative: Use rebase for cleaner history (advanced)
git rebase develop

# If conflicts occur during rebase
git status  # Check conflicted files
# Resolve conflicts in your editor
git add .
git rebase --continue

# Force push after rebase (only for feature branches!)
git push --force-with-lease origin feature/ai-meal-suggestions
```

#### 4. Complete Feature
```bash
# Final sync and push before creating PR
git checkout develop
git pull origin develop
git checkout feature/ai-meal-suggestions
git merge develop
git push origin feature/ai-meal-suggestions

# Create Pull Request via GitHub/Azure DevOps
# - Target branch: develop
# - Include comprehensive description
# - Link related issues
# - Add screenshots/demos for UI changes
# - Ensure all tests pass
```

### Release Workflow

#### 1. Prepare Release
```bash
# Create release branch from latest develop
git checkout develop
git pull origin develop
git checkout -b release/v1.2.0

# Update version numbers in relevant files
# - Update package.json version
# - Update appsettings.json version
# - Update assembly versions
# - Update API documentation version

# Update CHANGELOG.md with new features and fixes
# Generate release notes from commit history

git add .
git commit -m "chore(release): prepare v1.2.0 release

- Update version numbers to 1.2.0
- Update CHANGELOG.md with new features:
  * AI-powered meal suggestions with family personas
  * Enhanced recipe search with filters
  * Weekly menu planning automation
- Update API documentation for new endpoints
- Prepare release notes"

git push -u origin release/v1.2.0
```

#### 2. Release Testing and Bug Fixes
```bash
# During release testing, fix any critical bugs
git add .
git commit -m "fix(release): resolve meal suggestion timeout issue

- Increase AI service timeout from 30s to 60s
- Add better error handling for slow responses
- Improve user feedback during suggestion generation"

git push origin release/v1.2.0
```

#### 3. Deploy Release
```bash
# Merge to main for production deployment
git checkout main
git pull origin main
git merge --no-ff release/v1.2.0

# Create annotated tag for release
git tag -a v1.2.0 -m "Release version 1.2.0

New Features:
- AI-powered meal suggestions with family persona modeling
- Enhanced recipe search with advanced filters
- Weekly menu planning automation
- Family member preference management

Bug Fixes:
- Improved AI service reliability and timeout handling
- Fixed mobile responsive issues in recipe view
- Resolved duplicate ingredient suggestions

Breaking Changes:
- Updated API authentication flow (see migration guide)
"

# Push main and tags
git push origin main --tags

# Merge back to develop to keep branches in sync
git checkout develop
git pull origin develop
git merge --no-ff release/v1.2.0
git push origin develop

# Clean up release branch
git branch -d release/v1.2.0
git push origin --delete release/v1.2.0
```

### Hotfix Workflow

#### 1. Create Emergency Fix
```bash
# Create hotfix from current production (main)
git checkout main
git pull origin main
git checkout -b hotfix/ai-service-crash-fix

# Implement the critical fix
# Update version number (patch increment: 1.2.0 ? 1.2.1)
# Update CHANGELOG.md with hotfix details

git add .
git commit -m "fix(critical): resolve AI service crash on null family data

- Add null checks in GeminiAiService.GenerateSuggestions
- Improve error handling for invalid family member data
- Add fallback suggestions when AI service fails
- Add monitoring alerts for AI service failures

Fixes #456 - Critical production issue affecting 15% of users"

git push -u origin hotfix/ai-service-crash-fix
```

#### 2. Deploy Hotfix
```bash
# Merge to main for immediate production deployment
git checkout main
git merge --no-ff hotfix/ai-service-crash-fix

# Create patch release tag
git tag -a v1.2.1 -m "Hotfix version 1.2.1 - Critical AI service fix"

git push origin main --tags

# Merge to develop to keep fix in development
git checkout develop
git merge --no-ff hotfix/ai-service-crash-fix
git push origin develop

# Clean up hotfix branch
git branch -d hotfix/ai-service-crash-fix
git push origin --delete hotfix/ai-service-crash-fix
```

## Commit Message Standards

### Conventional Commits Format
We follow the Conventional Commits specification for clear, semantic commit messages:

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Commit Types
- **feat**: New feature for users
- **fix**: Bug fix for users
- **docs**: Documentation changes
- **style**: Code style changes (formatting, missing semicolons, etc.)
- **refactor**: Code refactoring without changing functionality
- **test**: Adding or updating tests
- **chore**: Maintenance tasks, dependency updates, build changes
- **perf**: Performance improvements
- **ci**: CI/CD pipeline changes
- **build**: Build system changes
- **revert**: Reverting previous commits

### Scope Examples for MealPrep
- **ai**: AI integration, Gemini service, meal suggestions
- **api**: Backend API endpoints and services
- **ui**: Frontend user interface components
- **auth**: Authentication and authorization
- **db**: Database changes, migrations, schema
- **recipe**: Recipe management features
- **family**: Family member management
- **menu**: Menu planning functionality
- **search**: Search and filtering features

### Commit Message Examples

#### Feature Commits
```bash
git commit -m "feat(ai): add family persona-based meal suggestions

- Implement FamilyPersona model with dietary preferences
- Add Gemini AI integration for personalized suggestions
- Include family fit scoring algorithm based on preferences
- Add fallback suggestions when AI service is unavailable

The AI now considers each family member's dietary restrictions,
liked/disliked ingredients, and cuisine preferences to generate
highly personalized meal suggestions.

Closes #123
Co-authored-by: Jane Developer <jane@example.com>"
```

#### Bug Fix Commits
```bash
git commit -m "fix(api): resolve recipe search timeout with large datasets

- Optimize database query with proper indexing on recipe.name
- Add query timeout configuration (30s default)
- Implement pagination for search results (20 items per page)
- Improve error handling for search service failures

Previously, searching through 1000+ recipes would timeout.
Now searches complete in under 200ms even with large datasets.

Fixes #456"
```

#### Breaking Change Commits
```bash
git commit -m "feat(auth)!: update JWT token structure for enhanced security

BREAKING CHANGE: JWT token format has changed. Clients must update
to handle new token structure with additional security claims.

- Add family context to JWT tokens for row-level security
- Implement refresh token rotation for better security
- Add device tracking for suspicious login detection
- Update token expiration to 1 hour (was 24 hours)

Migration Guide:
1. Update client token parsing to handle new claims
2. Implement refresh token handling
3. Update API calls to include device information

Closes #789"
```

#### Documentation Commits
```bash
git commit -m "docs(api): update endpoint documentation for v1.2

- Add comprehensive examples for AI suggestion endpoints
- Update authentication flow diagrams
- Include rate limiting information for all endpoints
- Add troubleshooting guide for common API errors

Updated Swagger documentation available at /api/docs"
```

## Pull Request Guidelines

### PR Creation Checklist
Before creating a pull request:

- [ ] Branch is up to date with target branch (`develop` or `main`)
- [ ] All tests are passing (unit, integration, E2E)
- [ ] Code follows project coding standards
- [ ] New features include appropriate tests
- [ ] Documentation is updated for API changes
- [ ] CHANGELOG.md is updated (for releases)
- [ ] Commit messages follow conventional format
- [ ] No merge commits in feature branch (rebase if needed)
- [ ] Large features are broken into reviewable chunks

### PR Template
```markdown
## Description
Brief description of the changes and their purpose.

### Related Issue
Closes #[issue_number]

## Type of Change
- [ ] ?? Bug fix (non-breaking change which fixes an issue)
- [ ] ? New feature (non-breaking change which adds functionality)
- [ ] ?? Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] ?? Documentation update
- [ ] ?? Chore (maintenance, dependencies, build changes)
- [ ] ? Performance improvement

## AI/ML Changes
- [ ] Prompt engineering changes
- [ ] Model parameter updates
- [ ] New AI service integrations
- [ ] Training data modifications

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] E2E tests added/updated
- [ ] Manual testing completed
- [ ] Performance testing (if applicable)
- [ ] AI suggestion quality testing (if applicable)

## Screenshots/Demos
Add screenshots, GIFs, or video demos for UI changes.

## Breaking Changes
List any breaking changes and migration instructions.

## Deployment Notes
Special deployment considerations or database migrations required.

## Checklist
- [ ] My code follows the style guidelines of this project
- [ ] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] Any dependent changes have been merged and published
```

### PR Review Process

#### For Authors
1. **Self-Review**: Review your own code before requesting review
2. **Clear Description**: Provide comprehensive PR description
3. **Link Issues**: Connect PR to related issues and requirements
4. **Add Context**: Include screenshots, demos, or additional context
5. **Respond Promptly**: Address review feedback quickly
6. **Keep Updated**: Rebase or merge latest changes from target branch

#### For Reviewers
1. **Timely Reviews**: Aim to review within 24-48 hours
2. **Constructive Feedback**: Provide specific, actionable feedback
3. **Test Locally**: Pull and test changes locally when needed
4. **Check Standards**: Verify coding standards and conventions
5. **Security Focus**: Pay special attention to security implications
6. **AI Quality**: For AI features, test suggestion quality and accuracy

## Advanced Git Techniques

### Interactive Rebase for Clean History
```bash
# Clean up commits before merging
git rebase -i HEAD~3

# In the editor, choose actions:
# pick = use commit as-is
# reword = use commit but edit message
# edit = use commit but stop for amending
# squash = use commit but combine with previous
# fixup = like squash but discard commit message
# drop = remove commit

# Example: Squash fixup commits
pick a1b2c3d feat(ai): implement meal suggestions
squash e4f5g6h fix typo in suggestion algorithm
squash h7i8j9k update tests for meal suggestions
```

### Cherry-picking for Selective Merges
```bash
# Apply specific commit to current branch
git cherry-pick a1b2c3d

# Cherry-pick range of commits
git cherry-pick a1b2c3d..e4f5g6h

# Cherry-pick with edit (to modify message)
git cherry-pick -e a1b2c3d
```

### Stashing for Context Switching
```bash
# Save current work temporarily
git stash push -m "WIP: implementing family persona feature"

# List all stashes
git stash list

# Apply and remove most recent stash
git stash pop

# Apply specific stash without removing
git stash apply stash@{1}

# Drop specific stash
git stash drop stash@{0}

# Stash only staged changes
git stash push --staged -m "staged changes for review"
```

### Bisecting for Bug Detection
```bash
# Find the commit that introduced a bug
git bisect start
git bisect bad                # Current commit has the bug
git bisect good v1.1.0       # Last known good version

# Git will checkout commits for testing
# Test each commit and mark as good or bad
git bisect good              # or git bisect bad

# When finished, Git will identify the problematic commit
git bisect reset             # Return to original state
```

## Repository Maintenance

### Branch Cleanup
```bash
# List merged branches
git branch --merged develop

# Delete merged feature branches
git branch -d feature/completed-feature

# Delete remote tracking branches for deleted remotes
git remote prune origin

# Delete remote branches
git push origin --delete feature/old-feature

# Clean up local references
git gc --prune=now
```

### Tag Management
```bash
# Create annotated tag for releases
git tag -a v1.2.0 -m "Release version 1.2.0 with AI meal suggestions"

# Push all tags to remote
git push origin --tags

# Push specific tag
git push origin v1.2.0

# List all tags
git tag -l

# Delete local tag
git tag -d v1.2.0

# Delete remote tag
git push origin --delete v1.2.0
```

### Repository Statistics
```bash
# Commit count by author
git shortlog -sn

# File change frequency
git log --pretty=format: --name-only | sort | uniq -c | sort -rg | head -10

# Code contribution by author
git log --pretty=format:"%an" | sort | uniq -c | sort -nr

# Commits per month
git log --pretty=format:"%ad" --date=format:"%Y-%m" | sort | uniq -c
```

## Troubleshooting Common Issues

### Merge Conflicts
```bash
# When merge conflicts occur
git status                   # See conflicted files

# Resolve conflicts manually or use merge tool
git mergetool

# After resolving all conflicts
git add .
git commit                   # Complete the merge

# Or abort the merge
git merge --abort
```

### Undoing Changes
```bash
# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Undo last commit (keep changes unstaged)
git reset HEAD~1

# Undo last commit (discard changes completely)
git reset --hard HEAD~1

# Undo specific file changes
git checkout -- filename

# Revert a commit (safe for shared branches)
git revert a1b2c3d
```

### Lost Commits Recovery
```bash
# View reflog to find lost commits
git reflog

# Recover lost commit
git checkout a1b2c3d
git checkout -b recovery-branch

# Or cherry-pick the lost commit
git cherry-pick a1b2c3d
```

### Large File Issues
```bash
# Remove large file from history
git filter-branch --force --index-filter \
'git rm --cached --ignore-unmatch large-file.bin' \
--prune-empty --tag-name-filter cat -- --all

# Or use BFG Repo-Cleaner (faster)
java -jar bfg.jar --strip-blobs-bigger-than 50M
```

## Workflow Automation

### Git Hooks
```bash
# Pre-commit hook to run tests
#!/bin/sh
# .git/hooks/pre-commit
npm test
dotnet test
```

### GitHub Actions Integration
```yaml
# .github/workflows/feature-branch.yml
name: Feature Branch CI
on:
  push:
    branches-ignore: [main, develop]
  pull_request:
    branches: [develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Run tests
      run: |
        dotnet test
        npm test
```

This comprehensive Git workflow ensures clean code history, effective collaboration, and reliable release management for the MealPrep project while supporting the unique needs of AI-powered meal planning development.

*This workflow should be adapted as the team grows and processes evolve.*