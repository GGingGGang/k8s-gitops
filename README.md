# k8s-gitops

앱 레이어 GitOps 진입점. MSA 서비스의 **배포 상태** — k8s 매니페스트 + ArgoCD Application CR — 를 소유한다.

앱 레포(`svc-*`)는 소스 코드 + `Dockerfile` + `Jenkinsfile` 만 갖는다. *무엇을 어떻게 배포하는지*는 본 레포가 소유한다 (ArgoCD config vs source code 분리).

```
k8s-gitops/
├── argocd/
│   ├── project.yaml       # AppProject `apps`
│   ├── root.yaml          # app-of-apps `apps-root`
│   └── apps/
│       └── core.yaml      # 서비스별 Application → manifests/<svc>
└── manifests/
    └── core/              # 서비스별 k8s 리소스 (deployment/service/httproute/kustomization)
```

흐름: 앱 레포 push → Jenkins(Kaniko) → GHCR → `deployBump` 가 `manifests/<svc>/kustomization.yaml` 의 이미지 태그를 불변 SHA 로 bump → ArgoCD 자동 sync.

부트스트랩 · 검증 · 결정: [`argocd/README.md`](./argocd/README.md).
