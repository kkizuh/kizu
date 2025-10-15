# CI/CD (index)

Мини-оглавление — коротко, по делу.

## 1) Типовой пайплайн

- **Build → Test → Scan → Sign → Push → Deploy**
    
    - _Build:_ кэш зависимостей, reproducible.
        
    - _Test:_ unit/integration + linters.
        
    - _Scan:_ **trivy** (image/code), **SBOM** (syft/bom).
        
    - _Sign:_ **cosign** (keyless или ключ в KMS).
        
    - _Push:_ в **registry** с **digest pinning**.
        
    - _Deploy:_ staging → prod, с **manual approval**.
        

## 2) Политики качества/безопасности

- **Запрет `latest`**: тэг = `app:1.4.2` + `app@sha256:…`.
    
- **Scan-gate:** падаем, если `HIGH/CRITICAL` (allowlist для явных исключений).
    
- **SBOM обязателен**: артефакт в репо/registry (OCI artifact).
    
- **Подписи образов (cosign)** + в проде **verify** (Admission Policy).
    
- **Secrets** только через Vault/CI secrets, **не** в `ENV/ARG`.
    
- **Reproducible builds**: фиксированные версии, lock-файлы.
    

## 3) Бранчи/релизы

- `main` → прод, `develop` → стейдж.
    
- **Feature** → PR/MR → review + status checks.
    
- **Release tags** `vX.Y.Z` → immutable.
    
- **Git tags = версия** контейнера/чарта.
    

## 4) Артефакты/кэш

- Кэш: зависимости (pip/npm/maven/go).
    
- Артефакты: отчёты тестов, **SBOM**, **trivy**-отчёты, покрытие.
    

## 5) Среды/деплой

- **Staging = прод-конфиг без секрета прод-данных.**
    
- Blue/Green или Canary (Ingress/Service weights).
    
- Авто-**rollback** по health/probe/метрикам.
    

## 6) Наблюдаемость/алерты

- Логи сборки (retention), алерты по **failed pipeline**.
    
- В проде: **SLO**-гейты (ошибки/латентность) для автопаузы раскатки.
    

---

## Мини-шаблоны

### GitHub Actions (Docker build → scan → sign → push)

```yaml
name: ci
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # для cosign keyless
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build (tag by sha)
        run: |
          docker buildx build -t ghcr.io/org/app:${{ github.sha }} \
            -t ghcr.io/org/app:${{ github.ref_name }} --push .
      - name: Trivy image scan (fail on HIGH)
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ghcr.io/org/app:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'HIGH,CRITICAL'
      - name: SBOM (syft) → upload
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/org/app:${{ github.sha }}
          artifact-name: sbom.spdx.json
      - name: Cosign sign (keyless)
        run: cosign sign ghcr.io/org/app@$(docker buildx imagetools inspect ghcr.io/org/app:${{ github.sha }} | awk '/Digest/{print $2}')
```

### GitLab CI (gate + deploy)

```yaml
stages: [build, test, scan, release, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE/app
  TAG: $CI_COMMIT_SHORT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE:$TAG .
    - docker push $IMAGE:$TAG
  only: [merge_requests, main]

scan:
  stage: scan
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 $IMAGE:$TAG
    - syft $IMAGE:$TAG -o spdx-json > sbom.json
  artifacts:
    paths: [sbom.json]
  needs: [build]

release:
  stage: release
  script:
    - cosign sign $IMAGE:$TAG
  needs: [scan]
  only: [tags]

deploy_prod:
  stage: deploy
  when: manual
  script:
    - helm upgrade --install app chart/ --set image.repository=$IMAGE --set image.tag=$TAG
  environment: production
  needs: [release]
```

---

## Чек-лист «что точно сделать»

-  Нет `latest`, только версии + **digest**.
    
-  **Trivy**: fail на `HIGH/CRITICAL`.
    
-  **SBOM** генерится и хранится.
    
-  **Cosign** подпись и **verify** на входе в кластер.
    
-  Secrets из **Vault/CI secrets**, **никогда** в Dockerfile/ENV.
    
-  Rollback сценарий проверен (и документирован).
    
-  Стейдж = как прод (конфиг/лимиты/пробы), только безопасные данные.
    
-  Логи/метрики пайплайна и деплоя подключены к алертам.