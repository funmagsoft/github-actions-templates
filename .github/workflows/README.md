# Reusable Workflows Documentation

## 📦 `build.yml` - CI Workflow for Java Services

Reusable workflow dla CI (Continuous Integration) serwisów Java Spring Boot.

### Funkcje

1. **Maven Build & Test**
   - Checkout kodu
   - Setup Java 21 (Eclipse Temurin)
   - Maven: `mvn clean package`
   - Cache: `.m2/repository`

2. **Docker Build & Push**
   - Login do Azure Container Registry (OIDC)
   - Build multi-stage Dockerfile
   - Tag: `{app_name}:{SHORT_SHA}` (7 znaków SHA)
   - Push do ACR

3. **Output**
   - `image_tag`: Short SHA zbudowanego obrazu

### Inputs

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `app_name` | string | ✅ | Nazwa aplikacji (np. `greeting-service`) |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AZURE_CLIENT_ID` | ✅ | Azure Service Principal Client ID |
| `AZURE_TENANT_ID` | ✅ | Azure Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | ✅ | Azure Subscription ID |

### Variables (GitHub Environments)

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `ACR_NAME` | ✅ | Nazwa ACR (bez `.azurecr.io`) | `hycomcminternal` |
| `ACR_LOGIN_SERVER` | ✅ | Pełny URL ACR | `hycomcminternal.azurecr.io` |

### Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `image_tag` | Short SHA (7 znaków) | `abc1234` |

### Przykład użycia

```yaml
# greeting-service/.github/workflows/cicd.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: funmagsoft/github-actions-templates/.github/workflows/build.yml@main
    with:
      app_name: greeting-service
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy-to-dev:
    name: "Deploy to Dev (GitOps)"
    needs: build
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: Trigger GitOps deployment
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.GITOPS_PAT }}
          repository: funmagsoft/gitops
          event-type: deploy
          client-payload: |
            {
              "app_name": "greeting-service",
              "environment": "dev",
              "image_tag": "${{ needs.build.outputs.image_tag }}",
              "acr_server": "${{ vars.ACR_LOGIN_SERVER }}"
            }
```

### Flow diagram

```
┌─────────────────────────────────────────────────────────────┐
│ greeting-service (repo serwisu)                              │
│                                                              │
│  Push to main                                                │
│      ↓                                                       │
│  .github/workflows/cicd.yml                                  │
│      ↓                                                       │
│  Job: build                                                  │
│      uses: github-actions-templates/build.yml                │
│      ↓                                                       │
│      ├─ Maven build + test                                   │
│      ├─ Docker build                                         │
│      ├─ Push to ACR: greeting-service:abc1234               │
│      └─ Output: image_tag = "abc1234"                       │
│      ↓                                                       │
│  Job: deploy-to-dev                                          │
│      repository_dispatch → gitops repo                       │
│      payload: { app_name, environment, image_tag }           │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ gitops (repo deploymentów)                                   │
│                                                              │
│  Trigger: repository_dispatch                                │
│      ↓                                                       │
│  .github/workflows/deploy.yml                                │
│      ↓                                                       │
│      ├─ Update apps/greeting-service/values-dev.yaml         │
│      │  (set image.tag = "abc1234")                          │
│      ├─ Git commit                                           │
│      ├─ Azure OIDC login                                     │
│      ├─ kubectl config (AKS)                                 │
│      └─ helm upgrade --install                               │
│            -n dev                                            │
│            -f values-dev.yaml                                │
│      ↓                                                       │
│  Pod restarted in AKS namespace 'dev'                        │
└─────────────────────────────────────────────────────────────┘
```

### Kiedy workflow się uruchamia?

Workflow `build.yml` jest **reusable** (`workflow_call`), więc:
- ❌ NIE uruchamia się bezpośrednio
- ✅ Jest wywoływany przez workflows w repozytoriach serwisów

W serwisie (np. `greeting-service`), warunki są dziedziczone:
- `on: push` → `if: github.event_name == 'push'` → deploy do dev
- `on: pull_request` → build & test, bez deployu

### Warunki Azure OIDC

Azure AD App Registration musi mieć **federated credentials** dla:
```
Subject: repo:funmagsoft/{service-name}:ref:refs/heads/main
Issuer: https://token.actions.githubusercontent.com
Audience: api://AzureADTokenExchange
```

Przykład (dla greeting-service):
```bash
az ad app federated-credential create \
  --id {APP_OBJECT_ID} \
  --parameters '{
    "name": "GitHubGreetingServiceMain",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:funmagsoft/greeting-service:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

---

## 🔧 Troubleshooting

### Build failuje na Maven

**Problem**: Maven dependencies nie są dostępne.

**Rozwiązanie**:
1. Sprawdź `pom.xml` - czy wszystkie dependencies są dostępne w Maven Central
2. Sprawdź network w runners (czy proxy nie blokuje)
3. Sprawdź logi: GitHub Actions → build → Maven output

### Docker push failuje

**Problem**: `unauthorized: authentication required`

**Rozwiązanie**:
1. Sprawdź czy OIDC credentials są poprawne:
   ```bash
   az ad app federated-credential list --id {APP_ID}
   ```
2. Sprawdź czy Subject match:
   - Expected: `repo:funmagsoft/greeting-service:ref:refs/heads/main`
   - W logach: "subject claim - repo:..."
3. Sprawdź czy Azure SP ma rolę `AcrPush` na ACR

### Output `image_tag` jest pusty

**Problem**: `needs.build.outputs.image_tag` jest undefined.

**Rozwiązanie**:
1. Sprawdź czy `build.yml` ma sekcję `outputs`
2. Sprawdź czy job `build` eksportuje output:
   ```yaml
   outputs:
     image_tag: ${{ steps.short-sha.outputs.sha }}
   ```

---

## 📚 Więcej informacji

- **GitOps Documentation**: https://github.com/funmagsoft/gitops
- **Helm Charts**: https://github.com/funmagsoft/helm
- **Example Service**: greeting-service, hello-service

