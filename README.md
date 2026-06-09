# github-workflows-templates

Repository pubblico contenente **steps** e **workflow riusabili** per GitHub Actions, condivisi tra tutti i progetti Pomiager.

---

## Struttura e convenzioni di naming

```
.github/workflows/
  <tool>-step-<azione>.yml          # step riusabile per uno strumento specifico
  <lang>-step-<tool>-<azione>.yml   # step riusabile specifico per tecnologia + strumento
  <lang>-workflow-<nome>.yml        # workflow riusabile (catena di step)
```

**Esempi:**
| File | Tipo | Descrizione |
|------|------|-------------|
| `dotnet-step-build-test.yml` | step | Build e test di un progetto .NET |
| `dotnet-step-docker-build-and-push-image.yml` | step | Build e push immagine Docker per progetti .NET |
| `azure-step-deploy-app-service.yml` | step | Deploy su Azure App Service via OIDC |
| `docker-step-deploy.yml` | step | Deploy di un container Docker (SSH / Kubernetes) |
| `dotnet-workflow-common.yml` | workflow | Catena common per progetti .NET (build + test) |

> I riferimenti interni tra step e workflow usano path locali senza versione (`./.github/workflows/...`).  
> I repository che usano questi template referenziano sempre una versione specifica (`@v1.0.0` o commit hash).

---

## Step disponibili

### `dotnet-step-build-test.yml`

Esegue restore, build e test di un progetto/soluzione .NET.

**Input:**

| Nome | Obbligatorio | Default | Descrizione |
|------|:---:|---------|-------------|
| `dotnet-version` | no | `10.0.x` | Versione SDK .NET |
| `working-directory` | no | `.` | Directory radice della soluzione |
| `build-configuration` | no | `Release` | Configurazione build (`Release` / `Debug`) |

**Esempio d'uso da un altro repository:**
```yaml
jobs:
  build-test:
    uses: Pomiager/github-workflows-templates/.github/workflows/dotnet-step-build-test.yml@v1.0.0
    with:
      dotnet-version: "10.0.x"
      working-directory: "src"
```

---

### `azure-step-deploy-app-service.yml`

Pubblica un progetto .NET e lo deploya su Azure App Service usando autenticazione **OIDC** (senza secrets di lunga durata).

**Input:**

| Nome | Obbligatorio | Default | Descrizione |
|------|:---:|---------|-------------|
| `environment` | **sì** | — | Nome GitHub environment (`test`, `production`, …). Abilita i reviewer obbligatori. |
| `app-name` | **sì** | — | Nome dell'Azure App Service |
| `dotnet-version` | no | `10.0.x` | Versione SDK .NET |
| `project-path` | no | `.` | Path al file `.csproj` da pubblicare |

**Secrets richiesti:**

| Nome | Descrizione |
|------|-------------|
| `azure-client-id` | Client ID dell'App Registration Azure |
| `azure-tenant-id` | Tenant ID Azure AD |
| `azure-subscription-id` | Subscription ID Azure |

**Esempio d'uso:**
```yaml
jobs:
  deploy:
    uses: Pomiager/github-workflows-templates/.github/workflows/azure-step-deploy-app-service.yml@v1.0.0
    with:
      environment: test
      app-name: mia-app-test
      project-path: src/MiaApp/MiaApp.csproj
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

### `dotnet-step-docker-build-and-push-image.yml`

Build di un'immagine Docker per un progetto .NET e push su un container registry.

**Input:**

| Nome | Obbligatorio | Default | Descrizione |
|------|:---:|---------|-------------|
| `image-name` | **sì** | — | Nome immagine senza registry (es. `my-app`) |
| `image-tag` | **sì** | — | Tag immagine (es. `test-abc1234`, `v1.2.3`) |
| `registry-url` | **sì** | — | URL registry (es. `ghcr.io/Pomiager`) |
| `dockerfile-path` | no | `./Dockerfile` | Path al Dockerfile |
| `build-context` | no | `.` | Contesto build Docker |

**Secrets richiesti:** `registry-username`, `registry-password`

**Output:** `full-image-tag` — riferimento completo dell'immagine pushata

---

### `docker-step-deploy.yml`

Deploy di un container Docker. Placeholder con tre implementazioni commentate (SSH, Azure Web App, Kubernetes).

**Input:** `environment`, `image-tag`, `service-name`  
**Secrets opzionali:** `deploy-ssh-host`, `deploy-ssh-user`, `deploy-ssh-key`

---

## Workflow disponibili

### `dotnet-workflow-common.yml`

Catena common per progetti .NET: chiama `dotnet-step-build-test.yml` internamente.  
Da usare nei progetti quando il `common.yml` locale deve semplicemente delegare al template senza personalizzazioni.

```yaml
jobs:
  common:
    uses: Pomiager/github-workflows-templates/.github/workflows/dotnet-workflow-common.yml@v1.0.0
    with:
      dotnet-version: "10.0.x"
