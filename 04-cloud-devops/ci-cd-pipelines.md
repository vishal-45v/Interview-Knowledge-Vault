# CI/CD Pipelines for Spring Boot

---

## GitHub Actions Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: mycompany/myapp

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: maven
    
    - name: Run tests
      run: mvn test -Dspring.profiles.active=test
      env:
        SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
        SPRING_DATASOURCE_PASSWORD: test
    
    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: test-results
        path: target/surefire-reports/

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: maven
    
    - name: Build JAR
      run: mvn package -DskipTests
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        build-args: |
          BUILD_DATE=${{ github.event.repository.updated_at }}
          GIT_COMMIT=${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - name: Deploy to ECS
      run: |
        aws ecs update-service \
          --cluster prod-cluster \
          --service myapp \
          --force-new-deployment \
          --region us-east-1
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Blue-Green Deployment Strategy

```
Current state:
  Load Balancer → Blue (v1.0) ← serving 100% traffic

Deploy v1.1 to Green environment:
  Blue (v1.0) ← serving 100% traffic
  Green (v1.1) ← deployed, not serving traffic

Run smoke tests on Green:
  curl https://green.internal/api/health → 200 OK

Switch traffic:
  Load Balancer → Green (v1.1) ← serving 100% traffic
  Blue (v1.0) ← standby (for quick rollback)

Rollback (if needed):
  Load Balancer → Blue (v1.0) ← restored in seconds
```

---

## Canary Deployment Strategy

```
Phase 1: 5% of traffic to new version
  95%  → v1.0  (stable)
  5%   → v1.1  (canary)
  Monitor: error rates, latency, business metrics
  
Phase 2: 25% if metrics look good
Phase 3: 50%
Phase 4: 100% (full rollout)

Implementation in Kubernetes with Argo Rollouts:
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 10m}
      - setWeight: 25
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 5m}
```

---

## Pipeline Best Practices

1. **Fail fast**: Run linting and unit tests first, integration tests after
2. **Parallel stages**: Run independent jobs in parallel (unit test, security scan, etc.)
3. **Artifact immutability**: Build once, deploy many (same artifact to staging and production)
4. **Environment promotion**: Dev → Staging → Production (never rebuild between environments)
5. **Secrets management**: Never hardcode secrets, use secret stores
6. **Rollback strategy**: Always have a way to rollback, test it regularly
