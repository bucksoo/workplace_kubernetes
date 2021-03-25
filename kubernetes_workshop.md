# 쿠버네티스 사용기 #1

쿠버네티스를 이번에 처음 사용하면서 기억할 만한 사항들을 기록합니다.
그리고 **내가 이해했던 방식과 경험**을 기록합니다.

1. 쿠버네티스 정리
2. 시나리오별 쿠버네티스 명령어

## 1. 쿠버네티스 정리

내가 이해하고 알고 있는 부분 틀렸을 수도 있지만 정리합니다.

* 백지에서 시작한다.
* 기반은 AWS이다. 이걸 잘하면 본인 로컬에서도 잘 될것이다.

### 1-1. 쿠버네티스 관리용 리눅스 AWS 환경 설정

어째든 쿠버네티스 관련 작업을 aws에서 할려면 서버가 하나 있어야 한다.(맥북처럼)
이 서버는 실제 쿠버네티스 인프라와는 관계가 없는 서버이다. ( 쿠버네티스 api 호출을 위한 관리용 클라이언트 서버라고 보면 된다.)

1. EC2를 올린다. 리눅스로 올려야 한다. Name:  k8s-mgmt-server 로 한다.

   1. 일단 만들때 기본으로 만든다. 만약 VPC를 다른 곳에 해야 한다면 VPC만큼은 EC2보다 먼저 만들고 진행해야 한다.

   2. EC2를 올린 이후에 프로그램들을 설치한다. ssh를 통해 접속시 pem file을 chmod 600 해줘야 한다.

   3. ```shell
      # awscli 설치 ( aws 명령어 툴 )
      cd ~
      curl https://s3.amazonaws.com/aws-cli/awscli-bundle.zip -o awscli-bundle.zip
      unzip awscli-bundle.zip
      sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
      rm -rf aws*
      aws --version
      
      # kubectl 설치(쿠버네티스 api 툴 -> 쿠버네티스 클러스터에 보내는 요청)
      cd ~
      curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      chmod +x ./kubectl
      sudo mv ./kubectl /usr/local/bin/kubectl
      kubectl version
      
      # kops 설치(클러스터 자체를 만들기 위한 툴)
      curl -LO  https://github.com/kubernetes/kops/releases/download/1.15.0/kops-linux-amd64
      chmod +x kops-linux-amd64
      sudo mv kops-linux-amd64 /usr/local/bin/kops
      kops version
      ```

2. 만든 후 IAM 도 생성한다.

   1. 결과론 적으로 ROUTE53, EC2, IAM, S3 full access 등의 권한이 필요하다.
   2. 이는 이 Role이 노드만들고 dns설정하고 s3만들고 하는 작업들을 진행하기 때문이다.
   3. User 는 새로 만든다. vinapay / vinapay123! 로 만들고 programtic, console access 다 설정한다. 그룹은 넣지 않는다.
   4. vinapay-kubernetes-role이라는 이름으로 역할을 만들고 EC2와 User에 할당한다.
      1. 일단 User에 할당하는 것은 잠시 보류.( policy 도 관계가 있어서)

3. S3 버킷 생성

   S3 버킷 : kops를 통해 만든 상태를 저장하기 위함. 다음에 또 만들어야 하므로.

   ```shell
   aws s3 mb s3://vinapay.k8s.vina.ga
   export KOPS_STATE_STORE=s3://vinapay.k8s.vina.ga
   ```

4. Route53 생성

   **이미 만들어 져 있으니 그대로 둔다. 기존에 있던 api.proto.k8s.vinapay.ga 것들이 어떻게 만들어지는지 보자** 

   ```shell
   Routeh53 --> hosted zones --> created hosted zone  
   Domain Name: vinapay.ga
   Type: Public으로 한다. 아무래도 외부에서 접속해야 할수 있으므로. 추후엔 불가로 하는게 좋을듯.
   ```

5. k8s-mgmt-server 를 위한 ssh key 생성

   ```shell
   ssh-keygen # 비밀번호 없이 생성
   ```

이정도면 된 듯 하다. 이제 쿠버로 간다.

### 1-2. 이정도면 AWS 환경은 되었다. 이제 쿠버네티스를 주무르자

