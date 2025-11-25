# Porównanie: build.yml vs build-and-deploy.yml

## build.yml (NOWY - CI only)

**Przeznaczenie**: Tylko CI - build, test, push do ACR
**Używany w**: Migracji GitOps (Faza 1+)

### Co robi:
1. ✅ Maven build + testy
2. ✅ Docker build
3. ✅ Push do ACR (tylko na push do main)
4. ✅ Output: `image_tag` (SHORT_SHA)

### Czego NIE robi:
- ❌ Deployment do AKS (to robi GitOps workflow w osobnym repo)

---

## build-and-deploy.yml (STARY - CI+CD w jednym)

**Przeznaczenie**: CI + CD w jednym workflow
**Używany w**: Przed migracją GitOps (legacy)

### Co robi:
1. ✅ Maven build + testy
2. ✅ Docker build
3. ✅ Push do ACR
4. ✅ **Deployment do AKS** (helm upgrade)

---

## Kluczowe różnice

| Aspekt | build.yml (nowy) | build-and-deploy.yml (stary) |
|--------|------------------|------------------------------|
| **Linie kodu** | 113 | 150 |
| **Maven build** | ✅ | ✅ |
| **Docker build+push** | ✅ | ✅ |
| **Deploy do AKS** | ❌ (robi GitOps) | ✅ |
| **Output image_tag** | ✅ | ❌ |
| **Secrets AKS** | ❌ (nie potrzebne) | ✅ (wymagane) |

---

## Migracja

### Faza 1-2 (Dual mode)
- Oba workflow istnieją
- Serwisy mogą używać któregokolwiek
- `build-and-deploy.yml` jako fallback

### Faza 3+ (Po migracji)
- `build.yml` - aktywny
- `build-and-deploy.yml` - deprecated (może być usunięty)

---

## Zachowane elementy (IDENTYCZNE w obu)

✅ `env.ACR_NAME` i `env.ACR_LOGIN_SERVER` - **POPRAWKA PO REVIEW**
✅ Logika PR vs Push (`github.event_name`)
✅ Cache Maven
✅ OIDC do Azure
✅ Tagowanie `sha` + `latest`


