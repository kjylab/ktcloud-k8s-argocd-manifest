# ktcloud-k8s-argocd-manifest

ArgoCD **App of Apps** 패턴으로 클러스터 전체 애플리케이션을 선언적으로 관리하는 GitOps 레포.

## 개념

### App of Apps 패턴
ArgoCD Application이 다른 ArgoCD Application들을 관리하는 계층 구조.

```
root-app (Setup/)
├── addons-app (ApplicationSet) → Postgresql, Redis, Kafka, EbsCsi, Prometheus, Loki, Tempo
├── charts-app (ApplicationSet) → Traefik
└── apps-app  (ApplicationSet) → msa-market (각 마이크로서비스)
```

`root-app` 하나만 클러스터에 등록하면 나머지 모든 앱이 자동으로 배포된다.

### ApplicationSet
동일한 템플릿으로 여러 Application을 한 번에 생성하는 ArgoCD 리소스. `elements` 리스트에 항목을 추가하면 앱이 자동 생성된다.

### Sync 정책
- `automated.prune: true` — Git에서 삭제된 리소스는 클러스터에서도 삭제
- `automated.selfHeal: true` — 클러스터 상태가 Git과 달라지면 자동 복원
- `ServerSideApply: true` — 대용량 CRD(kube-prometheus-stack 등) 적용 시 필요

## 디렉토리 구조

```
bootstrap/
  root-app.yaml          # 클러스터에 최초 1회 수동 등록하는 진입점

Setup/                   # root-app이 감시하는 디렉토리
  Addons-app.yaml        # 인프라 애드온 ApplicationSet
  Charts-app.yaml        # Helm 차트 ApplicationSet
  Apps-app.yaml          # 마이크로서비스 ApplicationSet

Addons/                  # 인프라 애드온 (각각 Helm wrapper 차트)
  Postgresql/
  Redis/
  Kafka/
  EbsCsi/
  Prometheus/            # kube-prometheus-stack (Prometheus + Grafana)
  Loki/                  # 로그 수집 (Loki + Promtail)
  Tempo/                 # 분산 트레이싱

Charts/
  Traefik/               # Ingress Controller + NLB

Apps/
  msa-market/            # 마이크로서비스 ApplicationSet
```

## 초기 설치 (1회)

```bash
kubectl apply -f bootstrap/root-app.yaml
```

이후 모든 변경은 Git push만으로 적용된다.

## 새 애드온 추가 방법

1. `Addons/<이름>/Chart.yaml` 생성 (Helm wrapper 차트)
2. `Addons/<이름>/values.yaml` 생성
3. `Setup/Addons-app.yaml`의 `elements`에 항목 추가
4. Git push → ArgoCD 자동 sync

## 주요 접속 정보

| 서비스 | 주소 |
|--------|------|
| ArgoCD UI | `http://<NLB>:30090` |
| Grafana | `http://<NLB>/grafana` (ID: admin / PW: admin1234) |

## 관련 레포

| 레포 | 역할 |
|------|------|
| [my-market-msa-manifest](https://github.com/kjylab/my-market-msa-manifest) | 마이크로서비스 공통 Helm 차트 |
| [my-msa-manifest-values](https://github.com/kjylab/my-msa-manifest-values) | 서비스별 Helm values |
