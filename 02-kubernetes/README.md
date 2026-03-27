# Kubernetes Add-ons

이 폴더는 Talos bootstrap 이후 Kubernetes 레이어에서 적용하는 add-on과 운영 메모를 단계별 문서로 정리하는 공간입니다.

현재 문서 순서:

1. [`01-cilium.md`](./01-cilium.md)
2. [`02-metrics-server.md`](./02-metrics-server.md)
3. [`03-metallb.md`](./03-metallb.md)
4. [`04-traefik.md`](./04-traefik.md)
5. [`05-domain-cloudflare-external-dns.md`](./05-domain-cloudflare-external-dns.md)

읽는 순서도 위와 동일하게 가져가는 것을 권장합니다.

## 왜 분리했는가

- Talos 이후 단계는 add-on 별로 설치 이유와 검증 포인트가 다르다.
- Cilium, `metrics-server`, MetalLB, ingress 같은 후속 작업을 한 문서에 계속 누적하면 다시 찾기 어려워진다.
- 번호를 붙여 두면 실제 설치 순서와 문서 흐름을 맞추기 쉽다.

## 현재 기준

- Cluster Name: `homelab-cluster`
- Kubernetes Version: `v1.35.0`
- Control Plane Endpoint: `https://192.168.1.10:6443`
- Pod CIDR: `10.244.0.0/16`
- Service CIDR: `10.96.0.0/12`

## 현재 권장 순서

1. Cilium
2. `metrics-server`
3. MetalLB
4. Traefik
5. Domain + ExternalDNS
6. `cert-manager`
7. 스토리지 연동

## 참고 문서

- Talos bootstrap 문서: [`../01-talos-os/README.md`](../01-talos-os/README.md)
- 루트 개요 문서: [`../README.md`](../README.md)
