# MetalLB

이 문서는 `metrics-server` 이후 단계로 MetalLB를 설치하고, 홈랩 LAN에서 `LoadBalancer` 타입 Service에 외부 IP를 붙이는 절차를 정리합니다.

## 왜 지금 설치하는가

이 저장소는 베어메탈 1대 위에 Talos VM 3대로 구성한 홈랩 클러스터입니다.  
이 환경에서는 클라우드 제공 LoadBalancer가 없기 때문에, `Service type=LoadBalancer`를 만들어도 외부 IP가 자동으로 생기지 않습니다.

MetalLB를 올리면 다음이 가능해집니다.

- `LoadBalancer` 서비스에 홈 네트워크 IP 할당
- 이후 ingress controller 외부 노출
- NAS / 홈 네트워크 장비에서 클러스터 서비스 접근 단순화

## 왜 Layer2 모드인가

현재 홈랩 기준에서는 Layer2 모드가 가장 자연스럽습니다.

- 같은 L2 네트워크(`192.168.1.0/24`) 안에서 동작
- 별도 BGP 라우터 연동이 필요 없음
- 학습용 단일 물리 호스트 환경에 맞게 구성이 단순함

MetalLB Layer2는 ARP로 VIP를 광고하므로, 홈 네트워크에서 가장 이해하기 쉬운 출발점입니다.

## 사전 조건

이 문서는 아래 상태를 전제로 합니다.

- 모든 노드가 `Ready`
- Cilium 정상 구동
- `metrics-server` 정상 동작
- 홈 네트워크에서 비어 있는 LoadBalancer용 IP 범위를 하나 정해 둠

먼저 확인:

```bash
kubectl get nodes
kubectl top nodes
```

## IP 대역 계획

이 저장소의 현재 토폴로지 기준 예시 외부 IP는 `192.168.1.2`입니다.

즉 가장 단순한 시작은 아래 둘 중 하나입니다.

1. 단일 IP로 시작: `192.168.1.2/32` 개념으로 `192.168.1.2-192.168.1.2`
2. 작은 풀로 시작: `192.168.1.2-192.168.1.5`

학습용 홈랩이라면 처음에는 단일 IP 또는 2~4개 정도의 작은 풀로 시작하는 편이 안전합니다.

중요한 점:

- 이 범위는 DHCP 자동 할당 범위와 겹치지 않아야 합니다.
- 라우터, NAS, 하이퍼바이저, VM 고정 IP와도 겹치면 안 됩니다.
- 나중에 ingress, 테스트용 LoadBalancer 서비스를 더 만들 계획이면 작은 풀로 시작하는 편이 편합니다.

이 문서에서는 예시로 `192.168.1.2-192.168.1.5`를 사용합니다.  
실제 환경에서 이미 `192.168.1.2`를 다른 장비가 쓰고 있다면 반드시 다른 범위로 바꿔 적용합니다.

## 설치 방법

MetalLB 공식 manifest를 적용합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

설치 후 파드를 확인합니다.

```bash
kubectl -n metallb-system get pods -o wide
```

기대 상태:

- `controller` Deployment가 `Running`
- `speaker` DaemonSet이 각 노드에 배치

참고:

- MetalLB 공식 설치 문서는 `kube-proxy` IPVS 모드일 때 `strictARP`가 필요할 수 있다고 설명합니다.
- 현재 저장소는 Talos 단계에서 `kube-proxy`를 비활성화하고 Cilium이 이를 대체하므로, 그 조치는 현재 흐름에서는 적용 대상이 아닙니다.

## IPAddressPool 과 L2Advertisement 설정

MetalLB는 설치만으로는 동작하지 않고, 어떤 IP를 배정할지와 어떻게 광고할지를 별도로 설정해야 합니다.

여기서 특히 중요한 것은 `L2Advertisement`입니다.

- `IPAddressPool`
  - 어떤 VIP 범위를 쓸지 결정
- `L2Advertisement`
  - 그 VIP를 어떤 방식과 어떤 인터페이스로 광고할지 결정

홈랩처럼 VM이 여러 NIC를 가지는 환경에서는 `L2Advertisement.interfaces`를 명시하는 편이 훨씬 안전합니다.  
이 저장소의 Talos 노드는 아래처럼 NIC 역할을 분리하고 있습니다.

- `enp1s0`: 관리망 `192.168.1.0/24`
- `enp2s0`: 내부망 `10.123.0.0/24`

MetalLB Layer2 VIP는 홈 LAN에서 ARP로 보여야 하므로, 일반적으로 `192.168.1.x`가 붙은 관리망 NIC에서 광고되어야 합니다.  
즉 현재 저장소 기준 예시는 `enp1s0`을 명시하는 쪽이 자연스럽습니다.

### 왜 `L2Advertisement`를 명시적으로 적는가

이 문서에서는 `IPAddressPool`만 만들지 않고 `L2Advertisement`를 함께 명시합니다.

