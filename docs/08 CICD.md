# CI/CD

Un pipeline CI/CD automatise la vérification et le déploiement du code à chaque push. Il garantit que le code qui part en production est testé, formaté et fonctionnel — sans intervention manuelle.

---

## Sommaire

- [Principe](#principe)
- [GitHub Actions](#github-actions)
- [GitLab CI](#gitlab-ci)
- [Azure Pipelines](#azure-pipelines)
- [Stratégies de déclenchement](#stratégies-de-déclenchement)
- [Bonnes pratiques](#bonnes-pratiques)

---

## Principe

**CI (Continuous Integration)** — à chaque push, le pipeline vérifie automatiquement :
- que le code est bien formaté (Ruff)
- qu'il n'y a pas d'erreurs de lint
- que les tests passent (pytest)
- que les dépendances sont cohérentes

**CD (Continuous Deployment)** — si la CI passe sur la branche principale, le pipeline déploie automatiquement l'application en production (ou en staging).

```
Push → CI (lint + tests) → ✅ merge → CD (déploiement)
```

---

## GitHub Actions

Les workflows GitHub Actions sont des fichiers YAML placés dans `.github/workflows/`.

### Workflow CI minimal

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Installer uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "latest"

      - name: Installer Python
        run: uv python install

      - name: Installer les dépendances
        run: uv sync --frozen

      - name: Vérification du formatage (Ruff)
        run: uv run ruff format --check .

      - name: Lint (Ruff)
        run: uv run ruff check .

      - name: Tests (pytest)
        run: uv run pytest
```

### Workflow avec matrix (tester sur plusieurs versions Python)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4

      - name: Installer uv
        uses: astral-sh/setup-uv@v3

      - name: Installer Python ${{ matrix.python-version }}
        run: uv python install ${{ matrix.python-version }}

      - name: Installer les dépendances
        run: uv sync --frozen

      - name: Ruff format
        run: uv run ruff format --check .

      - name: Ruff lint
        run: uv run ruff check .

      - name: Tests
        run: uv run pytest --tb=short
```

### Workflow CI + CD (build et push d'une image Docker)

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v3

      - run: uv python install

      - run: uv sync --frozen

      - run: uv run ruff format --check .

      - run: uv run ruff check .

      - run: uv run pytest

  cd:
    needs: ci
    runs-on: ubuntu-latest
    # Ne se déclenche que sur main (pas sur les PR)
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Se connecter à Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build et push de l'image Docker
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: monorganisation/mon-projet:latest
```

### Gérer les secrets dans GitHub Actions

Les secrets (clés API, tokens) se configurent dans `Settings → Secrets and variables → Actions` du dépôt. Ils sont ensuite accessibles dans les workflows via `${{ secrets.NOM_DU_SECRET }}`.

```yaml
- name: Déployer
  env:
    API_KEY: ${{ secrets.API_KEY }}
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: uv run python deploy.py
```

---

## GitLab CI

Le pipeline GitLab CI est défini dans un fichier `.gitlab-ci.yml` à la racine du projet.

### Pipeline CI minimal

```yaml
# .gitlab-ci.yml
default:
  image: python:3.12-slim

stages:
  - lint
  - test

variables:
  UV_VERSION: "latest"

before_script:
  - pip install uv
  - uv sync --frozen

lint:
  stage: lint
  script:
    - uv run ruff format --check .
    - uv run ruff check .

test:
  stage: test
  script:
    - uv run pytest --tb=short
```

### Pipeline CI + CD avec Docker

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

default:
  image: python:3.12-slim

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

lint:
  stage: lint
  before_script:
    - pip install uv
    - uv sync --frozen
  script:
    - uv run ruff format --check .
    - uv run ruff check .

test:
  stage: test
  before_script:
    - pip install uv
    - uv sync --frozen
  script:
    - uv run pytest --tb=short
  coverage: '/TOTAL.*\s+(\d+%)$/'

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  only:
    - main
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE

deploy:
  stage: deploy
  only:
    - main
  script:
    - echo "Déploiement de $DOCKER_IMAGE"
    # Commande de déploiement spécifique à votre infrastructure
```

### Gérer les secrets dans GitLab CI

Les secrets se configurent dans `Settings → CI/CD → Variables`. Les variables marquées comme "Masked" n'apparaissent pas dans les logs.

```yaml
deploy:
  stage: deploy
  script:
    - export API_KEY=$API_KEY  # variable définie dans les settings GitLab
    - uv run python deploy.py
```

---

## Azure Pipelines

Le pipeline Azure DevOps est défini dans un fichier `azure-pipelines.yml` à la racine du projet. Il s'intègre nativement avec Azure Repos mais fonctionne aussi avec GitHub.

### Pipeline CI minimal

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: "3.12"
    displayName: Installer Python

  - script: pip install uv
    displayName: Installer uv

  - script: uv sync --frozen
    displayName: Installer les dépendances

  - script: uv run ruff format --check .
    displayName: Vérification du formatage (Ruff)

  - script: uv run ruff check .
    displayName: Lint (Ruff)

  - script: uv run pytest --tb=short --junitxml=junit/test-results.xml
    displayName: Tests (pytest)

  - task: PublishTestResults@2
    inputs:
      testResultsFiles: "junit/test-results.xml"
    displayName: Publier les résultats de tests
    condition: always()
```

### Pipeline CI + CD avec Docker

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

variables:
  IMAGE_NAME: monorganisation/mon-projet
  IMAGE_TAG: $(Build.BuildId)

stages:
  - stage: CI
    displayName: Intégration continue
    jobs:
      - job: lint_and_test
        displayName: Lint et tests
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: "3.12"

          - script: pip install uv
            displayName: Installer uv

          - script: uv sync --frozen
            displayName: Installer les dépendances

          - script: uv run ruff format --check .
            displayName: Ruff format

          - script: uv run ruff check .
            displayName: Ruff lint

          - script: uv run pytest --tb=short --junitxml=junit/test-results.xml
            displayName: pytest

          - task: PublishTestResults@2
            inputs:
              testResultsFiles: "junit/test-results.xml"
            condition: always()

  - stage: CD
    displayName: Déploiement
    dependsOn: CI
    # Ne se déclenche que sur main (pas sur les PR)
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: build_and_push
        displayName: Build et push Docker
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: Docker@2
            displayName: Build et push de l'image
            inputs:
              command: buildAndPush
              repository: $(IMAGE_NAME)
              dockerfile: Dockerfile
              containerRegistry: mon-service-connection-docker
              tags: |
                $(IMAGE_TAG)
                latest
```

### Gérer les secrets dans Azure Pipelines

Les secrets se configurent dans `Pipelines → Library → Variable groups` ou directement dans les paramètres du pipeline (`Edit → Variables`). Les variables marquées comme secrètes sont masquées dans les logs.

```yaml
variables:
  - group: mon-groupe-de-variables  # Variable group défini dans Library

steps:
  - script: uv run python deploy.py
    displayName: Déployer
    env:
      API_KEY: $(API_KEY)           # variable secrète injectée
      DATABASE_URL: $(DATABASE_URL)
```

Les connexions à des services externes (Docker Hub, Azure Container Registry, Azure App Service…) se gèrent via les **Service Connections** dans `Project Settings → Service connections` — elles évitent de stocker des credentials directement dans les pipelines.

### Protéger la branche `main`

Dans Azure DevOps : `Project Settings → Repositories → Policies → Branch policies`. Activer **Require a minimum number of reviewers** et **Check for linked work items**, puis ajouter le pipeline CI comme **Build validation** obligatoire avant tout merge.

---

## Stratégies de déclenchement

### Déclencher sur des branches spécifiques

```yaml
# GitHub Actions
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# GitLab CI
test:
  only:
    - main
    - develop
    - merge_requests

# Azure Pipelines
trigger:
  branches:
    include:
      - main
      - develop
pr:
  branches:
    include:
      - main
```

### Déclencher uniquement sur certains fichiers

```yaml
# GitHub Actions
on:
  push:
    paths:
      - "src/**"
      - "tests/**"
      - "pyproject.toml"

# Azure Pipelines
trigger:
  paths:
    include:
      - src/**
      - tests/**
      - pyproject.toml
```

### Déclencher sur un planning (cron)

```yaml
# GitHub Actions — tous les jours à 6h UTC
on:
  schedule:
    - cron: "0 6 * * *"

# Azure Pipelines — tous les jours à 6h UTC
schedules:
  - cron: "0 6 * * *"
    displayName: Run quotidien
    branches:
      include:
        - main
    always: true
```

---

## Bonnes pratiques

**Garder la CI rapide**

Un pipeline qui dure 20 minutes décourage les commits fréquents. Viser moins de 5 minutes pour la CI :
- utiliser le cache des dépendances
- paralléliser les jobs indépendants
- séparer les tests lents (tests d'intégration) dans un job séparé déclenché moins souvent

**Mettre en cache les dépendances**

```yaml
# GitHub Actions
- name: Installer uv avec cache
  uses: astral-sh/setup-uv@v3
  with:
    enable-cache: true
```

```yaml
# GitLab CI
test:
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - .venv/
  before_script:
    - pip install uv
    - uv sync --frozen
```

```yaml
# Azure Pipelines
- task: Cache@2
  inputs:
    key: 'uv | "$(Agent.OS)" | pyproject.toml'
    path: .venv
  displayName: Cache des dépendances uv
```

**Ne jamais écrire de secrets en clair dans les fichiers de pipeline**

```yaml
# À ne jamais faire
- run: curl -H "Authorization: Bearer sk-1234..." https://api.example.com

# À faire
- run: curl -H "Authorization: Bearer ${{ secrets.API_KEY }}" https://api.example.com
```

**Protéger la branche `main`**

Sur GitHub et GitLab, activer la **protection de branche** pour que tout merge sur `main` nécessite que la CI passe. Cela empêche de merger du code cassé.

- GitHub : `Settings → Branches → Add rule → Require status checks to pass`
- GitLab : `Settings → Repository → Protected branches`
- Azure DevOps : `Project Settings → Repositories → Policies → Build validation`
