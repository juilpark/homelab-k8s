# Talos OS Bootstrap

이 폴더는 홈랩 Kubernetes 클러스터를 Talos OS로 부트스트랩하기 위한 설정과 작업 메모를 정리하는 공간입니다.  
범위는 Talos 이미지 준비, machine config patch, `apply-config`, `bootstrap`, kubeconfig 추출까지의 초기 구성입니다.
클러스터가 올라온 뒤 설치하는 Kubernetes add-on은 별도 폴더에서 관리하는 것을 기본 방향으로 둡니다.

## 목표

- Talos OS 기반으로 Kubernetes 클러스터 초기 구성을 준비한다.
- 생성 파일과 수기 작성 patch 파일의 역할을 분리한다.
- 공통 bootstrap patch와 노드별 patch의 책임을 분리한다.
- Cilium kube-proxy-free 구성을 위한 Talos 쪽 선행 설정을 문서화한다.
- 나중에 다시 봐도 같은 흐름으로 재현할 수 있도록 명령과 의도를 함께 기록한다.

## 전제

- Cluster Name: `homelab-cluster`
- Control Plane Endpoint: `https://192.168.1.10:6443`
- 설치 디스크: `/dev/vda`
- 노드 구성:
  - `talos-control-1`
  - `talos-worker-1`
  - `talos-worker-2`

네트워크 기준:

- 관리망 / 외부 접근망: `192.168.1.0/24`
- 노드 내부망: `10.123.0.0/24`
- Pod CIDR: `10.244.0.0/16`
- Service CIDR: `10.96.0.0/12`

네트워크와 전체 배치는 루트 README 및 [`../00-topology/`](../00-topology) 자료를 기준으로 맞춥니다.

## 현재 폴더 구조

```text
01-talos-os
|-- README.md
`-- configs
    |-- bootstrap-common.patch.yaml
    |-- controlplane-1.patch.yaml
    |-- worker-1.patch.yaml
    |-- worker-2.patch.yaml
    |-- controlplane.yaml        # generated, git ignored
    |-- worker.yaml              # generated, git ignored
    `-- talosconfig              # generated, git ignored
```

## 사전 준비

### 1. `talosctl` 설치

macOS 기준:

```bash
brew install talosctl
```

필요 시 설치 확인:

```bash
talosctl version
```

## Talos 기본 설정 생성

현재 사용 중인 생성 명령은 아래와 같습니다.

```bash
talosctl gen config homelab-cluster https://192.168.1.10:6443 \
  --install-disk /dev/vda \
  -o configs/
```

이 명령으로 생성되는 대표 파일:

- `configs/talosconfig`
- `configs/controlplane.yaml`
- `configs/worker.yaml`

이 파일들은 Talos 기본 골격으로 보고, 저장소에는 가능한 한 민감 정보가 포함된 생성 결과를 직접 커밋하지 않습니다.  
Public Repo 운영 원칙에 따라 실제 `talosconfig`, kubeconfig, 인증서, 키 파일은 `.gitignore`로 제외합니다.

## Patch 파일 운영 방식

현재 방향은 "공통 생성 파일 + 공통 bootstrap patch + 노드별 patch" 방식입니다.

- 생성 파일:
  - Talos가 기본으로 제공하는 machine 설정의 출발점
- 공통 bootstrap patch:
  - 모든 노드에 공통으로 적용할 Talos/Kubernetes 기본값
  - Cilium kube-proxy-free 선행 설정
  - metrics-server 호환을 위한 kubelet certificate rotation 같은 공통 옵션
- patch 파일:
  - 각 VM의 hostname
  - 고정 IP 또는 네트워크 인터페이스 설정
  - node role별 차이
  - VIP, nameserver, gateway, install 관련 개별 조정값

현재 확인된 patch 파일:

- `configs/bootstrap-common.patch.yaml`
- `configs/controlplane-1.patch.yaml`
- `configs/worker-1.patch.yaml`
- `configs/worker-2.patch.yaml`

현재 공통 bootstrap patch에 포함한 항목:

- `cluster.network.cni.name: none`
  - Talos 기본 CNI를 배포하지 않고, 부트스트랩 후 Cilium을 설치하기 위한 설정
