---
layout: post
title: Cloud Build와 Argo CD를 이용한 CI/CD 파이프라인
categories: CI/CD
tags: [Cloud Build, Artifact Registry, Cloud Source Repositories]
banner: /assets/images/2024-01-18-cicd/architecture.png
---

CI/CD 파이프라인은 소프트웨어 개발 및 배포 과정을 자동화하기 위한 일련의 프로세스입니다. CI는 Continuous Integration의 약자이고, CD는 Continuous Deployment 또는 Continuous Delivery의 약자로 빠르게 변화하는 소프트웨어 개발 환경에 대응하여 코드 통합, 테스트, 배포 등의 작업을 자동화하고 지속적으로 소프트웨어를 개발하고 배포하는 방식으로 높은 품질의 소프트웨어를 신속하게 제공할 수 있도록 합니다.

이 글에서는 Google Cloud의 Cloud Build와 Argo CD를 통해 GKE 클러스터에 배포 파이프라인을 구축하는 방법에 대해서 알아보겠습니다. 

<img src='/assets/images/2024-01-18-cicd/architecture.png'>

흐름은 다음과 같습니다. 

**Continuous Ingergration**
* Cloud Source Repositories에서 소스 코드와 쿠버네티스 매니페스트를 관리합니다. 
* 소스 코드가 저장되어 있는 리포지토리에서 Push 이벤트가 발생하면 Build가 트리거됩니다.
* Cloud Build는 컨테이너 이미지를 빌드하고 Artifact Registry에 저장하며, Manifest 파일이 저장되어 있는 리포지토리의 이미지 태그를 수정합니다. 

**Continuous Deployment**
* Argo CD는 매니페스트 저장소의 변경을 감지하고 저장소의 상태와 쿠버네티스 클러스터의 상태를 동기화하요 자동으로 배포를 수행합니다. 

