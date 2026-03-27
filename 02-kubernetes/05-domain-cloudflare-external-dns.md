# Domain And ExternalDNS

이 문서는 Traefik 다음 단계로 사용자 도메인을 클러스터에 연결하고, Cloudflare DNS를 ExternalDNS로 자동 관리하는 절차를 정리합니다.

## 전제

이 문서는 아래 조건을 가정합니다.

- 도메인 예시: `example.com`
- DNS 호스팅: Cloudflare
- Traefik가 정상 동작
- MetalLB가 Traefik service에 외부 IP를 정상 할당

여기서 `example.com`은 문서 예시용 placeholder입니다.  
실제 적용 시에는 본인이 소유한 실제 도메인으로 바꿔서 사용해야 합니다.

## 왜 지금 도메인 문서를 두는가

이 시점에는 이미 아래가 준비돼 있습니다.

- Cilium
- `metrics-server`
- MetalLB
- Traefik

이제 실제 사용자가 접근할 이름을 붙일 차례입니다.  
그리고 이후 `cert-manager`를 붙일 때도, 어떤 호스트명을 어떤 Ingress에 연결할지 먼저 정리되어 있어야 흐름이 자연스럽습니다.

## 목표 구조

- `echo.example.com` 같은 호스트명을 만든다
- ExternalDNS가 Cloudflare에 A 레코드를 자동 생성/갱신하게 한다
- Ingress의 hostname과 실제 DNS 레코드가 맞물리게 한다

흐름:

```text
Ingress host -> ExternalDNS -> Cloudflare DNS -> MetalLB external IP -> Traefik -> Service
```

## 테스트 전략

이 문서는 실제 앱을 바로 연결하기보다, `echo-test` 같은 짧은 수명 테스트 리소스로 먼저 검증한 뒤 정리하는 흐름을 권장합니다.

이유:

- DNS, Ingress, ExternalDNS 문제를 실제 앱 문제와 분리할 수 있음
- 실패해도 정리 범위가 명확함
- 이후 `cert-manager`를 붙이기 전까지 최소 리소스로 경로를 검증할 수 있음

즉 이 단계에서는 아래 리소스를 잠깐 만들었다가, 검증이 끝나면 지우는 것을 기본 흐름으로 삼습니다.

- `deployment/echo-test`
- `service/echo-test`
- `ingress/echo-test`

## 1. Cloudflare API Token Secret 만들기

민감 정보는 저장소에 커밋하지 않고, 클러스터 Secret으로만 넣습니다.

```bash
kubectl create namespace external-dns

kubectl -n external-dns create secret generic cloudflare-api-token \
  --from-literal=apiToken=YOUR_CLOUDFLARE_API_TOKEN
```

주의:

- 실제 토큰 문자열은 문서나 git에 남기지 않습니다.
- 토큰 값 앞뒤에 공백/개행이 들어가면 Cloudflare 인증 에러가 날 수 있습니다.

## 2. ExternalDNS 설치

Helm repo 추가:

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update
```

예시 설치 명령:

```bash
helm upgrade --install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --set 'sources[0]=service' \
  --set 'sources[1]=ingress' \
  --set provider.name=cloudflare \
  --set env[0].name=CF_API_TOKEN \
  --set env[0].valueFrom.secretKeyRef.name=cloudflare-api-token \
  --set env[0].valueFrom.secretKeyRef.key=apiToken \
  --set txtOwnerId=homelab-cluster \
  --set policy=upsert-only \
  --set registry=txt \
  --set 'domainFilters[0]=example.com' \
  --set 'extraArgs[0]=--cloudflare-dns-records-per-page=5000'
```

핵심 의미:

- `sources`
  - `service`, `ingress`를 DNS 소스로 사용
- `provider.name=cloudflare`
  - Cloudflare DNS provider 사용
- `CF_API_TOKEN`
  - Cloudflare API Token으로 인증
- `txtOwnerId=homelab-cluster`
  - TXT ownership 충돌 방지용 식별자
- `policy=upsert-only`
  - 처음에는 삭제보다 생성/수정 중심으로 운영해 실수를 줄임
- `registry=txt`
  - ownership 추적을 TXT 레코드로 저장
- `domainFilters`
  - `example.com` 존만 관리 대상으로 제한

## 3. 테스트 앱과 Service 준비

`04-traefik.md`에서 테스트 리소스를 이미 지웠다면 여기서 다시 잠깐 만듭니다.

```bash
kubectl delete ingress echo-test --ignore-not-found
kubectl delete svc echo-test --ignore-not-found
kubectl delete deployment echo-test --ignore-not-found