- `cluster.proxy.disabled: true`
  - `kube-proxy`를 비활성화해서 Cilium이 Service 처리 기능을 대체하도록 준비
- `machine.kubelet.extraArgs.rotate-server-certificates: "true"`
  - 이후 `metrics-server`, `kubectl exec`, `kubectl logs`가 kubelet serving certificate를 정상적으로 사용할 수 있게 하기 위한 선행 설정
  - 이 설정만으로 serving certificate 승인이 끝나는 것은 아니며, bootstrap 이후 Kubernetes 단계에서 `kubelet-serving-cert-approver` 같은 approver를 함께 준비해야 한다

## 현재 진행 상태

- Talos 기본 config 생성 완료
- 공통 bootstrap patch 작성 완료
- `controlplane-1`, `worker-1`, `worker-2` patch 작성 완료
- `apply-config`, `bootstrap`, kubeconfig 추출을 위한 기본 작업 순서 문서화 완료
- 실제 실행 결과와 운영 중 주의사항은 클러스터 구성 후 계속 보강 예정

## 추천 작업 순서

현재 저장소 상태를 기준으로 다음 순서로 진행하는 것이 자연스럽습니다.

1. 각 VM의 네트워크/hostname/install 설정 최종 점검
2. patch를 base config에 합쳐 적용용 machine config 렌더링
3. 각 노드에 `apply-config` 수행
4. Control Plane bootstrap 수행
5. kubeconfig 추출 및 `kubectl` 연결 확인
6. 부트스트랩 완료 후 별도 Kubernetes add-on 폴더에서 Cilium, MetalLB, ingress, 스토리지 같은 후속 구성 추가

여기서 주의할 점:

- Talos 쪽에서 `rotate-server-certificates`를 켜는 것은 kubelet serving certificate 요청을 가능하게 하는 사전 준비다.
- 실제 `kubernetes.io/kubelet-serving` CSR 승인 흐름은 Kubernetes add-on 단계에서 마무리된다.
- 따라서 Cilium 설치 후에는 `kubelet-serving-cert-approver` 여부도 함께 확인하는 편이 좋다.

## 적용 전 체크리스트

실제 적용 전에 아래 항목을 먼저 확인합니다.

- Talos ISO 또는 PXE 부팅 환경이 각 VM에 준비되어 있는지
- VM NIC 매핑이 아래와 같은지
  - `enp1s0`: `br0`에 연결된 관리망
  - `enp2s0`: `talos-os-net`에 연결된 내부망
- 노드별 IP가 아래 값과 일치하는지
  - `talos-control-1`: `192.168.1.11`, `10.123.0.11`
  - `talos-worker-1`: `192.168.1.21`, `10.123.0.21`
  - `talos-worker-2`: `192.168.1.22`, `10.123.0.22`
- Control Plane VIP `192.168.1.10`을 다른 장비가 사용하고 있지 않은지
- 로컬 PC에서 각 VM의 Talos maintenance API로 붙을 수 있는지

## 적용용 Machine Config 만들기

저장소에 있는 `controlplane.yaml`, `worker.yaml`은 생성된 base config이고, 실제 적용 전에는 공통 bootstrap patch와 node patch를 순서대로 합쳐서 최종 machine config를 렌더링해서 사용하는 방식이 깔끔합니다.

예시는 `/tmp`에 렌더링 결과를 만드는 형태로 기록합니다.

```bash
talosctl machineconfig patch configs/controlplane.yaml \
  -p @configs/bootstrap-common.patch.yaml \
  -p @configs/controlplane-1.patch.yaml \
  -o /tmp/controlplane-1.yaml

talosctl machineconfig patch configs/worker.yaml \
  -p @configs/bootstrap-common.patch.yaml \
  -p @configs/worker-1.patch.yaml \
  -o /tmp/worker-1.yaml

talosctl machineconfig patch configs/worker.yaml \
  -p @configs/bootstrap-common.patch.yaml \
  -p @configs/worker-2.patch.yaml \
  -o /tmp/worker-2.yaml
```

원하면 결과를 눈으로 한 번 확인합니다.

```bash
sed -n '1,220p' /tmp/controlplane-1.yaml
sed -n '1,220p' /tmp/worker-1.yaml
sed -n '1,220p' /tmp/worker-2.yaml
```

