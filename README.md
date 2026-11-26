# Домашнее задание к занятию «Helm»

## Задание 1. Подготовить Helm-чарт для приложения

### 1. Упаковать приложение в чарт для деплоя в разные окружения

Для выполнения задания использовалось следующее окружение:

- Kubernetes — v1.35.6;
- MicroK8s;
- Helm — v3.19.2;
- Ubuntu 22.04 LTS.

Создан Helm-чарт `netology-app`, содержащий два компонента приложения: frontend и backend.

Структура Helm-чарта:

```
netology-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment-frontend.yaml
    ├── deployment-backend.yaml
    ├── service-frontend.yaml
    └── service-backend.yaml
```

Основные файлы чарта:

- [Chart.yaml](netology-app/Chart.yaml)
- [values.yaml](netology-app/values.yaml)

Для проверки корректности Helm-чарта выполнена команда:

```
microk8s helm3 lint ./netology-app
```

Результат:

```
==> Linting ./netology-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

Helm-чарт успешно прошёл проверку без ошибок.

<img width="678" height="167" alt="image" src="https://github.com/user-attachments/assets/13811d3c-61bb-4402-8f0b-834808a9ee22" />

---

### 2. Каждый компонент приложения деплоится отдельным Deployment

Приложение состоит из двух компонентов:

- frontend — nginx;
- backend — wbitt/network-multitool.

Для каждого компонента создан отдельный Deployment и Service типа ClusterIP.

Манифесты Deployment:

- [deployment-frontend.yaml](netology-app/templates/deployment-frontend.yaml)
- [deployment-backend.yaml](netology-app/templates/deployment-backend.yaml)

Манифесты Service:

- [service-frontend.yaml](netology-app/templates/service-frontend.yaml)
- [service-backend.yaml](netology-app/templates/service-backend.yaml)

Для проверки генерации манифестов выполнена команда:

```
microk8s helm3 template test-release ./netology-app
```

В результате Helm сформировал два Deployment и два Service.

Для формирования уникальных имён ресурсов используется переменная:

```
name: {{ .Release.Name }}-frontend
```

Благодаря этому несколько экземпляров приложения могут быть развёрнуты в одном namespace без конфликта имён.

<img width="546" height="897" alt="image" src="https://github.com/user-attachments/assets/fdede71b-c470-44c1-a3a3-8b2d6bb64537" />

---

### 3. Изменить образ приложения через переменные чарта для изменения версии

Настройки образов вынесены в файл `values.yaml`.

```
frontend:
  replicaCount: 1
  image:
    repository: nginx
    tag: "1.27"
  service:
    port: 80

backend:
  replicaCount: 1
  image:
    repository: wbitt/network-multitool
    tag: "latest"
  service:
    port: 80
```

В шаблоне Deployment используется следующая конструкция:

```
image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
```

Для изменения версии образа при установке приложения используется параметр:

```
--set-string frontend.image.tag=1.26
```

Таким образом, один Helm-чарт позволяет разворачивать приложения с различными версиями образов без изменения шаблонов Kubernetes.

**Результат:** Helm-чарт успешно создан, проверен и подготовлен для развёртывания приложения в различных окружениях.

---

## Задание 2. Запустить две версии в разных неймспейсах

### 1. Проверить чарт и запустить несколько копий приложения

Для развёртывания приложений созданы два namespace:

```
kubectl create namespace app1
kubectl create namespace app2
```

Проверка:

```
kubectl get namespaces
```

Результат:

```
NAME   STATUS
app1   Active
app2   Active
```

---

### 2. Запустить одну версию в namespace app1, вторую версию в том же namespace и третью версию в namespace app2

Установка первой версии приложения:

```
microk8s helm3 install app-v1 ./netology-app \
  --namespace app1 \
  --set-string frontend.image.tag=1.27
```

Установка второй версии приложения:

```
microk8s helm3 install app-v2 ./netology-app \
  --namespace app1 \
  --set-string frontend.image.tag=1.26
```

Установка третьей версии приложения:

```
microk8s helm3 install app-v3 ./netology-app \
  --namespace app2 \
  --set-string frontend.image.tag=1.28
```

В результате были установлены три Helm-релиза:

| Релиз | Namespace | Frontend | Backend |
|---|---|---|---|
| app-v1 | app1 | nginx:1.27 | multitool:latest |
| app-v2 | app1 | nginx:1.26 | multitool:latest |
| app-v3 | app2 | nginx:1.28 | multitool:latest |

---

### 3. Продемонстрировать результат

#### Проверка Helm-релизов

Выполнена команда:

```
microk8s helm3 list -A
```

Результат (релизы развёрнутого приложения):

```
NAME     NAMESPACE   REVISION   STATUS     CHART
app-v1   app1        1          deployed   netology-app-0.1.0
app-v2   app1        1          deployed   netology-app-0.1.0
app-v3   app2        1          deployed   netology-app-0.1.0
```

Все три релиза успешно установлены и находятся в состоянии `deployed`.

<img width="983" height="95" alt="image" src="https://github.com/user-attachments/assets/eefa4dd4-1b04-4f59-81fa-f4798afaa43b" />

#### Проверка ресурсов в namespace app1

```
kubectl get pods,deployments,svc -n app1
```

В namespace `app1` успешно работают четыре Pod:

```
app-v1-backend-5bdbb99b5-5fzwl     1/1   Running
app-v1-frontend-67f95bdfb8-vwt2l   1/1   Running
app-v2-backend-9948f9cc8-t7dd8     1/1   Running
app-v2-frontend-68c64dccfd-x9p4l   1/1   Running
```

Также созданы четыре Deployment и четыре Service.

<img width="526" height="236" alt="image" src="https://github.com/user-attachments/assets/4db6e91f-6579-4e05-b8b1-0e465d9cb5dd" />

#### Проверка ресурсов в namespace app2

```
kubectl get pods,deployments,svc -n app2
```

В namespace `app2` успешно работают два Pod:

```
app-v3-backend-65494c5dc5-9tsv6    1/1   Running
app-v3-frontend-7cff48dc75-ltxsb  1/1   Running
```

Также созданы два Deployment и два Service.

<img width="519" height="162" alt="image" src="https://github.com/user-attachments/assets/63ea03c9-3114-45e9-a4ee-8b709b2e4680" />

#### Проверка версий образов

Для проверки версий образов выполнены команды:

```
kubectl get deployments -n app1 \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image'
```

Результат:

```
NAME              IMAGE
app-v1-backend    wbitt/network-multitool:latest
app-v1-frontend   nginx:1.27
app-v2-backend    wbitt/network-multitool:latest
app-v2-frontend   nginx:1.26
```

Проверка второго namespace:

```
kubectl get deployments -n app2 \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image'
```

Результат:

```
NAME              IMAGE
app-v3-backend    wbitt/network-multitool:latest
app-v3-frontend   nginx:1.28
```

<img width="563" height="156" alt="image" src="https://github.com/user-attachments/assets/b8e817c9-654f-4468-9342-fcc92cea180e" />

Несмотря на одинаковое значение `APP VERSION` в выводе Helm, фактические версии образов nginx различаются, что подтверждается выводом kubectl.

**Результат:** три экземпляра приложения с различными версиями nginx успешно развёрнуты с помощью одного Helm-чарта в двух namespace.
