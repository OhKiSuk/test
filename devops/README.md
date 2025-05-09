# k8s-workshop

### Table of Contents
#### [k8s-workshop](#k8s-workshop)
  ##### [DevOps](#DevOps)
    1. [소개](#소개)
    2. [환경 준비](#환경-준비)
    3. [설치](#설치)
    4. [실행](#실행)
    5. [CI 구성](CI-구성)
    6. [CD 구성](#CD-구성)
    7. [테스트](#테스트)

### DevOps

----
#### 1. 소개
본 문서는 Kubernetes 클러스터를 기반으로 CI/CD 환경을 구성하는 Hands-on Workshop이다.

본 문서에서 사용된 구성은 다음과 같다.
- k3s
- ingress-nginx
- jenkins(클러스터 외부에 설치)
- argocd
- mysql
- simple-api
- sample-api
- github(외부 서비스)
- docker hub(외부 서비스)

##### Workshop 목표
최종적으로 미리 작성된 Ansible 코드를 이용하여 위와 같이 테스트를 위한 Kubernetes 환경을 로컬에 구성하고, 샘플 어플리케이션(simple-api, sample-api)이 로컬에서 동작하는 것을 확인하여 Kubernetes를 이해하고 향후 개발 및 운영을 위한 기반으로 삼는다.

----
#### 2. 환경 준비
```markdown
**(2025.05.09) 하드웨어 사양**

CPU: 4Core 8Thread
RAM: 16GB
SSD: 512GB

**(2025.05.09) 환경 구성 시 필요한 소프트웨어 목록**
- VirtualBox
- Vagrant
- Putty or MobaXTerm(for Windows)
```
##### 2-1. VM 생성
다음과 같이 Vagrant(VirtualBox) 기반으로 VM을 두 대 생성한다. Vagrant 파일을 위치시킬 디렉토리을 생성하되 각각의 Vagant 파일을 다른 경로에 위치시킨다.
```shell
# Mac/Linux
mkdir -p /Your/Projcet/Path
cd /Your/Project/Path

# Windows
mkdir C:\Your\Project\Path
cd C:\Your\Project\Path

git clone https://github.com/oscka/k8s-workshop.git
```
###### Agent-Ansible1(VM1)
- Ansible 코드를 받아 Target 서버에 설치를 수행한다.
```markdown
- 파일 위치: vagrant\vbox\ansible1\Vagrant
- 권장 사양: 
	- CPU: 4Core
	- RAM: 4GB
- (2025.05.09) 최소 사양(Vagrant 파일 수정)
	- CPU: 1Core
	- RAM: 2GB(2,048MB)

- 계정 정보
	- ID: vagrant
	- Password: vagrant
```
###### Target-Ansible2(VM2)
- Jenkins 및 k3s 기반 클러스터. sample-api가 실행된다.
```markdown
- 파일 위치: vagrant\vbox\ansible2\Vagrant
- 권장 사양: 
	- CPU: 8Core
	- RAM: 8GB
- (2025.05.09) 최소 사양(Vagrant 파일 수정)
	- CPU: 3Core
	- RAM: 6GB(6,144MB)

- 계정 정보
	- ID: vagrant
	- Password: vagrant
```
##### 2-2. Vagrant 파일 생성 및 수정 후 Vagrant로 VM을 생성
- 각각의 Vagrantfile 위치에서 실행한다.
```shell
# 1. Vargrantfile 생성
vagrant init

# 2. Vagrantfile을 실행하여 VM 생성
vagrant up
```
##### 2-3. VM 생성 후 각 환경에 SSH로 접속되는 지 확인
```shell
# 1. sshd_config 설정
sudo vi /etc/ssh/sshd_config

# PasswordAuthentication 활성화 no → yes
...
PasswordAuthentication yes
...

# 2. cloudimg settings 설정
sudo vi /etc/ssh/sshd_config.d/60-cloudimg-settings-conf

# PasswordAuthentication 활성화 no → yes
...
PasswordAuthentication yes
...

# 3. sshd 변경 사항 적용
sudo systemctl restart ssh
```
###### 참고 사항
- VM1을 Windows의 WSL 환경이나 직접 Host에 Linux 설치 후 해당 환경 내에서 수행해도 된다.
- 각 VM을 띄우는 Vagrantfile 형식은 링크된 Vagrantfile을 참고한다.
- 로컬 환경이 최소사양을 충족하지 못할 경우, WSL 환경 또는 Linux 환경에서 사용하는 것을 권장한다.(Vagrant는 오래된 툴이다 보니, 예전에는 대안이 없었는데 현재는 대안도 많고 많이 느려졌습니다.)
- 로컬 - VM1 - VM2 간의 SSH 및 HTTP 통신이 원활해야 한다.
- VM 재생성 시 known_host, authorized_keys에 이전 host, client의 키가 남아 있다.
----
#### 3. 설치
##### 3-1. Ansible 설치
```shell
# Agent 서버의 패키지를 최신 버전으로 업데이트
sudo apt update && sudo apt upgrade -y

# Ansible 설치(in Ansible1)
sudo apt install ansible

# Ansible 설치 이후 script 실행 등 정상 동작이 안될 경우
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt update
sudo apt install ansible
```
##### 3-2. Agent에서 SSH Key를 생성하고 Target에 등록
```
# 1. SSH Key 생성
ssh-keygen

# 2. Target 서버에 SSH Key 등록
ssh-copy-id vagrant@192.168.56.11

# 3. 접속 확인
ssh vagrant@192.168.56.11
```
##### 3-3. Starter-kit 설치
- Ansible에서는 설치 대상(Target) 정보를 인벤토리(Inventory)라는 개념으로 관리하며, 인벤토리(해당 문서 내에서는 hosts-vm이다.) 정보를 설치 대상 VM 정보에 맞게 설정해야 한다.
- playbooks/hosts-vm 파일의 내용을 아래와 같이 수정하여 Ansible이 작업을 실행할 수 있게 설정한다.
```shell
# 1. msa-starter-kit을 clone
git clone https://github.com/oscka/msa-starter-kit.git

# 2. hosts-vm 내용 수정
sudo vi ~/msa-starter-kit/playbooks/hosts-vm
...
step1 ansible_host=192.168.56.11 ansible_user=vagrant ansible_port=22 ansible_ssh_private_key_file=/home/vagrant/.ssh/id_rsa
...

# 3. Target에 정상적으로 접근할 수 있는 지 확인
sudo ./show-ping-test.sh
```
----
##### 4. 실행
```
# DevOps ENV Install(/w Ansible)
sudo ./run-play.sh "tool-basic, helm-repo, k3s, ingress-nginx, jenkins, docker, argocd, mysql, simple-api-argocd"
```
##### 4-1. 실행 결과
- 파란색 흐름- CI(통합 빌드)
	- 원격 저장소가 Jenkins의 job 트리거 조건(webhook, push 등)을 만족 시 소스코드를 받아 빌드, dockerizing하고 docker hub에 push한다.
- 빨간색 흐름 - CD(배포)
	- Argocd가 Gitops에 변경된 버전을 확인하여 정의된 배포 전략에 따라 배포를 수행한다.
![[201487394-ebf3a507-aa51-4cb1-87e3-08b283a868fe.png]]
##### 참고 사항
###### Jenkins 구성 관련
- Jenkins의 경우 job 실행 속도 문제로 클러스터 밖의 환경에 별도로 설치하는 방법을 기준으로 한다.
```shell
# 2024.06 기준
# 1. jenkins 실행을 위한 설치
sudo apt-get install openjdk-11-jdk

# 2. apt key 추가
wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -

# 3. apt address 추가
echo deb http://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list

# 4. apt key 등록
# 해당 키의 경우 2026년 3월 26일까지 유효하므로 이후엔 새로운 키가 필요하다.
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 5BA31D57EF5975CA
```
###### 오류 발생 시 재실행 관련
- 특정 Ansible Task Install 실패 시 Target(Ansible2)에 설치된 파일들이 남아 있다. 재설치 시 secret 생성, Argocd Password 설정 등의 작업을 다시 실행하기 때문에 이전 작업들과 충돌한다.
- 작성된 Ansible task script가 재실행 가능할 경우 재실행 해도 상관 없으나, 재실행이 안되거나 Uninstall script가 별도로 있는 경우 VM을 재생성하여 처음부터 다시 수행하거나, Uninstall한 뒤 다시 Install을 수행한다.

----
#### 5. CI 구성
##### 5-1. 사전 준비
다음 프로젝트를 fork하여 각자의 계정에 프로젝트를 생성한다.
- sample-api - [https://github.com/oscka/sample-api.git](https://github.com/oscka/sample-api.git)
- sample-gitops - [https://github.com/oscka/sample-gitops.git](https://github.com/oscka/sample-gitops.git)
향후 진행할 샘플에서 연결할 프로젝트는 개인 계정에 fork한 프로젝트를 기반으로 한다.
- sample-api - [https://github.com/{{개인ID}}/sample-api.git](https://github.com/%7B%7B%EA%B0%9C%EC%9D%B8ID%7D%7D/sample-api.git)
- sample-gitops - [https://github.com/{{개인ID}}/sample-gitops.git](https://github.com/%7B%7B%EA%B0%9C%EC%9D%B8ID%7D%7D/sample-gitops.git)
fork된 Repository 내에서 환경 구성에 필요한 소스 코드 및 파일을 다음과 같이 수정한다.
###### sample-api(파일 수정 후 Tag 0.0.1로 Release)
1. Jenkinsfile
```yaml
def PROJECT_NAME = "sample-api"
def gitUrl = "https://github.com/{{Github Username}}/${PROJECT_NAME}.git"
def gitOpsUrl = "https://github.com/{{Github Username}}/sample-gitops.git"
def opsBranch = "main"
/////////////////////////////
pipeline {
    environment {
		 PATH = "$PATH:/usr/local/bin/"  //maven, skaffold, argocd,jq path
    }
    agent any
    stages {
        stage('Build') {
            steps {
                checkout scm: [
                        $class: "GitSCM",
                        userRemoteConfigs: [[url: "${gitUrl}",
                        credentialsId: "git-credential" ]],     //credential 이름이 jenkins에 등록된 이름과 동일해야 함
                        branches: [[name: "refs/tags/${TAG}"]]],
                    poll: false
                script{
                    docker.withRegistry("","imageRegistry-credential"){   //credential 이름이 jenkins에 등록된 이름과 동일해야 함, jenkins에 docker deploy 권한 필요
                        sh "skaffold build -p dev -t ${TAG}"
                    }
                    // mac local 일때만 사용 linux 환경에서는 docker.withRegistry 사용
                    //sh "skaffold build -p dev -t ${TAG}"
                }
            }
        }
        stage('workspace clear'){
            steps {
                cleanWs()
            }
        }

        stage('GitOps update') {
            steps {
	            print "======kustomization.yaml tag update====="
	            withCredentials([
		            gitUsernamePassword(credentialsId: 'git-credential', gitToolName: 'Default')
                    ]) {
                        sh """
                        git clone ${gitOpsUrl}
                        cd ./sample-gitops/sample-api/rolling-update-no-istio
                        kustomize edit set image {{DockerHub Username}}/sample-api:${TAG}
                        # 로컬외에는 주석 제거한다
                        git config --global user.email "admin@demo.com"
                        git config --global user.name "admin"
                        git add .
                        git commit -am 'update image tag ${TAG}'
                        git remote set-url --push origin ${gitOpsUrl}
                        git push origin ${opsBranch}
                        """
                    }
                    print "git push finished !!!"

                }                
                
        }

    }

}
```
2. /src/main/resources/config/application-dev.yml
```yaml
##############
### dev
##############

server:
 port: 8080 

spring:
  datasource:    
    driverClassName: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://mysql-svc.db.svc:3306/sampledb?autoReconnect=true&useUnicode=true&characterEncoding=utf8
    username: oscer
    password: oscer1234
    hikari: 
      pool-name: hikari-cp
      maximum-pool-size: 30
      minimum-idle: 2
      data-source-properties: 
          cachePrepStmts: true
          prepStmtCacheSize: 250
          prepStmtCacheSqlLimit: 2048
          useServerPrepStmts: true
  jpa:    
    show-sql: true
    properties:
      hibernate:
        format-sql: true
    hibernate:
        ddl-auto: none    

logging:
  level:  
    com.example.sampleapi: info  

logging:
  level:  
    com.example.sampleapi: info 
```
3. /src/main/resources/config/application.yml
```
spring:
  application:
    name: sample-api
  profiles:
    active: dev
```
###### sample-gitops(파일 수정 후 Tag 0.0.1로 Release)
1. sample-api/rolling-update-no-istio/sample-api-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-api
  namespace: api  
  labels:
    app: sample-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-api
  template:
    metadata:      
      labels:
        app: sample-api
    spec:
      containers:
        - name: sample-api
          image: {{DockerHub Username}}/sample-api
          ports:
          - name: http
            containerPort: 8080
```
2. sample-api/rolling-update-no-istio/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- sample-api-deployment.yaml
- sample-api-svc.yaml
- sample-api-ingress.yaml

images:
- name: {{DockerHub Username}}/sample-api
  newTag: 0.0.1
```
##### 5-2. Jenkins 환경 구성
- 참고 사항
	- Jenkins의 경우 job 실행 속도 문제로 클러스터 밖의 환경에 별도로 설치하도록 Ansible script가 구성되어 있으며, 설치만 수행하고 있다.
	- 설치 이후, 아래의 작업들을 수행해야 한다.
```shell
"""
1. Containerize를 위하여 Jenkins 계정에 Docker 실행권한 부여(Docker Daemon 재시작, 재로그인 후 반영)
"""
sudo usermod -aG docker jenkins
sudo service docker restart

"""
2. 계정 생성 및 비밀번호 변경
	- Jenkins의 초기 관리자 비밀번호를 확인
"""
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

"""
3. 로그인 후 플러그인 설치
	- Git Parameter, Workspace Cleanup, Docker Pipeline
	- Docker Pipeline의 경우 초기 설정 시 선택할 수 없으므로 초기 설정 이후 Jenkins 관리 메뉴에서 설치할 것
	- 플러그인 설치가 안될 경우 Jenkins 서비스를 재시작한 후, 다시 시도
	- 계정 정보
		- ID: admin
		- Password: admin1234
"""
# sudo systemctl restart jenkins

"""
4. credentials 생성
	4-1. Jenkins 관리 > Credentials()
		- git-credential
			- Kind: Username and Password
			- Scope: Global
			- Username: Github Username
			- Password: AccessToken(classic token, repo 권한 부여)
			- ID: git-credential
			- Description: git-credential
		- imageRegistry-credential
			- Kind: Username and Password
			- Scope: Global
			- Username: DockerHub Username
			- Password: DockerHub Password or Personal AccessToken
			- ID: imageRegistry-credential
			- Description: imageRegistry-credential
"""

"""
5. job 생성
	5-1. Dashboard > 새로운 Item
		- item name: sample-api-build
		- item type: pipeline
	5-2. Configure > '이 빌드는 매개변수가 있습니다.' 선택
	5-3. 매개변수 추가 > 'Git Parameter' 선택
	5-4. Git Parameter 설정
		- Name: TAG
		- Description: TAG(or default)
		- Default Value: 0.0.1(or default TAG Version)
	5-5. Pipeline > 'Pipeline Script from SCM' 선택
		- Credentials > 'git-credential' 선택
		- Branch to build > Branch Specifier (blank for 'any') > 본인 branch 이름(EX. */develop)
		- Pipeline Script > Jenkinsfile
#      
"""

"""
6. 빌드 도구 설치
	- skaffold, kustomize 설치(Agent에서 명령 실행)
	- jenkins 계정에서 docker 실행이 가능해야 한다.
"""
sudo ./run-play.sh "skaffold, kustomize"
```
##### 5-3. Job 실행
----
#### 6. CD 구성
- CI가 완료되면
	1. MSA 애플리케이션은 이미지 형태로 Containerize되어 DockerHub에 Deploy된다.
	2. 또한, 동시에 gitops Repository의 대상 애플리케이션 버전을 업데이트한다.
```shell
"""
1. Argocd에 로그인하여 Settings > Repository 메뉴에서 새로운 Repository를 등록(Connect Repo)한다.
	- 접속 주소
		- https://argocd.192.168.56.11.sslip.io/
		- 계정 정보
			- ID: admin
			- Password: admin1234
	1-1. Choose Your Connection Method > 'VIA HTTPS' 선택
	1-2. Project > 'default' 선택
	1-3. GitOps Address(Repository URL) > https://github.com/{{Github Username}}/sample-gitops.git
		- HTTPS 방식으로 연결 시 Github ID와 AccessToken이 필요
	1-4. Username > Github Username
	1-5. Password > Git AccessToken(환경 구축 중 발급받은 Token 사용 가능)
"""

"""
2. 배포를 위한 APP 등록
	2-1. Application > 'New APP' 클릭
	2-2. Application Name > 'sample-api' 입력
	2-3. Project Name > 'default' 선택
	2-4. Repository URL > 'https://github.com/{{Github Username}}/sample-gitops.git' 선택
	2-5. Revision > 'main' 입력
	2-6. Path > 'sample-api/rolling-update-no-istio' 선택
	2-7. Cluster URL > 'https://kubernetes.default.svc' 선택
	2-8. Namespace > 'api' 입력
		- 맨 아래 부분에는 Kustomize를 통해 기 입력된 항목들이 조회된다.
	2-9. Create 후 Refresh, SYNC 버튼을 한 번씩 클릭
		- gitops의 내용들이 클러스터에 Deploy 된다.
"""

"""
3. 애플리케이션의 버전을 올린 뒤 tag의 버전을 수정
	- 버전을 올리기 위해 소스 코드를 수정한다.
"""
git add ./*;git commit -m "version up";git push
git tag 0.0.2;git push

"""
4. Jenkins에서 해당 job을 찾아 빌드를 수행(ex. 0.0.2 선택)

5. Argocd에서 Sync하여 버전이 제대로 적용됐는 지 확인 
"""
```
----
#### 7. 테스트
생성한 애플리케이션의 테스트를 위해 다음과 같은 k8s manifast를 저장하여 클러스터에 적용한다. sample-gitops 프로젝트의 rolling-update-no-istio 경로의 리소스들이 생성된다. 아래 ingress는 자동으로 생성되지 않을 경우 생성하고 요청이 제대로 가는 지 확인한다.
######  7-1. sample-api-ingress.yaml 수정
```shell
sudo vi /var/lib/jenkins/workspace/sample-api-build/sample-gitops/sample-api/rolling-update-no-istio/sample-api-ingress.yaml
```
###### (Example) sample-api-ingress.yaml 내용
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-api-ingress
  namespace: api
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: "sample-api.192.168.56.11.sslip.io"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: sample-api-svc
            port:
              number: 8080
```
###### 7-2. 리소스 생성
```shell
sudo k apply -f ./sample-api-ingress.yaml -n api
```
###### 7-3. 생성된 다음 주소를 호출하고 테스트
[http://sample-api.192.168.56.11.sslip.io/swagger-ui.html](http://sample-api.192.168.56.11.sslip.io/swagger-ui.html])
###### 7-4. DB 접속 확인
- 같이 설정된 ingress를 통하여 다음과 같이 확인할 수 있다.
```shell
sudo apt install mysql-client-core-8.0(or mariadb-client-core-10.3)

mysql -h mysql.192.168.56.11.sslip.io -u root -p
```
