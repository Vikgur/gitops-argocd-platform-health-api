# Оглавление

- [О проекте](#о-проекте)  
  - [Связь с другими репозиториями и запуск](#связь-с-другими-репозиториями-и-запуск)  
- [Архитектура](#архитектура)  
  - [apps/](#apps)  
    - [argo-rollouts.yaml — blue/green и canary деплой](#argo-rolloutsyaml--bluegreen-и-canary-деплой)  
    - [argocd-image-updater.yaml — автоматическое обновление образов](#argocd-image-updateryaml--автоматическое-обновление-образов)  
    - [cert-manager.yaml — автоматический TLS](#cert-manageryaml--автоматический-tls)  
    - [external-secrets.yaml — секреты из облака](#external-secretsyaml--секреты-из-облака)  
    - [ingress-nginx.yaml — точка входа](#ingress-nginxyaml--точка-входа)  
    - [monitoring.yaml — метрики и алерты](#monitoringyaml--метрики-и-алерты)  
    - [policy-engine.yaml — политики безопасности](#policy-engineyaml--политики-безопасности)   
  - [argocd-image-updater](#argocd-image-updater)  
    - [argocd-image-updater-config.yaml](#argocd-image-updater-configyaml)  
    - [secret.yaml](#secretyaml)  
    - [Шаги интеграции в проект health-api](#шаги-интеграции-в-проект-health-api)
- [Внедренные DevSecOps практики](#внедренные-devsecops-практики)  
  - [Безопасность, linting и валидация](#безопасность-linting-и-валидация)  
    - [.yamllint.yml](#yamllintyml)  
    - [.checkov.yaml](#checkovyaml)  
    - [policy/argo/secure-apps.rego](#policyargosecure-appsrego)  
    - [.gitleaks.toml](#gitleakstoml)  
    - [.pre-commit-config.yaml](#pre-commit-configyaml)  

---

# О проекте

Этот репозиторий — **App of Apps** для продовой GitOps-платформы [`health-api`](https://github.com/vikgur/health-api-for-microservice-stack). Он описывает системные Argo CD-приложения (`kind: Application`) для базовых компонентов, необходимых **до запуска микросервисов**:

- `argo-rollouts` — стратегии выкладки  
- `argocd-image-updater` — автоматическое обновление тегов образов по стратегии и верификации подписи  
- `cert-manager` — автоматический TLS  
- `external-secrets` — секреты из облака  
- `ingress-nginx` — точка входа  
- `kube-prometheus-stack` — мониторинг  
- `kyverno` — политики безопасности 

Платформа живёт **дольше**, чем любые сервисы, и не зависит от бизнес-логики. Её можно обновлять независимо.

> Хотя кластер на текущем этапе обслуживает только `health-api`, его архитектура изначально спроектирована как **многоуровневая и масштабируемая платформа** — с чётким разделением на системный и прикладной уровни, готовая к подключению новых сервисов без изменения ядра.

## Связь с другими репозиториями и запуск

- Репозиторий **не клонируется напрямую** при инициализации кластера через [ansible-gitops-bootstrap-health-api](https://github.com/vikgur/ansible-gitops-bootstrap-health-api).
- Вместо этого он подключается **как дочернее приложение** из [argocd-config-health-api](https://github.com/Vikgur/argocd-config-health-api) — через объект `Application`, описанный в [apps/platform-apps.yaml](https://github.com/Vikgur/argocd-config-health-api/-/blob/main/apps/platform-apps.yaml).
- Это позволяет:
  - централизованно управлять всеми источниками в [argocd-config-health-api](https://github.com/Vikgur/argocd-config-health-api),
  - отделить платформу от пользовательских сервисов,
  - инициализировать Argo CD **одним единственным приложением**, которое рекурсивно разворачивает весь стек.

Таким образом, Argo CD сам клонирует и синхронизирует этот репозиторий **по ссылке**, без участия Ansible.

---

# Архитектура

Каждый файл в директории `apps/` — это отдельное Argo CD-приложение (`Application`) для платформенного компонента. Ниже — краткое описание назначения каждого.

## `apps/`

### `argo-rollouts.yaml` — blue/green и canary деплой

**Назначение:**  
Устанавливает `argo-rollouts` для продвинутых стратегий выкладки.  
Поддерживает `blueGreen`, `canary`, `step analysis`, ручной промоушен и откаты.

### `argocd-image-updater.yaml` — автоматическое обновление образов

**Назначение:**  
Устанавливает компонент Argo Image Updater через Helm-чарт.  
Этот сервис отслеживает новые теги образов в container registry, проверяет подписи (`cosign`) и обновляет Argo CD Application с актуальными версиями.  
Работает совместно с манифестами в директории `argocd-image-updater/` (ConfigMap и Secret).

### `cert-manager.yaml` — автоматический TLS

**Назначение:**  
Устанавливает `cert-manager`, который автоматически получает и обновляет TLS-сертификаты (например, от Let's Encrypt).  
Нужен для HTTPS и безопасного взаимодействия с внешними сервисами.

### `external-secrets.yaml` — секреты из облака

**Назначение:**  
Устанавливает `external-secrets` — компонент, который синхронизирует Kubernetes `Secret`-ы с внешними Secret Manager (например, Yandex Cloud, HashiCorp Vault).  
Избавляет от хранения секретов в Git.

### `ingress-nginx.yaml` — точка входа

**Назначение:**  
Устанавливает `ingress-nginx` — главный Ingress-контроллер.  
Обрабатывает внешний HTTP/HTTPS трафик и маршрутизирует его по сервисам.

### `monitoring.yaml` — метрики и алерты

**Назначение:**  
Устанавливает `kube-prometheus-stack`: Prometheus, Grafana, node-exporter, алерты и дешборды.  
Даёт видимость кластера и сервисов, поддерживает метрики для rollout-стратегий.

### `policy-engine.yaml` — политики безопасности

**Назначение:**  
Устанавливает `kyverno` — движок Policy-as-Code.  
Применяет политики к Kubernetes-объектам: проверка аннотаций, запрет привилегированных контейнеров, enforce-логика и т.д.

---

## `argocd-image-updater`

Конфигурация и секреты для работы Argo Image Updater.

### `argocd-image-updater-config.yaml`

**Назначение:**  
Задаёт глобальные настройки Argo Image Updater.  
Определяет используемые container registry, параметры аутентификации и проверку подписей образов (`cosign`).  
Применяется в namespace `argocd`.

### `secret.yaml`

**Назначение:**  
Содержит учётные данные для доступа к GitHub Container Registry (GHCR).  
Используется Argo Image Updater для чтения информации о тегах и связанных подписях.  
Создаётся как Kubernetes Secret в namespace `argocd`.

### Шаги интеграции в проект health-api

1. **Dockerfile**  
   Добавляются стандартные OCI-метки (`source`, `revision`, `created`) для каждого сервиса (backend, frontend).  
   Эти метки позволяют Argo Image Updater отслеживать происхождение и свежесть образов.

2. **Helm values**  
   В репозитории `helm-blue-green-canary-gitops-health-api` для каждого сервиса (например, `helm/values/backend.yaml`) настраиваются параметры `image.repository` и аннотации Argo Image Updater.  
   Это указывает AIU, какие образы обновлять и по каким правилам.

3. **ConfigMap**  
   В репозитории `gitops-argocd-platform-health-api` создаётся `argocd-image-updater-config.yaml` с глобальной конфигурацией.  
   Он описывает, с какими registry работает AIU и где хранится ключ cosign для верификации подписей.

4. **Secret**  
   В том же репозитории добавляется `Secret` с учётными данными для доступа к GHCR.  
   Этот секрет используется AIU для скачивания информации о тегах и подписях.

---

# Внедренные DevSecOps практики

В этот репозиторий внедрён набор DevSecOps-инструментов, обеспечивающих контроль качества и безопасности GitOps-манифестов.

## Безопасность, linting и валидация

### `.yamllint.yml`

**Назначение:**  
Проверка синтаксиса и стиля YAML-файлов.  
Предотвращает ошибки форматирования и обеспечивает единый стандарт оформления.

---

### `.checkov.yaml`

**Назначение:**  
Анализ Kubernetes-манифестов с помощью Checkov.  
Выявляет небезопасные практики: использование `latest` образов, отсутствие ресурсов, некорректные Ingress и т.д.

---

### `policy/argo/secure-apps.rego`

**Назначение:**  
OPA-политики для Argo CD `Application`.  
Запрещают:  
- использование `targetRevision: HEAD`,  
- отключение `prune` и `selfHeal` в `syncPolicy`.  
Гарантируют корректную и безопасную конфигурацию Argo CD-приложений.

---

### `.gitleaks.toml`

**Назначение:**  
Поиск секретов в коммитах и файлах.  
Предотвращает утечку токенов, паролей и других чувствительных данных в Git.

---

### `.pre-commit-config.yaml`

**Назначение:**  
Автоматический запуск проверок перед коммитом (`yamllint`, `checkov`, `gitleaks`).  
Позволяет выявлять проблемы ещё до попадания изменений в репозиторий.

---

> В результате обеспечивается базовое покрытие DevSecOps: от синтаксиса и стиля YAML до поиска секретов и применения политик безопасности для Argo CD.
