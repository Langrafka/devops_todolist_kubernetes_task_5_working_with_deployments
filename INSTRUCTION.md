How to deploy the Todoapp to Kubernetes

1. Створення Namespace

Всі ресурси, пов'язані з застосунком, будуть розгорнуті у просторі імен mateapp.

kubectl create namespace mateapp


2. Перевірка та Розгортання Маніфестів

2.1. Standalone Pod для перевірки (Pod Parity): pod.yml

Для дотримання вимоги паритету конфігурація Deployment Pod Template є ідентичною вмісту окремого Pod Manifest (pod.yml).

kubectl apply -f pod.yml -n mateapp
# Видаліть Pod після перевірки, щоб уникнути конфліктів
kubectl delete pod todoapp-standalone -n mateapp


2.2. Deployment: deployment.yml

Застосуйте Deployment, який створює 2 репліки застосунку.

kubectl apply -f deployment.yml -n mateapp


Обґрунтування ресурсних запитів (Requests) та лімітів (Limits):

CPU Requests (100m) / Limits (250m):

Request (100m): Мінімально гарантовані ресурси для планувальника.

Limit (250m): Обмежує використання 25% ядра.

Memory Requests (128Mi) / Limits (256Mi):

Request (128Mi): Базовий мінімум пам'яті.

Limit (256Mi): Обмежує споживання пам'яті, запобігаючи OOM-Killed.

Обґрунтування стратегії RollingUpdate:
Обрано стратегію RollingUpdate для оновлень без простою (Zero Downtime).

maxUnavailable: 1: Гарантує, що при replicas: 2 мінімум один под завжди залишається робочим.

maxSurge: 1: Дозволяє створити один додатковий под, забезпечуючи плавний перехід.

2.3. Horizontal Pod Autoscaler: hpa.yml

Застосуйте HPA. Увага: Для роботи HPA потрібен Metrics Server (для CPU та Memory метрик).

kubectl apply -f hpa.yml -n mateapp


Обґрунтування HPA:

minReplicas: 2: Забезпечує високу доступність (HA).

maxReplicas: 5: Встановлює верхню межу для масштабування, контролюючи витрати.

averageUtilization: 70%: Поріг для ініціації масштабування до досягнення критичного навантаження. Працює відносно встановлених requests.

3. Встановлення та Перевірка Service

3.1. Service: service.yml

Створіть NodePort Service для зовнішнього доступу.

kubectl apply -f service.yml -n mateapp


3.2. Перевірка Metrics Server (вимоги HPA)

Перевірте, чи встановлено Metrics Server та чи доступні метрики:

# Перевірка наявності Metrics Server
kubectl get deployment metrics-server -n kube-system

# Перевірка доступності метрик (повинні бути значення, а не <unknown>)
kubectl top nodes
kubectl top pods -n mateapp


3.3. Перевірка HPA

Перевірте статус HPA. Увага: Для спрацьовування масштабування необхідне навантаження.

kubectl get hpa -n mateapp
kubectl describe hpa todoapp -n mateapp


4. Тестування доступу та масштабування

4.1. Отримання IP та Port

Отримайте динамічний NodePort:

NODE_PORT=$(kubectl get svc todoapp-nodeport -n mateapp -o jsonpath='{.spec.ports[0].nodePort}')


Отримайте IP робочої ноди (залежить від кластера):

# Для Minikube використовуйте:
NODE_IP=$(minikube ip)

# Для інших кластерів:
# NODE_IP=$(kubectl get nodes -o wide | awk 'NR>1 {print $6; exit}') 


Експортуйте URL:

echo "Доступний за адресою: http://$NODE_IP:$NODE_PORT"


4.2. Перевірка Health та API

Протестуйте доступ до кінцевих точок (/api/liveness та /api/readiness мають повертати HTTP 200/200/503).

echo "Перевірка головної сторінки:"
curl -I http://$NODE_IP:$NODE_PORT/

echo "Перевірка API Liveness (повертає 200, якщо процес живий):"
curl http://$NODE_IP:$NODE_PORT/api/liveness

echo "Перевірка API Readiness (повертає 200/503 відповідно до затримки):"
curl http://$NODE_IP:$NODE_PORT/api/readiness


4.3. Тест навантаження (HPA Scaling)

Запустіть простий цикл для генерації навантаження (викликає ~100% CPU навантаження на 10-15 секунд) для тестування HPA:

# У новому терміналі:
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- [http://todoapp-nodeport.mateapp.svc.cluster.local:80](http://todoapp-nodeport.mateapp.svc.cluster.local:80); done"


Негайно перевірте стан Pods та HPA в іншому терміналі:

kubectl get pods -n mateapp --watch
kubectl get hpa -n mateapp


Ви маєте побачити, як кількість реплік зросте до 5 (maxReplicas). Після зупинки навантаження (Ctrl+C в терміналі з load-generator) HPA автоматично поверне кількість реплік до 2.