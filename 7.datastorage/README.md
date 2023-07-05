# Подключаем ceph в кластер kubernetes

* [1. rbd](rbd/README.md)
* [2. cephfs](cephfs/README.md)
* [3. fileshare](fileshare/README.md)


## !! Будем работать на двух серверах: node-1 и master-1 под root !!
## !! Все комадны `kubectl` выполняются на master-1 !!
## !! Команды `ceph` и `rbd` выполняются на node-1 !!

* [1. Устанавливаем CSI Driver для Ceph RBD](rbd/README.md)
* [2. Устанавливаем CSI Driver для Ceph FS](cephfs/README.md)
* [3. Запускаем приложение fileshare](fileshare/README.md)