**사전 요구 사항**
- 배포할 애플리케이션 코드
- [GKE(Google Kubernetes Engine) 클러스터](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster?hl=ko)
- [Google Cloud CLI 설치](https://cloud.google.com/sdk/docs/install-sdk?hl=ko)

사전 요구 사항이 충족되지 않는 경우 첨부한 링크를 참조하여주세요. 

## CI 파이프라인 설정

### Artifact Registry

Artifact Registry는 Google Cloud에서 제공되는 컨테이너 이미지 및 패키지(Maven, npm) 등을 저장 및 관리하는 서비스입니다. Google Cloud의 CI/CD 서비스와 통합하여 자동화된 파이프라인을 구축할 수도 있습니다. 

여기서는 Cloud Build에서 빌드된 이미지를 저장하기 위해 사용됩니다. 

#### Artifact Registry 생성

Artifact Registry 저장소를 생성하기 위해서는 **Artifact Registry 저장소 관리자**(roles/artifactregistry.repoAdmin) IAM 역할이 필요합니다.

1. [Artifact Registry Repositories](https://console.cloud.google.com/artifacts?project=mystore-poc) 페이지로 이동 후 [CREATE REPOSITORY] 버튼 클릭
2. 생성할 Repository 정보 입력

| 구분 | 내용 | 비고 |
|--|--|--|
| Name | 리포지토리명 | |
| Format | Docker | 저장할 패키지 유형 |
| Mode | Standard | 비공개 아티팩트에 대한 일반적인 저장소 |
| Location type | Region: asia-northeast3 (Seoul) | |

3. [CREATE] 버튼을 클릭하여 Repository 생성

<img src='/assets/images/2024-01-18-cicd/artifact-registry.png'>

생성된 Repository에 저장될 이미지의 이름 형식은 다음과 같습니다.  
<span style='color:#0067A3'>LOCATION</span>-docker.pkg.dev/<span style='color:#0067A3'>PROJECT-ID</span>/<span style='color:#0067A3'>REPOSITORY</span>/<span style='color:#0067A3'>IMAGE</span>:<span style='color:#0067A3'>TAG</span>  

### Cloud Source Repositories

Cloud Source Repositories는 Google Cloud에서 호스팅되는 비공개 Git 저장소입니다. Cloud Build와 연결하여 Repository에 저장된 소스 코드를 가져가 사양에 맞게 빌드를 실행하고 컨테이너 이미지를 생성하도록 할 수 있습니다.

> Cloud Build는 Cloud Source Repositories 뿐만 아니라 Cloud Storage, GitHub, Bitbucket과도 통합할 수 있습니다.

여기서는 아래와 같이 다른 용도로 사용할 2개의 저장소가 필요합니다. 
* **app 저장소**: 애플리케이션 소스 코드를 저장
* **manifest 저장소**: 쿠버네티스에 배포할 매니페스트 파일을 저장

#### App Repository 생성

Cloud Source Repositories에서 코드 저장소를 생성하기 위해서는 **소스 저장소 관리자**(roles/source.admin) IAM 역할이 필요합니다. 

1. [Cloud Source Repositories](https://source.cloud.google.com/repos) 페이지로 이동 후 오른쪽 상단의 [Add repository] 버튼 클릭
2. [Create new repository] > [Continue]를 클릭하여 빈 저장소 생성
3. 생성할 Repository 정보 입력

| 구분 | 내용 |
|--|--|
| Repository Name | app-repo |
| Project | 프로젝트명 |

4. [CREATE] 버튼을 클릭하여 Repository 생성

#### Manifest Repository 생성

App Repository와 동일한 방식으로 저장소를 생성해 줍니다. 

1.  [Cloud Source Repositories](https://source.cloud.google.com/repos) 페이지로 이동 후 오른쪽 상단의 [Add repository] 클릭
2.  [Create new repository] > [Continue]를 클릭하여 빈 저장소 생성
3.  생성할 Repository 정보 입력

| 구분 | 내용 |
|--|--|
| Repository Name | manifest-repo |
| Project | 프로젝트명 |

4. [CREATE] 버튼을 클릭하여 Repository 생성

<img src="/assets/images/2024-01-18-cicd/source-repositories.png">

#### 로컬 인증 설정

로컬에서 Cloud Source Repositories에 액세스하거나 상호작용 하기 위해서는 SSH나 gcloud CLI를 통한 인증 설정이 필요합니다. 

##### SSH를 통한 인증

1. 저장소에 액세스하려는 로컬 시스템에 새 SSH 키 생성
```bash
ssh-keygen -t rsa -C "your_email@example.com"
```
2. [SSH 키 관리 페이지](https://source.cloud.google.com/user/ssh_keys?hl=ko)로 이동 후 [Register SSH key] 클릭
3. 등록할 SSH 키 정보 입력

| 구분 | 내용 | 비고 |
|--|--|--|
| Key Name | *name for key* | |
| Key | *public key* | ssh- 또는 ecdsa-로 시작 |

##### gcloud CLI를 통한 인증
1. `gcloud init` 명령어 실행
2. 안내에 따라 인증 설정

```bash
➜  ~ gcloud init
Welcome! This command will take you through the configuration of gcloud.

Settings from your current configuration [default] are:
core:
  account: minjung.lee@mz.co.kr
  disable_usage_reporting: 'False'
  project: project-name

Pick configuration to use:
 [1] Re-initialize this configuration [default] with new settings 
 [2] Create a new configuration
Please enter your numeric choice:   
...
```

### Cloud Build

Cloud Build는 소스 코드를 빌드하고 테스트하며 Google Cloud의 다양한 서비스에 배포할 수 있도록 하는 Google Cloud 서비스입니다. 
앞서 생성해준 Cloud Source Repositories에서 소스 코드를 가져와 컨테이너 이미지를 생성할 수 있습니다. 

#### Cloud Build Trigger 생성

1. [Cloud Build Triggers](https://console.cloud.google.com/cloud-build/triggers) 페이지로 이동 후 [CREATE TRIGGER] 버튼 클릭
2. 생성할 Trigger 정보 입력

| 구분 | 내용 |
|--|--|
| Name | *Cloud Build Trigger명* |
| Region | asia-northeast3 (Seoul) |
| Event | Push to a branch|

**Source**
| 구분 | 내용 |
|--|--|
| Repository | app-repo |
| Branch | ^master$ |

<img src="/assets/images/2024-01-18-cicd/cloud-build-event.png">

**Configuration**
| 구분 | 내용 |
|--|--|
| Type | Cloud Build configuration file (yaml or json) |
| Location | Repository |
| Cloud Build configuration file location | / cloudbuild.yaml |

<img src="/assets/images/2024-01-18-cicd/cloud-build-configuration.png">

3. [CREATE] 버튼을 클릭하여 Trigger 생성

<img src="/assets/images/2024-01-18-cicd/cloud-build-trigger.png">

#### cloudbuild.yaml 파일 생성

1. app-repo 루트 디렉토리에 cloudbuild.yaml 파일 생성
2. 하단의 코드 입력

**Docker 이미지 빌드**

도커 빌더 이미지를 사용하여 이미지를 빌드합니다. '-t' 옵션을 통해 앞서 생성해준 Artifact Registry Repository를 지정하고 태그로는 현재 Git 커밋의 짧은 해시값을 설정해줍니다. 

```yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'LOCATION-docker.pkg.dev/PROJECT-ID/REPOSITORY/IMAGE-NAME:$SHORT_SHA', '.'] 
```

**Manifest Repository 업데이트**

Manifest가 저장된 Git Repository를 클론한 후 컨테이너 이미지 태그를 업데이트 해줍니다. 변경사항은 커밋해준 후 리모트 저장소의 master 브랜치에 push합니다. 

```yaml
- name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: /bin/sh
  args:
  - '-c'
  - |
    gcloud source repos clone manifest-repo && \
    cd manifest-repo && \
    git checkout master && \
    git config user.email $(gcloud auth list --filter=status:ACTIVE --format='value(account)') && \
    cd overlays/dev && \
    kustomize edit set image LOCATION-docker.pkg.dev/PROJECT-ID/REPOSITORY/IMAGE-NAME=LOCATION-docker.pkg.dev/PROJECT-ID/REPOSITORY/IMAGE-NAME:$SHORT_SHA
    git add . && \
    git commit -m "image 변경" && \
    git push origin master

options:
  logging: CLOUD_LOGGING_ONLY
```

**Artifact Registry에 이미지 Push**

빌드한 이미지를 Artifact Registry에 push합니다. 

```yaml
images:
- 'LOCATION-docker.pkg.dev/PROJECT-ID/REPOSITORY/IMAGE-NAME:$SHORT_SHA'
```
>**steps** : 빌드 파이프라인에서 실행할 단계를 정의  
**name** : 사용할 빌더 이미지를 지정 (언어 및 도구가 설치된 컨테이너 이미지로 해당 이미지 내에서 작업이 실행됨)   
**args** : 빌드를 실행할 때 전달할 명령어 및 옵션을 정의    
**images** : 컨테이너 이미지를 repository에 push하는 단계를 정의

## CI 파이프라인 설정

### Argo CD

GKE 클러스터 내에 애플리케이션 배포 자동화를 위해 Argo CD를 사용해줍니다. Argo CD는 Git 저장소의 변경사항을 감지하고 클러스터의 상태와 Git 저장소의 상태를 동기화 해줍니다. 

#### Argo CD CLI 설치

하단의 링크를 참고하여 Argo CD CLI를 설치합니다.   
https://argo-cd.readthedocs.io/en/stable/cli_installation/ 

#### Argo CD 설치

Argo CD를 Helm을 통해 설치할 것이기 때문에 Helm이 설치되어 있지 않은 경우 [헬름 설치하기 문서](https://helm.sh/ko/docs/intro/install/)를 참고하여 설치 후 다음 단계를 진행해줍니다.

Helm은 쿠버네티스 패키지 매니저입니다. Helm을 이용하지 않고도 Argo CD를 배포할 수 있지만, Helm을 사용하면 간편하게 설치할 수 있으며 구성을 더 편하게 관리하고 커스터마이징할 수 있도록 해줍니다.

1. Helm에 Argo CD Repository 추가

```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

2. Argo CD 배포

```bash
kubectl create namespace argocd
helm -n argocd install argocd argo/argo-cd
```

3. Argo CD Web UI를 외부에서 접근할 수 있도록 서비스를 LoadBalancer 타입으로 변경

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

```bash
➜  ~  k get svc -n argocd
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                     AGE 
argocd-server    LoadBalancer   10.28.3.107    EXTERNAL-IP   80:32057/TCP,443:30134/TCP   17h
```

4. Argo CD 로그인 및 패스워드 변경

```bash
argocd admin initial-password -n argocd # 초기 비밀번호 확인
argocd login <ARGOCD_SERVER> # 위에서 확인한 비밀번호를 가지고 Argo CD에 로그인
argocd account update-password # 비밀번호 변경
```

<img src="/assets/images/2024-01-18-cicd/argocd-ui.png" style="width: 100%">


#### Argo CD 연결
1. ArgoCD에서 private 저장소인 Soucre Repository에 액세스하기 위해 Service Account 생성 및 해당 SA의 account key 발급
* argocd-sa@mystore-poc.iam.gserviceaccount.com SA를 생성하여 Source Repository Admin 권한 부여
* IAM & Admin Service Accounts 페이지에서 argocd-sa 클릭 
[KEYS] > [ADD KEY] > [Create new key] > Key type JSON > CREATE
> SA Private Key는 유출되지 않도록 관리에 유의 필요

<img src="/assets/images/2024-01-18-cicd/argocd-sa.png">

2. Source Repository 연결

* GUI를 이용한 연결

| 구분 | 내용 |
|--|--|
| Project | default |
| Repository URL | https://source.developers.google.com/p/*PROJECT-NAME*/r/*REPOSITORY-NAME* |
| GCP Service Account Key | *access-key.json 내용 입력* |

<img src="/assets/images/2024-01-18-cicd/argocd-repo.png">

* CLI를 이용한 연결
```bash
argocd repo add https://source.developers.google.com/p/PROJECT-NAME/r/REPOSITORY-NAME --gcp-service-account-key-path access-key.json
```

<img src="/assets/images/2024-01-18-cicd/argocd-repo-check.png">

3. App 연결

| 구분 | 내용 |
|--|--|
| Application Name | *Application-Name* |
| Sync Policy | Manual |
| Repository URL | https://source.developers.google.com/p/*PROJECT-NAME*/r/*REPOSITORY-NAME* |
| Path | overlays/dev |
| Cluster URL | https://kubernetes.default.svc |  
| Namepsace | default |

<img src="/assets/images/2024-01-18-cicd/argocd-app-summary.png">

Argo CD에 Repository와 Application 연결 구성을 완료하고 상태가 Synced인 경우 애플리케이션이 배포되고 쿠버네티스 리스소가 생성된 것을 확인할 수 있습니다.

```bash
Name:               argocd/APP_NAME
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://EXTERNAL_IP/applications/APP_NAME
Repo:               https://source.developers.google.com/p/PROJECT-NAME/r/REPOSITORY-NAME
Target:             master
Path:               overlays/dev
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to master (41e12b7)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME               STATUS  HEALTH   HOOK  MESSAGE
apps   Deployment  default    APP_NAME  Synced  Healthy        deployment.apps/DEPLOY-NAME configured
```

```bash
➜  ~ k get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
DEPLOY_NAME   1/1     1            1           6d2h
```

<img src="/assets/images/2024-01-18-cicd/argocd-app.png">

## CI/CD 테스트

Cloud Source Repository에 저장된 파일 변경사항을 푸시할 때마다 SHORT-SHA 태그와 함께 Cloud Build에서 이미지가 빌드되며 빌드된 이미지는 Artifact Registry에 저장됩니다. 

하단의 사진에서 Git 커밋에 대한 해시 값이 `312adf3`인 것을 볼 수 있습니다. 해당 해시 값은 생성될 컨테이너 이미지의 태그로 사용됩니다. 

<img src="/assets/images/2024-01-18-cicd/build-test.png">

Artifact Registry에 저장된 가장 최신의 이미지와 kustomization 파일의 이미지가 `312adf3`` 태그 값을 가지고 있는 것을 확인할 수 있습니다.

<img src="/assets/images/2024-01-18-cicd/image-test.png">
<img src="/assets/images/2024-01-18-cicd/kustomization-test.png">

Argo CD에서 변경사항을 감지하고 OutOfSync 상태인 것을 확인할 수 있습니다. 

<img src="/assets/images/2024-01-18-cicd/outofsync.png">

Argo CD Sync 설정 시 Deployment의 이미지가 `312adf3` 태그를 가진 이미지로 수정되어 배포되고 Synced 상태로 바뀐 것을 볼 수 있습니다. 

```bash
➜  ~ k rollout history deploy DEPLOY_NAME --revision 2
deployment.apps/DEPLOY_NAME with revision #2
Pod Template:
  Labels:	app=hDEPLOY_NAME
	pod-template-hash=587dc5b646
  Containers:
   test-app:
    Image:	asia-northeast3-docker.pkg.dev/PROJECT/REPOSITORY/CONTAINER:312adf3
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

<img src="/assets/images/2024-01-18-cicd/synced.png">

## Reference

클라우드 소스 저장소 로컬 인증 설정  
<https://cloud.google.com/source-repositories/docs/authentication?hl=ko>

Argo CD Getting Started  
<https://argo-cd.readthedocs.io/en/stable/getting_started/>

Cloud Build를 사용한 GitOps 방식의 지속적 배포  
<https://cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build?hl=ko> 