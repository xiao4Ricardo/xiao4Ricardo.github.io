---
layout:     post
title:      "CI/CD Pipeline 最佳实践"
subtitle:   "Building Reliable Continuous Integration and Deployment Pipelines"
date:       2026-01-22 09:00:00
author:     "Tony L."
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - DevOps
    - CI/CD
    - Software Engineering
    - GitHub Actions
---

## 为什么 CI/CD 如此重要？

CI/CD 不是可选的奢侈品，而是现代软件开发的基础设施。没有 CI/CD 的团队就像没有自动化测试的代码——也许现在能跑，但迟早会出问题。

**CI（持续集成）**：每次代码提交自动触发构建和测试，尽早发现问题。

**CD（持续部署/交付）**：将通过测试的代码自动部署到生产环境（持续部署）或准备好待手动发布（持续交付）。

## GitHub Actions 实战

以一个 Spring Boot + Vue 3 项目为例，构建完整的 CI/CD Pipeline。

### Backend CI

```yaml
# .github/workflows/backend-ci.yml
name: Backend CI

on:
  push:
    branches: [main, develop]
    paths: ['backend/**']
  pull_request:
    branches: [main]
    paths: ['backend/**']

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Run tests
        working-directory: backend
        run: ./gradlew test
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: testdb
          DB_USER: test
          DB_PASSWORD: test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: backend/build/reports/tests/
```

### Frontend CI

```yaml
# .github/workflows/frontend-ci.yml
name: Frontend CI

on:
  push:
    branches: [main, develop]
    paths: ['frontend/**']
  pull_request:
    branches: [main]
    paths: ['frontend/**']

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: frontend
        run: npm ci

      - name: Type check
        working-directory: frontend
        run: npx vue-tsc --noEmit

      - name: Lint
        working-directory: frontend
        run: npm run lint

      - name: Unit tests
        working-directory: frontend
        run: npx vitest run --coverage

      - name: Build
        working-directory: frontend
        run: npm run build
```

## Pipeline 设计原则

### 1. 快速反馈

Pipeline 的第一个原则是**速度**。开发者提交代码后应该在几分钟内得到反馈。

优化策略：
- **并行执行**：lint、type-check、unit-test 可以并行跑
- **增量构建**：只构建变更部分（monorepo 用 path filter）
- **缓存依赖**：npm cache、Gradle cache、Docker layer cache
- **分层测试**：快速测试先跑，慢速测试后跑

### 2. 构建一次，部署多处

同一个构建产物（Docker image / JAR）部署到所有环境（dev → staging → production），只改变配置。

```yaml
build:
  steps:
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
        docker push registry/myapp:${{ github.sha }}

deploy-staging:
  needs: build
  steps:
    - name: Deploy to staging
      run: kubectl set image deployment/myapp
           myapp=registry/myapp:${{ github.sha }}
           -n staging

deploy-production:
  needs: deploy-staging
  environment: production  # 需要审批
  steps:
    - name: Deploy to production
      run: kubectl set image deployment/myapp
           myapp=registry/myapp:${{ github.sha }}
           -n production
```

### 3. 环境一致性

"在我电脑上能跑"是最经典的问题。Docker 解决了这个问题：

```dockerfile
# Multi-stage build
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew bootJar

FROM eclipse-temurin:17-jre
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4. 安全扫描

将安全扫描集成到 Pipeline 中：

```yaml
security:
  steps:
    - name: Dependency check
      run: ./gradlew dependencyCheckAnalyze

    - name: SAST scan
      uses: github/codeql-action/analyze@v3

    - name: Secret scan
      uses: trufflesecurity/trufflehog@main
```

### 5. 数据库迁移

数据库变更应该是 Pipeline 的一部分：

```yaml
migrate:
  steps:
    - name: Run Flyway migration
      run: ./gradlew flywayMigrate
      env:
        FLYWAY_URL: jdbc:postgresql://db:5432/myapp
```

## 监控与回滚

### 部署后验证

```yaml
post-deploy:
  steps:
    - name: Health check
      run: |
        for i in {1..30}; do
          if curl -s http://myapp/actuator/health | grep -q UP; then
            echo "Health check passed"
            exit 0
          fi
          sleep 2
        done
        echo "Health check failed"
        exit 1

    - name: Smoke test
      run: |
        curl -f http://myapp/api/ping || exit 1
```

### 自动回滚

```yaml
    - name: Rollback on failure
      if: failure()
      run: kubectl rollout undo deployment/myapp -n production
```

## 常见问题与解决方案

| 问题 | 解决方案 |
|------|----------|
| Pipeline 太慢 | 并行化 + 缓存 + 增量构建 |
| Flaky tests | 隔离测试环境 + 重试机制 + 定期清理 |
| Secret 泄露 | GitHub Secrets + 环境变量注入 |
| 依赖冲突 | Lock file + 定期更新 + Dependabot |
| 环境不一致 | Docker + Infrastructure as Code |

## 总结

好的 CI/CD Pipeline 应该是：快速的、可靠的、安全的、可观测的。它不需要一步到位——从最基本的"提交触发测试"开始，逐步添加代码检查、安全扫描、自动部署和监控。最重要的是**开始做**，然后持续改进。

## References

1. Humble, J. & Farley, D. (2010). "Continuous Delivery." Addison-Wesley.
2. GitHub Actions Documentation. https://docs.github.com/en/actions
3. Fowler, M. (2006). "Continuous Integration." martinfowler.com.
