# Cilium

이 문서는 Talos bootstrap 이후 Kubernetes 레이어에서 가장 먼저 적용하는 Cilium 설치와 검증 절차를 정리합니다.  
이후 단계인 `metrics-server`, MetalLB, ingress, 스토리지 같은 구성은 `02-kubernetes/` 폴더의 다음 문서들로 이어갑니다.

## 왜 지금 `NotReady` 인가

현재 클러스터 상태:

```text
talos-control-1   NotReady
talos-worker-1    NotReady
talos-worker-2    NotReady
```

이 상태는 지금 흐름에서는 이상 징후라기보다 예상된 결과에 가깝습니다.

`01-talos-os/configs/bootstrap-common.patch.yaml`에서 아래 설정을 이미 적용했기 때문입니다.

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

의미는 다음과 같습니다.

- Talos 기본 CNI를 설치하지 않는다.
- `kube-proxy`도 비활성화한다.
- 따라서 Cilium이 설치되기 전까지 Pod 네트워킹과 Service 처리가 완성되지 않는다.
- 그 결과 노드는 `NotReady`로 보이는 것이 정상이다.

즉 지금 단계의 첫 번째 Kubernetes 작업은 Cilium 설치입니다.

## 이 폴더의 역할

- Talos 이후 단계의 Kubernetes 설정을 Talos 문서와 분리해서 관리한다.
- add-on 별로 왜 필요한지, 어떤 값으로 설치했는지, 어디서 적용했는지를 남긴다.
- public repo 원칙에 맞게 민감값 없이 재현 가능한 절차를 유지한다.

## 현재 기준

- Cluster Name: `homelab-cluster`
- Kubernetes Version: `v1.35.0`
- Control Plane Endpoint: `https://192.168.1.10:6443`
- Pod CIDR: `10.244.0.0/16`
- Service CIDR: `10.96.0.0/12`
- Talos 쪽 선행 조건:
  - `cluster.network.cni.name: none`
  - `cluster.proxy.disabled: true`
  - `machine.kubelet.extraArgs.rotate-server-certificates: "true"`

## 추천 진행 순서

1. `kubectl`과 `helm`이 현재 클러스터를 볼 수 있는지 확인한다.
2. `kubelet-serving-cert-approver`를 먼저 설치한다.
3. Cilium을 kube-proxy replacement 모드로 설치한다.
4. 노드가 `Ready`로 바뀌는지 확인한다.
5. Cilium 상태를 검증한다.
6. 그 다음에 `metrics-server`, MetalLB, ingress controller 순서로 확장한다.

## 사전 준비

Talos 단계에서 kubeconfig를 이미 받아왔다고 가정합니다.

예시:

```bash
export KUBECONFIG=/tmp/homelab-kubeconfig
kubectl get nodes -o wide
```

`helm`이 없다면 macOS 기준으로 설치:

```bash
brew install helm
```

설치 확인:

```bash
helm version
kubectl cluster-info
```

## 1. `kubelet-serving-cert-approver`를 먼저 두는 이유

Talos 쪽에서는 이미 아래 설정으로 kubelet serving certificate rotation을 켜 두었습니다.

```yaml
machine:
  kubelet:
    extraArgs:
      rotate-server-certificates: "true"
```

하지만 이 설정은 "kubelet이 serving CSR을 만들 수 있게 준비하는 단계"까지입니다.  
실제 `kubernetes.io/kubelet-serving` CSR이 `Approved,Issued` 되려면 approver가 필요합니다.

이걸 Cilium 뒤에 미루면 아래 확인 작업에서 바로 막힐 수 있습니다.

- `kubectl exec`
- `kubectl logs`
- `cilium-dbg status`
- 이후 `metrics-server`

즉 Cilium 자체는 떠도, 운영 검증에 꼭 필요한 kubelet API 경로가 TLS 에러로 막힐 수 있습니다.  
그래서 이 저장소에서는 `kubelet-serving-cert-approver`를 Cilium보다 먼저 두는 흐름이 더 자연스럽습니다.

먼저 CSR 상태를 확인합니다.

```bash
kubectl get csr
```

여기서 `kubernetes.io/kubelet-serving` signer의 CSR이 `Pending`이라면 approver를 먼저 설치해야 합니다.

설치:

```bash
kubectl apply -f https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
```

설치 후 확인:

```bash
kubectl get csr
```

기대 상태:

- kubelet serving CSR이 `Approved,Issued`
- 이후 `kubectl exec`, `kubectl logs`, `metrics-server`가 정상 동작

## 2. Cilium부터 시작하는 이유

이 저장소는 처음부터 Cilium을 전제로 Talos 설정을 준비했습니다.

- Talos 기본 CNI를 쓰지 않음
- `kube-proxy`를 꺼 둠
- 이후 Service 처리까지 Cilium이 맡도록 구성

이렇게 해두면 네트워크 관련 책임이 Cilium으로 모이고, 나중에 Gateway API, Hubble, eBPF 기반 기능을 확장할 때도 흐름이 자연스럽습니다.

학습용 홈랩 관점에서는 "기본 CNI 하나, 그 위에 나중에 또 바꾸기"보다 처음부터 방향을 정해 두는 편이 더 단순합니다.

## 3. Cilium 설치

이 저장소의 현재 방향은 "Talos는 Talos대로 유지하고, Kubernetes add-on은 bootstrap 이후 Helm으로 명시적으로 설치"입니다.