## 최초 적용 순서

최초 설치 시점에는 Talos가 maintenance mode에 있기 때문에 `--insecure`로 machine config를 밀어 넣는 흐름을 사용합니다.

### 1. Control Plane에 config 적용

```bash
talosctl apply-config \
  --insecure \
  --nodes 192.168.1.11 \
  --file /tmp/controlplane-1.yaml
```

### 2. Worker들에 config 적용

```bash
talosctl apply-config \
  --insecure \
  --nodes 192.168.1.21 \
  --file /tmp/worker-1.yaml

talosctl apply-config \
  --insecure \
  --nodes 192.168.1.22 \
  --file /tmp/worker-2.yaml
```

config가 적용되면 노드가 재설정되면서 지정한 IP/hostname으로 올라옵니다.

## Talos 클라이언트 설정

초기 적용 이후에는 생성된 `configs/talosconfig`를 사용해서 Talos API에 인증된 방식으로 접속합니다.

먼저 현재 작업 디렉터리 기준으로 `talosconfig`를 명시하는 습관을 들이면 헷갈림이 적습니다.

```bash
export TALOSCONFIG="$(pwd)/configs/talosconfig"
```

이후 Talos client context의 endpoint와 node 목록을 맞춥니다.

초기에는 control plane 실제 IP로 접근하고, 안정화 후에는 VIP로 바꿔도 됩니다.

```bash
talosctl --talosconfig "$TALOSCONFIG" config endpoint 192.168.1.11
talosctl --talosconfig "$TALOSCONFIG" config node 192.168.1.11 192.168.1.21 192.168.1.22
```

추후 VIP가 안정적으로 동작하면 endpoint를 아래처럼 바꿀 수 있습니다.

```bash
talosctl --talosconfig "$TALOSCONFIG" config endpoint 192.168.1.10
```

## 첫 부팅 이후 진행 순서

### 1. Control Plane bootstrap

이 클러스터는 control plane이 1대이므로 `talos-control-1`에서 etcd bootstrap을 한 번 수행합니다.

```bash
talosctl --talosconfig "$TALOSCONFIG" bootstrap \
  --nodes 192.168.1.11 \
  --endpoints 192.168.1.11
```

이 작업은 최초 클러스터 생성 시 한 번만 수행합니다.

### 2. Talos 상태 확인

노드가 보이는지 확인합니다.

```bash
talosctl --talosconfig "$TALOSCONFIG" get members \
  --nodes 192.168.1.11
```

전체 건강 상태도 확인합니다.

```bash
talosctl --talosconfig "$TALOSCONFIG" health \
  --nodes 192.168.1.11 \
  --control-plane-nodes 10.123.0.11 \
  --worker-nodes 10.123.0.21 \
  --worker-nodes 10.123.0.22 \
  --endpoints 192.168.1.11
```

VIP 사용으로 전환했다면 `--endpoints 192.168.1.10`으로 바꿔도 됩니다.

주의:

- `talosctl health`에서 `--worker-nodes`나 `--control-plane-nodes`에 쉼표로 여러 IP를 묶어 넣으면 버전에 따라 에러가 날 수 있습니다.
- 가장 안전한 방식은 위 예시처럼 같은 플래그를 여러 번 반복해서 주는 것입니다.
- `talosconfig`에 기본 node가 여러 개 잡혀 있으면 `health`가 `multiple nodes` 에러를 낼 수 있습니다.
- 이 경우 위 예시처럼 `--nodes 192.168.1.11`로 실제 Talos API 요청 대상을 하나로 고정하면 정상 동작합니다.
- `--nodes`와 `--endpoints`는 Talos API에 실제로 접속할 주소입니다.
- 반면 `--control-plane-nodes`와 `--worker-nodes`는 health 체크에서 멤버십을 비교할 노드 IP인데, 현재 클러스터에서는 `10.123.0.0/24` 내부망 IP 기준으로 맞추는 것이 자연스럽습니다.

### 3. kubeconfig 가져오기

Kubernetes API가 올라오면 admin kubeconfig를 가져옵니다.

현재 저장소를 public repo로 관리하므로, kubeconfig는 저장소 안에 커밋하지 않는 경로에 두거나 임시 파일로 받는 것을 권장합니다.