kubectl create deployment echo-test --image=ealen/echo-server
kubectl expose deployment echo-test --port=80 --target-port=80
kubectl get deployment,pods,svc,endpoints -l app=echo-test -o wide
```

기대 상태:

- `echo-test` pod가 `Running`
- `echo-test` service가 생성됨
- endpoints가 비어 있지 않음

## 4. Ingress 에 실제 도메인 붙이기

예시로 `echo.example.com`을 사용합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-test
  annotations:
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"
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

핵심:

- `rules[].host`의 `echo.example.com`을 ExternalDNS가 읽음
- Cloudflare proxy는 처음에는 `false`로 시작

임시 적용:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-test
  annotations:
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"
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

## 5. DNS 레코드 생성 확인

먼저 Kubernetes 쪽 확인:

```bash
kubectl get ingress
kubectl describe ingress echo-test
kubectl -n external-dns logs deployment/external-dns
```

그 다음 Cloudflare 쪽 확인:

- `echo.example.com` A 레코드 생성 여부
- 대상 IP가 Traefik service 외부 IP와 같은지

로컬에서도 확인:

```bash
dig echo.example.com
nslookup echo.example.com
```

## 6. 실제 요청 확인

DNS 전파 후 확인:

```bash
curl http://echo.example.com
```

정상이라면 `Ingress -> Traefik -> Service` 흐름이 도메인 이름으로도 동작하는 것입니다.

## 7. 테스트 리소스 정리

이 단계가 끝나면 테스트용 앱과 Ingress는 정리해 두는 편이 좋습니다.

삭제 순서:

```bash
kubectl delete ingress echo-test
kubectl delete svc echo-test
kubectl delete deployment echo-test
```

확인:

```bash
kubectl get ingress
kubectl get deployment,svc,pods -l app=echo-test
```

중요:

- `policy=upsert-only`로 설치했다면 Cloudflare의 A/TXT 레코드는 자동 삭제되지 않을 수 있습니다.
- 즉 Kubernetes 리소스를 지운 뒤에도 DNS 레코드가 Cloudflare에 남아 있을 수 있습니다.

처음 테스트 단계에서는 이 점이 오히려 안전합니다.  
실수로 운영 레코드를 지우지 않게 해주기 때문입니다.

정리할 것:

1. Kubernetes의 `echo-test` Ingress/Service/Deployment 삭제
2. Cloudflare에 남은 `echo.example.com` A 레코드 확인
3. 필요하면 Cloudflare에서 테스트 레코드 수동 삭제
4. TXT ownership 레코드도 테스트용이면 같이 정리

## 8. 잘 안 될 때 먼저 볼 것

### 1. Cloudflare에 레코드가 안 생기는 경우

```bash
kubectl -n external-dns get pods
kubectl -n external-dns logs deployment/external-dns
```

### 2. 레코드는 생겼는데 IP가 기대와 다른 경우

```bash
kubectl -n traefik get svc
kubectl get ingress echo-test -o yaml
```

### 3. DNS는 맞는데 HTTP 응답이 없는 경우

```bash
kubectl describe ingress echo-test
kubectl get svc echo-test
kubectl get endpoints echo-test
kubectl -n traefik logs deployment/traefik --tail=100
```

## 현재 단계의 판단

이 저장소에서는 아래 기준이면 성공으로 봅니다.

- ExternalDNS 파드 정상
- Cloudflare API 인증 정상
- `echo.example.com` A 레코드 자동 생성 성공
- 레코드가 Traefik 외부 IP를 가리킴
- `curl http://echo.example.com` 성공
- 테스트 리소스를 정리한 뒤 어떤 리소스와 레코드가 남는지 이해하고 통제 가능

## 다음에 할 작업

1. `cert-manager`
2. HTTPS/TLS 붙이기
3. 실제 앱별 도메인 구조 정리

## 참고 문서

- Traefik 문서: [`./04-traefik.md`](./04-traefik.md)
- Cilium 문서: [`./01-cilium.md`](./01-cilium.md)
- Metrics Server 문서: [`./02-metrics-server.md`](./02-metrics-server.md)
- MetalLB 문서: [`./03-metallb.md`](./03-metallb.md)
- Talos bootstrap 문서: [`../01-talos-os/README.md`](../01-talos-os/README.md)
- ExternalDNS Cloudflare 문서: <https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/cloudflare/>
- ExternalDNS annotations 문서: <https://kubernetes-sigs.github.io/external-dns/latest/docs/annotations/annotations/>
