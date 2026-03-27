# Traefik

이 문서는 MetalLB 이후 단계로 Traefik을 ingress controller로 설치하고, 홈랩에서 HTTP/HTTPS 진입점을 만드는 절차를 정리합니다.

## 왜 Traefik을 선택했는가

현재 저장소 기준에서 Traefik은 가장 실용적인 선택입니다.

- `ingress-nginx`는 retirement가 공지되어 새 설치 대상으로는 부담이 크다.
- `Cilium Gateway API`는 개념적으로 좋지만, 현재 홈랩 환경에서 MetalLB와 조합 시 디버깅 비용이 컸다.
- Traefik은 Kubernetes `Ingress` 방식으로 돌아가면서도 설치와 운영 흐름이 비교적 단순하다.

즉 이 저장소는 지금부터 Traefik을 "실제 홈랩 진입점"의 기본 방향으로 사용합니다.

## 현재 목표

이 단계의 목표는 아래입니다.

- Traefik controller를 `LoadBalancer` 서비스로 노출
- MetalLB가 Traefik에 외부 IP를 할당
- `Ingress` 리소스로 앱 라우팅
- 이후 `ExternalDNS`, `cert-manager`와 자연스럽게 연결

## 사전 조건

이 문서는 아래 상태를 전제로 합니다.

- 모든 노드가 `Ready`
- Cilium 정상 구동
- `metrics-server` 정상 동작
- MetalLB 정상 동작
- 기존 Gateway API 실험 리소스는 정리 완료

먼저 확인:

```bash
kubectl get nodes
kubectl get svc -A
kubectl -n metallb-system get pods
```

## 설치 방식

이 저장소에서는 Helm 설치를 기준으로 합니다.

이유:

- 설치/업그레이드가 단순하다.
- values 파일로 옮기기 쉽다.
- IngressClass, Service 타입, published service 설정을 명확하게 관리할 수 있다.

## 1. Helm repo 추가

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

## 2. Traefik 설치

예시 설치 명령:

```bash
helm upgrade --install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set service.type=LoadBalancer \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=true \
  --set ingressClass.name=traefik \
  --set providers.kubernetesIngress.ingressClass=traefik \
  --set providers.kubernetesIngress.publishedService.enabled=true
```

핵심 의미:

- `service.type=LoadBalancer`
  - MetalLB가 외부 IP를 할당할 수 있게 한다.
- `ingressClass.enabled=true`
  - Traefik용 IngressClass 생성
- `ingressClass.isDefaultClass=true`
  - 기본 IngressClass로 사용한다.
- `ingressClass.name=traefik`
  - IngressClass 이름을 `traefik`으로 고정한다.
- `providers.kubernetesIngress.ingressClass=traefik`
  - Traefik가 처리할 IngressClass를 명시한다.
- `providers.kubernetesIngress.publishedService.enabled=true`
  - ExternalDNS나 Ingress status에 Traefik service 외부 IP가 반영되도록 돕는다.

## 3. 설치 직후 확인

```bash
kubectl -n traefik rollout status deployment/traefik
kubectl -n traefik get pods -o wide
kubectl -n traefik get svc
kubectl get ingressclass
```

기대 상태:

- `deployment/traefik` rollout 완료
- Traefik service가 `LoadBalancer`
- `EXTERNAL-IP`에 MetalLB IP 할당
- `IngressClass traefik` 생성

## 4. 테스트 앱 배포

간단한 echo 서버를 테스트 대상으로 사용합니다.

이미 같은 이름의 테스트 리소스가 남아 있다면 먼저 정리하고 다시 만듭니다.

```bash
kubectl delete ingress echo-test --ignore-not-found
kubectl delete svc echo-test --ignore-not-found
kubectl delete deployment echo-test --ignore-not-found
```

그 다음 생성:

```bash
kubectl create deployment echo-test --image=ealen/echo-server
kubectl expose deployment echo-test --port=80 --target-port=80
```

확인:

```bash
kubectl get deployment,pods,svc,endpoints -l app=echo-test -o wide
```

## 5. 테스트용 Ingress 생성

예시 manifest:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-test
spec:
  ingressClassName: traefik
  rules:
    - host: echo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo-test
                port:
                  number: 80
```

임시 적용:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-test
spec:
  ingressClassName: traefik
  rules:
    - host: echo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo-test
                port:
                  number: 80
EOF
```

확인:

```bash
kubectl get ingress
kubectl describe ingress echo-test
```

## 6. 접근 테스트

먼저 Traefik service 외부 IP 확인:

```bash
kubectl -n traefik get svc
```

예를 들어 외부 IP가 `192.168.1.2`라면:

```bash
curl -H 'Host: echo.example.com' http://192.168.1.2
```

정상이라면 echo 서버 JSON 응답이 돌아옵니다.

## 7. 테스트 리소스 정리

Traefik 자체는 다음 단계에서도 계속 사용하지만, `echo-test` 앱과 테스트용 Ingress는 확인이 끝나면 정리하는 편이 좋습니다.

정리 대상:

- `deployment/echo-test`
- `service/echo-test`
- `ingress/echo-test`

삭제:

```bash
kubectl delete ingress echo-test
kubectl delete svc echo-test
kubectl delete deployment echo-test
```

정리 확인:

```bash
kubectl get ingress
kubectl get deployment,svc,pods -l app=echo-test
```

기대 상태:

- 테스트용 `Ingress echo-test`가 없어짐
- `echo-test` 앱 리소스가 더 이상 남아 있지 않음
- `traefik` namespace의 controller와 service는 그대로 유지됨

즉 이 단계에서 지우는 것은 "테스트 대상 앱"이고, 실제 ingress controller인 Traefik는 남겨 둡니다.

## 8. 잘 안 될 때 먼저 볼 것

### 1. `EXTERNAL-IP` 가 비어 있는 경우

```bash
kubectl -n traefik get svc
kubectl -n metallb-system get pods
kubectl -n metallb-system get ipaddresspools
kubectl -n metallb-system get l2advertisements
```

### 2. Ingress 는 있는데 응답이 없는 경우

```bash
kubectl get ingress
kubectl describe ingress echo-test
kubectl get svc echo-test
kubectl get endpoints echo-test
kubectl -n traefik logs deployment/traefik --tail=100
```

### 3. `Host` 헤더 없이 확인하고 있었던 경우

Ingress는 host 기반 라우팅이므로 아래처럼 확인해야 합니다.

```bash
curl -H 'Host: echo.example.com' http://<TRAEFIK_EXTERNAL_IP>
```

## 현재 단계의 판단

이 저장소에서는 아래 기준이면 성공으로 봅니다.

- `deployment/traefik` rollout 완료
- Traefik service에 MetalLB 외부 IP 할당 성공
- `IngressClass traefik` 생성 확인
- 테스트 Ingress로 HTTP 요청 성공
- 테스트가 끝난 뒤 `echo-test` 리소스를 깨끗하게 정리할 수 있음

## 다음에 할 작업

1. 사용자 도메인 연결
2. ExternalDNS 설정
3. `cert-manager`

## 참고 문서

- Cilium 문서: [`./01-cilium.md`](./01-cilium.md)
- Metrics Server 문서: [`./02-metrics-server.md`](./02-metrics-server.md)
- MetalLB 문서: [`./03-metallb.md`](./03-metallb.md)
- Talos bootstrap 문서: [`../01-talos-os/README.md`](../01-talos-os/README.md)
- Traefik Kubernetes Ingress 문서: <https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/>
- Traefik Helm chart 문서: <https://doc.traefik.io/traefik/getting-started/install-traefik/>
