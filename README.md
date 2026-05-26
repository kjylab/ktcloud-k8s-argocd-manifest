# ktcloud-k8s-argocd-manifest

ArgoCD **App of Apps** 패턴으로 클러스터 전체 애플리케이션을 선언적으로 관리하는 GitOps 레포.

## 인프라 구성

- **클라우드**: AWS (EC2 기반 자체 구축)
- **프로비저닝**: Terraform (S3 backend: `troica-tfstate-troica-2026-jyupk`) + Ansible
- **Kubernetes**: kubeadm 부트스트랩
- **Ingress**: Traefik + NLB
- **CNI**: Calico

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
    templates/
      kafka-exporter.yaml   # Consumer lag 모니터링 (danielqsj/kafka-exporter)
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

| 서비스 | 접속 방법 |
|--------|----------|
| ArgoCD UI | SSH 터널 후 `http://localhost:8080` |
| Grafana | `http://<NLB>/grafana` (admin / admin1234) |
| Prometheus | `kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090` |

### ArgoCD SSH 터널

ArgoCD는 NodePort(30090)로만 노출, NLB 미연결. 아래 터널로 접근:

```bash
ssh -L 8080:10.0.4.236:30090 -i ~/.ssh/ktcloud-bastion-node-key ec2-user@16.184.35.141 -N
```

## 관측성 스택

| 구성요소 | 역할 | 네임스페이스 |
|----------|------|------------|
| Prometheus | 메트릭 수집 | monitoring |
| Grafana | 대시보드 | monitoring |
| Loki + Promtail | 로그 수집 | monitoring |
| Tempo | 분산 트레이싱 | monitoring |
| kafka-exporter | Kafka consumer lag | kafka |

### Loki ↔ Tempo 연동
- Loki derivedFields: 로그의 traceId를 추출해 Tempo 링크 생성
- Tempo tracesToLogsV2: traceId로 연관 로그 조회

## 주요 설정 특이사항

- Traefik `publishedService.enabled: true` 필수 (없으면 Ingress ADDRESS 없어서 ArgoCD health check 실패)
- kube-prometheus-stack 같은 대용량 CRD는 `ServerSideApply=true` 필요
- kafka-exporter: bitnami kafka chart v29에는 내장 metrics 미지원 → `Addons/Kafka/templates/`에 직접 추가

## 관련 레포

| 레포 | 역할 |
|------|------|
| [my-market-msa-manifest](https://github.com/kjylab/my-market-msa-manifest) | 마이크로서비스 공통 Helm 차트 |
| [my-msa-manifest-values](https://github.com/kjylab/my-msa-manifest-values) | 서비스별 Helm values |
| [msa-provisioning](https://github.com/kjylab/msa-provisioning) | Terraform + Ansible 인프라 프로비저닝 |