- 어떤 IP를 배정할지와 어떤 인터페이스로 광고할지를 분리해서 읽을 수 있음
- 멀티 NIC VM 환경에서 실수를 줄일 수 있음
- 나중에 IP 풀이나 광고 정책이 바뀌어도 diff가 명확함

특히 이 저장소처럼 다음 조건이 같이 있는 경우에는 `L2Advertisement`를 생략하지 않는 편이 좋습니다.

- Talos 노드에 관리망 NIC와 내부망 NIC가 분리되어 있음
- Unraid/libvirt bridge 위에서 VM이 동작함
- 홈 네트워크에서 ARP 응답이 실제로 보여야 `LoadBalancer`가 열림

즉 `IPAddressPool`은 "무슨 IP를 쓸지", `L2Advertisement`는 "그 IP를 어느 인터페이스로 광고할지"를 문서화한다고 생각하면 이해하기 쉽습니다.

예시 manifest:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.2-192.168.1.5
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - homelab-pool
  interfaces:
    - enp1s0
```

임시로 바로 적용하려면:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.2-192.168.1.5
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - homelab-pool
  interfaces:
    - enp1s0
EOF
```

중요:

- `L2Advertisement`에 `spec`이 아예 없으면 구성은 생성돼도 실제 광고 동작이 기대와 다를 수 있습니다.
- `interfaces`를 지정하지 않으면 멀티 NIC 환경에서 어느 네트워크로 VIP를 광고할지 모호해질 수 있습니다.
- 실제 NIC 이름은 반드시 노드에서 확인하는 편이 안전합니다. 문서상 예시는 `enp1s0`이지만, 환경에 따라 `eth0`, `ens18` 등으로 다를 수 있습니다.
- VIP는 홈 LAN에 보여야 하므로, `10.123.0.0/24` 내부망 NIC가 아니라 `192.168.1.0/24` 관리망 NIC를 지정해야 합니다.

실제 NIC 이름 확인 예시:

```bash
talosctl -n 192.168.1.11 get links
talosctl -n 192.168.1.21 get links
talosctl -n 192.168.1.22 get links
```

확인:

```bash
kubectl -n metallb-system get ipaddresspools
kubectl -n metallb-system get l2advertisements
kubectl -n metallb-system get l2advertisement homelab-l2 -o yaml
```

기대 상태:

- `IPAddressPool homelab-pool` 존재
- `L2Advertisement homelab-l2` 존재
- `spec.ipAddressPools`에 `homelab-pool` 포함
- `spec.interfaces`에 관리망 NIC 포함

추가로 보면 좋은 것:

```bash
kubectl -n metallb-system get all
kubectl -n metallb-system get pods -o wide
```

여기서 `speaker`가 각 노드에 배치되어 있어야 Layer2 광고가 자연스럽게 동작합니다.

## 동작 확인

가장 쉬운 검증은 `LoadBalancer` 타입 테스트 서비스를 하나 만드는 것입니다.

예시:

```bash
kubectl create deployment nginx-lb-test --image=nginx
kubectl expose deployment nginx-lb-test \
  --port=80 \
  --target-port=80 \
  --type=LoadBalancer
```

외부 IP가 붙는지 확인:

```bash
kubectl get svc nginx-lb-test
kubectl get svc -A
kubectl describe svc nginx-lb-test
```

기대 상태:

- `EXTERNAL-IP`에 `192.168.1.2`~`192.168.1.5` 중 하나가 할당
- `kubectl describe svc nginx-lb-test` 이벤트에 `IPAllocated`가 보임
- 정상이라면 `nodeAssigned` 이벤트도 함께 보일 수 있음

광고 노드까지 확인하려면:

```bash
kubectl -n metallb-system get servicel2statuses -o wide
```

기대 상태:

- `nginx-lb-test` 같은 서비스 이름이 보임
- 어떤 노드가 해당 VIP를 Layer2로 광고하는지 확인 가능
- 문제가 생겼을 때 "IP는 배정됐지만 실제 광고는 안 됨" 상태를 구분하는 데 도움이 됨

홈 네트워크 다른 장비나 로컬 PC에서 확인:

```bash
curl http://<EXTERNAL-IP>
```

응답이 오면 MetalLB Layer2 구성이 기본적으로 성공한 것입니다.

테스트가 끝나면 정리:

```bash
kubectl delete svc nginx-lb-test
kubectl delete deployment nginx-lb-test
```

## 잘 안 될 때 먼저 볼 것

### 1. EXTERNAL-IP 가 `pending` 인 경우

```bash
kubectl -n metallb-system get pods
kubectl -n metallb-system get ipaddresspools
kubectl -n metallb-system get l2advertisements
kubectl -n metallb-system get l2advertisement homelab-l2 -o yaml
kubectl describe svc nginx-lb-test
```

주로 보는 포인트:

- IP pool이 실제로 생성됐는지
- pool 주소가 다른 장비와 충돌하지 않는지
- speaker 파드가 각 노드에서 정상인지
- `L2Advertisement.spec.ipAddressPools`가 pool 이름과 맞는지
- `L2Advertisement.spec.interfaces`가 실제 관리망 NIC 이름과 맞는지