1. 클러스터 환경생성 : 모든 것은 클러스터 부터 시작한다. 클러스터는 쉽게 "프로젝트"라고 이해하자.

   **아래 zones는 리전이 아니다 (중요) , 가용영역이다. **

   ```shell
   kops create cluster --cloud=aws --zones=ap-northeast-2a --name=dev.k8s.vinapay.ga --dns-zone=vinapay.ga --dns public
   ```

   이렇게 클러스터를 만든다는 것은 여러 서버군을 만든다는 것이다. 클러스터에 존재하는 여러 서버들의 목적은 따로 공부할것.

   이 클러스터를 생성하면 기본적으로 aws에서 master 와 node 들 ec2가 자동으로 생긴다.

2. 클러스터 환경확인 : 1번 create cluster는 소위 말하는 ***클러스터 환경*** 을 생성하는 것이다. 실제 서버를 생성하는 것은 아니다. 

   그리고 생성된 환경정보는 위에 정의한 s3에 저장된다. 

   ```shell
   # edit this cluster with: kops edit cluster dev.k8s.vinapay.ga
   # edit your node instance group: kops edit ig --name=dev.k8s.vinapay.ga nodes
   # edit your master instance group: kops edit ig --name=dev.k8s.vinapay.ga master-ap-northeast-2a
   ```

   즉 위의 명령어로 알아서 잘 조절하자. 일반적으로 연습때는 t2.micro 정도만 수정한다.

3. 클러스터 실제 생성:

   ```
   kops update cluster dev.k8s.vinapay.ga --yes
   
   kops validate cluster
   kubectl get nodes --show-labels
   ssh -i ~/.ssh/id_rsa admin@api.dev.k8s.vinapay.ga
   addon 읽을거리: https://github.com/kubernetes/kops/blob/master/docs/addons.md
   ```

   




## 2. 시나리오별 쿠버네티스 명령어

어떤 상황에서 어떤걸 하고 싶을때 등을 정리합니다.
환경은 아래와 같습니다.

* aws ec2에 쿠버네티스를 올린다.
* 물론 mac에 설치해서도 가능하다.

어플리케이션쪽은 기존 Java & Spring 개발자분들이 쓰시는 기술 스택과 크게 차이가 안날것 같습니다.  
이외 나머지 기술들을 선택했던 이유들을 하나씩 소개하겠습니다.

### 1-1. 쿠버네티스 클러스터 안에서 pod에 curl명령어를 날리고 싶을때

* pod가 올라가있는데 동일한 pod레벨안에서 curl 또는 nslookup등을 날리고 싶을때 사용

잽싸게 curl 이미지를 dockerhub에서 받아 올리고 curl 명령어를 하면됨.
(사실 curl을 설치하는 것보다 빠른것 같다.)

```
kubectl run curl --image=radial/busyboxplus:curl -i --tty # 이렇게 하면 이미지 받고 바로 해당 이미지 root계정으로 접속됨.
exit # 일단 할꺼 다 하고 나왔다 해도. 다식 들어가면 됨.
kubectl get pods # pod 명 확인
kubectl attach curl -i # pod 접속
kubectl delete pod curl # 다 했으면 삭제
```

  
IntelliJ의 ```.http```에 대한 상세한 설명은 이전에 작성한 글이 있으니 참고하셔도 좋을것 같습니다.

* [IntelliJ의 .http를 사용해 Postman 대체하기](https://jojoldu.tistory.com/266)

## 5. API 문서 자동화

주문, 빌링, 정산, 회원, 쿠폰, 메인 프론트 등 여러 팀에서 포인트의 API를 사용해야 했기 때문에 API문서는 필수였습니다.  
**수동으로 작성하는 것은 언젠간 코드와 문서간에 간격이 발생**해서 문서 자동화에 대해 고민했었습니다.  
API 문서 자동화에 관한 솔루션들은 많습니다.

* [Swagger](https://swagger.io/)
* [Apidoc](http://apidocjs.com/)
* [Spring Rest Docs](https://docs.spring.io/spring-restdocs/docs/2.0.3.BUILD-SNAPSHOT/reference/html5/)

이 중에서 저희는 [Spring Rest Docs](https://spring.io/projects/spring-restdocs)를 선택했습니다.  
나머지 문서 자동화 솔루션들에는 개인적으로 생각하는 큰 단점들이 있었습니다.  

Apidoc의 경우 2가지 단점이 있었습니다.

* 문서를 사실상 수동으로 작성하는 것과 마찬가지라 문서와 코드가 따로 작동합니다 
* Apidoc을 위한 어노테이션이 프로덕션 코드를 오염시켰습니다

Swagger의 경우도 비슷한 단점이 존재했습니다.




