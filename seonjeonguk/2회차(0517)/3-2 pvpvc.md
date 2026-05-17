# 3-2. PV/PVC - MariaDB Deployment 복구

## 문제 요약

`mariadb` namespace에 있던 MariaDB Deployment가 삭제되었다.  
기존 데이터가 보존된 PersistentVolume(PV)을 재사용하여 MariaDB Deployment를 다시 복구해야 한다.

### 요구사항
- 기존 PV는 이미 존재하며 재사용해야 한다.
- `mariadb` namespace에 PVC를 생성한다.
  - PVC 이름: `mariadb`
  - Access Mode: `ReadWriteOnce`
  - Storage: `250Mi`
- `~/mariadb-deploy.yaml` 파일을 수정하여 생성한 PVC를 사용하도록 한다.
- 수정한 Deployment를 적용한다.
- MariaDB Deployment가 정상적으로 Running / Stable 상태인지 확인한다.

---

## 핵심 개념

### PV와 PVC
- **PV(PersistentVolume)**: 클러스터에 존재하는 실제 저장소 자원
- **PVC(PersistentVolumeClaim)**: Pod가 사용할 저장소를 요청하는 리소스

Pod는 PV를 직접 사용하는 것이 아니라,  
**PVC를 통해 PV를 사용한다.**

```text
Pod → PVC → PV → 실제 저장소

## 2. PVC 생성

문제 조건에 맞게 `mariadb` namespace에 PVC를 생성한다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
  storageClassName: local-path
```

```bash
kubectl apply -f mariadb-pvc.yml
```

PVC가 기존 PV와 정상적으로 연결되었는지 확인한다.

```bash
kubectl get pvc -n mariadb
kubectl get pv
```

---

## 3. Deployment 수정 및 적용

문제에서 제공한 `~/mariadb-deploy.yaml` 파일을 수정한다.

기존 Deployment는 `mariadb-pvc`라는 PVC를 참조하고 있으므로,  
문제에서 생성한 PVC 이름인 `mariadb`로 변경한다.

```yaml
volumes:
  - name: mysql-persistent-storage
    persistentVolumeClaim:
      claimName: mariadb
```

수정한 Deployment를 적용한다.

```bash
kubectl apply -f ~/mariadb-deploy.yaml
```

---

## 4. 최종 확인

MariaDB Deployment와 Pod가 정상적으로 실행 중인지 확인한다.

```bash
kubectl get pods -n mariadb
kubectl get deploy -n mariadb
```

정상적으로 완료되면 아래 상태를 확인할 수 있다.

```text
PVC        Bound
PV         Bound
Pod        Running
Deployment READY 1/1
```

## 실제 출력 결과

[cka0001@k8s-master 2회차]$ kubectl get all -n mariadb
NAME                           READY   STATUS    RESTARTS   AGE
pod/mariadb-5b556fdf59-24ss2   1/1     Running   0          2m44s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mariadb   1/1     1            1           17m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/mariadb-5b556fdf59   1         1         1       2m44s
replicaset.apps/mariadb-7b6f6cc6b5   0         0         0       17m

