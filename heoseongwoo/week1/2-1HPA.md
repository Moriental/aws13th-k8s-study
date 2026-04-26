1. autoscale 고정 및 HPA의 yaml파일 생성(cpu 사용량50% 이상일 때 파드 최대 4개까지 증가)
[cka0001@k8s-master ~]$ kubectl config set-context --current --namespace=autoscale
Context "kubernetes-admin@kubernetes" modified.
[cka0001@k8s-master ~]$ kubectl autoscale deployment apache-server --cpu-percent=50 --min=1 --max=4 --dry-run=client -o yaml > hpa.yaml

2. 만들어진 hpa.yaml 파일 수정(30초 대기 후 생성하도록 설정)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-server
  namespace: autoscale
spec:
  maxReplicas: 4
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-server
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30

3. 설정 적용 및 확인
   [cka0001@k8s-master ~]$ kubectl apply -f hpa.yaml
horizontalpodautoscaler.autoscaling/apache-server created
[cka0001@k8s-master ~]$ kubectl describe hpa apache-server
Name:                                                  apache-server
Namespace:                                             autoscale
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Sun, 26 Apr 2026 16:02:55 +0900
Reference:                                             Deployment/apache-server
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  2% (1m) / 50%
Min replicas:                                          1
Max replicas:                                          4
Behavior:
  Scale Up:
    Stabilization Window: 0 seconds
    Select Policy: Max
    Policies:
      - Type: Pods     Value: 4    Period: 15 seconds
      - Type: Percent  Value: 100  Period: 15 seconds
  Scale Down:
    Stabilization Window: 30 seconds
    Select Policy: Max
    Policies:
      - Type: Percent  Value: 100  Period: 15 seconds
Deployment pods:       1 current / 1 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:           <none>
