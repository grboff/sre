**init** 
Создаем кластер, который «знает» о существовании двух датацентров. В реальном мире это были бы разные физические площадки, соединенные оптоволокном. В нашем kind - это разные Docker-контейнеры с метками.

# 1. Создаем кластер
kind create cluster --config kind-config.yaml --name simulation

# 2. Создаем неймспейсы (наша проектная сетка)
kubectl create namespace helloworld-nt-sb-mg
kubectl create namespace helloworld-nt-sb-gm
kubectl create namespace monitoring-core
kubectl create namespace vault # Задел на следующий этап

# 3. Проверяем разметку нод
kubectl get nodes --show-labels | grep datacenter

| Node | Status | Roles | Age | Version | Arch | OS | Datacenter | Hostname | Segment |
|------|--------|-------|-----|---------|------|----|----|----------|---------|
| simulation-worker | Ready | `<none>` | 86s | v1.27.3 | arm64 | linux | mg | simulation-worker | sandbox |
| simulation-worker2 | Ready | `<none>` | 85s | v1.27.3 | arm64 | linux | gm | simulation-worker2 | sandbox |


Почему?
Isolation: Даже если нода mg выйдет из строя (имитация падения ЦОДа), плечо в gm останется живым.
Scheduling
Naming Convention

Что дальше?

Интеграция с Vault: Ты не просто хранишь пароли в коде, ты используешь annotations для динамической подкачки секретов.

Infrastructure Awareness: Через nodeSelector ты управляешь тем, в каком "ЦОДе" физически запустится агент.

Dry Principle: Один чарт подходит и для mg, и для gm, и для sandbox, и для prod.

**progress**
Развернут кластер kind, имитирующий два географически распределенных ЦОДа (mg — Мегацод, gm — Гигацод) через метки нод (node-labels).

Внедрена строгая схема именования неймспейсов: <service>-<env>-<segment>-<dc>, что позволяет автоматизировать деплой и разграничивать доступы.

Развернут HashiCorp Vault с включенным методом аутентификации Kubernetes.

OpSec: Все секреты и политики переведены на анонимизированные идентификаторы (hellouniverse), чтобы исключить утечку чувствительных данных.

Настроены роли Vault, жестко привязанные к ServiceAccount и Namespace. Это исключает возможность одного плеча системы украсть секреты другого.

В изолированном контуре monitoring-core развернута VictoriaMetrics.

Настроена внутренняя маршрутизация (DNS), имитирующая централизованный сервис сбора метрик для всей организации.
Vault Sidecar Injection. Агент сам получает секреты при старте, не имея их в своем образе.


kubectl get pod -n helloworld-nt-sb-mg -o wide
NAME                   READY   STATUS     RESTARTS   AGE   IP           NODE                NOMINATED NODE   READINESS GATES
hwa-5b8bb75db6-5ttlq   0/1     Init:0/1   0          75s   10.244.1.3   simulation-worker   <none>           <none>

agent: (view) vault.read(secret/data/hellouniverse/helloworld/config): no secret exists at secret/data/hellouniverse/helloworld/config (retry attempt 1 after "250ms")

vault kv get secret/hellouniverse/helloworld/config
No value found at secret/data/hellouniverse/helloworld/config
/ $ 


vault kv enable-versioning secret/
KV-v2 Migration: Проведена миграция движка секретов на версию 2 с поддержкой версионирования.

перезаписываем секрет 

**to do krasivo**
Проект имитирует работу двух ЦОДов: MG (Мегацод) и GM (Гигацод).
Secret Management: HashiCorp Vault (инъекция секретов через Sidecar).

Observability: VictoriaMetrics (единое хранилище метрик для обоих плеч).

CI/CD: Jenkins в Kubernetes, использующий динамические агенты и конфигурацию из subsystems.json.

**КОмпоненты:**
1. Конфигурация: subsystems.json
Пайплайн читает этот файл и понимает, какие подсистемы нужно развернуть:
id: уникальное имя плеча.
dc: идентификатор датацентра (mg/gm).
namespace: целевой неймспейс в K8s.

2. Пайплайн: Jenkinsfile
Изолирован: Запускается в Docker-контейнере dtzar/helm-kubectl.
Идемпотентен: Сам создает неймспейсы и обновляет релизы (не падает, если они уже есть).
Безопасен: Использует @NonCPS для парсинга JSON, чтобы избежать ошибок сериализации.

**how to start** 
Шаг 1: Подготовка кластера и прав
Если кластер kind запущен, убедись, что у Jenkins есть права:
# Даем права админа агенту Jenkins (важно для создания неймспейсов)
kubectl create clusterrolebinding jenkins-agent-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=jenkins:default

Шаг 2: Core 
helm install vm-storage victoria-metrics/victoria-metrics-single -n monitoring-core --create-namespace
helm install vault hashicorp/vault -n vault --set "server.dev.enabled=true" --set "injector.enabled=true" --create-namespace

Шаг 3: Запуск Автоматизации
Зайди в Jenkins: http://localhost:8081 (после port-forward).
Build Now 
Результат: В кластере появятся два неймспейса helloworld-nt-sb-mg и ...-gm с работающими агентами.

Как проверить метрики?
kubectl port-forward svc/vm-storage-victoria-metrics-single-server 8428 -n monitoring-core

лейблами datacenter="mg" и datacenter="gm".

Next Step: Network Policies