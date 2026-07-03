# argocd — 앱 레이어 GitOps 진입점

MSA 서비스(현재 `core`)를 ArgoCD 로 배포하는 app-of-apps. 플랫폼 인프라와 **별도 AppProject(`apps`)** 로 분리.

본 GitOps 레포는 배포 상태를 소유한다: `argocd/` 에 **Application CR**, `manifests/<svc>/` 에 **실제 매니페스트**(deployment/service/httproute/kustomization). 앱 레포는 소스 코드 + `Dockerfile` + `Jenkinsfile` 만 갖는다.

```
k8s-gitops/
├── argocd/
│   ├── project.yaml        # AppProject `apps` — 소스 레포 + 대상 NS 화이트리스트
│   ├── root.yaml           # app-of-apps `apps-root` — apps/*.yaml recurse, auto-sync
│   └── apps/
│       └── core.yaml       # core Application → manifests/core, dest core NS
└── manifests/
    └── core/               # deployment / service / httproute / kustomization
```

## 1. 전제 조건

- ArgoCD 동작 (`cicd` NS)
- 대상 NS 존재 (`core`)
- 대상 NS 에 `ghcr-pull` Secret (GHCR private 이미지 pull). PAT 재발급 없이 `build/ghcr-push` 복사:
  ```bash
  cfg=$(kubectl get secret ghcr-push -n build -o go-template='{{index .data ".dockerconfigjson"}}')
  kubectl create secret generic ghcr-pull -n core --type=kubernetes.io/dockerconfigjson \
    --from-literal=.dockerconfigjson="$(echo "$cfg" | base64 -d)" --dry-run=client -o yaml | kubectl apply -f -
  ```

## 2. 설치

부트스트랩은 일회성 `kubectl apply` (self-managed adopt). 이후 `apps-root` 가 git 의 `apps/*.yaml` 을 auto-sync.

```bash
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/root.yaml
```

`apps-root` 가 `apps/core.yaml` 을 발견 → `core` Application 생성 → `manifests/core` 를 `core` NS 에 sync.

## 3. 검증

```bash
kubectl get appproject apps -n cicd
kubectl get application apps-root core -n cicd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
kubectl get pods,svc,httproute -n core
```

## 4. 결정

### 인프라 레포와 분리된 전용 GitOps 레포

앱 레이어 app-of-apps 를 인프라 레포에서 들어내 전용 레포로 격리. *배포 상태(매니페스트 + Application)* 는 본 GitOps 레포, *소스 코드* 는 앱 레포, *인프라/플랫폼* 은 인프라 레포 — 세 관심사를 레포 경계로 분리. ArgoCD 공식 *config vs source code 분리* 의 연장.

### 매니페스트를 앱 레포가 아니라 GitOps 레포가 소유

이미지 태그 bump(CI 산출물)와 소스 코드(사람 산출물)를 같은 레포에 두면 빌드 루프·머지 노이즈·권한 과다가 섞인다. 배포 상태를 본 레포로 들어내면 앱 레포는 소스만, 본 레포는 desired state 만 갖는다. CI 는 `manifests/<svc>/kustomization.yaml` 의 태그만 bump.

### `platform` 과 분리된 `apps` AppProject

`apps` 의 `sourceRepos` 는 본 GitOps 레포로 한정, `destinations` 는 서비스 NS(`core`/`batch`/`login`) + Application CR 이 사는 `cicd` 로 한정. 인프라 레포가 앱 NS 를, 앱 레포가 인프라를 건드리지 못하게 권한 경계를 프로젝트로 강제.

### auto-sync (prune + selfHeal) — 앱은 켬

앱 레이어 Application 은 `automated` 활성. 이미지 태그 bump 커밋 → 자동 배포가 GitOps 루프의 목적. 반면 Jenkins 는 emptyDir 라 sync = pod 재기동 = 빌드 즉사 → 수동 유지. *디스럽션 비용이 다르면 sync 정책도 다르다.*

### 이미지 private → `ghcr-pull`

GHCR 패키지가 private(anonymous pull 401)이라 서비스 NS 마다 `imagePullSecrets`. 시크릿 값은 git 미커밋 — `build/ghcr-push` 를 복사(전제 조건). 후속 OpenBao/ESO 이관 대상.

## 5. 주의 사항

### 부트스트랩 의존

`project.yaml`/`root.yaml` 은 ArgoCD 가 자기 자신을 관리하기 전 단계라 최초 1회 `kubectl apply` 필요. 이후는 git 이 진실원천 — `apps/*.yaml` 추가/삭제는 push 만 하면 `apps-root` 가 반영.

### 새 서비스 추가

`manifests/<svc>/` 생성(deployment/service/httproute/kustomization) + `apps/<svc>.yaml` 에 Application 추가(`manifests/<svc>` 지시) + 대상 NS 생성 + `apps` AppProject `destinations` 에 NS 추가 + 해당 NS 에 `ghcr-pull` 복사. `apps-root` 가 auto-sync 로 흡수.

### 이미지 태그가 `:latest` 면 재배포 안 됨

kustomization `images.newTag` 가 `latest` 같은 mutable 태그면 ArgoCD 가 git desired(`:latest`)와 live(`:latest`)를 문자열 비교 → digest 가 바뀌어도 Synced 로 보고 재배포 안 함. CI 가 불변 SHA 태그를 git 에 bump 해야 루프가 닫힘.
