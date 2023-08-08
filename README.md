# EKS 환경에 WordPress 및 MySQL 구성하기
해당 프로젝트는 구름 KDT 쿠버네티스 전문가 양성 과정에서 진행한 세미 프로젝트입니다.
<br/>Stateful, Stateless 애플리케이션의 이해와 AWS EKS 클러스터 환경에서의 배포 실습을 목표로 합니다.

<br/>



## 프로젝트 목차

- [진행상황](#progress)
- [요구사항](#requirements)
    - [k8s 클러스터 구성](#k8s-cluster)
    - [Stateful WordPress 리소스 구성](#wordpress-mysql)
    - [Stateful MySQL 리소스 구성](#stateful-mysql)
    - [Stateless 애플리케이션 배포](#stateless-app)
- [Statefule/Stateless](#stateful-stateless)
- [EKS 클러스터 구성](#eks-cluster)
- [Stateful MySQL 리소스 구성](#stateful-mysql-config)
- [Stateless 애플리케이션 배포하기](#stateless-app-deploy)
- [질문사항](#questions)
- [회고](#retrospective)


<br/>

<a name="progress"></a>
## 진행상황
<aside>
💡

- 요구사항 전체 완료
- 요구사항 외 Argo CD, Argo Rollouts, GitOps 로 Blue/Green 배포하기 완료
- 카카오 멘토님들과 QnA 및 회고 진행

</aside>

<br/>


<a name="requirements"></a>
## 요구사항


<a name="k8s-cluster"></a>
### EKS 클러스터 구성

- 작업 환경은 개인별로 편한걸 사용해주시면 됩니다. (AWS, GCP 등)
    - [x]  EKS 클러스터 구성 ([참고링크](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/getting-started-console.html#eks-create-cluster))
        - [x]  노드 provisioning 2개 이상
        

<a name="wordpress-mysql"></a>
### Stateful WordPress 리소스 구성 추가

- [x]  Deployment로 배포
- [x]  Service와 Ingress(ALB)를 생성하여 wordpress app을 클러스터 외부로 노출 ([참고 문서](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html))
- [x]  HPA를 설정하여 Autoscailing이 가능하도록 정의 (Metric Server 설치)
- [x]  resources, livenessProbe를 정의

<a name="stateful-mysql"></a>
### Stateful MySQL 리소스 구성 추가

- [x]  StatefulSet으로 배포
- [x]  replicas는 2 이상으로 정의
- [x]  resources, livenessProbe를 정의
- [x]  ~~Secret을 생성하여 root 패스워드 설정~~
- [x]  ~~PVC를 이용하여 스토리지와 연결~~
- [x]  ~~Headless Service를 생성하여 mysql app과 연결~~

<a name="stateless-app"></a>
### Stateless 애플리케이션 배포하기

- stateless 애플리케이션 배포하기 (https://kubernetes.io/ko/docs/tutorials/stateless-application/)
    - [x]  디플로이먼트로 배포
        - [x]  scale up, down 해보기
        - [x]  특정 노드에 고정하여 배포하기(NodeAffinity)
    - [x]  NodePort, Port-forward로 노출하여 접근하기
    - [x]  Recreate, RollingUpdate의 동작 이해하고 BlueGreen 배포 구현해보기
    - [x]  (권장) Load Balancer 생성하기

<br/>


<a name="stateful-stateless"></a>
## stateful/stateless
    
    <aside>
    💡 state → 애플리케이션의 상태 정보
    pod의 상태정보 저장 및 필요 여부에 따라 stateful/stateless로 구분
     stateful/stateless 에 따라 배포 방식이 약간 다름
    
    pod의 이름, DNS(네트워크 식별자), 배포/업데이트/스케일링 순서, 볼륨(disk) 정보 등
    
    stateful pod : 안정적이고 작업 순서가 보장되며 고유한 상태를 유지
    
    stateless pod : 비교적 가볍고 언제든지 새롭게 pod가 재시작 되어도 문제 없음
    
    stateless → deployment
    stateful → statefulset
     - 순차적 기동, pod 이름이 시퀀셜, 특정 pod 재시작 되어도 기존의 pvc, pv 와 연결됨
    
    일반 API 서버, 애플리케이션은 deployment | DB는 statefulset
    
    </aside>


<br/>

<a name="eks-cluster"></a>
## EKS 클러스터 구성하기

- kubectl 설치
- eksctl 설치
- AWS CLI 자격증명
    - 액세스 키 발급
    - [AWS CLI 자격증명 구성](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-authentication-user.html#:~:text=%ED%8C%8C%EC%9D%BC%20%EC%A7%81%EC%A0%91%20%ED%8E%B8%EC%A7%91-,aws%20configure%20%EC%82%AC%EC%9A%A9,-%EC%9D%BC%EB%B0%98%EC%A0%81%EC%9D%B8%20%EC%9A%A9%EB%8F%84%EC%97%90%EC%84%9C%20aws)
        
        ```bash
        aws configure --profile goorm-project
        AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
        AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
        Default region name [None]: us-west-2
        Default output format [None]: json
        
        aws configure set profile goorm-project
        aws configure get profile
        goorm-project
        ```
        
    - AWS CLI 자격증명 확인
        
        ```bash
        aws sts get-caller-identity
        {
            "UserId": "AIDAS7LKAHOQUXMYEPBZK",
            "Account": "204770130849",
            "Arn": "arn:aws:iam::204770130849:user/goorm-kdt-041"
        }
        ```
        

- IAM 역할 생성
    
    ```bash
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "eks.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    
    aws iam create-role \
      --role-name myAmazonEKSClusterRole \
      --assume-role-policy-document file://"eks-cluster-role-trust-policy.json"
    
    aws iam attach-role-policy \
      --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy \
      --role-name myAmazonEKSClusterRole
    ```
    

- VPC 생성
    
    ```bash
    aws cloudformation create-stack \
      --region ap-northeast-2 \
      --stack-name my-eks-vpc-stack \
      --profile goorm-project \
      --template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
    {
        "StackId": "arn:aws:cloudformation:ap-northeast-2:204770130849:stack/my-eks-vpc-stack/90163260-04f0-11ee-9686-063ea541767a"
    }
    ```
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aca6fa2f-f3ce-4a5e-92d7-7e6911ad3a93/Untitled.png)
    

- EKS 클러스터 생성
    
    ```bash
    aws eks create-cluster --region ap-northeast-2 --name my-cluster --kubernetes-version 1.26 \
       --role-arn arn:aws:iam::204770130849:role/myAmazonEKSClusterRole \
       --resources-vpc-config subnetIds=subnet-01eb5c6f369468677,subnet-09174ca3aee78f668,subnet-0d12d5d244a194236,subnet-0e179eff96927a64b,securityGroupIds=sg-0377d9d826656b335
    ```
    
- 클러스터와 컴퓨터를 통신하도록 구성 및 확인
    
    ```bash
    aws eks update-kubeconfig --region ap-northeast-2 --name my-cluster
    
    kubectl get svc
    NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   35m
    ```
    
    - [권한 오류](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/troubleshooting.html#unauthorized:~:text=TroubleshootEKSWorkerNode%EB%A5%BC%20%EC%B0%B8%EC%A1%B0%ED%95%98%EC%84%B8%EC%9A%94.-,%EA%B6%8C%ED%95%9C%EC%9D%B4%20%EC%97%86%EA%B1%B0%EB%82%98%20%EC%95%A1%EC%84%B8%EC%8A%A4%EA%B0%80%20%EA%B1%B0%EB%B6%80%EB%90%A8(kubectl),-kubectl%20%EB%AA%85%EB%A0%B9%EC%9D%84%20%EC%8B%A4%ED%96%89%ED%95%A0)
        - 처음에 AWS console 에서 root 계정으로 EKS 구성 시 AWS CLI 자격 인증 과정에서 문제가 있었음 → console 에서는 생성자 지정을 별도로 못해서 문제>??
        - console 에서 생성한 EKS 클러스터 삭제 후 AWS CLI로 생성하니 kubectl 명령어로 EKS 클러스터와 통신이 정상적으로됨
        
        ```bash
        kubectl get node
        error: You must be logged in to the server (Unauthorized)
        ```
        
- 노드 추가
    
    ```bash
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "ec2.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    
    aws iam create-role \
      --role-name myAmazonEKSNodeRole \
      --assume-role-policy-document file://"node-role-trust-policy.json"
    
    aws iam attach-role-policy \
      --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy \
      --role-name myAmazonEKSNodeRole
    aws iam attach-role-policy \
      --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly \
      --role-name myAmazonEKSNodeRole
    aws iam attach-role-policy \
      --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
      --role-name myAmazonEKSNodeRole
    ```
    
    - AWS console 에서 EKS 클러스터에 노드그룹 추가
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7aaffc8f-fee2-460b-907e-10298fc707cb/Untitled.png)
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/65cc6685-c56c-47c8-8465-1ac7896efc53/Untitled.png)
    
    ```bash
    kubectl get node
    NAME                                                 STATUS   ROLES    AGE   VERSION
    ip-192-168-139-149.ap-northeast-2.compute.internal   Ready    <none>   16s   v1.26.4-eks-0a21954
    ip-192-168-220-162.ap-northeast-2.compute.internal   Ready    <none>   15s   v1.26.4-eks-0a21954
    ```


<br/>

<a name="stateful-mysql-config"></a>
## Stateful WordPress 및 MySQL 구성하기


- EBS 스토리지 클래스 생성
    
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: ebs-storage-class
    provisioner: kubernetes.io/aws-ebs
    ```
    
- [EBS CSI 드라이버 설치](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
    - 자격 증명 공급자 추가
        
        ```bash
        aws eks describe-cluster --name my-cluster --query "cluster.identity.oidc.issuer" --output text
        https://oidc.eks.ap-northeast-2.amazonaws.com/id/9BEA1958342BFE917B097229E12C6C29
        ```
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/44c0c8df-a299-429d-8d7c-caea51aa1eba/Untitled.png)
        
    
    - IAM Role 생성
        
        ```bash
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": {
                "Federated": "arn:aws:iam::204770130849:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/9BEA1958342BFE917B097229E12C6C29"
              },
              "Action": "sts:AssumeRoleWithWebIdentity",
              "Condition": {
                "StringEquals": {
                  "oidc.eks.ap-northeast-2.amazonaws.com/id/9BEA1958342BFE917B097229E12C6C29:aud": "sts.amazonaws.com",
                  "oidc.eks.ap-northeast-2.amazonaws.com/id/9BEA1958342BFE917B097229E12C6C29:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
                }
              }
            }
          ]
        }
        
        aws iam create-role \
          --role-name AmazonEKS_EBS_CSI_DriverRole \
          --assume-role-policy-document file://"json4.json"
        
        aws iam attach-role-policy \
          --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
          --role-name AmazonEKS_EBS_CSI_DriverRole
        ```
        
    
    - Amazon EBS CSI 추가 기능 추가
        
        ```bash
        aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver \
          --service-account-role-arn arn:aws:iam::204770130849:role/AmazonEKS_EBS_CSI_DriverRole
        
        aws eks describe-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver --query "addon.addonVersion" --output text
        ```
        
        - (선택)Amazon EBS CSI 드라이버를 Amazon EKS 추가 기능으로 업데이트
            
            ```bash
            aws eks describe-addon-versions --addon-name aws-ebs-csi-driver --kubernetes-version 1.25 \
              --query "addons[].addonVersions[].[addonVersion, compatibilities[].defaultVersion]" --output text
            
            aws eks update-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver --addon-version v1.19.0-eksbuild.2 \
              --resolve-conflicts PRESERVE
            ```
            
    
    - EBS CSI 서비스 계정에 IAM Role arn 지정
        
        ```bash
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          creationTimestamp: "2023-06-07T11:10:05Z"
          labels:
            app.kubernetes.io/component: csi-driver
            app.kubernetes.io/managed-by: EKS
            app.kubernetes.io/name: aws-ebs-csi-driver
            app.kubernetes.io/version: 1.19.0
          name: ebs-csi-controller-sa
          namespace: kube-system
          resourceVersion: "40631"
          uid: f1fa6e17-c0e0-436d-b3cc-46f0f27a937d
          annotations:
            eks.amazonaws.com/role-arn: arn:aws:iam::204770130849:role/AmazonEKS_EBS_CSI_DriverRole
        ```
        
    
- PVC 생성
    - yaml 파일 작성
        
        ```yaml
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: wp-pv-claim
          labels:
            app: wordpress
        spec:
          storageClassName: ebs-storage-class
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
        ---
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: mysql-pv-claim
          labels:
            app: wordpress
        spec:
          storageClassName: ebs-storage-class
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
        ```
        
    - PVC 생성 및 bound 확인
        
        ```bash
        kubectl apply -f pvc.yaml
        
        kubectl get pvc                                         
        NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
        mysql-pv-claim   Bound    pvc-9b0a2ba3-b043-4d02-88d2-a6f7ceb935b6   5Gi        RWO            ebs-storage-class   9m7s
        wp-pv-claim      Bound    pvc-f7198520-081e-465b-a0c0-19a4fa5f5f0c   5Gi        RWO            ebs-storage-class   9m7s
        ```
        
    - CSI 드라이버 설치 전 pvc describe
        
        ```bash
        Normal  ExternalProvisioning  1s (x6 over 47s)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "ebs.csi.aws.com" or manually created by system administrator
        ```
        
    
- 시크릿 생성
    - MySQL Docker 이미지를 사용할 때, **`MYSQL_ROOT_PASSWORD`**라는 환경 변수를 사용하여 MySQL의 **`root`** 사용자의 초기 비밀번호를 설정
    
    ```bash
    kubectl create secret generic mysql-pass --from-literal=password=1234
    ```
    
- WordPress Deployment 생성
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: wordpress
      labels:
        app: wordpress
    spec:
      selector:
        matchLabels:
          app: wordpress
          tier: frontend
      template:
        metadata:
          labels:
            app: wordpress
            tier: frontend
        spec:
          containers:
          - image: wordpress:4.8-apache
            name: wordpress
            resources:
              requests:
                memory: "64Mi"
                cpu: "250m"
              limits:
                memory: "128Mi"
                cpu: "500m"
            env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
            ports:
            - containerPort: 80
              name: wordpress
            volumeMounts:
            - name: wordpress-persistent-storage
              mountPath: /var/www/html
          volumes:
          - name: wordpress-persistent-storage
            persistentVolumeClaim:
              claimName: wp-pv-claim
    ```
    
- MySQL StatefulSet 생성
    
    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: wordpress-mysql
      labels:
        app: wordpress
    spec:
      selector:
        matchLabels:
          app: wordpress
          tier: mysql
      replicas: 2
      template:
        metadata:
          labels:
            app: wordpress
            tier: mysql
        spec:
          containers:
          - image: mysql:5.6
            name: mysql
            livenessProbe:
              exec:
                command: ["mysqladmin", "ping"]
              initialDelaySeconds: 30
              periodSeconds: 10
              timeoutSeconds: 5
            resources:
              requests:
                memory: "300Mi"
                cpu: "250m"
              limits:
                memory: "500Mi"
                cpu: "500m"
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
            ports:
            - containerPort: 3306
              name: mysql
            volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
          volumes:
          - name: mysql-persistent-storage
            persistentVolumeClaim:
              claimName: mysql-pv-claim
    ```
    
- MySQL Headless 서비스 생성
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: wordpress-mysql
      labels:
        app: wordpress
    spec:
      ports:
        - port: 3306
      selector:
        app: wordpress
        tier: mysql
      clusterIP: None
    ```
    
    - Headless 서비스 생성 전 wordpress pod log
        
        ```bash
        Warning: mysqli::mysqli(): php_network_getaddresses: getaddrinfo failed: Name or service not known in - on line 22
        MySQL Connection Error: (2002) php_network_getaddresses: getaddrinfo failed: Name or service not known
        
        Warning: mysqli::mysqli(): (HY000/2002): php_network_getaddresses: getaddrinfo failed: Name or service not known in - on line 22
        
        Warning: mysqli::mysqli(): php_network_getaddresses: getaddrinfo failed: Name or service not known in - on line 22
        
        Warning: mysqli::mysqli(): (HY000/2002): php_network_getaddresses: getaddrinfo failed: Name or service not known in - on line 22
        
        MySQL Connection Error: (2002) php_network_getaddresses: getaddrinfo failed: Name or service not known
        
        MySQL Connection Error: (2002) php_network_getaddresses: getaddrinfo failed: Name or service not known
        
        Warning: mysqli::mysqli(): php_network_getaddresses: getaddrinfo failed: Name or service not known in - on line 22
        
        Warning: mysqli::mysqli(): (HY000/2002): php_network_getaddresses: getaddrinfo failed: Name or service not known in - on line 22
        ```
        
- WordPress 서비스 및 Ingress 생성
    - [로드밸런서 서비스 YAML 작성](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)
        
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: wordpress
          labels:
            app: wordpress
        spec:
          ports:
            - port: 80
          selector:
            app: wordpress
            tier: frontend
          type: LoadBalancer
        ```
        
    
    - 로드밸런서 생성 확인
        
        ```bash
        kubectl get svc
        NAME              TYPE           CLUSTER-IP     EXTERNAL-IP                                                                   PORT(S)        AGE
        kubernetes        ClusterIP      10.100.0.1     <none>                                                                        443/TCP        16h
        wordpress         LoadBalancer   10.100.59.48   af290fda958874b19ac9e435ac4ebe43-877871056.ap-northeast-2.elb.amazonaws.com   80:32688/TCP   2m37s
        wordpress-mysql   ClusterIP      None           <none>                                                                        3306/TCP       8h
        ```
        
    
    - 로드밸런서 테스트
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dcfc4058-eca0-47b9-aa95-c3382be5bdf6/Untitled.png)
        
    
    - [Ingress 컨트롤러 설치](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) / [Ingress Nginx 컨트롤러](https://github.com/kubernetes/ingress-nginx/blob/main/README.md#readme)
        - 다른 Ingree Controller 는 IngreeClass 생성이 필요할 수 있으나 nignx 는 default IngressClass 가 있음
        
        ```bash
        brew install helm
        
        helm upgrade --install ingress-nginx ingress-nginx \
          --repo https://kubernetes.github.io/ingress-nginx \
          --namespace ingress-nginx --create-namespace
        
        kubectl get all -n ingress-nginx
        NAME                                            READY   STATUS    RESTARTS   AGE
        pod/ingress-nginx-controller-6b448794df-d86td   1/1     Running   0          72s
        
        NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                  PORT(S)                      AGE
        service/ingress-nginx-controller             LoadBalancer   10.100.93.139   a61571edf24c14f31b20a13e76b79b92-28163619.ap-northeast-2.elb.amazonaws.com   80:30118/TCP,443:30394/TCP   72s
        service/ingress-nginx-controller-admission   ClusterIP      10.100.54.133   <none>                                                                       443/TCP                      72s
        
        NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/ingress-nginx-controller   1/1     1            1           72s
        
        NAME                                                  DESIRED   CURRENT   READY   AGE
        replicaset.apps/ingress-nginx-controller-6b448794df   1         1         1       72s
        ```
        
    
    - [Ingress 생성](https://kubernetes.io/docs/concepts/services-networking/ingress/)
        
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: minimal-ingress
          annotations:
            nginx.ingress.kubernetes.io/rewrite-target: /
        spec:
          ingressClassName: nginx
          rules:
          - http:
              paths:
              - path: /wordpress
                pathType: Prefix
                backend:
                  service:
                    name: wordpress  
                    port:
                      number: 80
        ```
        
    
    - Ingress 테스트
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/170fcf5c-6ff5-4c8f-89f1-089b658aa668/Untitled.png)
        
- Metrics 서버 설치 및 WordPress HPA 적용
    - [Metrics 서버 설치](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/metrics-server.html)
        
        ```bash
        kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
        
        kubectl get deployment metrics-server -n kube-system   
        NAME             READY   UP-TO-DATE   AVAILABLE   AGE
        metrics-server   1/1     1            1           53s
        ```
        
    
    - [WordPress HPA 적용](https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
        
        ```bash
        kubectl top pod wordpress-7c6d6ddc4d-9774d
        NAME                         CPU(cores)   MEMORY(bytes)   
        wordpress-7c6d6ddc4d-9774d   1m           20Mi
        
        kubectl describe pod wordpress-7c6d6ddc4d-9774d | grep cpu   
              cpu:     500m
              cpu:     250m
        
        kubectl autoscale deployment wordpress --cpu-percent=50 --min=1 --max=10
        horizontalpodautoscaler.autoscaling/wordpress autoscaled
        
        kubectl get hpa
        NAME        REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
        wordpress   Deployment/wordpress   <unknown>/50%   1         10        0          11s
        ```
        
    
    - HPA 테스트
        - 부하 주는 pod 를 생성하여 wordpress 로 계속 요청
        - replicas 가 4개까지 올라간 후 변동이 없어서 요청 중단하고 replicas 1이 될때까지 모니터링
        
        ```bash
        kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://af290fda958874b19ac9e435ac4ebe43-877871056.ap-northeast-2.elb.amazonaws.com; done"
        
        kubectl get hpa -w
        NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
        wordpress   Deployment/wordpress   0%/50%    1         10        1          139m
        wordpress   Deployment/wordpress   22%/50%   1         10        1          140m
        wordpress   Deployment/wordpress   174%/50%   1         10        1          140m
        wordpress   Deployment/wordpress   168%/50%   1         10        4          140m
        wordpress   Deployment/wordpress   76%/50%    1         10        4          141m
        wordpress   Deployment/wordpress   20%/50%    1         10        4          141m
        wordpress   Deployment/wordpress   14%/50%    1         10        4          141m
        wordpress   Deployment/wordpress   22%/50%    1         10        4          141m
        wordpress   Deployment/wordpress   22%/50%    1         10        4          142m
        wordpress   Deployment/wordpress   10%/50%    1         10        4          142m
        wordpress   Deployment/wordpress   21%/50%    1         10        4          142m
        wordpress   Deployment/wordpress   8%/50%     1         10        4          142m
        wordpress   Deployment/wordpress   0%/50%     1         10        4          143m
        wordpress   Deployment/wordpress   0%/50%     1         10        4          146m
        wordpress   Deployment/wordpress   0%/50%     1         10        2          146m
        wordpress   Deployment/wordpress   0%/50%     1         10        2          147m
        wordpress   Deployment/wordpress   0%/50%     1         10        1          147m
        wordpress   Deployment/wordpress   1%/50%     1         10        1          178m
        wordpress   Deployment/wordpress   0%/50%     1         10        1          179m
        wordpress   Deployment/wordpress   0%/50%     1         10        1          3h1m
        wordpress   Deployment/wordpress   1%/50%     1         10        1          3h1m
        wordpress   Deployment/wordpress   0%/50%     1         10        1          3h2m
        
        kubectl get pod -w
        NAME                         READY   STATUS    RESTARTS        AGE
        wordpress-7c6d6ddc4d-9774d   1/1     Running   92 (151m ago)   10h
        wordpress-mysql-0            1/1     Running   0               147m
        wordpress-mysql-1            1/1     Running   0               146m
        load-generator               0/1     Pending   0               0s
        load-generator               0/1     Pending   0               0s
        load-generator               0/1     ContainerCreating   0               0s
        load-generator               1/1     Running             0               5s
        wordpress-7c6d6ddc4d-k47jk   0/1     Pending             0               0s
        wordpress-7c6d6ddc4d-k47jk   0/1     Pending             0               0s
        wordpress-7c6d6ddc4d-psbjc   0/1     Pending             0               0s
        wordpress-7c6d6ddc4d-cqlwk   0/1     Pending             0               0s
        wordpress-7c6d6ddc4d-k47jk   0/1     ContainerCreating   0               0s
        wordpress-7c6d6ddc4d-psbjc   0/1     Pending             0               0s
        wordpress-7c6d6ddc4d-cqlwk   0/1     Pending             0               0s
        wordpress-7c6d6ddc4d-psbjc   0/1     ContainerCreating   0               0s
        wordpress-7c6d6ddc4d-cqlwk   0/1     ContainerCreating   0               0s
        wordpress-7c6d6ddc4d-cqlwk   1/1     Running             0               2s
        wordpress-7c6d6ddc4d-k47jk   1/1     Running             0               2s
        wordpress-7c6d6ddc4d-psbjc   1/1     Running             0               2s
        load-generator               0/1     Error               0               2m31s
        load-generator               0/1     Error               0               2m33s
        load-generator               0/1     Terminating         0               2m33s
        load-generator               0/1     Terminating         0               2m33s
        wordpress-7c6d6ddc4d-psbjc   1/1     Terminating         0               5m31s
        wordpress-7c6d6ddc4d-k47jk   1/1     Terminating         0               5m31s
        wordpress-7c6d6ddc4d-k47jk   0/1     Terminating         0               5m31s
        wordpress-7c6d6ddc4d-psbjc   0/1     Terminating         0               5m31s
        wordpress-7c6d6ddc4d-psbjc   0/1     Terminating         0               5m32s
        wordpress-7c6d6ddc4d-psbjc   0/1     Terminating         0               5m32s
        wordpress-7c6d6ddc4d-k47jk   0/1     Terminating         0               5m34s
        wordpress-7c6d6ddc4d-k47jk   0/1     Terminating         0               5m34s
        wordpress-7c6d6ddc4d-cqlwk   1/1     Terminating         0               7m1s
        wordpress-7c6d6ddc4d-cqlwk   0/1     Terminating         0               7m2s
        wordpress-7c6d6ddc4d-cqlwk   0/1     Terminating         0               7m2s
        wordpress-7c6d6ddc4d-cqlwk   0/1     Terminating         0               7m2s
        
        kubectl top pod
        NAME                         CPU(cores)   MEMORY(bytes)   
        load-generator               6m           0Mi             
        wordpress-7c6d6ddc4d-9774d   33m          120Mi           
        wordpress-7c6d6ddc4d-cqlwk   72m          73Mi            
        wordpress-7c6d6ddc4d-k47jk   100m         81Mi            
        wordpress-7c6d6ddc4d-psbjc   52m          82Mi            
        wordpress-mysql-0            14m          466Mi           
        wordpress-mysql-1            14m          460Mi
        ```
        
    
<br/>

<a name="stateless-app-deploy"></a>
## Stateless 애플리케이션 배포하기


- Redis 리더 Deployment 실행
    
    ```yaml
    # SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: redis-leader
      labels:
        app: redis
        role: leader
        tier: backend
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: redis
      template:
        metadata:
          labels:
            app: redis
            role: leader
            tier: backend
        spec:
          containers:
          - name: leader
            image: "docker.io/redis:6.0.5"
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 6379
    ```
    
    ```bash
    kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-deployment.yaml
    
    kubectl get pod
    NAME                            READY   STATUS    RESTARTS         AGE
    redis-leader-5596fc7b68-2sjpf   1/1     Running   0                60s
    ```
    
- Redis 리더 서비스 생성
    
    ```yaml
    # SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-leader
      labels:
        app: redis
        role: leader
        tier: backend
    spec:
      ports:
      - port: 6379
        targetPort: 6379
      selector:
        app: redis
        role: leader
        tier: backend
    ```
    
    ```bash
    kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-service.yaml
    
    kubectl get svc
    NAME              TYPE           CLUSTER-IP     EXTERNAL-IP                                                                   PORT(S)        AGE
    kubernetes        ClusterIP      10.100.0.1     <none>                                                                        443/TCP        21h
    redis-leader      ClusterIP      10.100.33.69   <none>                                                                        6379/TCP       4s
    wordpress         LoadBalancer   10.100.59.48   af290fda958874b19ac9e435ac4ebe43-877871056.ap-northeast-2.elb.amazonaws.com   80:32688/TCP   4h59m
    wordpress-mysql   ClusterIP      None           <none>                                                                        3306/TCP       13h
    ```
    
- 2개의 Redis 팔로워를 실행
    
    ```yaml
    # SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: redis-follower
      labels:
        app: redis
        role: follower
        tier: backend
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: redis
      template:
        metadata:
          labels:
            app: redis
            role: follower
            tier: backend
        spec:
          containers:
          - name: follower
            image: gcr.io/google_samples/gb-redis-follower:v2
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 6379
    ```
    
    ```bash
    kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-deployment.yaml
    
    kubectl get pod
    NAME                              READY   STATUS    RESTARTS         AGE
    redis-follower-74d9c98c76-fbrsh   1/1     Running   0                12s
    redis-follower-74d9c98c76-vpn45   1/1     Running   0                12s
    redis-leader-5596fc7b68-2sjpf     1/1     Running   0                3m56s
    wordpress-7c6d6ddc4d-9774d        1/1     Running   92 (5h19m ago)   13h
    wordpress-mysql-0                 1/1     Running   0                5h15m
    wordpress-mysql-1                 1/1     Running   0                5h14m
    ```
    
- Redis 팔로워 서비스 생성
    
    ```yaml
    # SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-follower
      labels:
        app: redis
        role: follower
        tier: backend
    spec:
      ports:
        # the port that this service should serve on
      - port: 6379
      selector:
        app: redis
        role: follower
        tier: backend
    ```
    
    ```bash
    kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-service.yaml
    
    kubectl get svc
    NAME              TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)        AGE
    kubernetes        ClusterIP      10.100.0.1       <none>                                                                        443/TCP        21h
    redis-follower    ClusterIP      10.100.112.249   <none>                                                                        6379/TCP       4s
    redis-leader      ClusterIP      10.100.33.69     <none>                                                                        6379/TCP       3m19s
    wordpress         LoadBalancer   10.100.59.48     af290fda958874b19ac9e435ac4ebe43-877871056.ap-northeast-2.elb.amazonaws.com   80:32688/TCP   5h3m
    wordpress-mysql   ClusterIP      None             <none>                                                                        3306/TCP       13h
    ```
    
- 방명록 프론트엔드를 실행
    - [특정 노드에 배포하기](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)
        - 노드에 라벨링하거나, 노드명을 바로 지정 가능
    
    ```bash
    kubectl label nodes ip-192-168-220-162.ap-northeast-2.compute.internal affinity=test  
    node/ip-192-168-220-162.ap-northeast-2.compute.internal labeled
    
    kubectl get node -L affinity                  
    NAME                                                 STATUS   ROLES    AGE   VERSION               AFFINITY
    ip-192-168-139-149.ap-northeast-2.compute.internal   Ready    <none>   20h   v1.26.4-eks-0a21954   
    ip-192-168-220-162.ap-northeast-2.compute.internal   Ready    <none>   20h   v1.26.4-eks-0a21954   test
    ```
    
    ```yaml
    # SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend
    spec:
      replicas: 2
      selector:
        matchLabels:
            app: guestbook
            tier: frontend
      template:
        metadata:
          labels:
            app: guestbook
            tier: frontend
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: affinity
                    operator: In
                    values:
                    - test
          containers:
          - name: php-redis
            image: gcr.io/google_samples/gb-frontend:v5
            env:
            - name: GET_HOSTS_FROM
              value: "dns"
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 80
    ```
    
    ```bash
    kubectl get deployment
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    frontend         2/2     2            2           24s
    redis-follower   2/2     2            2           17m
    redis-leader     1/1     1            1           21m
    wordpress        1/1     1            1           19h
    
    kubectl get pod -o wide
    NAME                              READY   STATUS    RESTARTS         AGE     IP                NODE                                                 NOMINATED NODE   READINESS GATES
    frontend-77c548ff95-jnh8l         1/1     Running   0                48s     192.168.193.33    ip-192-168-220-162.ap-northeast-2.compute.internal   <none>           <none>
    frontend-77c548ff95-shcg7         1/1     Running   0                48s     192.168.247.215   ip-192-168-220-162.ap-northeast-2.compute.internal   <none>           <none>
    redis-follower-74d9c98c76-fbrsh   1/1     Running   0                17m     192.168.171.196   ip-192-168-139-149.ap-northeast-2.compute.internal   <none>           <none>
    redis-follower-74d9c98c76-vpn45   1/1     Running   0                17m     192.168.246.184   ip-192-168-220-162.ap-northeast-2.compute.internal   <none>           <none>
    redis-leader-5596fc7b68-2sjpf     1/1     Running   0                21m     192.168.209.243   ip-192-168-220-162.ap-northeast-2.compute.internal   <none>           <none>
    wordpress-7c6d6ddc4d-9774d        1/1     Running   92 (5h37m ago)   13h     192.168.159.179   ip-192-168-139-149.ap-northeast-2.compute.internal   <none>           <none>
    wordpress-mysql-0                 1/1     Running   0                5h33m   192.168.222.50    ip-192-168-220-162.ap-northeast-2.compute.internal   <none>           <none>
    wordpress-mysql-1                 1/1     Running   0                5h32m   192.168.176.88    ip-192-168-139-149.ap-northeast-2.compute.internal   <none>           <none>
    ```
    
- 프론트엔드 서비스를 노출하고 확인
    - private subnet에 EKS 클러스터를 구축해서, Ingress 에 NodePort 서비스를 지정해주기로..
    
    ```yaml
    # SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend
      labels:
        app: guestbook
        tier: frontend
    spec:
      type: NodePort
      ports:
        # the port that this service should serve on
      - port: 80
        nodePort: 30007
      selector:
        app: guestbook
        tier: frontend
    ```
    
    ```bash
    kubectl get svc frontend
    NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
    frontend   NodePort   10.100.122.35   <none>        80:30007/TCP   49s
    
    kubectl port-forward svc/frontend 8080:80
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    
    kubectl get node -o wide
    NAME                                                 STATUS   ROLES    AGE   VERSION               INTERNAL-IP       EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
    ip-192-168-139-149.ap-northeast-2.compute.internal   Ready    <none>   20h   v1.26.4-eks-0a21954   192.168.139.149   <none>        Amazon Linux 2   5.10.179-166.674.amzn2.x86_64   containerd://1.6.19
    ip-192-168-220-162.ap-northeast-2.compute.internal   Ready    <none>   20h   v1.26.4-eks-0a21954   192.168.220.162   <none>        Amazon Linux 2   5.10.179-166.674.amzn2.x86_64   containerd://1.6.19
    ```
    
    - Ingress rule 수정
    
    ```bash
    kubectl edit ingress minimal-ingress
    
    - http:
          paths:
          - backend:
              service:
                name: frontend
                port:
                  number: 80
            path: /
            pathType: Prefix
    ```
    
    - 테스트
        - 80 (O)
        - 작동은 되나 NodePort에 대한 실습 의도대로 이뤄진 건 아닌 것 같음;;
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/afe3d859-0128-4427-a637-0517d3107525/Untitled.png)
    
- 방명록 프론트엔드 scale up, down
    
    ```bash
    kubectl get pod         
    NAME                              READY   STATUS    RESTARTS         AGE
    frontend-77c548ff95-jnh8l         1/1     Running   0                40m
    frontend-77c548ff95-shcg7         1/1     Running   0                40m
    
    kubectl get deployment
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    frontend         2/2     2            2           40m
    
    kubectl scale deployment frontend --replicas=5
    deployment.apps/frontend scaled
    
    kubectl get pod                               
    NAME                              READY   STATUS    RESTARTS         AGE
    frontend-77c548ff95-52pmb         1/1     Running   0                64s
    frontend-77c548ff95-gjgfv         1/1     Running   0                64s
    frontend-77c548ff95-jnh8l         1/1     Running   0                41m
    frontend-77c548ff95-shcg7         1/1     Running   0                41m
    frontend-77c548ff95-vl575         1/1     Running   0                64s
    
    kubectl get deployment                        
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    frontend         5/5     5            5           41m
    redis-follower   2/2     2            2           58m
    redis-leader     1/1     1            1           62m
    wordpress        1/1     1            1           20h
    
    kubectl scale deployment frontend --replicas=3
    deployment.apps/frontend scaled
    
    kubectl get pod                               
    NAME                              READY   STATUS    RESTARTS         AGE
    frontend-77c548ff95-52pmb         1/1     Running   0                111s
    frontend-77c548ff95-jnh8l         1/1     Running   0                42m
    frontend-77c548ff95-shcg7         1/1     Running   0                42m
    
    kubectl get deployment                        
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    frontend         3/3     3            3           42m
    ```
    
- [Recreate, RollingUpdate 이해](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)하고 Blue/Green 배포해보기
    - **Recreate**
        - 새 포드가 생성되기 전에 모든 기존 포드가 종료됩니다 `.spec.strategy.type==Recreate`
        - 다운타임 발생
    - **롤링 업데이트 배포**
        - 배포는 롤링 업데이트 방식으로 포드를 업데이트합다 `.spec.strategy.type==RollingUpdate`. `maxSurge`, `maxUnavailable` 롤링 업데이트 프로세스를 지정 하고 제어할 수 있음
        - ex) n개의 신버전 배포 완료되면, 구버전 n개가 삭제
        - 구버전과 신버전이 동시에 서비스 되는 구간이 존재
    - Blue Green 배포
        - 블루와 그린을 모두 배포하여 그린에 대한 안정성이 검증되면 그린으로 트래픽을 모두 전환, 미검증 시 블루로 롤백하는 전략
    - Blue, Green Deployment 생성
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend
    spec:
      replicas: 2
      selector:
        matchLabels:
            app: guestbook
            tier: frontend
            version: v4
      template:
        metadata:
          labels:
            app: guestbook
            tier: frontend
            version: v4
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: affinity
                    operator: In
                    values:
                    - test
          containers:
          - name: php-redis
            image: gcr.io/google_samples/gb-frontend:v4
            env:
            - name: GET_HOSTS_FROM
              value: "dns"
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 80
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend2
    spec:
      replicas: 2
      selector:
        matchLabels:
            app: guestbook
            tier: frontend
            version: v5
      template:
        metadata:
          labels:
            app: guestbook
            tier: frontend
            version: v5
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: affinity
                    operator: DoesNotExist
          containers:
          - name: php-redis
            image: gcr.io/google_samples/gb-frontend:v5
            env:
            - name: GET_HOSTS_FROM
              value: "dns"
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 80
    ```
    
    ```bash
    kubectl get deployment -o wide
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                                       SELECTOR
    frontend         2/2     2            2           12s    php-redis    gcr.io/google_samples/gb-frontend:v4         app=guestbook,tier=frontend,version=v4
    frontend2        2/2     2            2           12s    php-redis    gcr.io/google_samples/gb-frontend:v5         app=guestbook,tier=frontend,version=v5
    ```
    
    - Blue 버전을 Select 하는 로드밸런서 서비스 생성
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend-load-balancer
      labels:
        app: guestbook
        tier: frontend
    spec:
      ports:
        - port: 80
      selector:
        app: guestbook
        tier: frontend
        version: v4
      type: LoadBalancer
    ```
    
    ```bash
    kubectl get frontend-load-balancer
    NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)        AGE
    frontend-load-balancer   LoadBalancer   10.100.193.112   a702694a6dfb1471f989f3a3269cfb42-1845485982.ap-northeast-2.elb.amazonaws.com   80:30385/TCP   81s
    ```
    
    - Green 버전을 Select 하도록 서비스 수정
    
    ```yaml
    version: v4 
    -> 
    version: v5
    
    kubectl get svc frontend-load-balancer -o wide 
    NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)        AGE   SELECTOR
    frontend-load-balancer   LoadBalancer   10.100.193.112   a702694a6dfb1471f989f3a3269cfb42-1845485982.ap-northeast-2.elb.amazonaws.com   80:30385/TCP   16m   app=guestbook,tier=frontend,version=v5
    ```
    
    - Blue 버전 삭제
    
    ```bash
    kubectl delete deployment frontend
    deployment.apps "frontend" deleted
    ```
    
- Argo CD, Argo Rollouts, GitOps 로 Blue/Green 배포하기
    - [Argo CD 설치](https://argo-cd.readthedocs.io/en/stable/#quick-start:~:text=%C2%B6-,Quick%20Start,-%C2%B6)
        
        ```bash
        kubectl create namespace argocd
        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
        
        kubectl get all -n argocd
        NAME                                                    READY   STATUS    RESTARTS   AGE
        pod/argocd-application-controller-0                     1/1     Running   0          33s
        pod/argocd-applicationset-controller-5c64658ff9-d5p6n   1/1     Running   0          33s
        pod/argocd-dex-server-78659867d9-jn82j                  1/1     Running   0          33s
        pod/argocd-notifications-controller-749b96fd4d-xwwqd    1/1     Running   0          33s
        pod/argocd-redis-74cb89f466-wgg79                       1/1     Running   0          33s
        pod/argocd-repo-server-69746cbd47-zjtlk                 1/1     Running   0          33s
        pod/argocd-server-6c87596fcf-55v4z                      1/1     Running   0          33s
        
        NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
        service/argocd-applicationset-controller          ClusterIP   10.100.114.37    <none>        7000/TCP,8080/TCP            34s
        service/argocd-dex-server                         ClusterIP   10.100.49.75     <none>        5556/TCP,5557/TCP,5558/TCP   34s
        service/argocd-metrics                            ClusterIP   10.100.44.30     <none>        8082/TCP                     34s
        service/argocd-notifications-controller-metrics   ClusterIP   10.100.132.209   <none>        9001/TCP                     34s
        service/argocd-redis                              ClusterIP   10.100.196.194   <none>        6379/TCP                     33s
        service/argocd-repo-server                        ClusterIP   10.100.254.175   <none>        8081/TCP,8084/TCP            33s
        service/argocd-server                             ClusterIP   10.100.26.167    <none>        80/TCP,443/TCP               33s
        service/argocd-server-metrics                     ClusterIP   10.100.69.178    <none>        8083/TCP                     33s
        
        NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/argocd-applicationset-controller   1/1     1            1           33s
        deployment.apps/argocd-dex-server                  1/1     1            1           33s
        deployment.apps/argocd-notifications-controller    1/1     1            1           33s
        deployment.apps/argocd-redis                       1/1     1            1           33s
        deployment.apps/argocd-repo-server                 1/1     1            1           33s
        deployment.apps/argocd-server                      1/1     1            1           33s
        
        NAME                                                          DESIRED   CURRENT   READY   AGE
        replicaset.apps/argocd-applicationset-controller-5c64658ff9   1         1         1       33s
        replicaset.apps/argocd-dex-server-78659867d9                  1         1         1       33s
        replicaset.apps/argocd-notifications-controller-749b96fd4d    1         1         1       33s
        replicaset.apps/argocd-redis-74cb89f466                       1         1         1       33s
        replicaset.apps/argocd-repo-server-69746cbd47                 1         1         1       33s
        replicaset.apps/argocd-server-6c87596fcf                      1         1         1       33s
        
        NAME                                             READY   AGE
        statefulset.apps/argocd-application-controller   1/1     33s
        ```
        
    
    - Argo CD 서비스 로드밸런서로 변경
        
        ```bash
        kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
        service/argocd-server patched
        
        kubectl get svc argocd-server -n argocd 
        NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)                      AGE
        argocd-server   LoadBalancer   10.100.26.167   a991359cceb3045748301d9b2d74564e-1969146517.ap-northeast-2.elb.amazonaws.com   80:31074/TCP,443:30484/TCP   28m
        ```
        
    - Argo CD 로그인
        
        ```bash
        kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
        I3h6-wjKHJeeCMa3
        ```
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a65c0bdd-1ae3-4ae6-851c-cfd8c09bf239/Untitled.png)
        
    
    - [GitOps 를 위한 gitea 설치](https://docs.gitea.com/next/installation/install-on-kubernetes#:~:text=Installation%20with%20Helm%20(on%20Kubernetes))
        - [helm gitea 설정](https://gitea.com/gitea/helm-chart/)
        
        ```bash
        helm repo add gitea-charts https://dl.gitea.io/charts/
        "gitea-charts" has been added to your repositories
        helm install gitea gitea-charts/gitea
        coalesce.go:175: warning: skipped value for memcached.initContainers: Not a table.
        NAME: gitea
        LAST DEPLOYED: Thu Jun  8 23:48:43 2023
        NAMESPACE: default
        STATUS: deployed
        REVISION: 1
        NOTES:
        1. Get the application URL by running these commands:
          echo "Visit http://127.0.0.1:3000 to use your application"
          kubectl --namespace default port-forward svc/gitea-http 3000:3000
        
        kubectl create namespace gitea
        namespace/gitea created
        
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: gitea-pvc
          namespace: gitea
        spec:
          storageClassName: ebs-storage-class
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
        ---
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: gitea-postgres-pvc
          namespace: gitea
        spec:
          storageClassName: ebs-storage-class
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
        
        kubectl get pvc -n gitea
        NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
        gitea-postgres-pvc   Bound    pvc-0b931324-8733-4f69-8cc5-c1eb3bbbc7c5   5Gi        RWO            ebs-storage-class   11s
        gitea-pvc            Bound    pvc-a18b20ab-27bb-4fce-81c5-7b3766c0d515   5Gi        RWO            ebs-storage-class   11s
        
        helm show values gitea-charts/gitea -n gitea > values.yaml
        
        ~~helm install gitea \
        --set adminUsername=gitea \
        --set giteaPassword=gitea \
        --set persistence.existingClaim=gitea-pvc \
        --set persistence.existingPostgresClaim=gitea-postgres-pvc \
        --set service.type=NodePort \
        --set service.nodePorts.http=30001 \
        oci://registry-1.docker.io/bitnamicharts/gitea -n gitea
        gitea-charts/gitea -n gitea~~
        
        helm install gitea gitea-charts/gitea -n gitea -f values.yaml -n gitea
        
        kubectl get all -n gitea
        NAME                                   READY   STATUS    RESTARTS   AGE
        pod/gitea-0                            1/1     Running   0          42s
        pod/gitea-memcached-8666cf9db5-g8dhd   1/1     Running   0          42s
        pod/gitea-postgresql-0                 1/1     Running   0          42s
        
        NAME                          TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)          AGE
        service/gitea-http            LoadBalancer   10.100.199.207   aba2e8ad2b259487a942c58ae42187b7-965532667.ap-northeast-2.elb.amazonaws.com   3000:31329/TCP   42s
        service/gitea-memcached       ClusterIP      10.100.150.236   <none>                                                                        11211/TCP        42s
        service/gitea-postgresql      ClusterIP      10.100.36.140    <none>                                                                        5432/TCP         42s
        service/gitea-postgresql-hl   ClusterIP      None             <none>                                                                        5432/TCP         42s
        service/gitea-ssh             ClusterIP      None             <none>                                                                        22/TCP           42s
        
        NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/gitea-memcached   1/1     1            1           42s
        
        NAME                                         DESIRED   CURRENT   READY   AGE
        replicaset.apps/gitea-memcached-8666cf9db5   1         1         1       42s
        
        NAME                                READY   AGE
        statefulset.apps/gitea              1/1     42s
        statefulset.apps/gitea-postgresql   1/1     42s
        
        # gitea-http 서비스를 80으로 진입 시 3000으로 포트포워딩 되도록 설정
        kubectl get svc gitea-http -n gitea
        NAME         TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)        AGE
        gitea-http   LoadBalancer   10.100.199.207   aba2e8ad2b259487a942c58ae42187b7-965532667.ap-northeast-2.elb.amazonaws.com   80:31329/TCP   7m5s
        ```
        
        - 접속 확인
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/74a01ce7-63ab-4772-8336-b771b32cb9bc/Untitled.png)
        
        - GitOps를 위한 레포지토리 생성 및 매니페스트 작성
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d1f20211-a991-4de8-9fe4-6925c3b581db/Untitled.png)
        
        - Argo CD 배포 확인
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/80ed4771-12d3-4ec5-ba92-e046a6b00d34/Untitled.png)
        
        ```bash
        # yaml version v4 -> version v5 변경
        
        # before
        kubectl get deployment -o wide
        NAME             READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                       SELECTOR
        frontend         1/1     1            1           20s   php-redis    gcr.io/google_samples/gb-frontend:v4         app=guestbook,tier=frontend
        
        # after
        kubectl get deployment -o wide
        NAME             READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                                       SELECTOR
        frontend         1/1     1            1           2m13s   php-redis    gcr.io/google_samples/gb-frontend:v5         app=guestbook,tier=frontend
        ```
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7a8aee0a-590d-4989-871e-355f5fa9a885/Untitled.png)
        
    
    - [Argo Rollouts 설치](https://argoproj.github.io/argo-rollouts/#:~:text=Relic%2C%20Graphite%2C%20InfluxDB-,%EB%B9%A0%EB%A5%B8%20%EC%8B%9C%EC%9E%91,-%C2%B6)
        
        ```bash
        kubectl create namespace argo-rollouts
        kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
        
        kubectl get all -n argo-rollouts      
        NAME                                READY   STATUS    RESTARTS   AGE
        pod/argo-rollouts-998d8d858-hzn48   1/1     Running   0          17s
        
        NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
        service/argo-rollouts-metrics   ClusterIP   10.100.136.29   <none>        8090/TCP   17s
        
        NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/argo-rollouts   1/1     1            1           17s
        
        NAME                                      DESIRED   CURRENT   READY   AGE
        replicaset.apps/argo-rollouts-998d8d858   1         1         1       17s
        ```
        
    
    - 플러그인 설치
        
        ```bash
        curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-amd64
        chmod +x ./kubectl-argo-rollouts-darwin-amd64
        sudo mv ./kubectl-argo-rollouts-darwin-amd64 /usr/local/bin/kubectl-argo-rollouts
         
        kubectl argo rollouts version
        ```
        
    
    - [Argo Rollouts 이용해서 Blue/Green 배포해보기](https://gomgomshrimp.oopy.io/posts/2)
        
        ```bash
        apiVersion: argoproj.io/v1alpha1
        kind: Rollout
        metadata:
          name: rollout-bluegreen
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: rollout-bluegreen
          template:
            metadata:
              labels:
                app: rollout-bluegreen
            spec:
              containers:
              - name: php-redis
                image: gcr.io/google_samples/gb-frontend:v5
        
                imagePullPolicy: Always
                ports:
                - containerPort: 80
          strategy:
            blueGreen:
              activeService: rollout-bluegreen-active
              previewService: rollout-bluegreen-preview
              autoPromotionEnabled: false
         
        ---
        kind: Service
        apiVersion: v1
        metadata:
          name: rollout-bluegreen-active
        spec:
          type: NodePort
          selector:
            app: rollout-bluegreen
          ports:
          - protocol: TCP
            port: 80
            targetPort: 80
            nodePort: 30081
         
        ---
        kind: Service
        apiVersion: v1
        metadata:
          name: rollout-bluegreen-preview
        spec:
          type: NodePort
          selector:
            app: rollout-bluegreen
          ports:
          - protocol: TCP
            port: 80
            targetPort: 80
            nodePort: 30082
        ```
        
        ```bash
        Name:            rollout-bluegreen
        Namespace:       default
        Status:          ✔ Healthy
        Strategy:        BlueGreen
        Images:          gcr.io/google_samples/gb-frontend:v4 (stable, active)
        Replicas:
          Desired:       1
          Current:       1
          Updated:       1
          Ready:         1
          Available:     1
        
        NAME                                          KIND        STATUS     AGE    INFO
        ⟳ rollout-bluegreen                           Rollout     ✔ Healthy  6m27s  
        └──# revision:1                                                             
           └──⧉ rollout-bluegreen-dd78f6dcd           ReplicaSet  ✔ Healthy  6m27s  stable,active
              └──□ rollout-bluegreen-dd78f6dcd-9t4ld  Pod         ✔ Running  6m27s  ready:1/1
        
        kubectl get replicaset -o wide
        NAME                          DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                                       SELECTOR
        rollout-bluegreen-dd78f6dcd   1         1         1       7m51s   php-redis    gcr.io/google_samples/gb-frontend:v4         app=rollout-bluegreen,rollouts-pod-template-hash=dd78f6dcd
        
        # version 4->5 변경
        
        Name:            rollout-bluegreen
        Namespace:       default
        Status:          ॥ Paused
        Message:         BlueGreenPause
        Strategy:        BlueGreen
        Images:          gcr.io/google_samples/gb-frontend:v4 (stable, active)
                         gcr.io/google_samples/gb-frontend:v5 (preview)
        Replicas:
          Desired:       1
          Current:       2
          Updated:       1
          Ready:         1
          Available:     1
        
        NAME                                           KIND        STATUS     AGE  INFO
        ⟳ rollout-bluegreen                            Rollout     ॥ Paused   12m  
        ├──# revision:2                                                            
        │  └──⧉ rollout-bluegreen-666cc68b6c           ReplicaSet  ✔ Healthy  59s  preview
        │     └──□ rollout-bluegreen-666cc68b6c-j5dst  Pod         ✔ Running  59s  ready:1/1
        └──# revision:1                                                            
           └──⧉ rollout-bluegreen-dd78f6dcd            ReplicaSet  ✔ Healthy  12m  stable,active
              └──□ rollout-bluegreen-dd78f6dcd-9t4ld   Pod         ✔ Running  12m  ready:1/1
        ```
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f5b7ba20-eb7e-4e3c-ac16-cdaaa3081734/Untitled.png)
        
    
- Istio Ingress Gateway 로 크로스 네임스페이스 라우팅해보기 (Ingress Controller 교체)
    
    → 파이널 프로젝트에서 진행
    
- 프로메테우스, 그라파나 구성
    
    → 파이널 프로젝트에서 진행
    
- fluntd, ElasticSearch, kibana 로깅  구성
    
    → 파이널 프로젝트에서 진행


<br/>

<a name="questions"></a>
## 질문사항


- [x]  Step2,3 에서 WordPress 와 MySQL(StatefulSet replica=2) 연동 시MySQL 파드에서 종종 에러 발생하며 죽는 현상이 있었음
    - ~~`InnoDB: Check that you do not already have another mysqld process using the same InnoDB data or log files`~~
    - ~~두 개의 파드에서 동일한 PVC를 사용해서 그런 것 같아 volumeClaimTemplates 를 이용하여 각 파드에 다른 PVC 가 매핑되도록 처리했음~~
    - ~~이 경우 데이터의 일관성이 어떻게 보장되는지와 가용성을 위해서는 어떻게 구성해야하는지 궁금합니다.~~
- [x]  실무에서는 어떤 배포 전략을 선호하고, 어떻게 진행하시는지 궁금합니다!
- [x]  하나의 클러스터에 여러 모듈을 설치하다 보니까 로드밸런서를 너무 많이 생성한 것 같습니다. 각각 다른 진입점을 갖고 있어서 불편하네요.. 이 방식이 잘못된 느낌은 오지만 어떻게 해야할지 모르겠습니다..!! (ArgoCD, gitea, 기타 애플리케이션 등을 외부로 노출하기 위해)
    - 현재는 하나의 Ingress 에서 동일 namespace의 서비스로는 라우팅이 가능한 상태 ( nginx-ingress-controller)
    - Private Subnet 으로 구성된 클러스터로, NodePort 로 불가한 상태
    - [x]  효율적으로 Ingress 하나로 다른 네임스페이스의 ClusterIP 서비스를 라우팅 하는 방법이 궁금합니다. 아니면 실무에서 이런 경우 어떤식으로 구성하시는지 궁금합니다.

<br/>

<a name="retrospective"></a>
## 회고


- 전에는 hostpath 로만 volume mount 해봤는데, EBS CSI 드라이버 설치하여 스토리지 클래스로 PV 동적 프로비저닝까지 해봐서 의미가 있음
- Recreate, RollingUpdate 배포 전략에 대해 이해하고, Blue/Green 배포 전략에 대해 한 번 더 리마인드하고 실습까지 해볼 수 있어서 좋았음
- 실습하며 kubectl 명령어를 이것 저것 시도해볼 수 있어서 좋았음
- 조금 더 심화하지 못해 아쉽지만, 세미 프로젝트 과정을 거치게 됨으로써 다음 파이널 프로젝트에서 한 단계 심화된 결과물을 만들어낼 수 있을 것 같아서 개인적으로  만족스러운 세미 프로젝트였음