### 2. IP 는 붙었는데 접속이 안 되는 경우

```bash
kubectl get endpoints nginx-lb-test
kubectl get pods -o wide
kubectl -n metallb-system logs daemonset/speaker
kubectl -n metallb-system get servicel2statuses -o wide
```

주로 보는 포인트:

- 실제 backend pod가 Ready인지
- ARP 응답이 홈 네트워크에서 정상 반영되는지
- 클라이언트 쪽 ARP 캐시가 오래 남아 있지 않은지
- MetalLB가 실제로 어떤 서비스를 어떤 노드에서 광고 중인지
- `servicel2statuses`에 대상 서비스가 보이는지

추가 확인 예시:

클라이언트 또는 라우터 쪽에서:

```bash
arp -an | grep <EXTERNAL-IP>
ping -c 3 <EXTERNAL-IP>
```

OpenWRT 같은 라우터가 있다면:

```bash
ip neigh | grep <EXTERNAL-IP>
```

기대 상태:

- VIP에 대한 MAC 주소가 학습됨
- `FAILED`가 아니라 REACHABLE/STALE 같은 상태로 보임

참고:

- ping 자체가 응답하지 않아도, 먼저 ARP가 학습되는지 보는 것이 더 중요합니다.
- 홈랩에서는 HTTP 문제처럼 보여도 실제 원인은 ARP 광고 누락인 경우가 자주 있습니다.

### 3. 일반 LoadBalancer 서비스는 되는데 특정 서비스만 안 되는 경우

이 경우는 MetalLB 전체 문제라기보다, 해당 서비스가 MetalLB 입장에서 "광고 가능한 서비스"로 보이지 않는 문제일 수 있습니다.

먼저 비교:

```bash
kubectl -n metallb-system get servicel2statuses -o wide
kubectl get endpointslice -A
```

특히 CNI나 ingress controller가 자동 생성한 `LoadBalancer` 서비스는 일반 앱 서비스와 구조가 다를 수 있으므로, 일반 `LoadBalancer` 테스트 서비스와 비교하는 것이 도움이 됩니다.

예를 들어:

- 일반 테스트 서비스는 `servicel2statuses`에 나타나고 접속도 성공
- 특정 컨트롤러가 만든 서비스는 `EXTERNAL-IP`는 있어도 `servicel2statuses`에 안 보이거나 실제 ARP 광고가 안 될 수 있음

이 경우에는 MetalLB 전체가 망가졌다기보다, 해당 서비스가 MetalLB가 기대하는 형태와 다를 가능성을 먼저 의심하는 편이 좋습니다.

### 4. 멀티 NIC / 하이퍼바이저 환경에서 계속 안 되는 경우

홈랩에서 Unraid, libvirt, bridge 환경을 쓰면 Kubernetes 리소스만 보고는 원인을 놓치기 쉽습니다.

먼저 확인:

```bash
talosctl -n 192.168.1.11 get links
kubectl -n metallb-system get l2advertisement homelab-l2 -o yaml
kubectl -n metallb-system get servicel2statuses -o wide
```

주로 보는 포인트:

- `interfaces`에 지정한 NIC 이름이 실제 노드 NIC와 일치하는지
- VIP를 광고하는 노드가 실제로 잡혔는지
- 일반 `LoadBalancer` 서비스는 되는데 특정 서비스만 안 되는지

가능하면 라우터나 하이퍼바이저에서도 같이 봅니다.

- 라우터의 `ip neigh`
- 하이퍼바이저 bridge의 `tcpdump`

예를 들어 `tcpdump`에서 `who-has <VIP>` 요청만 보이고 응답이 전혀 없으면, 애플리케이션이 아니라 Layer2 광고 경로 문제로 좁힐 수 있습니다.

## 현재 단계의 판단

이 저장소에서는 아래 기준이면 성공으로 봅니다.

- `metallb-system/controller` 정상
- `speaker` DaemonSet 정상
- `IPAddressPool`, `L2Advertisement` 생성 완료
- `L2Advertisement`에 올바른 pool / interface 지정 완료
- 테스트 `LoadBalancer` 서비스에 LAN IP 할당 성공
- 해당 IP로 실제 접속 가능

## 다음에 할 작업

1. ingress controller
2. `cert-manager`
3. 스토리지 연동

## 참고 문서

- Cilium 문서: [`./01-cilium.md`](./01-cilium.md)
- Metrics Server 문서: [`./02-metrics-server.md`](./02-metrics-server.md)
- Talos bootstrap 문서: [`../01-talos-os/README.md`](../01-talos-os/README.md)
- MetalLB 공식 설치 문서: <https://metallb.io/installation/>
- MetalLB 공식 설정 문서: <https://metallb.io/configuration/>
- MetalLB Layer2 개념 문서: <https://metallb.io/concepts/layer2/>
