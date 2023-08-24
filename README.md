# validation-server
## introduction
* 데이터 라벨링 스튜디오에서 라벨링한 데이터를 검증하는 API를 띄우기 위한 서버입니다.
* Kubernetes 환경에서 동작하며, [validation-api](https://github.com/cilverjoo/validation-api)와 연동하여 동작합니다.
* validation-api에서 develop, master 브랜치에 변경사항이 배포되면 github action이 이미지를 빌드하여 azure acr에 배포한 후  validation-server 레포에서 kustomize를 활용하여 app/overlays/dev, app/overlays/production의  kustomization.yaml 파일을 변경합니다.
* validation-server 레포는 ArgoCD에 연동되어 app/overlays/dev 위치에서 변경사항이 발생하면 Kuternetes 환경의 리소스와 레포의 내용을 자동으로 sync 합니다.


## Kustomize
* Kustomize는 패치(Patch)를 사용하여 기존 표준 구성 파일을 방해하지 않고 환경별 구성 사항을 변경하는 도구이다.
* base에 위치한 yaml파일을 기준으로 overlays 아래의 prod, staging, dev 등 운영 환경에 맞춰 config를 바꿔가며 배포할 수 있다.

### Kustomize directory 구조
```
app
├── base
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays
    ├── dev
    │   ├── cpu_count.yaml
    │   ├── replica_count.yaml
    │   └── kustomization.yaml
    └── production
        ├── cpu_count.yaml
        ├── replica_count.yaml
        └── kustomization.yaml
```
* base와 overlay를 같은 경로에 두고, overlays 하위에 dev, production과 같이 환경별로 나눠서 변경할 패치 파일를 넣어준다.
* kustomize build ${kustomization.yaml의 경로} 를 실행하면, kustomization.yaml 파일 안에 지정된 base의 위치와 변경할 패치 정보를 입력받아 생성한 yaml 파일 정보를 리턴한다.

## ArgoCD

### ArgoCD install
```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd -f values.yaml
```

### ArgoCD ingress 시 주의사항
```
configs:
    params:
        # -- Run server without TLS
        server.insecure: true
        server.rootpath: /argocd

server:
    ingress:
        # -- Enable an ingress resource for the Argo CD server
        enabled: true
```
- argoCD는 자체 tls 설정이 들어있어서 외부 nginx-controller에서 접근 시 304 또는 504에러가 발생할 수 있다.
- 내부 server에서 insecure 옵션을 줘서 TLS 설정을 제거해주고, server의 ingress 설정을 true로 바꿔줘야 한다.
```
spec:
    template:
        spec:
            containers:
                - args:
                    - /nginx-ingress-controller
                    - --publish-service=$(POD_NAMESPACE)/basic-ingress-nginx-controller
                    ...        
                    - --enable-ssl-passthrough # 이부분 추가
```
* ArgoCD의 ingress 설정에서 ssl-passthrough 설정값을 true로 주는 경우, nginx-controller에서 위의 설정 또한 같이 변경해줘야 한다.

### ArgoCD Google OAuth
* [Google 개발자 콘솔](https://console.cloud.google.com/apis/credentials)에서 google oauth 인증을 위한 프로젝트를 생성한다.
* 자세한 내용은 [ArgoCD 공식문서](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/google/#openid-connect-using-dex) 참고!
* ingress로 ArgoCD로 연결하려는 주소에  /api/dex/callback 가 추가된 주소를 redirect uri로 하여 oauth client를 발급받는다.
* 여기서 발급받은 client-id, client-secret을 dex-server에 추가한다.

kuberenetes에서 직접 수정할 때
```
kubectl edit cm argocd-cm -n <ArgoCD 네임스페이스>
    
    
data:
  ...
  url: <INGRESS_URL>
  dex.config: |
    connectors:
    - config:
        issuer: https://accounts.google.com
        clientID: XXXXXXXXXXXXX.apps.googleusercontent.com
        clientSecret: XXXXXXXXXXXXX
        issuer: https://accounts.google.com
      type: oidc
      id: google
      name: Google
  ...
```

values.yaml에서 수정할 때
```
configs:
    # Dex configuration
    dex.config: |
      connectors:
      - config:
          issuer: https://accounts.google.com
          clientID: XXXXXXXXXXXXX.apps.googleusercontent.com
          clientSecret: XXXXXXXXXXXXX
          issuer: https://accounts.google.com
        type: oidc
        id: google
        name: Google
```