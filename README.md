# Домашнее задание к занятию «Запуск приложений в K8S» - Kadancev- Vladimir

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.

## Решение:
Манифест с ошибкой: [deployment-multi-broken.yaml](deployment-multi-broken.yaml)

Под переходит в статус `1/2 Error`, в логе контейнера `multitool` видно:

```
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address in use)
```

**Причина.** Контейнеры одного пода делят одно сетевое пространство (один IP и один набор портов).
И nginx, и multitool по умолчанию слушают порт 80, поэтому второй контейнер не может занять порт.

Нужно перевести multitool на другой порт через переменные окружения `HTTP_PORT=8080` и `HTTPS_PORT=11443`.

Исправленный манифест: [deployment-multi.yaml](deployment-multi.yaml)

Скриншот ошибки:

![Ошибка Address in use](img/task1-error.png)


2. После запуска увеличить количество реплик работающего приложения до 2.
## Решение:
Команды:

```bash
kubectl get pods
kubectl scale deployment multi-app --replicas=2
kubectl get pods
```

Количество подов до (1) и после (2) масштабирования:

![Масштабирование](img/task1-scale.png)

3. Продемонстрировать количество подов до и после масштабирования.
## Решение:
Манифест: [service-multi.yaml](service-multi.yaml)

Service `multi-svc` (ClusterIP) пробрасывает порт 80 на nginx и порт 8080 на multitool. В Endpoints адреса обеих реплик.

4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
## Решение:
Манифест отдельного пода с multitool: [pod-multitool.yaml](pod-multitool.yaml)

Команды:

```bash
kubectl describe svc multi-svc
kubectl exec multitool-client -- curl -s multi-svc:80
kubectl exec multitool-client -- curl -s multi-svc:8080
```

На порт 80 отвечает nginx, на порт 8080 отвечает multitool.

![Service и curl](img/task1-svc-curl.png)

5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.
------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
## Решение:
Манифесты:
- Deployment: [deployment-init.yaml](deployment-init.yaml)
- Service: [service-init.yaml](service-init.yaml)

Init-контейнер на `busybox:1.28` в цикле проверяет через `nslookup`, что DNS-имя Service `nginx-init-svc` разрешается.
Пока Service не создан, основной контейнер nginx не стартует.
Под в статусе `Init:0/1`, в логе init-контейнера повторяется `waiting for service`:

```bash
kubectl get pods -l app=nginx-init
kubectl logs -l app=nginx-init -c wait-for-svc --tail=3
```

![Состояние до](img/task2-before.png)



2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
## Решение:
```bash
kubectl apply -f service-init.yaml
kubectl get pods -l app=nginx-init -w
kubectl get svc,endpoints nginx-init-svc
```

После создания Service init-контейнер завершился, и под прошёл `Init:0/1` → `PodInitializing` → `Running`.

![Состояние после](img/task2-after.png)
3. Создать и запустить Service. Убедиться, что Init запустился.
## Решение:

4. Продемонстрировать состояние пода до и после запуска сервиса.
## Решение:

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------
