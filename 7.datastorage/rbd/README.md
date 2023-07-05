# Устанавливаем ceph CSI driver для Ceph RBD

### Проверяем состояние кластера

1) С sbox заходим на node-1

```bash
ssh node-1.s<номер вашего логина>
sudo -s
```
2) Проверяем что цеф живой

```bash
ceph health
ceph -s
```

### Создаем пул в цефе для RBD дисков

3) Запускаем на на node-1

```bash
ceph osd pool create kube 32
ceph osd pool application enable kube rbd
```

### Устанавливаем ceph CSI driver для RBD

4) Добавляем репозиторий с чартом, получаем набор переменных чарта ceph-csi-rbd

```
helm repo add ceph-csi https://ceph.github.io/csi-charts

mkdir -p /srv/ceph
cd /srv/ceph

helm inspect values ceph-csi/ceph-csi-rbd > cephrbd.yml
```

### Заполняем переменные в cephrbd.yml

5) выполняем на node-1, чтобы узнать необходимые параметры:

```bash
#  - clusterID: "<cluster-id>"

ceph fsid
```
```bash
#     monitors:
#       - "<MONValue1>"
#       - "<MONValue2>"

ceph mon dump
```

6) Правим файл cephrbd.yml

Заносим свои значение clusterID, и адреса мониторов.
Включаем создание политик PSP, и увеличиваем таймаут на создание дисков
Список изменений в файле cephrbd.yml. Опции в разделах nodeplugin и provisioner уже есть в файле, их надо исправить так, как показано ниже.

```
csiConfig:
  - clusterID: "bcd0d202-fba8-4352-b25d-75c89258d5ab"
    monitors:
      - "v2:172.18.8.5:3300/0,v1:172.18.8.5:6789/0"
      - "v2:172.18.8.6:3300/0,v1:172.18.8.6:6789/0"
      - "v2:172.18.8.7:3300/0,v1:172.18.8.7:6789/0"

nodeplugin:
  podSecurityPolicy:
    enabled: true

provisioner:
  replicaCount: 1
  podSecurityPolicy:
    enabled: true
```

При необходимости можно свериться с файлом rbd/cephrbd-values-example.yml

### Устанавливаем чарт

7) Выполняем команду

```
helm upgrade -i ceph-csi-rbd ceph-csi/ceph-csi-rbd -f cephrbd.yml -n ceph-csi-rbd --create-namespace
```

### Создаем секрет и storageclass

8) Создаем пользователя в ceph, с правами записи в пул kube 

Запускаем на node-1

```bash
ceph auth get-or-create client.rbdkube mon 'profile rbd' osd 'profile rbd pool=kube'
```

9) Смотрим ключ доступа для пользователя rbdkube

```bash
ceph auth get-key client.rbdkube
```

вывод:
```
AQCO9NJbhYipKRAAMqZsnqqS/T8OYQX20xIa9A==
```

10) Заполняем манифесты

выполняем на master-1, подставляем значение ключа в манифест секрета.

```bash
cd ~/slurm/practice/7.ceph/rbd/
vi secret.yaml

# Заносим значение ключа в 
# userKey: AQBRYK1eo++dHBAATnZzl8MogwwqP/7KEnuYyw==
```

11) Создаем секрет

```bash
kubectl apply -f secret.yaml
```

12) Получаем id кластера ceph

Выполняем на node-1

```bash
ceph fsid
```

13)  Заносим clusterid в storageclass.yaml

Выполняем на master-1

```
vi storageclass.yaml
# clusterID: bcd0d202-fba8-4352-b25d-75c89258d5ab
```

14) Создаем storageclass

```bash
kubectl apply -f storageclass.yaml
```

### Проверяем как работает.

15) Создаем pvc, и проверем статус и наличие pv

```bash
kubectl apply -f pvc.yaml

kubectl get pvc
kubectl get pv
```

### Проверяем создание тома в цеф

16) Получаем список томов в пуле и просматриваем информацию о томе

Выполняем на node-1

```bash
# список томов в пуле kube
rbd ls -p kube

# информация о созданном томе
rbd -p kube info csi-vol-eb3d257d-8c6c-11ea-bff5-6235e7640653
```

### Пробуем как работает resize

17) Изменяем размер тома в манифесте pvc.yaml и применяем

```bash
vi pvc.yaml

resources:
  requests:
    storage: 2Gi

kubectl apply -f pvc.yaml
```

18) Немножко ждем и проверяем размер тома в ceph, pv, pvc

```bash
rbd -p kube info csi-vol-eb3d257d-8c6c-11ea-bff5-6235e7640653

kubectl get pv
kubectl get pvc
```

### Разбираемся почему у pvc размер не изменился

19) Смотрим его манифест в yaml

```bash
kubectl get pvc rbd-pvc -o yaml
```

То увидим сообщение message: Waiting for user to (re-)start a pod to finish file system resize of volume on node.
type: FileSystemResizePending
Необходимо смонтировать том, для того чтобы увеличить на нем файловую систему. А у нас этот PVC/PV не используется никаким подом.

20) Создаем под, который использует этот PVC/PV, и смотрим на размер, указанный в pvc

```bash
kubectl apply -f pod.yaml

kubectl get pvc
```
