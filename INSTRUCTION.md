How to deploy the Todoapp to Kubernetes

1. Створення Namespace

Всі ресурси, пов'язані з застосунком, будуть розгорнуті у просторі імен mateapp.

kubectl create namespace mateapp


2. Розгортання Маніфестів

2.1. Deployment: deployment.yml

Застосуйте Deployment, який створює 2 репліки застосунку.

kubectl apply -f deployment.yml -n mateapp


Обґрунтування ресурсних запитів (Requests) та лімітів (Limits):

CPU Requests (100m) / Limits (250m):

Request (100m): Мінімально гарантовані ресурси для планувальника.

Limit (250m): Обмежує використання 25% ядра, захищаючи вузол від неконтрольованих сплесків.

Memory Requests (128Mi) / Limits (256Mi):

Request (128Mi): Базовий мінімум пам'яті для стабільної роботи.

Limit (256Mi): Обмежує споживання пам'яті, запобігаючи OOM-Killed (видаленню через нестачу пам'яті).

Обґрунтування стратегії RollingUpdate:
Обрано стратегію RollingUpdate для оновлень без простою (Zero Downtime).

maxUnavailable: 1: Гарантує, що при replicas: 2 мінімум один под завжди залишається робочим.

maxSurge: 1: Дозволяє створити один додатковий под, забезпечуючи плавний перехід.

2.2. Horizontal Pod Autoscaler: hpa.yml

Застосуйте HPA. Увага: Для роботи HPA у кластері має бути встановлений Metrics Server.

kubectl apply -f hpa.yml -n mateapp


Обґрунтування HPA:

minReplicas: 2: Відповідає вимозі завдання (стан спокою) та забезпечує високу доступність (HA).

maxReplicas: 5: Встановлює верхню межу для масштабування, контролюючи витрати.

averageUtilization: 70%: Поріг для ініціації масштабування до досягнення критичного навантаження. Він працює для CPU та Memory (відносно встановлених requests).

3. Створення Service

Створіть Service (наприклад, NodePort, який відправляє трафік на порт 8000 контейнера) для зовнішнього доступу та застосуйте його.

# Припускаємо, що service.yml створює NodePort Service
kubectl apply -f service.yml -n mateapp


4. Тестування доступу

Отримайте URL для доступу через NodePort:

NODE_PORT=$(kubectl get svc todoapp -n mateapp -o jsonpath='{.spec.ports[0].nodePort}')
echo "Доступний за адресою: http://<NODE_IP>:$NODE_PORT"


Для локального тестування (якщо Service має назву todoapp):

kubectl port-forward svc/todoapp 8000:8000 -n mateapp


Застосунок буде доступний за адресою http://localhost:8000.