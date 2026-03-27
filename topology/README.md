# Cluster Topology

이 폴더는 홈랩 Kubernetes 클러스터의 네트워크/노드 구조를 시각적으로 관리하는 공간입니다.

## 파일

- [`homelab-topology.drawio`](./homelab-topology.drawio): draw.io 원본 편집 파일
- [`homelab-topology.drawio.png`](./homelab-topology.drawio.png): README 등에 표시하기 위한 내보내기 이미지

## 편집 기준

- 원본 수정은 [draw.io](https://draw.io)에서 진행합니다.
- 이미지 변경이 있으면 `.drawio`와 `.png`를 함께 갱신합니다.
- 이 다이어그램은 아래 네트워크 구분을 명확히 보여주도록 유지합니다:
  - `192.168.1.0/24`: 관리망 및 외부 접근망
  - `10.123.0.0/24`: Talos / Kubernetes 노드 내부망
  - `10.244.0.0/16`: Pod CIDR
  - `10.96.0.0/12`: Service CIDR

## 문서 동기화

토폴로지에서 아래 항목이 바뀌면 루트 문서도 함께 업데이트합니다.

- 노드명 / 호스트네임
- VIP, LoadBalancer IP
- 관리망 또는 내부망 CIDR
- Pod CIDR / Service CIDR
