1. 네임 스페이스 확인
[cka0001@k8s-master ~]$ kubectl get pod --all-namespaces
NAMESPACE         NAME                                        READY   STATUS    RESTARTS         AGE
calico-system     calico-apiserver-5c57668876-t995p           1/1     Running   0                3h53m
calico-system     calico-apiserver-5c57668876-tlddt           1/1     Running   0                3h53m
calico-system     calico-kube-controllers-58bf77846b-mmc9l    1/1     Running   0                3h52m
calico-system     calico-node-9jt5d                           1/1     Running   1 (106m ago)     3h53m
calico-system     calico-typha-84dc677674-x8hxn               1/1     Running   0                3h53m
calico-system     csi-node-driver-sfxz6                       2/2     Running   0                3h52m
calico-system     goldmane-5bb799dbd6-56d2v                   1/1     Running   0                3h53m
calico-system     whisker-6d48d8c8c8-wpqlf                    2/2     Running   0                3h50m
cert-manager      cert-manager-5776fd696b-2wbnd               1/1     Running   5 (8m39s ago)    3h49m
cert-manager      cert-manager-cainjector-5dc4bf4cdf-hlfgc    1/1     Running   4 (8m39s ago)    3h49m
cert-manager      cert-manager-webhook-6ff7dcb868-d2cs4       1/1     Running   0                3h49m
ingress-nginx     ingress-nginx-controller-5f54fcdbd7-xwb5f   1/1     Running   0                3h53m
kube-system       coredns-66bc5c9577-27m5r                    1/1     Running   0                3h53m
kube-system       coredns-66bc5c9577-pwggs                    1/1     Running   0                3h53m
kube-system       etcd-k8s-master                             1/1     Running   0                3h54m
kube-system       kube-apiserver-k8s-master                   1/1     Running   0                3h54m
kube-system       kube-controller-manager-k8s-master          1/1     Running   8 (8m39s ago)    3h54m
kube-system       kube-proxy-t2tdz                            1/1     Running   0                3h53m
kube-system       kube-scheduler-k8s-master                   1/1     Running   8 (8m39s ago)    3h54m
kube-system       metrics-server-5b57c547cb-vj7m8             1/1     Running   10 (2m39s ago)   3h53m
nginx-gateway     ngf-nginx-gateway-fabric-869fb69457-xnpsb   1/1     Running   7 (8m39s ago)    3h49m
nginx-static      nginx-static-7f74676cb8-sfw7n               1/1     Running   0                7m28s
tigera-operator   tigera-operator-8f8bdf4dd-xr5n4             1/1     Running   7 (8m39s ago)    3h53m

2. configmap(nginx-static) 파일 확인
[cka0001@k8s-master ~]$ kubectl get configmap -n nginx-static nginx-config -o yaml
apiVersion: v1
data:
  nginx.conf: |
    events {}

    http {
      server {
        listen 443 ssl;
        server_name web.k8s.local;

        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;

        ssl_protocols TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location / {
          return 200 "OK\n";
        }
      }
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2026-04-26T06:18:52Z"
  name: nginx-config
  namespace: nginx-static
  resourceVersion: "18610"
  uid: 7f1400b8-26c9-45ec-9ab7-a78ae0a65464

3. configmap 파일 수정
[cka0001@k8s-master ~]$ kubectl edit configmap -n nginx-static nginx-config
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  nginx.conf: |
    events {}

    http {
      server {
        listen 443 ssl;
        server_name web.k8s.local;

        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location / {
          return 200 "OK\n";
        }
      }
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2026-04-26T06:18:52Z"
  name: nginx-config
  namespace: nginx-static
  resourceVersion: "19711"
  uid: 7f1400b8-26c9-45ec-9ab7-a78ae0a65464

4. 실행 확인
[cka0001@k8s-master ~]$ curl -k --tls-max 1.2 https://web.k8s.local:30007
curl: (35) error:0A00042E:SSL routines::tlsv1 alert protocol version
[cka0001@k8s-master ~]$ kubectl rollout restart deployment nginx-static -n nginx-static
deployment.apps/nginx-static restarted
[cka0001@k8s-master ~]$ curl -k --tls-max 1.2 https://web.k8s.local:30007
OK
