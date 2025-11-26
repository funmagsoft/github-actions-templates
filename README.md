# GitHub Actions Templates

Reusable workflows dla serwisów Java.

## 🔄 Workflows

### `build.yml` - CI dla serwisów Java (ZALECANY - GitOps)

Build, test i push Docker image do ACR.

**Użycie w serwisie**:
```yaml
jobs:
  build:
    uses: funmagsoft/github-actions-templates/.github/workflows/build.yml@main
    with:
      app_name: my-service
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy-to-dev:
    needs: build
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.GITOPS_PAT }}
          repository: funmagsoft/gitops
          event-type: deploy
          client-payload: |
            {
              "app_name": "my-service",
              "environment": "dev",
              "image_tag": "${{ needs.build.outputs.image_tag }}",
              "acr_server": "${{ vars.ACR_LOGIN_SERVER }}"
            }
```

**Outputs**:
- `image_tag` - Short SHA (7 znaków) zbudowanego obrazu

**Dokumentacja**: `.github/workflows/README.md`

---

### `build-and-deploy.yml` - Legacy CI/CD (DEPRECATED)

⚠️ **DEPRECATED** - Używaj `build.yml` + GitOps.

Ten workflow jest utrzymywany tylko dla serwisów nieprzenoszonych na GitOps.

---

## 📚 Dokumentacja

Pełna dokumentacja workflows: [.github/workflows/README.md](.github/workflows/README.md)

## 🔗 Powiązane repozytoria

- **GitOps**: https://github.com/funmagsoft/gitops
- **Helm Charts**: https://github.com/funmagsoft/helm
- **Przykładowe serwisy**: 
  - greeting-service
  - hello-service
