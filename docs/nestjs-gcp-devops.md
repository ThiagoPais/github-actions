# NestJS to GCP Cloud Run - CI/CD Pipeline

Este documento descreve como configurar e utilizar o workflow reutilizável de CI/CD para aplicações **NestJS** com deploy automatizado no **Google Cloud Run**, utilizando Artifact Registry e Cloud Build.

## 🚀 O que este workflow faz?

1. **Code Validations:** Valida a versão no `package.json` (Exige `-rc`, `-alpha` ou `-beta` na branch `develop` e versão estável na `master`).
2. **Auto-PR:** Se uma versão estável for detectada na branch `develop`, o workflow aborta o deploy e abre um **Pull Request automático** para a `master`.
3. **Docker Validations:** Faz o build da imagem Docker no GCP Cloud Build e publica no Artifact Registry.
4. **Deploy on GCP:** Atualiza o serviço no Cloud Run com a nova imagem.

---

## 🛠️ 1. Configuração no Google Cloud Platform (GCP)

Para que o GitHub Actions consiga se comunicar com o GCP sem precisar de chaves JSON estáticas (que são inseguras), utilizamos o **Workload Identity Federation (WIF)**.

### 1.1 Criar a Service Account de Deploy

Crie uma Service Account que o GitHub usará para executar a esteira:

```bash
gcloud iam service-accounts create github-deployer \
    --display-name="GitHub Actions Deployer"
```

### 1.2 Atribuir Permissões (IAM Roles)

Esta Service Account precisa de permissões estritas para construir a imagem e fazer o deploy. Execute os comandos abaixo substituindo `SEU_PROJECT_ID`:

```bash
export PROJECT_ID="SEU_PROJECT_ID"
export SA_EMAIL="github-deployer@${PROJECT_ID}.iam.gserviceaccount.com"

# Permite gerenciar o Cloud Run (necessário para mudar ingress para 'all')
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/run.admin"
# Permite atuar como a Service Account de computação
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/iam.serviceAccountUser"
# Permite escrever imagens no Artifact Registry
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/artifactregistry.writer"
# Permite enviar e gerenciar builds no Cloud Build
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/cloudbuild.builds.editor"
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/cloudbuild.builds.viewer"
# Permite gerenciar o bucket temporário de códigos do Cloud Build
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/storage.admin"
# Permite visualizar logs de build e usar as APIs do GCP
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/logging.viewer"
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member="serviceAccount:${SA_EMAIL}" --role="roles/serviceusage.serviceUsageConsumer"
```

### 1.3 Permissão de Secret Manager para o Cloud Run
O contêiner do Cloud Run roda utilizando a Default Compute Service Account. Ela precisa de acesso aos segredos.
Nota: Substitua `NUMERO_DO_PROJETO` pelo ID numérico do seu projeto no GCP.

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
    --member="serviceAccount:NUMERO_DO_PROJETO-compute@developer.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"
