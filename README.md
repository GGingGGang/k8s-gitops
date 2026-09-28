# k8s-gitops

애플리케이션 GitOps 저장소입니다. MSA 서비스의 **배포 상태**(Kubernetes 매니페스트와 Argo CD Application CR)를 관리합니다.

각 서비스 저장소는 애플리케이션 코드, `Dockerfile`, `Jenkinsfile`을 관리하고, 이 저장소는 어느 이미지 버전을 배포할지와 Kubernetes desired state를 관리합니다.

```
k8s-gitops/
├── argocd/
│   ├── project.yaml       # AppProject `apps`
│   ├── root.yaml          # app-of-apps `apps-root`
│   └── apps/              # 서비스별 Argo CD Application
│       ├── auth.yaml
│       ├── batch.yaml
│       ├── core.yaml
│       ├── notify.yaml
│       └── web.yaml
└── manifests/
    ├── auth/              # Deployment, Service, HTTPRoute, ServiceMonitor, Kustomization
    ├── batch/             # Deployment, Service, ServiceMonitor, Kustomization
    ├── core/              # Deployment, Service, HTTPRoute, ServiceMonitor, Kustomization
    ├── notify/            # Deployment, Service, ServiceMonitor, Kustomization
    └── web/               # Deployment, Service, HTTPRoute, Kustomization
```

서비스 이미지는 GHCR에서 가져오며, `manifests/<service>/kustomization.yaml`의 `images.newTag`에는 CI가 기록한 변경 불가능한 Git SHA 태그를 둡니다.

배포 흐름은 다음과 같습니다. Jenkins의 일반 `main` 빌드가 최신 소스 SHA인지 검증한 뒤 `ci/bump-<service>` Deployment PR을 생성하거나 갱신합니다. 이 PR이 `main`에 병합되면 Argo CD가 Git 변경을 감지하고 해당 Application을 자동 동기화합니다. Jenkins 빌드 성공만으로 배포가 완료되는 것은 아닙니다.

| 서비스 | namespace | 포함 리소스 | HTTP 노출 |
| --- | --- | --- | --- |
| `auth` | `auth` | Deployment, Service, HTTPRoute, ServiceMonitor | `api.ggang.cloud/v1/auth` → `auth:3000` (접두사 제거) |
| `batch` | `batch` | Deployment, Service, ServiceMonitor | 없음 |
| `core` | `core` | Deployment, Service, HTTPRoute, ServiceMonitor | `api.ggang.cloud/v1/core` → `core:8080` (접두사 제거) |
| `notify` | `notify` | Deployment, Service, ServiceMonitor | 없음 |
| `web` | `web` | Deployment, Service, HTTPRoute | `www.ggang.cloud/` → `web:8080` |

초기 설치, 사전 조건, 운영 확인과 서비스 추가 방법은 [argocd/README.md](./argocd/README.md)를 참고하세요.
