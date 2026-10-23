# Деплой в Beget Managed Kubernetes

Вариант v2: Hotels Booking System. Общий chart `helm/booking-service`
используется для всех четырёх сервисов с параметрами из `deploy/*-service.yaml`.
Внешние запросы идут через Traefik Ingress на Gateway. Все Service приложения
и PostgreSQL имеют тип ClusterIP.

## Доступ и секреты GitHub Actions

В Settings → Secrets and variables → Actions добавьте:

- `DOCKERHUB_USERNAME`: `mslinyakov`.
- `DOCKERHUB_TOKEN`: Docker Hub access token с правом записи.
- `KUBE_CONFIG`: полное содержимое YAML kubeconfig из Beget, без base64-кодирования.

Репозитории Docker Hub должны быть публичными. Kubeconfig должен позволять
управлять namespace, Secret, Deployment, Service, Ingress, ConfigMap и читать
список нод. Сам файл и токены нельзя коммитить.

## Входная точка (однократно)

Traefik уже установлен вручную. Его настройки сохранены в `deploy/traefik.yaml`.
При создании нового кластера установите его заново (команды из корня проекта):

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik \
  --kubeconfig /absolute/path/to/kubeconfig.yaml \
  --namespace traefik --create-namespace \
  -f deploy/traefik.yaml --wait --timeout 5m
```

Убедитесь, что у Service traefik появился внешний IP.

## Запуск CI/CD

Push в master запускает сборку, unit-тесты, проверку charts, публикацию четырёх
образов и деплой в namespace `test`. Также доступен ручной запуск workflow.
Pull request запускает только сборку и проверки.

Образы помечаются SHA коммита и `latest`. В кластер разворачивается именно SHA.
Workflow сам получает внешний IP Service traefik и передаёт его Newman как
`baseUrl`. Поскольку `education/autograding@v1` очищает окружение тестов,
workflow перед его запуском подставляет адрес API и явный путь kubeconfig
в команду `autograding.json` только в рабочей копии runner.
После деплоя тестовый сценарий отключает и включает Loyalty Service.
При завершении сценария исходное число реплик восстанавливается, в том числе
при падении теста. Для GitHub Classroom сохранён шаг `education/autograding`.

## PostgreSQL и данные

При первом деплое создаётся Secret `booking-database` со случайным паролем
и строками подключения. Он сохраняется при следующих деплоях. PostgreSQL
создаёт три базы: `reservations`, `payments`, `loyalties`. Сами сервисы при
запуске создают таблицы и исходные данные для тестов.

В Beget сейчас нет StorageClass, поэтому PostgreSQL закрепляется за одним
worker по метке `kubernetes.io/hostname` и хранит данные в hostPath
`/var/lib/rsoi-lab4/postgres`. Это учебная конфигурация: перезапуск Pod сохраняет
данные, но пересоздание/удаление worker приводит к их потере. Если worker
недоступен, PostgreSQL не переносится автоматически на другую ноду.
Не удаляйте Secret отдельно от данных PostgreSQL: новый пароль не изменит
пароль в уже инициализированной базе.

## Ручной деплой и проверка

После публикации образов (Helm, kubectl и Python 3 должны быть установлены):

```bash
./scripts/deploy.sh /absolute/path/to/kubeconfig.yaml <commit-sha> test
kubectl --kubeconfig /absolute/path/to/kubeconfig.yaml -n test get pods,svc,ingress
curl --fail http://<traefik-external-ip>/manage/health
```

Запуск тестового сценария из корня проекта после `npm install -g newman@6`:

```bash
BASE_URL=http://<traefik-external-ip> ./scripts/test-script.sh \
  v2 loyalty-service test /absolute/path/to/kubeconfig.yaml
```

Команды используют явный kubeconfig и не меняют локальный контекст по умолчанию.
