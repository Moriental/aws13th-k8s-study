# 3-2 task-pvpvc 정리

## 1. 문제 핵심 요구사항

MariaDB Deployment가 삭제된 상황에서, 기존 PersistentVolume(PV)을 재사용하여 MariaDB Deployment를 복구하는 문제이다.

핵심은 **데이터를 보존하기 위해 기존 PV를 삭제하거나 새로 만들지 않고, PVC를 통해 기존 PV에 다시 연결하는 것**이다.

---

## 2. 요구사항 정리

| 항목 | 요구사항 |
|---|---|
| Namespace | `mariadb` |
| PVC 이름 | `mariadb` |
| Access Mode | `ReadWriteOnce` |
| Storage | `250Mi` |
| 기존 PV | 재사용해야 함 |
| Deployment 파일 | `~/mariadb-deploy.yaml` 수정 |
| 최종 상태 | MariaDB Deployment Running & Stable |



## pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
  namespace: mariadb
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 250Mi

```




## Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: mariadb
spec:
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
        - name: mariadb
          image: busybox
          command: ['sh', '-c', 'echo "init data" > /var/lib/mysql/data.db && sleep 3600']
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mariadb-pvc


```





```
[root@k8s-master ~]# kubectl get all -n mariadb 
NAME                           READY   STATUS    RESTARTS   AGE
pod/mariadb-7b6f6cc6b5-vc7gp   1/1     Running   0          48m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mariadb   1/1     1            1           48m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/mariadb-7b6f6cc6b5   1         1         1       48m
------------------------------------------------------------------------

[root@k8s-master ~]# kubectl describe pvc -n mariadb 
Name:          mariadb-pvc
Namespace:     mariadb
StorageClass:  local-path
Status:        Bound
Volume:        mariadb-pv
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      250Mi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       mariadb-7b6f6cc6b5-vc7gp
Events:        <none>
------------------------------------------------

[root@k8s-master ~]# kubectl describe pv -n mariadb 
Name:            mariadb-pv
Labels:          <none>
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    local-path
Status:          Bound
Claim:           mariadb/mariadb-pvc
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        250Mi
Node Affinity:   <none>
Message:         
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /home/cka0001/mariadb
    HostPathType:  
Events:            <none>

```