이번 첫 설치는 control plane VIP `192.168.1.10:6443`를 Kubernetes API endpoint로 사용합니다.  
Talos 문서에는 KubePrism 기반 예시도 있지만, 현재 저장소 기준에서는 먼저 API VIP를 직접 지정하는 쪽이 이해하기 쉽고 의존성이 적습니다.

먼저 Helm repo를 추가합니다.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

그 다음 Cilium을 설치합니다.

```bash
helm upgrade --install cilium cilium/cilium \
  --version 1.19.2 \
  --namespace kube-system \
  --create-namespace \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.1.10 \
  --set k8sServicePort=6443 \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}"
```

이 값들의 핵심 의미:

- `ipam.mode=kubernetes`
  - Pod IP 관리를 Kubernetes 방식으로 맞춘다.
- `kubeProxyReplacement=true`
  - Talos에서 비활성화한 `kube-proxy` 역할을 Cilium이 대체한다.
- `k8sServiceHost=192.168.1.10`
  - Cilium이 kube-proxy 없이도 API server에 바로 붙을 수 있게 한다.
- `cgroup.autoMount.enabled=false`
  - Talos가 이미 제공하는 cgroup 마운트를 사용한다.
- `securityContext.capabilities.*`
  - Talos 환경에서 Cilium이 필요한 capability만 명시한다.

## 4. 설치 직후 확인

먼저 rollout 상태를 봅니다.

```bash
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system rollout status deploy/cilium-operator
```

그 다음 전체 파드를 확인합니다.

```bash
kubectl get pods -A
```

노드 상태가 바뀌는지 확인합니다.

```bash
kubectl get nodes -o wide
```

정상적으로 올라오면 기대 결과는 대략 아래 방향입니다.

- `talos-control-1`, `talos-worker-1`, `talos-worker-2`가 모두 `Ready`
- `kube-system` 네임스페이스에 `cilium` DaemonSet 파드가 각 노드에 배치
- `cilium-operator` Deployment가 Running

## 5. Cilium 동작 검증

설치가 끝났으면 kube-proxy replacement 상태를 한 번 확인합니다.

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep KubeProxyReplacement
```

좀 더 자세히 보고 싶다면:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose
```

노드가 `Ready`가 되지 않으면 아래 순서로 보는 것이 좋습니다.

```bash
kubectl -n kube-system get pods -o wide
kubectl -n kube-system logs ds/cilium
kubectl -n kube-system logs deploy/cilium-operator
kubectl describe node talos-control-1
```

## 6. 첫 설치 이후 메모

- 첫 설치 단계에서는 Hubble, Gateway API, BGP 같은 옵션을 바로 켜지 않고 기본 동작부터 확인하는 편이 안전합니다.
- Cilium이 안정화된 뒤에만 MetalLB, ingress, metrics-server 같은 다음 단계를 진행합니다.
- 이후 값을 더 정교하게 관리하고 싶다면 Helm CLI `--set` 나열 대신 별도 values 파일로 옮기는 것이 좋습니다.

## 7. `kubectl exec` 또는 `logs` 가 TLS 에러를 내는 경우

예를 들어 아래와 같은 에러가 날 수 있습니다.

```text
error sending request: Post "https://10.123.0.11:10250/exec/...": remote error: tls: internal error
```

이 증상은 Cilium 자체 장애라기보다, kube-apiserver가 각 노드의 kubelet API (`:10250`)에 붙을 때 사용할 kubelet serving certificate가 아직 제대로 승인되지 않았을 때 자주 보입니다.

현재 저장소는 Talos 쪽에서 아래 설정을 이미 켜 둔 상태입니다.

```yaml
machine:
  kubelet:
    extraArgs:
      rotate-server-certificates: "true"
```

하지만 이것만으로는 충분하지 않습니다.  
Talos 공식 가이드 기준으로는 `kubernetes.io/kubelet-serving` CSR을 자동 승인해 줄 `kubelet-serving-cert-approver`를 함께 두는 것이 자연스럽습니다.

먼저 CSR 상태를 확인합니다.

```bash
kubectl get csr
```

여기서 `kubernetes.io/kubelet-serving` signer의 CSR이 `Pending`으로 보이면 원인을 거의 확인한 셈입니다.

### 해결 방법

`kubelet-serving-cert-approver`를 설치합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
```

설치 후 다시 CSR을 확인합니다.

```bash
kubectl get csr
```

기대 상태:

- kubelet serving CSR이 `Approved,Issued`
- 이후 `kubectl exec`, `kubectl logs`, `metrics-server`가 정상 동작

그 다음 다시 확인:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose
```

참고로 이 이슈는 나중에 `metrics-server`를 설치할 때도 같은 뿌리 원인으로 이어질 수 있으므로, Cilium 직후에 해결해 두는 편이 좋습니다.

## 다음에 이어서 할 작업

현재 저장소 기준으로 다음 우선순위는 아래 순서가 자연스럽습니다.

1. `kubelet-serving-cert-approver`
2. `metrics-server`
3. MetalLB
4. ingress controller
5. `cert-manager`
6. 스토리지 연동

특히 `metrics-server`는 Talos 쪽에서 이미 kubelet certificate rotation을 켜 두었기 때문에, Cilium이 정상화된 뒤 바로 이어서 문서화하기 좋은 다음 단계입니다.

## 참고 문서

- Talos bootstrap 문서: [`../01-talos-os/README.md`](../01-talos-os/README.md)
- Cilium 공식 kube-proxy-free 가이드: <https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/>
- Talos 공식 Cilium 가이드: <https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium>
- Talos 공식 Metrics Server 가이드: <https://docs.siderolabs.com/kubernetes-guides/monitoring-and-observability/deploy-metrics-server>
