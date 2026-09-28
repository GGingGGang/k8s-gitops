# Argo CD 애플리케이션 GitOps

`argocd/`는 서비스 계층의 app-of-apps를 정의합니다. `apps-root` Application이 `apps/*.yaml`을 읽어 서비스별 Application을 만들고, 각 Application이 해당 `manifests/<service>` Kustomization을 서비스 namespace에 자동 동기화합니다.

```
k8s-gitops/
├── argocd/
│   ├── project.yaml        # AppProject `apps`: 소스 저장소와 배포 대상 namespace
│   ├── root.yaml           # app-of-apps `apps-root`: argocd/apps/*.yaml 관리
│   └── apps/
│       ├── auth.yaml       # manifests/auth → auth namespace
│       ├── batch.yaml      # manifests/batch → batch namespace
│       ├── core.yaml       # manifests/core → core namespace
│       ├── notify.yaml     # manifests/notify → notify namespace
│       └── web.yaml        # manifests/web → web namespace
└── manifests/
    ├── auth/               # deployment / service / httproute / servicemonitor / kustomization
    ├── batch/              # deployment / service / servicemonitor / kustomization
    ├── core/               # deployment / service / httproute / servicemonitor / kustomization
    ├── notify/             # deployment / service / servicemonitor / kustomization
    └── web/                # deployment / service / httproute / kustomization
```

## 사전 조건

- Argo CD가 `cicd` namespace에서 동작해야 합니다.
- `core`, `batch`, `auth`, `notify`, `web` namespace가 존재해야 합니다. `apps-root`는 `CreateNamespace=false`를 사용하므로 namespace를 만들지 않습니다.
- 각 서비스 namespace에는 private GHCR 이미지를 위한 `ghcr-pull` Secret이 있어야 합니다. 아래 예시는 `core`에 만드는 방법이며, 다른 서비스도 namespace만 바꿔 반복합니다.

  ```bash
  cfg=$(kubectl get secret ghcr-push -n build -o go-template='{{index .data ".dockerconfigjson"}}')
  kubectl create secret generic ghcr-pull -n core --type=kubernetes.io/dockerconfigjson \
    --from-literal=.dockerconfigjson="$(echo "$cfg" | base64 -d)" --dry-run=client -o yaml | kubectl apply -f -
  ```

- Deployment가 참조하는 서비스별 Secret도 준비해야 합니다. `auth`, `batch`, `core`는 `db-creds`를 사용하고, `auth`는 `jwt-signing-key`, `core`는 선택적인 `gemini-api-key`도 참조합니다.

## 설치

최초 한 번은 Argo CD가 자신을 관리하기 전의 bootstrap 단계이므로 아래 두 리소스를 적용합니다. 이후 `apps-root`가 Git의 `apps/*.yaml` 변경을 자동 반영합니다.

```bash
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/root.yaml
```

서비스별 Application은 `automated`, `prune`, `selfHeal`, `ServerSideApply=true`를 사용합니다. `apps-root`도 `automated`, `prune`, `selfHeal`을 사용하며, `syncOptions`에는 `CreateNamespace=false`가 설정되어 있습니다.

## 확인

```bash
kubectl get appproject apps -n cicd
kubectl get application apps-root auth batch core notify web -n cicd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
kubectl get pods,svc,httproute -n core
kubectl get pods,svc -n batch
kubectl get pods,svc,httproute -n auth
kubectl get pods,svc -n notify
kubectl get pods,svc,httproute -n web
```

ServiceMonitor가 있는 `auth`, `batch`, `core`, `notify`는 다음처럼 확인할 수 있습니다.

```bash
kubectl get servicemonitor -n auth
kubectl get servicemonitor -n batch
kubectl get servicemonitor -n core
kubectl get servicemonitor -n notify
```

## 이미지 배포 흐름

서비스 저장소의 Jenkins 파이프라인은 이 저장소의 `images.newTag`를 Git SHA로 바꾸고 GitOps `main`에 직접 push합니다. Argo CD는 이 Git 변경을 감지해 해당 Application을 자동 동기화합니다. Jenkins 성공은 GitOps push까지를 뜻하며, 실제 배포 완료는 Application의 Sync와 Health 상태를 확인해야 합니다.

`latest`처럼 바뀔 수 있는 태그는 Git의 desired state와 실제 이미지 digest가 달라도 동기화 상태에 드러나지 않을 수 있으므로 사용하지 않습니다.

## 서비스 추가

새 서비스에는 다음 변경이 함께 필요합니다.

1. `manifests/<service>/`에 `deployment.yaml`, `service.yaml`, `kustomization.yaml`을 추가하고, 관측이 필요하면 `servicemonitor.yaml`, HTTP 공개가 필요하면 `httproute.yaml`을 Kustomization `resources`에 포함합니다.
2. `argocd/apps/<service>.yaml`에 Application을 추가해 `manifests/<service>`와 배포 namespace를 연결합니다.
3. `argocd/project.yaml`의 `destinations`에 서비스 namespace를 추가합니다.
4. 해당 namespace와 필요한 Secret, `ghcr-pull` Secret을 준비합니다.

`apps-root`가 `apps/*.yaml`을 자동 동기화하므로 Application 파일을 추가하면 새 Application도 관리 대상에 들어갑니다.
