# Test akcji deploy-to-app-service-slots z prawdziwym Azure

Instrukcja krok po kroku. Wartosci `org/repo/branch` sa juz wstawione dla repo
**Medjai09/gh-actions-course**, galaz **main**.

Kroki 1-4 wykonujesz w **Azure Cloud Shell (Bash)** w przegladarce.
Krok 5 (sekrety) w ustawieniach repo na GitHub. Krok 6 to uruchomienie workflow.

---

## Krok 1 - Infrastruktura w Azure

```bash
# Zmienne (nazwy app/acr musza byc globalnie unikalne)
RG="rg-deployslots-test"
LOCATION="westeurope"
PLAN="plan-deployslots-test"
APP="app-deployslots-$RANDOM"
ACR="acrdeployslots$RANDOM"

az group create -n $RG -l $LOCATION

# Azure Container Registry
az acr create -n $ACR -g $RG --sku Basic --admin-enabled true

# App Service Plan - MUSI byc Standard/Premium (sloty nie dzialaja na Basic/Free)
az appservice plan create -n $PLAN -g $RG --is-linux --sku S1

# Web App for Containers (startowy obraz, zaraz podmieni go akcja)
az webapp create -n $APP -g $RG -p $PLAN \
  --deployment-container-image-name mcr.microsoft.com/azuredocs/aci-helloworld:latest

# Zapamietaj te wartosci - przydadza sie przy uruchamianiu workflow:
echo "APP  = $APP"
echo "ACR  = $ACR"
echo "RG   = $RG"
```

---

## Krok 2 - Obraz testowy z endpointem /health

```bash
mkdir -p ~/test-app && cd ~/test-app

cat > default.conf << 'EOF'
server {
    listen 80;
    server_name _;
    location /health {
        access_log off;
        add_header Content-Type application/json;
        return 200 '{"status":"UP"}';
    }
    location / {
        add_header Content-Type text/plain;
        return 200 'hello from test-app';
    }
}
EOF

cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
EOF

# Budowa obrazu w chmurze (lokalny Docker niepotrzebny)
az acr build -r $ACR -t myapp:1.0.0 .

# Weryfikacja
az acr repository show-tags -n $ACR --repository myapp -o table
```

---

## Krok 3 - Tozsamosc dla GitHub Actions (OIDC)

```bash
APP_NAME="gh-deployslots-test"
ORG="Medjai09"
REPO="gh-actions-course"
BRANCH="main"

# App registration + service principal
az ad app create --display-name "$APP_NAME"
APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)
az ad sp create --id "$APP_ID"
echo "CLIENT_ID (APP_ID) = $APP_ID"

# Federated credential - powiazanie repo+galaz z tozsamoscia
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters "{
    \"name\": \"gh-${BRANCH}\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:${ORG}/${REPO}:ref:refs/heads/${BRANCH}\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"
```

---

## Krok 4 - Nadanie uprawnien (RBAC)

```bash
SUB=$(az account show --query id -o tsv)
echo "SUBSCRIPTION_ID = $SUB"
TENANT=$(az account show --query tenantId -o tsv)
echo "TENANT_ID = $TENANT"

# Zarzadzanie App Service w resource group
az role assignment create --assignee "$APP_ID" --role Contributor \
  --scope "/subscriptions/$SUB/resourceGroups/$RG"

# Pobieranie obrazu z ACR
az role assignment create --assignee "$APP_ID" --role AcrPull \
  --scope "$(az acr show -n $ACR --query id -o tsv)"
```

---

## Krok 5 - Sekrety w GitHub

W repo: **Settings -> Secrets and variables -> Actions -> New repository secret**.
Dodaj trzy sekrety (wartosci z wypisow w krokach 3 i 4):

| Nazwa sekretu          | Wartosc                       |
|------------------------|-------------------------------|
| `AZURE_CLIENT_ID`      | `APP_ID` z kroku 3            |
| `AZURE_TENANT_ID`      | `TENANT_ID` z kroku 4         |
| `AZURE_SUBSCRIPTION_ID`| `SUBSCRIPTION_ID` z kroku 4   |

---

## Krok 6 - Uruchomienie workflow

1. Wypchnij pliki do repo (galaz `main`):
   ```bash
   git add .github/workflows/test-deploy-real-azure.yaml composite test-app
   git commit -m "Add real-Azure test for deploy-to-app-service-slots"
   git push origin main
   ```
2. GitHub -> zakladka **Actions** -> workflow **"Test Deploy to App Service Slots (Real Azure)"** -> **Run workflow**.
3. Wypelnij pola:
   - `resource-group`: `rg-deployslots-test`
   - `app-service-name`: wartosc `APP` z kroku 1
   - `image-uri`: `<ACR>.azurecr.io/myapp:1.0.0`
   - reszta domyslna (`/health`, slot `deploy`)
4. Obserwuj log. Oczekiwany wynik: **status `success`**, URL slotu, czas > 0.

Podglad logow aplikacji (rownolegle, w Cloud Shell):
```bash
az webapp log tail -n $APP -g $RG --slot deploy
```

---

## Scenariusze testowe

- **Sukces** -> obraz z dzialajacym `/health` -> status `success`, swap wykonany.
- **Rollback** -> ustaw `health-check-path` na nieistniejaca sciezke (np. `/nope`) ->
  status `health-check-failed`, slot zatrzymany.
- **Auto-tworzenie slotu** -> `az webapp deployment slot delete -n $APP -g $RG --slot deploy`
  przed uruchomieniem -> akcja sama odtworzy slot.

---

## Sprzatanie po testach

```bash
az group delete -n rg-deployslots-test --yes --no-wait
# opcjonalnie usun app registration
az ad app delete --id "$APP_ID"
```

---

## Czeste problemy

- **Plan Basic/Free** -> akcja padnie na tworzeniu slotu. Wymagany S1+.
- **Brak `id-token: write`** w jobie -> logowanie OIDC nie zadziala (jest juz w workflow).
- **Brak roli AcrPull** -> slot wystartuje, ale nie pobierze obrazu -> health check padnie.
- **Zly `subject` w federated credential** -> blad logowania. Musi byc dokladnie
  `repo:Medjai09/gh-actions-course:ref:refs/heads/main`.
- **Wolny start kontenera** -> zwieksz `startup-wait-seconds` / `health-check-timeout-seconds`.
