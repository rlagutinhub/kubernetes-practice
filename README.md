# kubernetes-practice
Kubernetes manifests examples

* backup etcd
> Хорошей идеей будет добавить подобную команду в cron и выполнять ее каждую ночь.
```
# etcdctl snapshot save file_to_save_snapshot
kubectl exec -n kube-system etcd-minikube.xxx.cluster.local -- /bin/sh -c 'ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /root/minikube-snapshot.db'
```
