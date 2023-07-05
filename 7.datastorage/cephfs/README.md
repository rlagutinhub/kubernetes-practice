# Устанавливаем ceph CSI driver для CephFS

1) Получаем переменные чарта ceph-csi-cephfs

```
cd /srv/ceph
helm inspect values ceph-csi/ceph-csi-cephfs >cephfs.yml
```

### Заполняем переменные в cephfs.yml

2) Узнаем необходимые параметры:

Выполняем на node-1

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

3) Правим файл cephfs.yml

Заносим свои значение clusterID, и адреса мониторов.
Включаем создание политик PSP и увеличиваем таймаут на создание дисков

## !! Внимание, адреса мониторов указываются в простой форме адреc:порт, потому что они передаются в модуль ядра для монтирования cephfs на узел.
## !! А модуль ядра еще не умеет работать с протоколом мониторов v2

> Список изменений в файле cephfs.yml. Опции в разделах nodeplugin и provisioner уже есть в файле, их надо исправить так как показано ниже.

```
csiConfig:
  - clusterID: "bcd0d202-fba8-4352-b25d-75c89258d5ab"
    monitors:
      - "172.18.8.5:6789"
      - "172.18.8.6:6789"
      - "172.18.8.7:6789"

nodeplugin:
  httpMetrics:
    enabled: true
    containerPort: 8091
  podSecurityPolicy:
    enabled: true

provisioner:
  replicaCount: 1
  podSecurityPolicy:
    enabled: true
```

> При необходимости можно свериться с файлом cephfs/cephfs-values-example.yml

### Устанавливаем чарт

4) Запускаем на master-1

```
helm upgrade -i ceph-csi-cephfs ceph-csi/ceph-csi-cephfs -f cephfs.yml -n ceph-csi-cephfs --create-namespace
```

### Создаем Secret и StorageClass

> Провизионер для CephFS создает отдельных пользователей для каждого pv, поэтому в документации написано, что необходимо права администратора в кластере ceph.
> Но мы создадим отдельного пользователя fs, с более ограниченными правами

5) Создаем пользователя для cephfs

Возвращаемся на node-1

```bash
ceph auth get-or-create client.fs mon 'allow r' mgr 'allow rw' mds 'allow rws' osd 'allow rw pool=cephfs_data, allow rw pool=cephfs_metadata'
```

6) Посмотреть ключ доступа для пользователя fs

```bash
ceph auth get-key client.fs
```

вывод:
```
AQCO9NJbhYipKRAAMqZsnqqS/T8OYQX20xIa9A==
```

7) Заполняем манифест secret.yml

Выполняем на master-1

```bash
cd ~/slurm/practice/7.ceph/cephfs/
vi secret.yaml
# Заносим значение ключа в
# adminID: fs
# adminKey: AQCO9NJbhYipKRAAMqZsnqqS/T8OYQX20xIa9A==
```

8) Создаем секрет

```bash
kubectl apply -f secret.yaml
```

9) Получаем id кластера ceph

Выполняем на node-1

```bash
ceph fsid
```

10) Заносим clusterid в storageclass.yaml

Выполняем на master-1

```
vi storageclass.yaml
# clusterID: bcd0d202-fba8-4352-b25d-75c89258d5ab
```

11) Создаем storageclass

```bash
kubectl apply -f storageclass.yaml
```

### Проверяем как работает.

12) Создаем pvc, и проверяем статус и наличие pv

```bash
kubectl apply -f pvc.yaml

kubectl get pvc
kubectl get pv
```

### Проверяем создание каталога в cephfs

13) монтируем CephFS на node-1

Выполняем на node-1

```bash
# Точка монтирования
mkdir -p /mnt/cephfs

# Создаем файл с ключом администратора
ceph auth get-key client.admin >/etc/ceph/secret.key

# Добавляем запись в /etc/fstab
# !! Изменяем ip адрес на адрес узла node-1
echo "172.<xx>.<yyy>.6:6789:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/secret.key,noatime,_netdev    0       2">>/etc/fstab

mount /mnt/cephfs
```

14) Идем в каталог /mnt/cephfs и смотрим что там есть

```
cd /mnt/cephfs
```

### Пробуем как работает resize

15) Изменяем размер тома в манифесте pvc.yaml

Возвращаемся на master-1

```bash
vi pvc.yaml

resources:
  requests:
    storage: 7Gi

kubectl apply -f pvc.yaml
```

16) Проверяем на node-1

```bash
yum install -y attr

getfattr -n ceph.quota.max_bytes <каталог-с-данными>
```