```

---

## Nota: variabile `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`

Tutti i job di questo template impostano:

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

**Perché:** Le GitHub Actions basate su JavaScript (come `actions/checkout`, `actions/setup-dotnet`, `azure/login`) girano su un runtime Node.js fornito dal runner.  
Le versioni recenti di queste action (necessarie per .NET 10+) richiedono **Node.js 24**, ma i runner GitHub potrebbero usare di default Node 20.  
Questa variabile forza l'uso di Node 24 ed evita warning o errori di compatibilità.

---

## Setup Azure OIDC

Questo template usa **OIDC (OpenID Connect)** per autenticarsi ad Azure senza salvare credenziali di lunga durata come secrets. GitHub genera un token temporaneo a ogni esecuzione del workflow.

### 1. Crea un App Registration in Azure

1. Vai su **portal.azure.com → Microsoft Entra ID → App registrations → New registration**
2. Nome: es. `github-actions-Pomiager` (uno per ambiente, oppure uno condiviso)
3. Clicca **Register**
4. Annota **Application (client) ID** e **Directory (tenant) ID**

### 2. Aggiungi una Federated Credential

Nella App Registration appena creata:

1. **Certificates & secrets → Federated credentials → Add credential**
2. Scenario: **GitHub Actions deploying Azure resources**
3. Compila i campi:

| Campo | Valore |
|-------|--------|
| Organization | `Pomiager` |
| Repository | nome del repo (es. `fanuc-kusarigate-web`) |
| Entity type | `Environment` |
| GitHub environment name | `test` oppure `production` |

> Crea una Federated Credential **separata** per ogni environment (`test`, `production`).  
> In alternativa usa Entity type `Branch` con `main` per ambienti senza protezione.

### 3. Assegna i permessi sull'App Service

1. Vai sull'**App Service** (o sul Resource Group che lo contiene)
2. **Access control (IAM) → Add role assignment**
3. Ruolo: **Website Contributor** _(sufficiente per il deploy)_
4. Assegna il ruolo all'App Registration creata al passo 1

### 4. Salva i secrets su GitHub

Nel repository del progetto: **Settings → Secrets and variables → Actions**

| Secret | Valore |
|--------|--------|
| `AZURE_CLIENT_ID` | Application (client) ID dell'App Registration |
| `AZURE_TENANT_ID` | Directory (tenant) ID |
| `AZURE_SUBSCRIPTION_ID` | ID della subscription Azure |

> Se usi service principal separati per test e produzione, puoi usare nomi distinti  
> (es. `AZURE_TEST_CLIENT_ID`, `AZURE_PROD_CLIENT_ID`) e adattare i workflow di progetto.

---

## Setup GitHub Environments

Gli environment GitHub permettono di aggiungere regole di protezione (reviewer obbligatori, branch policy) prima di ogni deploy.

Nel repository del progetto: **Settings → Environments → New environment**

| Environment | Configurazione consigliata |
|-------------|---------------------------|
| `test` | Nessun reviewer — deploy automatico |
| `production` | Aggiungi reviewer obbligatori — ogni deploy richiede approvazione manuale |

---

## Come usare in un nuovo progetto

Struttura minima dei workflow in un progetto `.NET` + Azure App Service:

```
.github/workflows/
  common.yml          ← build + test
  pull-request.yml    ← trigger su PR → chiama common
  main.yml            ← push su main → common → deploy test
  deploy-prod.yml     ← manuale → deploy produzione (con approvazione)
```

**`common.yml`**
```yaml
on:
  workflow_call:
jobs:
  build-test:
    uses: Pomiager/github-workflows-templates/.github/workflows/dotnet-workflow-common.yml@v1.0.0
    with:
      dotnet-version: "10.0.x"
```

**`pull-request.yml`**
```yaml
on:
  pull_request:
    branches: [main]
jobs:
  common:
    if: github.repository == 'Pomiager/<nome-repo>'
    uses: ./.github/workflows/common.yml
```

**`main.yml`**
```yaml
on:
  push:
    branches: [main]
jobs:
  common:
    if: github.repository == 'Pomiager/<nome-repo>'
    uses: ./.github/workflows/common.yml
  deploy-test:
    needs: common
    if: github.repository == 'Pomiager/<nome-repo>'
    uses: Pomiager/github-workflows-templates/.github/workflows/azure-step-deploy-app-service.yml@v1.0.0
    with:
      environment: test
      app-name: ${{ vars.TEST_APP_SERVICE_NAME }}
      project-path: src/MiaApp/MiaApp.csproj
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

**`deploy-prod.yml`**
```yaml
on:
  workflow_dispatch:
jobs:
  deploy-production:
    if: github.repository == 'Pomiager/<nome-repo>'
    uses: Pomiager/github-workflows-templates/.github/workflows/azure-step-deploy-app-service.yml@v1.0.0
    with:
      environment: production
      app-name: ${{ vars.PROD_APP_SERVICE_NAME }}
      project-path: src/MiaApp/MiaApp.csproj
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## Versioning

> **Non usare `@main` nei progetti in produzione.**

Quando il template è stabile, crea un tag:

```bash
git tag v1.0.0
git push origin v1.0.0
```

Poi nei progetti usa `@v1.0.0` oppure il commit hash completo (più sicuro):

```yaml
uses: Pomiager/github-workflows-templates/.github/workflows/azure-step-deploy-app-service.yml@<commit-hash>
```

Il commit hash garantisce che il workflow non cambi mai, anche se il tag viene spostato.
