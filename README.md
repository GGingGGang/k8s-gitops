# k8s-gitops

앱 레이어 GitOps 진입점. MSA 서비스를 ArgoCD app-of-apps 로 배포하는 **Application CR(포인터)** 만 보유한다.

실제 매니페스트(deployment/service/httproute/kustomization)는 각 서비스 레포 `deploy/k8s` 가 소유 — 본 레포는 *무엇을 배포할지(Application)* 만 선언한다 (ArgoCD config vs source code 분리).

```
k8s-gitops/
└── argocd/
    ├── project.yaml    # AppProject `apps`
    ├── root.yaml       # app-of-apps `apps-root`
    └── apps/
        └── core.yaml   # 서비스별 Application 포인터
```

부트스트랩 · 검증 · 결정: [`argocd/README.md`](./argocd/README.md).
