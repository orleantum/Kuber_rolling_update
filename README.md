# Лаб 7. Деплой сервисов в k8s (minikube)

# Задача: «Куберизировать» приложение Flask+Redis из предыдущей практики и обновить

1. Разворачиваем на vm мини-кластер k8s - minikube (Предварительно увеличиваем ресурсы vm)
2. (опционально: Подключаемся к кластеру с хостовой машины через приложение Lens)
3. Собираем образы будущих контейнеров и прокидываем их внутрь minikube
4. Создаем манифесты – описания приложений и сервисов
5. Разворачиваем deployment/ReplicaSet с application-серверами и БД и сервисы
6. Активируем балансировщик входящих запросов (служба LoadBalancer)
7. Проверяем работу сервисов 8. Обновляем веб сервисы на новую версию (RollingUpdate).

# Выполнение

### Скачаем исполняемый файл Minikube:

```Bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```


### Установим его в системную папку `/usr/local/bin`:

```Bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```


### Проверим доступность `minikube`:

```Bash
minikube version
```

![[{0B505879-282B-43D9-912C-1EA4F104F3E2}.png]]


### Запустим Minikube, выделив ему 2 ядра процессора и 3 ГБ оперативной памяти:

```Bash
minikube start --driver=docker --cpus=2 --memory=3000
```

<img width="944" height="442" alt="image" src="https://github.com/user-attachments/assets/dc08ccd1-dc56-4479-8e3a-0eff2eea653c" />



### Подготовка файлов и сборка образа

```Bash
mkdir -p ~/lab7/flask_redis
cd ~/lab7
cp ~/lab6/flask-redis/app.py flask_redis/
cp ~/lab6/flask-redis/requirements.txt flask_redis/
cp ~/lab6/flask-redis/Dockerfile flask_redis/
```

### Соберём первую версию образа (v1) стандартным докером:

```Bash
docker build -t flask:v1 flask_redis/
```


### Прокинем (загрузим) образ внутрь Minikube

```Bash
minikube image load flask:v1
```


### Создаем базовые манифесты приложений:

1. Создадим деплоймент для Redis (`redis-deployment.yaml`):

```Bash
nano redis-deployment.yaml
```

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:alpine
          ports:
            - containerPort: 6379
```


2. Создадим внутренний сервис для Redis (`redis-service.yaml`):

```Bash
nano redis-service.yaml
```

```YAML
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    app: redis
```


3. Создадим деплоймент для Flask (`flask-deployment.yaml`):

```Bash
nano flask-deployment.yaml
```

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask
          image: flask:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
```

4. Создадим внешний сервис для Flask типа LoadBalancer (`flask-service.yaml`):

```Bash
nano flask-service.yaml
```

```YAML
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: LoadBalancer
  selector:
    app: flask-app
  ports:
    - name: http
      port: 8080        # Внешний порт
      targetPort: 5000  # Порт внутри контейнера Flask
```


### Разворачиваем манифесты в кластер:

```Bash
kubectl apply -f .
kubectl get pods
```

<img width="402" height="99" alt="image" src="https://github.com/user-attachments/assets/acd06a25-c1d9-4fa6-8d36-840d05ef93cc" />

<img width="561" height="82" alt="image" src="https://github.com/user-attachments/assets/9cc273de-08cc-4a09-978b-8a4fa05b912f" />



### Активируем балансировщик:

- В Minikube служба LoadBalancer не получит внешний IP-адрес автоматически. Нам нужно запустить туннель
- Откроем новое (второе) окно терминала и запустим в нём туннель:

<img width="441" height="216" alt="image" src="https://github.com/user-attachments/assets/b943a8db-2187-4f0f-9ea9-3512c0c3b983" />


Вернёмся в первый терминал и проверим статус сервисов:

```Bash
kubectl get services
```

<img width="777" height="101" alt="image" src="https://github.com/user-attachments/assets/9317de37-e703-43b3-9fd4-cb98a6d7f114" />


У службы `flask-service` в колонке `EXTERNAL-IP` назначен адрес `10.106.70.101`:

<img width="515" height="43" alt="image" src="https://github.com/user-attachments/assets/f6b42957-67a6-4bd9-983a-be75163fe2f3" />



# Обновление на новую версию (`RollingUpdate`)

Суть `RollingUpdate` - обновить приложение без его остановки.

### Изменим наше Flask-приложение:

```Python
import time
import redis
from flask import Flask, Response
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

VISITS_COUNTER = Counter('flask_site_visits_total', 'Total number of site visits')

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    VISITS_COUNTER.inc()
    return f'Этот сайт открывали {count} раз(а). [Версия 2]\n'

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```


### Соберём новую версию образа (v2):

```Bash
docker build -t flask:v2 flask_redis/
```

<img width="771" height="535" alt="image" src="https://github.com/user-attachments/assets/153bbb22-577c-4373-89b7-d9c6b140a9a8" />



### Загрузим новый образ внутрь Minikube:

```Bash
minikube image load flask:v2
```


### Запустим RollingUpdate:

```Bash
kubectl set image deployment/flask-app flask=flask:v2
```


### Посмотреть процесс обновления:

```Bash
kubectl rollout status deployment/flask-app
```

<img width="629" height="42" alt="image" src="https://github.com/user-attachments/assets/2e10d913-cd0c-4480-8bcc-7e97805ddf24" />



### Проверим результат:

```Bash
curl http://localhost:8080
```

<img width="526" height="83" alt="image" src="https://github.com/user-attachments/assets/d4538b3e-46ab-4759-9291-ddd9773f51eb" />
