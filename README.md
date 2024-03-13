---
### 🔗 프로젝트 개요
![](https://velog.velcdn.com/images/yujeong-shin/post/a878c70d-5331-4168-9d29-a793e8615d89/image.png)


<br/>

---
### 🎨 배포 아키텍처 🖌️
![](https://velog.velcdn.com/images/yujeong-shin/post/3ebaab26-7e5b-4f0b-a818-2c172bd305e9/image.png)

* Front-End
1. 해당 Repo로 checkout
2. vue run/build를 위한 npm 설치
3. AWS CLI 자격 증명
4. 프론트 dist 폴더 S3 bucket에 업로드

* Back-End
1. 해당 Repo로 checkout
2. EKS 클러스터와 상호 작용하기 위한 kubectl 구성 파일 생성
3. docker로 생성한 백엔드 이미지 빌드 후 ECR에 올림
4. pod 생성 시 백엔드 이미지 배포

<br/>

#### 🤔 아키텍처 선정 이유 ❓

||docker compose|kubernetes|
|:---:|:---:|:---:|
|아키텍처|단일 호스트|다중 호스트|
|확장성| Scale-Up|Scale-Out|
|로드 밸런싱|직접 설정|자동 설정|
|HA|장애 발생 시 서비스 다운|장애 감지 및 자동 복구
|설정 편의성|서비스의 IP 주소나 도메인을 직접 지정|내부 서비스 디스커버리 자동 설정|

<br/>

* docker compose <br/>
단순한 개발 환경 또는 작은 규모의 애플리케이션에 적합

* kubernetes ✅<br/>
대규모 및 고가용성 애플리케이션 및 확장성이 요구되는 복잡한 시나리오에 적합


<br/>

---
### 📝 구성 스크립트
* Front-End
<details>
<summary> well-com-front pipeline </summary>

  ```
  pipeline{
    // 어떠한 agent로도 실행해도 된다는 의미
    agent any
    tools{
        // nodejs 플러그인 설치 및 nodejs20 에 대한 버전 설정
        nodejs 'nodejs20'
    }
    environment{
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_REGION = 'ap-northeast-2'
    }
    stages{
        stage('Checkout'){
            steps{
                git url: 'https://github.com/SonJM/Well-Com-Front.git',
                credentialsId: 'github_access_token',
                branch: 'main'
            }
        }
        stage('npm install'){
            steps{
                sh 'npm install'    
            }
        }
        stage('npm run build'){
            steps{
                sh 'npm run build'    
            }
        }
        stage('aws cli configure'){
            steps{
                sh '''
                aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                aws configure set default.region $AWS_REGION
                '''
            }
        }
        stage("deploy to S3 buckets") {
            steps{
                sh 'aws s3 cp ./dist s3://well-com-front/ --recursive'
            }
        }
    }
} 
  ```
</details>  


* Back-End
<details>
<summary> backend-depl-serve.yml </summary>

  ```
  apiVersion: apps/v1
kind: Deployment
metadata:
  name: wellcom-backend-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wellcom-backend
  template:
    metadata:
      labels:
        app: wellcom-backend
    spec:
      containers:
      - name: wellcom
        image: 346903264902.dkr.ecr.ap-northeast-2.amazonaws.com/wellcom:latest
        ports:
        - containerPort: 8080
        resources:
        # 컨테이너가 사용할 수 있는 리소스의 최대치
          limits:
            cpu: "1"
            memory: "500Mi"
        # 컨테이너가 시작될 때 보장받아야 하는 최소 자원
          requests:
            cpu: "0.5"
            memory: "250Mi"
        env:
        # RDS 시크릿
          - name: DB_HOST
            valueFrom:
              secretKeyRef:
                name: db-infos
                key: DB_HOST
          - name: DB_USERNAME
            valueFrom:
              secretKeyRef:
                name: db-infos
                key: DB_USERNAME
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: db-infos
                key: DB_PASSWORD
          # JWT 시크릿
          - name: JWT_SECRET_KEY
            valueFrom:
              secretKeyRef:
                name: jwt-infos
                key: JWT_SECRET_KEY
          - name: JWT_ACCESS_EXPIRE
            valueFrom:
              secretKeyRef:
                name: jwt-infos
                key: JWT_ACCESS_EXPIRE
          - name: JWT_REFRESH_EXPIRE
            valueFrom:
              secretKeyRef:
                name: jwt-infos
                key: JWT_REFRESH_EXPIRE
          # EMAIL 시크릿
          - name: GOOGLE_EMAIL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mail-infos
                key: GOOGLE_EMAIL_PASSWORD
          # OAUTH 시크릿
          - name: OAUTH_CLIENT_ID
            valueFrom:
              secretKeyRef:
                name: oauth-infos
                key: OAUTH_CLIENT_ID
          - name: OAUTH_CLIENT_SECRET
            valueFrom:
              secretKeyRef:
                name: oauth-infos
                key: OAUTH_CLIENT_SECRET
          - name: OAUTH_REDIRECT_URL
            valueFrom:
              secretKeyRef:
                name: oauth-infos
                key: OAUTH_REDIRECT_URL
          # S3 IMAGE 시크릿
          - name: S3_IMAGE_BUCKET
            valueFrom:
              secretKeyRef:
                name: s3-image-infos
                key: S3_IMAGE_BUCKET
          - name: S3_IMAGE_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: s3-image-infos
                key: S3_IMAGE_ACCESS_KEY
          - name: S3_IMAGE_SECRET_KEY
            valueFrom:
              secretKeyRef:
                name: s3-image-infos
                key: S3_IMAGE_SECRET_KEY
  
  apiVersion: v1
kind: Service
metadata:
  name: wellcom-backend-service
spec:
  # ClusterIP는 클러스터 내부에서만 접근가능한 Service를 생성
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 8080
  selector:
    app: wellcom-backend
  ```
</details>
<details>
<summary> ingress.yml </summary>

  ```
    # ingress-controller 설치 명령어-
  # kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/aws/deploy.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: well-com-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    # nginx.ingress.kubernetes.io/rewrite-target: /$1 # 첫번째 prefix제거
    cert-manager.io/cluster-issuer: letsencrypt-prod
  
 spec:
  # ingressClassName: nginx
  tls:
  - hosts:
    - "server.blisle.shop"
    secretName: well-com-tls
  rules:
    - host: server.blisle.shop
      http:
        paths:
          - path: / # 모든 URL요청을 wellcom-backend-service로 라우팅
            pathType: Prefix
            backend:
              service:
                name: wellcom-backend-service
                port:
                  number: 80
  ```  
</details>
<details>
<summary> ingress_cert.yml </summary>

  ```
 # https 인증서 적용 절차
 # cert-manager 생성
 # cert-manager 생성을 위한 cert-manager namespace 생성
 # 1-1. kubectl create namespace cert-manager
 # 1-2. Helm 설치
 # 1-3. cert-manager를 설치하기 위한 Jetstack Helm repository 추가
 # 명령어 : helm repo add jetstack https://charts.jetstack.io
 # 1-4. Helm repository 업데이트
 # 명령어 : helm repo update
 # 1-5. cert-manager 차트 설치
 # 명령어 : helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.5.0 --create-namespace --set installCRDs=true

 # 2.ClusterIssuer 생성
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
  # 인증서 서버 주소. 해당 서버의 리소스를 통해 인증서 발행
    server: https://acme-v02.api.letsencrypt.org/directory
    # 인증서 만료 또는 갱신 필요시 알람 email
    email: ksg3941234@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers: 
    - http01:
        ingress:
          class: nginx
 ---
 # 3. ClusterIssue를 사용하여 Certificate 리소스 생성 : Certificate 리소스 생성시에 인증서 발급
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: well-com-tls
  namespace: default
  annotations:
  kubectl.kubernetes.io/last-applied-configuration: ""
spec:
  secretName: well-com-tls
  duration: 2160h # 90-day
  renewBefore: 360h # 15-day
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: server.blisle.shop
  dnsNames:
  - server.blisle.shop
  ```
</details>
<details>
<summary> backend_deploy.yml </summary>

  ```
  name: deploy order order-backend

 on:
  push:
    branches:
      - main

 jobs:
  build-and-deploy:
      runs-on: ubuntu-latest
      steps:
          - name: checkout github
            uses: actions/checkout@v2

          - name: install kubectl
            uses: azure/setup-kubectl@v3
            with:
              version: "v1.25.9"
            id: install

          - name: configure aws
            uses: aws-actions/configure-aws-credentials@v1
            with:
              aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
              aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws-region: ap-northeast-2

          - name: update cluster infomation
            run: aws eks update-kubeconfig --name 1team-cluster --region ap-northeast-2

          - name: Login to ECR
            id: login-ecr
            uses: aws-actions/amazon-ecr-login@v1

          - name: build and push docker image to ecr
            env:
              REGISTRY: 346903264902.dkr.ecr.ap-northeast-2.amazonaws.com
              REPOSITORY: wellcom
              IMAGE_TAG: latest
            run: |
              docker build \
              -t $REGISTRY/$REPOSITORY:$IMAGE_TAG \
              -f ./Dockerfile .
              docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG

          - name: eks kubectl apply
            run: |
              kubectl apply -f ./k8s/backend-depl-serve.yml
              kubectl rollout restart deployment wellcom-backend-depl
  ```
</details>
  
<br/>

### ⭐ 배포 포인트

1️⃣ Jenkins Webhook 설정 시 ngrok 사용
<details>
<summary> ngrok 설정 방법 </summary>
  jenkins 서버를 local 환경에서 띄우다 보니 Github Webhook에 Public한 URL 지정 불가

➔ local 서버를 외부에서 접속할 수 있게 해주는 터널링 서비스 ngrok 사용하여, local에 띄운 jenkins 서버를 외부에서 들어올 수 있도록 설정
  
➔ 1분마다 polling 방식이 아닌, 코드 변동사항 "즉시" 반영 !
  
  
* ngrok 설정
![](https://velog.velcdn.com/images/yujeong-shin/post/e5f5e643-8964-4937-9bc6-8fcb69bab668/image.png)

  
* Jenkins에 Webhook 설정
  ![](https://velog.velcdn.com/images/yujeong-shin/post/1496311a-34fc-4b21-aa82-b2cb12aaf145/image.png)
  ![](https://velog.velcdn.com/images/yujeong-shin/post/e36e2b08-b859-4d7f-8e21-51b5d9082674/image.png)


* GitHub에 Webhook 설정
  ![](https://velog.velcdn.com/images/yujeong-shin/post/9454f9bd-6b21-4e46-8d75-fed6fe47b7d7/image.png)

  
* Github에서 Jenkins에게 push 유무 알리기 위해 둘을 연결
  ![](https://velog.velcdn.com/images/yujeong-shin/post/cc59dd56-f5d0-459f-9a2c-1fd92d565342/image.png)

  ![](https://velog.velcdn.com/images/yujeong-shin/post/6179d4a5-d82c-4889-982d-8ae35aef2442/image.png)

  
  
</details>


2️⃣ kubernetes 내장 secret 활용
<details>
<summary> kubernetes secret </summary>
  yml 파일에서 사용되는 환경 변수들을 모두 쿠버네티스 내장 secret에 저장

➔ git action에는 AWS 연결에 필요한 secret만 등록 & AWS 의존성 감소
  

![](https://velog.velcdn.com/images/yujeong-shin/post/4868730d-2bfc-4598-a07b-c2ba6c4c8bc9/image.png)


![](https://velog.velcdn.com/images/yujeong-shin/post/f071e8bb-7942-419a-9fcc-75d086cd2bdd/image.png)
</details>

</br>

---
### 💡 발생한 문제 및 해결 방법

* CORS 에러
<details>
<summary> CORS problem </summary>
  ![](https://velog.velcdn.com/images/yujeong-shin/post/3c62ef37-a495-4aa4-bfb6-13bad3f39364/image.png)
  <br/>
  Front URL에 대해 입장 허용

</details>

* 첫 접속은 무조건 localhost ?!
<details>
<summary> First request problem </summary>

  ```
 서버 배포 후, 클라이언트 접속 시 첫 접속은 서버의 localhost로 요청을 보내는 오류 발생
  새로고침 해주어야만 요청이 올바르게 처리됨.
  ```
  ingress에서 /로 들어오는 모든 URL요청을 라우팅하면서 생긴 문제.
  
  CloudFront의 기본값 루트 객체를 index.html ➔ www.blisle.shop으로 변경
  기본값 루트 객체는 URL 접속 시 이동해야 하는 HTML 파일명
  ![](https://velog.velcdn.com/images/yujeong-shin/post/55d802c7-be22-4421-bef0-c3d935e99d70/image.png)

</details>

* DB 한글 인코딩 문제
<details>
<summary> DB encode problem </summary>

  ```
  에러 메시지
  incorrect string value ' xed x85 x8c xec x8a xa4...' for column
  ```
  ALTER TABLE 테이블명 convert to charset utf8;
</details>

</br>

---
### 🐳 서비스 테스트 결과서

1. 회원가입
2. 로그인
3. 나눔 글 등록, 수정, 삭제
4. 나눔 게임 진행
5. 테이블 생성, 삭제
6. 테이블 예약, 예약 취소

➔ 시연