```

### 1.4 Configurar o WIF (Workload Identity Federation)
Configure um Pool e um Provider para o GitHub. (Siga a [documentação oficial do Google](https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines) para criar o provider).
Anote os dois valores gerados:

Provider ID: `projects/NUMERO/locations/global/workloadIdentityPools/MEU_POOL/providers/MEU_PROVIDER`

Service Account Email: O email da `github-deployer` criada no passo 1.1.

## 🔒 2. Configuração no GitHub

No repositório da sua aplicação (ex: `backend`), vá em **Settings > Secrets and variables > Actions**.

### 2.1 Repository Secrets

Adicione os seguintes segredos no nível do **Repositório** e nível de **Ambiente**:

- `WIF_PROVIDER`: O ID do provider gerado no GCP.

- `WIF_SERVICE_ACCOUNT`: O e-mail da service account (`github-deployer@...`).

### 2.2 Environments

Vá em Settings > Environments e crie os ambientes (ex: staging e production).
Isso é útil para exigir revisões manuais de aprovação antes que um deploy vá para produção.

## 📦 3. Preparando o Repositório da Aplicação

Sua aplicação NestJS precisa seguir algumas convenções para que o workflow funcione perfeitamente.

### 3.1 Dockerfile (Ignorar Husky)

Se você usa Husky ou outros scripts no package.json, evite que eles quebrem a instalação no ambiente de produção do Google. No seu Dockerfile, use a flag `--ignore-scripts`:

```dockerfile
# Exemplo de step no Dockerfile
RUN npm ci --omit=dev --ignore-scripts
```

### 3.2 Arquivos YAML do Cloud Run

Na raiz do seu projeto, crie uma pasta `infra/` com as definições do Cloud Run para cada ambiente (ex: infra/service-hml.yaml e infra/service-prd.yaml).
Importante: A imagem deve conter a string `IMAGE_TAG_PLACEHOLDER`, que será substituída dinamicamente pelo GitHub Actions.

Substitua os valores `{LOCATION}`, `{SEU_PROJECT_ID}`, `{SEU_REPOSITORY}` e `{SEU_NOME_DA_IMAGEM` no código para seus os respectivos valores.

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: nestjs-api-hml
spec:
  template:
    spec:
      containers:
        - image: {LOCATION}-docker.pkg.dev/{SEU_PROJECT_ID}/{SEU_REPOSITORY}/{SEU_NOME_DA_IMAGEM}:IMAGE_TAG_PLACEHOLDER
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: DATABASE_URL_HML
                  key: latest
```

Use esse arquivo para adicionar variáveis de ambientes no seu projeto também.

## 🚀 4. Arquivo Workflow (Como chamar)

Crie o arquivo `.github/workflows/main.yml` no repositório da sua aplicação, chamando o nosso workflow reutilizável:

```yaml
name: Backend CI/CD

on:
  push:
    branches: [ master, develop ]
  pull_request:
    branches: [ master ]
  workflow_dispatch:

permissions:
  contents: write       # Necessário para Git Tags
  id-token: write       # Necessário para autenticação WIF (GCP)
  pull-requests: write  # Necessário para criar o Auto-PR

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    
    # IMPORTANTE: Aponte para o caminho correto do repositório de Actions
    uses: SEU_USUARIO/SEU_REPO_ACTIONS/.github/workflows/nestjs-gcp-devops.yml@main
    
    with:
      environment: 'hml'
      project_id: 'seu-projeto-gcp'
      region: 'southamerica-east1'
      image_name: 'nestjs-api'
      repo_name: 'nome-do-repo-artifact-registry'
      gh_environment: 'staging'
      node_version: 20
    secrets:
      WIF_PROVIDER: ${{ secrets.WIF_PROVIDER }}
      WIF_SERVICE_ACCOUNT: ${{ secrets.WIF_SERVICE_ACCOUNT }}

  # Adicione um job similar para 'deploy-production' verificando a branch 'master'.
```

## ⚠️ 5. Erros Comuns e Como Resolver

| Erro / Log no GitHub | Causa Raiz | Como Resolver (Solução) |
| -------------------- | ---------- | ----------------------- |
| The user is forbidden from accessing the bucket [..._cloudbuild] | A SA do GitHub não tem permissão de Cloud Build ou Storage. | Conceda `roles/cloudbuild.builds.editor` e `roles/storage.admin` à SA github-deployer. |
| This tool can only stream logs if you are Viewer/Owner | A SA do GitHub não tem permissão para ler o log gerado pelo GCP. | Conceda `roles/logging.viewer` e certifique-se de que o workflow usa a flag `--suppress-logs` no comando gcloud builds submit. |
| husky: not found ou npm error code 127 | O npm tentou rodar scripts de desenvolvimento em ambiente de build no GCP. | Adicione `--ignore-scripts` ao comando `npm install` / `npm ci` dentro do seu Dockerfile. |
| Permission denied on secret: projects/.../secrets/... | O contêiner subiu, mas a conta de máquina do Google não pode ler o Secret Manager. | Conceda `roles/secretmanager.secretAccessor` à Compute Service Account (a conta que tem ...-compute@developer...). |
| O botão "Run workflow" não aparece no GitHub | O GitHub exige que a trigger manual exista na branch padrão. | Faça um push/merge do seu arquivo .yml contendo `workflow_dispatch`: diretamente para a branch master. |
| Deploy pula testes e encerra sucesso prematuro na branch develop | O sistema de Auto-PR detectou uma versão estável. | Comportamento esperado. O workflow abre o PR e pausa a esteira de develop. Os testes rodarão novamente quando o merge para master ocorrer. |
