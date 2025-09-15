# Оглавление

- [О проекте](#о-проекте)  
  - [Связь с другими репозиториями и запуск](#связь-с-другими-репозиториями-и-запуск)  
- [Архитектура](#архитектура)  
  - [apps/](#apps)  
    - [ingress-nginx.yaml — точка входа](#ingress-nginxyaml--точка-входа)  
    - [cert-manager.yaml — автоматический TLS](#cert-manageryaml--автоматический-tls)  
    - [argo-rollouts.yaml — bluegreen-и-canary-деплой](#argo-rolloutsyaml--bluegreen-и-canary-деплой)  
    - [external-secrets.yaml — секреты из облака](#external-secretsyaml--секреты-из-облака)  
    - [monitoring.yaml — метрики и алерты](#monitoringyaml--метрики-и-алерты)  
    - [policy-engine.yaml — политики безопасности](#policy-engineyaml--политики-безопасности)  

---

# О проекте

Этот репозиторий — **App of Apps** для продовой GitOps-платформы [`health-api`](https://github.com/vikgur/health-api-for-microservice-stack). Он описывает системные Argo CD-приложения (`kind: Application`) для базовых компонентов, необходимых **до запуска микросервисов**:

- `ingress-nginx` — точка входа  
- `cert-manager` — автоматический TLS  
- `argo-rollouts` — стратегии выкладки  
- `external-secrets` — секреты из облака  
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

### `ingress-nginx.yaml` — точка входа

**Назначение:**  
Устанавливает `ingress-nginx` — главный Ingress-контроллер.  
Обрабатывает внешний HTTP/HTTPS трафик и маршрутизирует его по сервисам.

---

### `cert-manager.yaml` — автоматический TLS

**Назначение:**  
Устанавливает `cert-manager`, который автоматически получает и обновляет TLS-сертификаты (например, от Let's Encrypt).  
Нужен для HTTPS и безопасного взаимодействия с внешними сервисами.

---

### `argo-rollouts.yaml` — blue/green и canary деплой

**Назначение:**  
Устанавливает `argo-rollouts` для продвинутых стратегий выкладки.  
Поддерживает `blueGreen`, `canary`, `step analysis`, ручной промоушен и откаты.

---

### `external-secrets.yaml` — секреты из облака

**Назначение:**  
Устанавливает `external-secrets` — компонент, который синхронизирует Kubernetes `Secret`-ы с внешними Secret Manager (например, Yandex Cloud, HashiCorp Vault).  
Избавляет от хранения секретов в Git.

---

### `monitoring.yaml` — метрики и алерты

**Назначение:**  
Устанавливает `kube-prometheus-stack`: Prometheus, Grafana, node-exporter, алерты и дешборды.  
Даёт видимость кластера и сервисов, поддерживает метрики для rollout-стратегий.

---

### `policy-engine.yaml` — политики безопасности

**Назначение:**  
Устанавливает `kyverno` — движок Policy-as-Code.  
Применяет политики к Kubernetes-объектам: проверка аннотаций, запрет привилегированных контейнеров, enforce-логика и т.д.