예시:

```bash
talosctl --talosconfig "$TALOSCONFIG" kubeconfig /tmp/homelab-kubeconfig \
  --nodes 192.168.1.11 \
  --endpoints 192.168.1.11 \
  --force \
  --merge=false
```

로컬 셸에서 사용:

```bash
export KUBECONFIG=/tmp/homelab-kubeconfig
kubectl get nodes -o wide
kubectl get pods -A
```

### 4. 첫 확인 포인트

처음 부팅 직후에는 아래 정도를 확인하면 충분합니다.

- `kubectl get nodes`에서 control plane 1대와 worker 2대가 모두 `Ready`인지
- `kubectl get pods -A`에서 주요 시스템 파드가 정상 구동되는지
- control plane VIP `192.168.1.10`으로 API 접근이 안정적인지
- 각 노드가 기대한 호스트네임으로 올라왔는지
- 노드 IP가 관리망과 내부망으로 예상대로 잡혔는지

## 첫 사용 이후 다음 단계

기본 부트스트랩이 끝나면 보통 아래 순서로 이어집니다.

1. 별도 Kubernetes add-on 폴더에서 Cilium 설치 및 네트워크 상태 점검
2. MetalLB 설치 및 `192.168.1.2` 같은 외부 노출 IP 구성
3. ingress controller 설치
4. 사용할 도메인 / 서브도메인 구조 정리
5. Longhorn, NFS CSI, metrics-server, cert-manager 같은 후속 구성 추가

현재 저장소에서는 이 다음 단계 문서를 [`../02-kubernetes/README.md`](../02-kubernetes/README.md) 아래 단계별 문서에서 이어서 관리합니다.
특히 kubelet serving CSR 승인과 `kubelet-serving-cert-approver` 관련 내용은 [`../02-kubernetes/01-cilium.md`](../02-kubernetes/01-cilium.md)에서 이어서 다룹니다.

## 부트스트랩 이후 분리할 영역

`01-talos-os/`는 Talos와 초기 Kubernetes 제어면이 살아나는 시점까지만 다룹니다.

다음 구성은 별도 폴더에서 이어서 관리하는 것을 권장합니다.

- Cilium 설치와 운영 값
- MetalLB
- ingress controller
- cert-manager
- external-dns
- Longhorn
- NFS CSI
- metrics-server
- monitoring stack

이렇게 분리하면 Talos machine config 변경과 Kubernetes add-on 변경의 경계를 명확하게 유지할 수 있습니다.

## 주의할 점

- `bootstrap`은 최초 클러스터 생성 시 한 번만 실행합니다.
- `configs/talosconfig`, kubeconfig, 인증서/키 파일은 저장소에 커밋하지 않습니다.
- patch 파일을 고친 뒤에는 base config에 다시 patch를 합쳐서 렌더링 파일을 새로 만든 다음 적용합니다.
- 공통 설정은 `bootstrap-common.patch.yaml`에, 노드별 차이는 node patch에 유지합니다.
- 노드에 이미 config가 적용된 뒤에는 상황에 따라 `apply-config`를 `--insecure` 없이 사용하게 됩니다.

## 나중에 문서화할 내용

아래 항목은 클러스터 구성이 진행되면 실제 실행 결과를 바탕으로 구체화할 예정입니다.

- `talosctl apply-config` 실행 순서
- 최초 bootstrap 타이밍과 대상 노드
- kubeconfig 추출 방법
- 노드 상태 확인 명령
- 장애가 아닌 "재현성" 관점에서의 재설치 절차

## 보안 및 커밋 정책

- 이 저장소는 Public Repo다.
- 실제 키, 인증서, kubeconfig, secret, token은 커밋하지 않는다.
- 예제가 필요하면 실제 값 대신 sanitized template 또는 예시 문서를 남긴다.
- 생성 파일 중 민감한 항목이 포함될 수 있는 산출물은 `.gitignore`로 관리한다.

## 메모

- 이 문서는 현재 작업 메모 성격이 강하다.
- 최종적으로는 루트 README와 별도로, 처음 Talos를 구성하는 사람도 따라갈 수 있는 단계별 가이드로 발전시키는 것을 목표로 한다.
