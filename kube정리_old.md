# Kubernetes

## Kubertnets cluster 기초를 먼저 살짝..

먼저 클러스터를 만들고 시작한다. 

어디서 만드는지? 당연히 EC2 서버를 하나 올리고 거기에 쿠버네티스를 작동시킬 프로그램들을 깔아서 진행한다.

나의 경우, k8s-management-server 로 이름을 보통 짓는다.

```shell
# kops create cluster 명령어를 사용한다.
kops create cluster --cloud=aws --zones=ap-south-1b --name=demo.k8s.valaxy.net --dns-zone=valaxy.net --dns private
```

그리고, 이렇게 하면 바로 만들어 지진 않는다. 위 명령어들로 클러스터 정보만 만드는 것이다.

```shell
kops update cluster
# 이게 실제로 만드는 명령어이다.
```

만든 클러스터 정보는 만드시 어디에 정의해두어야 한다. local에 해도 되는데 보통 s3에 저장을 한다. 

환경변수 명 :  KOPS_STATE_STORE 이다. 여기에 경로를 두어야 한다.

```shell
export KOPS_STATE_STORE=s3://proto.k8s.vinapay.ga
# 이렇게 export 를 하고 꼭 kops create, update를 진행해라.
```

**환경변수가 설정되어 있지 않으면 cluster 정보를 불러올수 없다.**

몇일이 지나 k8s-management-server에 들어가서 cluster 정보를 볼려고 해보았다.

```shell
kops get clusters
# 정보가 안나와서 난 없어진줄 알았다. 그래서 새로 만드는 말도 안되는 실수를 하였다.
KOPS_STATE_STORE=s3://proto.k8s.vinapay.ga
# 이렇게 하고 다시 하면 된다.
kops get clusters
NAME			CLOUD	ZONES
proto.k8s.vinapay.ga	aws	ap-northeast-2c
```



## 웹 UI 대쉬보드

쿠버네티스 리소스 상태를 볼수 있는 대쉬보드를 한번 설치해서 해본다.

현재 나는 외부에서 promotion app 에 접속이 안된다. 문제를 찾을 수가 없고 상태도 볼수 없다.

먼저 dashboard를 만든다. 이미 만들어 져 있는게 있을수 있으니 확인한다. 

```shell
k get pods -n kubernetes-dashboard
k get deployments -n kubernetes-dashboard
k get services -n kubernetes-dashboard
k get ingress -n kubernetes-dashboard
# 참고로 pods의 경우는 지울때 deployments를 지워야 지워지지 pods를 delete 하면 deployment 가 계속 살린다.
```

아무것도 없으면 설치한다.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
# 설치된거 확인도 위에 get 명령어로 하면된다. 참고로 ingress는 따로 설치할꺼다.
k apply -f dashboard-ingress.yaml
# yaml file은 아래와 같다.
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
spec:
  rules:
  - host: dashboard.vinapay.ga
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
...

# 만든뒤 ADDRESS 부분에 주소가 들어와야 하는데 시간이 좀 걸린다고 한다. watch option으로 보자.
k get ingress -n kubernetes-dashboard --watch 
NAME                HOSTS                  ADDRESS   PORTS   AGE
dashboard-ingress   dashboard.vinapay.ga             80      69s

```





