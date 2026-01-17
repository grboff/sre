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