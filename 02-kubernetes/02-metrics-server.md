# Metrics Server

이 문서는 Cilium 설치 이후 `metrics-server`를 올리는 절차를 정리합니다.  
목표는 `kubectl top`이 동작하도록 만들고, 이후 HPA나 간단한 리소스 모니터링의 기반을 준비하는 것입니다.

## 왜 지금 설치하는가

`metrics-server`는 지금 단계에서 가장 자연스러운 다음 작업입니다.

- Cilium이 올라와 노드와 기본 네트워크가 안정화되었다.
- Talos 쪽에서 이미 kubelet serving certificate rotation을 켜 두었다.
- `kubectl top nodes`, `kubectl top pods`로 클러스터 상태를 바로 확인할 수 있다.
- 나중에 HPA를 쓰려면 `metrics-server`가 필요하다.

## 사전 조건

이 문서는 아래 상태를 전제로 합니다.

- 모든 노드가 `Ready`
- Cilium이 정상 구동 중
- `kubelet-serving-cert-approver`가 설치되어 kubelet serving CSR이 `Approved,Issued`

먼저 확인:

```bash
kubectl get nodes
kubectl get csr
kubectl -n kube-system get pods -o wide
```

특히 `kubectl get csr`에서 `kubernetes.io/kubelet-serving` signer의 요청이 계속 `Pending`이면 먼저 [01-cilium.md](./01-cilium.md)의 approver 섹션을 다시 확인합니다.

## 설치 방법

가장 단순한 시작점은 공식 배포 manifest를 그대로 적용하는 것입니다.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

설치 후 파드가 뜨는지 확인합니다.

```bash
kubectl -n kube-system get deployment metrics-server
kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide
```

## 설치 직후 확인

먼저 rollout 상태를 봅니다.

```bash
kubectl -n kube-system rollout status deployment/metrics-server
```

그 다음 APIService 등록 상태를 확인합니다.

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
```

기대 상태:

- `AVAILABLE`가 `True`
- `metrics-server` 파드가 `Running`

## 실제 동작 확인

이제 `kubectl top`이 되는지 확인합니다.

```bash
kubectl top nodes
kubectl top pods -A
```

정상이라면 CPU와 메모리 사용량이 숫자로 출력됩니다.

## 잘 안 될 때 먼저 볼 것

### 1. APIService가 `False` 인 경우

```bash
kubectl describe apiservice v1beta1.metrics.k8s.io
kubectl -n kube-system logs deployment/metrics-server
```

### 2. kubelet TLS 관련 에러가 보이는 경우

`metrics-server`는 각 노드의 kubelet에 접속해 메트릭을 긁어옵니다.  
따라서 kubelet serving certificate가 제대로 승인되지 않았으면 여기서도 막힐 수 있습니다.

먼저 다시 확인:

```bash
kubectl get csr
```

그리고 `kubelet-serving-cert-approver`가 떠 있는지 확인:

```bash
kubectl get pods -A | grep approver
```

### 3. 아직 메트릭이 안 보이는 경우

설치 직후에는 수십 초 정도 시간이 걸릴 수 있습니다.

잠깐 기다렸다가 다시 확인:

```bash
kubectl top nodes
kubectl top pods -A
```

## 현재 단계의 판단

이 저장소에서는 우선 아래 기준이면 성공으로 봅니다.

- `deployment/metrics-server` rollout 완료
- `apiservice/v1beta1.metrics.k8s.io`가 `Available=True`
- `kubectl top nodes`가 정상 동작

이 단계가 끝나면 다음 후보는 보통 MetalLB입니다.

## 다음에 할 작업

1. MetalLB
2. ingress controller
3. `cert-manager`
4. 스토리지 연동

## 참고 문서

- Cilium 문서: [`./01-cilium.md`](./01-cilium.md)
- Talos bootstrap 문서: [`../01-talos-os/README.md`](../01-talos-os/README.md)
- Talos 공식 Metrics Server 가이드: <https://docs.siderolabs.com/kubernetes-guides/monitoring-and-observability/deploy-metrics-server>
- Metrics Server 공식 릴리스: <https://github.com/kubernetes-sigs/metrics-server/releases>
