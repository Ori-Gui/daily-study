# [2026-11-26] Kubernetes 무중단 배포와 프로브 — Deployment 롤링 업데이트·Service/Ingress 트래픽 경로부터 Readiness/Liveness/Startup Probe·preStop·Graceful Shutdown까지

> **한 줄 핵심 요약**: Kubernetes에서 "무중단"은 Deployment가 잘 굴러가서 되는 게 아니라, **① 준비된 파드만 엔드포인트에 넣는 Readiness Probe**, **② 종료 중인 파드를 엔드포인트에서 빼는 시간과 SIGTERM을 처리하는 시간을 맞춘 종료 시퀀스(preStop sleep + Graceful Shutdown)**, **③ 그 둘을 가능하게 하는 requests/limits 산정**이 같이 맞아야 성립합니다. `maxSurge`/`maxUnavailable`은 *몇 개를 동시에 갈아치울지*만 정할 뿐이고, **엔드포인트 갱신(kube-proxy 전파)과 SIGTERM 도착 사이의 시차**가 배포 때 502가 나는 실제 원인입니다. Liveness는 "내 프로세스가 살아 있는가"만 물어야 하고, DB 같은 외부 의존을 물으면 **장애 한 번에 전체 파드가 재시작 루프**에 빠집니다. (Docker 격리는 11-20, JVM 메모리는 11-18 노트와 이어집니다.)

---

## 1. 개념 및 동작 원리

### 1.1 `docker run` 한 대와 24/7 서비스의 차이

로컬에서 `docker run -p 8080:8080 app`으로 띄운 컨테이너와, 트래픽을 받는 서비스 사이에는 다음의 빈칸이 있습니다.

| 빈칸 | 손으로 하면 | Kubernetes가 하는 일 |
| --- | --- | --- |
| 여러 대 중 어디에 띄울까 | 서버마다 SSH 접속 | Scheduler가 노드 자원(request) 보고 배치 |
| 죽으면 누가 다시 띄우나 | 크론 + 스크립트, 사람 | kubelet이 컨테이너 상태 감시 후 재시작 |
| 새 버전 배포 | 접속해서 `docker pull` + 재시작(순단) | Deployment가 롤링 업데이트 |
| 파드 IP가 계속 바뀌는데 어떻게 부르나 | 수동 IP 관리 | Service가 가상 IP로 묶고 EndpointSlice로 갱신 |
| 트래픽 받을 준비 됐는지 | 사람이 확인 | Readiness Probe |

구조는 **선언(desired state)** 과 **조정(reconciliation)** 두 단어로 요약됩니다.

```text
[Control Plane]
  kube-apiserver  ← 모든 요청/상태의 단일 창구 (kubectl, 컨트롤러 모두 여기로)
  etcd            ← 클러스터 상태 저장 (여기가 죽으면 클러스터는 "동결"됨)
  kube-scheduler  ← Pod를 어느 노드에 놓을지 결정 (아직 실행은 안 함)
  kube-controller-manager ← Deployment/ReplicaSet/EndpointSlice 등 컨트롤 루프
[Worker Node]
  kubelet         ← apiserver가 준 PodSpec대로 컨테이너 실행·프로브 수행·재시작
  container runtime (containerd)
  kube-proxy      ← Service 가상 IP → Pod IP DNAT 규칙(iptables/IPVS) 유지
```

- **모든 컴포넌트는 apiserver를 통해서만 대화합니다.** kubelet이 "파드 3개 유지"를 직접 판단하지 않고, ReplicaSet 컨트롤러가 *현재 개수와 spec을 비교해서* 파드를 만들고 지웁니다. 그래서 `kubectl delete pod`를 해도 ReplicaSet이 있으면 다시 살아납니다(이게 정상 동작입니다).
- 이 구조 때문에 **"YAML에 쓴 것"과 "지금 실제로 도는 것"이 다를 수 있습니다.** 배포가 안 끝났는데 명령만 성공하는 경우가 그래서 생깁니다(2.6절의 `rollout status`가 필요한 이유).

### 1.2 오브젝트 계층: Pod → ReplicaSet → Deployment

| 오브젝트 | 책임 | 실무에서 직접 만드나 |
| --- | --- | --- |
| **Pod** | 컨테이너 1개 이상 + 볼륨 + 네트워크 네임스페이스. **스케줄·재시작의 최소 단위** | 거의 안 만듦(디버깅용 `kubectl run`) |
| **ReplicaSet** | 지정한 개수의 파드를 유지. 라벨 셀렉터로 관리 | 직접 안 만듦 |
| **Deployment** | ReplicaSet을 **버전별로** 관리. 롤아웃·롤백·이력 | **실무의 기본 단위** |
| **StatefulSet** | 순서·고유 이름·PVC 유지(DB, Kafka 등) | 상태 저장 워크로드일 때 |
| **DaemonSet** | 모든 노드에 1개씩(로그 수집, CNI, 모니터링 에이전트) | 인프라 컴포넌트일 때 |
| **Job/CronJob** | 완료가 목적인 작업, 주기 실행(배치) | 배치성 작업일 때 |

- **파드는 일회용(ephemeral)입니다.** 재생성되면 IP가 바뀌고, 노드 안의 로컬 파일은 사라집니다. 그래서 세션·업로드 파일·캐시를 파드 로컬 디스크에 두면 안 됩니다. 세션은 Redis, 파일은 S3/오브젝트 스토리지로 빼는 이유가 여기 있습니다(11-10 Redis 노트).
- **연결 고리는 전부 라벨입니다.** Deployment의 `spec.selector.matchLabels`는 Pod template의 `labels`와 반드시 일치해야 하고, Service의 `spec.selector`도 같은 라벨을 봅니다. 이 셋 중 하나가 어긋나면 배포는 "성공"인데 **엔드포인트가 0개인 서비스**가 됩니다.

```bash
# 라벨/셀렉터가 어긋났는지 확인하는 가장 빠른 명령
kubectl get deploy,rs,pod --show-labels
kubectl get endpointslices -l kubernetes.io/service-name=my-api   # ENDPOINTS 비어 있으면 셀렉터 불일치 또는 readiness 전멸
```

### 1.3 롤링 업데이트: `maxSurge` / `maxUnavailable`이 실제로 하는 일

Deployment 기본 전략은 `RollingUpdate`이고 기본값은 둘 다 **25%**입니다.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # spec.replicas 위로 몇 개까지 더 띄울 수 있나
    maxUnavailable: 0    # spec.replicas 아래로 몇 개까지 줄어도 되나
```

계산 규칙이 미묘합니다.

| 항목 | 값 | 계산 |
| --- | --- | --- |
| replicas | 4 | |
| maxSurge 25% | 1 | **올림** → 4 + 1 = 최대 5개 |
| maxUnavailable 25% | 1 | **내림** → 4 - 1 = 최소 3개 |

- **둘 다 0은 불가능합니다.** 새 파드를 하나도 못 띄우고 기존 파드도 못 내리면 교체 자체가 진행되지 않기 때문입니다.
- `maxUnavailable: 0` + `maxSurge: 1`은 **가용성을 최우선**하는 조합이지만, 노드 자원이 빡빡하면 새 파드가 `Pending`에 걸려 롤아웃이 진행되지 않습니다. 무중단을 원한다는 이유로 무조건 이 값을 쓰다가 배포가 멈추는 경우가 흔합니다.
- **`maxUnavailable`은 "준비된 파드"를 세는 값입니다. 여기에 Readiness Probe가 없으면 계산이 무의미해집니다.** 프로브가 없으면 컨테이너가 뜨자마자 Ready로 간주되어, 아직 스프링 컨텍스트가 안 올라온 파드가 트래픽을 받습니다.
- 관련 필드 네 개를 같이 기억합니다.

| 필드 | 기본값 | 의미 |
| --- | --- | --- |
| `minReadySeconds` | 0 | Ready가 된 뒤 이만큼 버텨야 "가용"으로 인정(조기 성공 방지) |
| `progressDeadlineSeconds` | 600 | 이 시간 동안 진행이 없으면 `ProgressDeadlineExceeded`로 실패 표시 |
| `revisionHistoryLimit` | 10 | 롤백용 ReplicaSet 보관 개수 |
| `terminationGracePeriodSeconds` | 30 | 종료 유예 시간(2.1절, preStop 포함) |

- 롤아웃은 **ReplicaSet을 새로 하나 만들고, 새 RS를 늘리면서 옛 RS를 줄이는** 방식입니다. 그래서 특정 시점에는 **두 버전이 동시에 트래픽을 받습니다.** 이 사실이 API 변경(필드 삭제, 응답 형식 변경)에서 장애를 만듭니다 — 구버전과 신버전이 같은 DB 스키마를 봐야 하므로 **스키마 변경은 항상 "확장 → 배포 → 수축" 3단계**로 나눠야 합니다(예: 컬럼 추가 → 양쪽 코드 배포 → 옛 컬럼 삭제).

### 1.4 트래픽 경로: Service와 Ingress

```text
사용자 → [Ingress Controller(Nginx)] → [Service ClusterIP] → [Pod A / Pod B / Pod C]
              L7: Host/Path/TLS              L4: DNAT (kube-proxy)
```

| 계층 | 무엇을 보나 | 특징 |
| --- | --- | --- |
| **Service (ClusterIP)** | IP + 포트 | 가상 IP. 실제로는 kube-proxy가 만든 iptables/IPVS 규칙이 Pod IP로 DNAT. **L4, 연결 단위 분산**(랜덤에 가까움) |
| **Service (NodePort)** | 노드 IP:30000~32767 | 외부 노출용. 실무에선 잘 안 씀 |
| **Service (LoadBalancer)** | 클라우드 LB | AWS면 NLB/CLB 프로비저닝 |
| **Ingress** | Host, Path, 헤더 | L7 라우팅·TLS 종료. **컨트롤러(Nginx/ALB)를 따로 설치해야 동작** |

- **Service의 로드밸런싱은 "연결" 단위입니다.** HTTP keep-alive로 연결을 재사용하는 클라이언트는 계속 같은 파드로 갑니다. 파드별 부하가 안 고르게 퍼지는 이유이고, 무중단 배포 관점에서는 **긴 연결이 남아 있는 파드가 종료될 때** 끊김이 생기는 원인이 됩니다.
- **Ingress는 L7이라 재시도·타임아웃을 줄 수 있습니다.** Nginx Ingress의 `proxy-next-upstream`, `proxy-read-timeout` 같은 설정으로 파드 종료 중 발생한 연결 오류를 흡수할 수 있습니다. 허나 이건 **가려주는 장치**지, 종료 시퀀스가 틀린 것을 고치지는 않습니다.
- **`readinessProbe`가 Ready가 아닌 파드는 EndpointSlice에서 빠집니다.** 이게 "무중단"의 절반입니다. 나머지 절반은 종료 쪽(1.6절)입니다.

### 1.5 프로브 세 가지 — 각각 무엇을 막는가

| 프로브 | 질문 | 실패 시 동작 | 잘못 설정하면 |
| --- | --- | --- | --- |
| **readinessProbe** | "지금 트래픽 받아도 되나?" | Service 엔드포인트에서 제외(파드는 유지) | 준비 전 파드가 트래픽을 받아 500, 또는 전 파드 제외로 **전면 503** |
| **livenessProbe** | "이 프로세스 살아 있나? 재시작해야 하나?" | 컨테이너 재시작 | 일시적 지연에 재시작 폭풍 → **장애 증폭** |
| **startupProbe** | "부팅 끝났나?" | (실패 시 재시작) | 없으면 느린 부팅이 liveness에 걸려 `CrashLoopBackOff` |

- **startupProbe가 성공하기 전까지 liveness·readiness는 평가되지 않습니다.** 그래서 "부팅이 최대 90초 걸린다"는 서비스는 `initialDelaySeconds`를 90으로 주는 대신 `startupProbe.failureThreshold * periodSeconds ≥ 90`으로 **부팅 시간만** 넉넉히 주고, liveness는 짧게 유지할 수 있습니다. 이 분리가 startupProbe의 존재 이유입니다.
- **liveness에 `/actuator/health`(전체)를 쓰면 안 됩니다.** 11-18 노트에서 본 것처럼 전체 health는 DB·Redis·디스크까지 봅니다. DB가 30초 흔들리면 **모든 파드가 동시에 재시작**되고, 재시작 중에는 커넥션이 더 몰려 DB가 더 죽는 하강 나선이 됩니다. liveness는 **자기 프로세스의 이벤트 루프/스레드가 응답하는지**만 봐야 합니다(Spring이면 `livenessState`).
- **readiness에 DB를 넣는 것도 위험합니다.** DB가 잠깐 느려지면 전 파드가 엔드포인트에서 빠져 503이 나고, 트래픽이 0이 되면 DB가 회복돼도 요청이 없어 다시 Ready 판정을 못 받는 교착이 생길 수 있습니다. **"내 프로세스가 요청을 처리할 수 있는 상태인가"** 가 readiness의 기준이고, 의존성 장애는 회로 차단기·재시도로 다루는 게 맞습니다.
- 프로브 수치의 실무 출발점(절대 정답 아님):

```yaml
startupProbe:              # 부팅 예산: 5s * 30 = 최대 150초
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  periodSeconds: 5
  failureThreshold: 30
readinessProbe:            # 트래픽 자격: 3회 연속 실패(6초)면 제외
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  periodSeconds: 2
  timeoutSeconds: 1
  failureThreshold: 3
livenessProbe:             # 프로세스 생존: 3회 연속 실패(30초)면 재시작
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
```

- `timeoutSeconds`가 `periodSeconds`보다 크면 프로브가 겹쳐 kubelet이 부하를 만들 수 있습니다. 타임아웃은 주기보다 작게 둡니다.

### 1.6 파드 종료 시퀀스 — 배포할 때 502가 나는 진짜 이유

파드를 지울 때(롤아웃, `kubectl delete pod`, 노드 드레인) 일어나는 일을 시간순으로 놓으면 이렇습니다.

```text
t=0.0  apiserver: Pod에 deletionTimestamp 설정 (이때부터 "Terminating")
       ├─ (A) endpoints controller: EndpointSlice에서 이 파드를 ready=false / terminating=true 로 갱신
       └─ (B) kubelet: 종료 절차 시작
t=0.0~ (B1) preStop hook 실행 (여기서 sleep 5 를 주는 이유)
t=5.0~ (B2) SIGTERM 전달 → 애플리케이션이 graceful shutdown (진행 중 요청 마무리 + 신규 수신 중단)
t=?    (B3) terminationGracePeriodSeconds 초과 시 SIGKILL (exit 137)
```

문제는 **(A)와 (B)가 동시에, 서로를 기다리지 않고** 진행된다는 점입니다.

- EndpointSlice가 갱신돼도 **각 노드의 kube-proxy가 규칙을 다시 쓰기까지 시간이 걸립니다**(수백 ms ~ 수 초, 클러스터 규모에 비례). 그 사이 새 연결이 이미 SIGTERM을 받은 파드로 들어갑니다 → **connection reset / 502**.
- 반대로 **SIGTERM을 받은 프로세스가 즉시 소켓을 닫으면**, 아직 남아 있던 keep-alive 연결과 처리 중 요청이 그대로 끊깁니다.
- 그래서 순서를 **인위적으로 벌립니다.**

```yaml
spec:
  terminationGracePeriodSeconds: 40   # preStop(5s) + graceful(최대 20s) + 여유
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]   # "엔드포인트 전파 대기"용. 로직을 넣는 자리가 아님
```

- **`preStop` 실행 시간은 유예 시간에 포함됩니다.** `terminationGracePeriodSeconds: 30`에 `sleep 25`를 넣으면 graceful shutdown에 5초밖에 못 씁니다. 위처럼 합계를 계산해서 잡습니다.
- **Spring Boot 쪽은 따로 켜야 합니다.** 기본값은 즉시 종료입니다.

```yaml
# application.yml
server:
  shutdown: graceful                 # SIGTERM 시 신규 요청 거부 + 진행 중 요청 완료 대기
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s  # 이 시간 넘으면 강제 종료
management:
  endpoint:
    health:
      probes:
        enabled: true                # /actuator/health/{liveness,readiness} 노출
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

- Spring Boot는 graceful shutdown이 시작되면 **ReadinessState를 `REFUSING_TRAFFIC`으로 바꿉니다.** 즉 `/actuator/health/readiness`가 503이 되어 kubelet이 "Ready 아님"으로 표시하고, 엔드포인트 제거가 한 번 더 촉진됩니다. 다만 이것도 kube-proxy 전파 시간을 없애지는 못하므로 **preStop sleep은 여전히 필요합니다.**
- 정리하면 무중단 배포의 실제 조건은 네 가지가 동시에 성립하는 것입니다.
  1. Readiness가 **실제 준비 완료** 시점에 켜진다(프로브 경로가 얕고 정확).
  2. `maxUnavailable`이 0이거나 최소 1 이상 여유가 있다.
  3. 종료 시 **엔드포인트 제거 → SIGTERM 순서가 벌어진다**(preStop sleep).
  4. 애플리케이션이 SIGTERM에 **진행 중 요청을 마무리하고** 종료한다(graceful shutdown).

### 1.7 requests / limits — 무중단의 전제 조건

```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }   # 스케줄링·HPA의 기준
  limits:   { cpu: "1",    memory: "768Mi" }   # 상한
```

| 구분 | CPU | Memory |
| --- | --- | --- |
| 초과 시 | **스로틀링**(CFS quota, 100ms 주기) — 죽지 않고 느려짐 | **OOMKilled**(exit 137) — 즉시 종료 |
| 압축 가능? | 가능(성능만 저하) | 불가능(회수하려면 죽여야 함) |
| requests 의미 | 스케줄러가 노드 여유로 판단 | 노드 메모리 압박 시 eviction 순위에 영향 |

- **CPU limit은 지연 스파이크의 원인**이 됩니다. 100ms 주기 quota를 다 쓰면 다음 주기까지 스레드가 멈추므로, 평균 CPU는 낮은데 p99 응답이 튀는 패턴이 나옵니다. JVM처럼 워밍업·GC가 몰리는 워크로드는 CPU limit을 아예 두지 않고 requests만으로 관리하는 선택지도 있습니다(노드 자원 격리가 약해지는 트레이드오프).
- **메모리 limit은 JVM에서 특히 조심해야 합니다.** 컨테이너 메모리에는 힙 외에 Metaspace, 스레드 스택, Direct Buffer, JIT 코드 캐시, GC 구조체가 모두 들어갑니다. `-Xmx`를 limit과 같게 잡으면 **반드시** 한계에 부딪힙니다.

```dockerfile
# limit 768Mi 기준: 힙은 65~70% 선에서 잡는다
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=65.0", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-jar", "/app/app.jar"]
```

  - `MaxRAMPercentage`는 컨테이너 인식(cgroup) 기반으로 힙을 잡습니다. `-Xmx512m` 하드코딩은 limit 변경 시 같이 안 고쳐져서 위험합니다. (11-18 JVM 노트의 컨테이너 메모리 산정과 같은 맥락)
  - `ExitOnOutOfMemoryError`를 주면 힙 OOM 시 프로세스가 죽고 kubelet이 재시작합니다. 안 주면 반쯤 죽은 상태로 liveness만 통과하며 500을 뱉는 경우가 생깁니다.
- **QoS 클래스**는 requests/limits 조합으로 자동 결정됩니다.

| QoS | 조건 | 노드 압박 시 |
| --- | --- | --- |
| Guaranteed | 모든 컨테이너에 requests == limits (CPU·메모리 모두) | 마지막에 축출 |
| Burstable | requests < limits (일부만 지정) | 중간 |
| BestEffort | 아무것도 지정 안 함 | **먼저 축출** |

### 1.8 HPA — 파드를 늘려도 안 빨라지는 경우

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }   # requests 대비 %
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 기본 5분. 진동 방지
```

- HPA의 CPU 사용률은 **`requests.cpu` 대비 비율**입니다. **requests를 안 넣으면 비율을 계산할 수 없어 HPA가 동작하지 않습니다**(`<unknown>` 타깃).
- 기본 동기화 주기 15초, scale-down 안정화 300초. 트래픽이 1분 주기로 출렁이면 스케일 인/아웃이 계속 반복되므로 `behavior`로 감쇠를 겁니다.
- **파드를 늘려도 커넥션 풀·DB가 병목이면 효과가 없습니다.** 파드 10개 × HikariCP 10개 = DB 커넥션 100개가 되어 DB가 먼저 죽습니다. 스케일 아웃 전에 **파드당 풀 크기 × 최대 파드 수 ≤ DB max_connections** 를 계산해야 합니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 실무형 Deployment 전체 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
  labels: { app: order-api }
spec:
  replicas: 4
  revisionHistoryLimit: 5
  minReadySeconds: 5                 # Ready 후 5초 버텨야 가용 인정
  progressDeadlineSeconds: 300
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0              # 가용성 우선(노드 여유 필요)
  selector:
    matchLabels: { app: order-api }  # 아래 template.labels와 반드시 동일
  template:
    metadata:
      labels: { app: order-api, version: v1 }
    spec:
      terminationGracePeriodSeconds: 40
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        fsGroup: 10001
      topologySpreadConstraints:     # 노드 장애 대비 분산
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app: order-api }
      containers:
        - name: api
          image: registry.example.com/order-api:1.8.3   # latest 금지, 불변 태그
          imagePullPolicy: IfNotPresent
          ports:
            - { name: http, containerPort: 8080 }
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: order-api-secret, key: db-password }
          resources:
            requests: { cpu: "500m", memory: "640Mi" }
            limits:   { memory: "768Mi" }    # CPU limit은 의도적으로 미설정(스로틀링 회피)
          startupProbe:
            httpGet: { path: /actuator/health/readiness, port: http }
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: http }
            periodSeconds: 2
            timeoutSeconds: 1
            failureThreshold: 3
            successThreshold: 1
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: http }
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
```

- **`successThreshold`는 readiness에서만 1보다 크게 쓸 수 있습니다.** liveness/startup은 1이어야 합니다(실패 → 성공 반복으로 재시작을 판정하는 구조라서).
- **`imagePullPolicy`**: 불변 태그에는 `IfNotPresent`, 태그를 덮어쓰는 운영 방식이면 `Always`(그러나 캐시 문제로 배포가 안 먹는 사고가 잦으므로 **digest 고정**이 안전: `image: repo/app@sha256:...`).
- `topologySpreadConstraints`는 노드/존에 파드가 몰리지 않게 합니다. 노드가 1대뿐이면 무의미하지만, 클라우드 멀티 AZ에서는 사실상 필수입니다.

### 2.2 Service / Ingress / PDB

```yaml
apiVersion: v1
kind: Service
metadata: { name: order-api }
spec:
  type: ClusterIP
  selector: { app: order-api }     # Pod labels와 일치
  ports:
    - { name: http, port: 80, targetPort: http }   # targetPort는 파드의 포트 이름
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-api
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-next-upstream: "error timeout http_502"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: order-api
                port: { name: http }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: order-api-pdb }
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: order-api }
```

- **`targetPort`에 이름을 쓰면** 컨테이너 포트를 바꿔도 Service를 안 고쳐도 됩니다.
- **PDB는 `kubectl drain`(자발적 중단)에만 적용됩니다.** `kubectl delete pod`나 롤아웃에는 PDB가 개입하지 않습니다. 그래서 노드 점검 때 PDB가 있으면 드레인의 `--timeout`을 넉넉히 주고, 없으면 **한 번에 여러 파드가 내려가** 순단이 납니다.
- `minAvailable`을 replicas와 같게 두면 드레인이 **영원히 막힙니다**(내릴 수 있는 파드가 0). `maxUnavailable: 1`이 더 안전한 경우가 많습니다.

### 2.3 함정 체크리스트

| 증상 | 원인 후보 | 확인 |
| --- | --- | --- |
| 배포 직후 간헐적 502/connection reset | preStop sleep 없음 + graceful shutdown 미설정 | `terminationGracePeriodSeconds`, `server.shutdown` |
| 새 버전 배포했는데 500 급증 | readiness가 얕아 준비 전 트래픽 유입, 또는 구/신 버전 공존 중 API·스키마 불일치 | 프로브 경로, 배포 중 스키마 변경 여부 |
| `CrashLoopBackOff` | startupProbe 예산 부족, liveness가 너무 짧음, 힙 OOM | `kubectl describe pod`의 `Last State: OOMKilled` |
| `OOMKilled` 반복 | `-Xmx`가 limit과 같음, Direct Buffer/스레드 스택 미계산 | `MaxRAMPercentage`, 스레드 수 |
| p99만 튐, 평균은 정상 | CPU limit 스로틀링, GC | `container_cpu_cfs_throttled_periods_total` |
| 서비스 응답 503, 파드는 Running | Service selector 불일치 또는 readiness 전멸 | `kubectl get endpointslices` |
| 배포가 `Pending`에서 멈춤 | 노드 자원 부족(maxSurge 파드 못 띄움) | `kubectl describe pod`의 `FailedScheduling` |
| HPA가 `<unknown>` | requests 미설정 또는 metrics-server 없음 | `kubectl top pods` |
| 롤아웃이 안 끝나고 `ProgressDeadlineExceeded` | 이미지 pull 실패, 프로브 영구 실패 | `kubectl rollout status`, 이벤트 |

### 2.4 검증 절차 — "됐다"를 어떻게 확인하나

```bash
# 1) 롤아웃이 실제로 끝났는지 (명령 성공 ≠ 배포 완료)
kubectl rollout status deployment/order-api --timeout=300s

# 2) 새 RS가 정상이고 옛 RS가 0인지
kubectl get rs -l app=order-api
kubectl get pods -l app=order-api -o wide

# 3) 엔드포인트가 기대한 개수인지 (readiness 전멸 감지)
kubectl get endpointslices -l kubernetes.io/service-name=order-api

# 4) 종료 중 파드가 실제로 트래픽을 안 받는지 — 배포 중에 반복 호출
for i in $(seq 1 200); do
  curl -s -o /dev/null -w "%{http_code}\n" https://api.example.com/health
  sleep 0.2
done | sort | uniq -c
# 기대: 200만. 502/000(연결 실패)이 섞이면 종료 시퀀스가 틀린 것

# 5) 문제 시 롤백
kubectl rollout undo deployment/order-api --to-revision=3
```

- **로컬에서도 재현할 수 있습니다.** kind/minikube에 Deployment + Service를 올리고 롤아웃 중 `curl` 루프를 돌리면, preStop 유무에 따라 502 개수가 확연히 달라집니다. 포트폴리오에서 "무중단 배포를 구현했다"고 쓸 거라면 **이 수치(preStop 전 502 n건 → 후 0건)** 를 남기는 게 설득력 있습니다.
- 프로브 자체도 검증 대상입니다. `readiness`가 실제로 503을 내는지 `kubectl exec`로 직접 호출해 확인하고, 의도적으로 DB 커넥션을 끊어 **liveness는 200, readiness는 503**이 되는지 봅니다.

### 2.5 배포 전략 비교

| 전략 | 방식 | 장점 | 단점 / 필요 조건 |
| --- | --- | --- | --- |
| **RollingUpdate** | 파드를 순차 교체 | 추가 인프라 불필요, 기본 제공 | 두 버전 공존 → **하위 호환 필수** |
| **Blue-Green** | 새 환경 전체를 띄우고 Service selector 스위치 | 즉시 롤백, 두 버전 동시 부하 테스트 | **자원 2배**, 스위치 순간 전환(세션·캐시 정합) |
| **Canary** | 소수 비율로 신버전 노출 후 확대 | 위험 최소화, 실측 기반 판단 | 트래픽 분할·지표 비교 도구 필요(Argo Rollouts, 서비스 메시) |

- 신입 포트폴리오 기준으로는 **RollingUpdate + readiness/liveness + graceful shutdown**을 정확히 설명할 수 있는 편이, Argo Rollouts를 이름만 아는 것보다 훨씬 강합니다. 면접에서 "왜 Blue-Green을 안 썼나"에 **"자원이 2배 필요하고, 이 프로젝트는 하위 호환이 깨지는 스키마 변경이 없어 롤링으로 충분했다"** 고 답할 수 있으면 됩니다.

### 2.6 ConfigMap/Secret — 배포와 얽히는 지점

- ConfigMap/Secret을 `env`로 주입하면 **파드 재시작 전까지 반영되지 않습니다.** 값만 바꿔도 롤아웃이 자동으로 일어나지 않습니다.
- 파일로 마운트(`volumeMounts`)하면 kubelet이 주기적으로 갱신하지만, **애플리케이션이 파일을 다시 읽지 않으면 무의미**합니다(스프링은 시작 시 읽음). 결국 **설정 변경 = 롤아웃**으로 취급하는 게 단순합니다.
- Secret은 base64일 뿐 암호화가 아닙니다. etcd 암호화·RBAC·외부 시크릿 관리(AWS Secrets Manager, Vault)로 보완합니다. Git에 평문 Secret을 커밋하지 않습니다.

---

## 3. 기술 면접 / 실전 예상 질문 & 답변 팁

- **Q. Kubernetes가 정확히 무엇을 해주나요?**
  - **A.** "선언한 상태를 유지하는 컨트롤 루프의 집합입니다. Deployment에 replicas 4를 선언하면 ReplicaSet 컨트롤러가 현재 개수를 비교해 파드를 만들고, kubelet이 노드에서 컨테이너를 유지합니다. 사람이 개별 서버에 접속해 재시작·배포하던 일을 API 오브젝트로 옮긴 것이고, 그래서 '파드를 지워도 다시 살아난다'는 동작이 정상입니다."

- **Q. 롤링 업데이트에서 502가 나는 이유를 설명해 보세요.**
  - **A.** "파드 삭제 시 엔드포인트 제거와 SIGTERM 전달이 병렬로 일어나고, 각 노드의 kube-proxy가 규칙을 갱신하기까지 시차가 있습니다. 그 사이 새 연결이 종료 중인 파드로 들어가 끊깁니다. 그래서 preStop에 sleep을 넣어 엔드포인트 전파 시간을 벌고, 애플리케이션은 graceful shutdown으로 진행 중 요청을 마무리합니다. preStop 시간은 terminationGracePeriodSeconds에 포함되므로 합계를 계산해 잡아야 합니다."

- **Q. Readiness와 Liveness의 차이는 무엇이고, 각각에 무엇을 넣어야 하나요?**
  - **A.** "Readiness는 '트래픽을 받을 자격'이라 실패하면 엔드포인트에서 빠지고 파드는 살아 있습니다. Liveness는 '프로세스를 재시작해야 하나'라 실패하면 컨테이너가 재시작됩니다. Liveness에는 자기 프로세스의 생존만 넣습니다. DB 같은 외부 의존을 liveness에 넣으면 DB가 잠깐 흔들릴 때 전 파드가 재시작되어 장애가 커집니다. Readiness도 전체 health 대신 준비 상태만 보는 경로를 씁니다."

- **Q. startupProbe는 왜 필요한가요?**
  - **A.** "부팅이 느린 애플리케이션에서 liveness의 initialDelaySeconds를 크게 주면, 일단 뜬 뒤의 장애 감지도 그만큼 느려집니다. startupProbe는 부팅 완료까지만 넉넉한 예산을 주고, 성공한 뒤에는 짧은 주기의 liveness로 감시하게 해줍니다. 이게 없으면 배포 직후 CrashLoopBackOff가 나기 쉽습니다."

- **Q. Service는 어떻게 파드를 찾고, 파드가 죽으면 어떻게 빠지나요?**
  - **A.** "Service는 라벨 셀렉터로 파드를 고르고, 엔드포인트 컨트롤러가 Ready인 파드 IP를 EndpointSlice에 씁니다. kube-proxy가 그 목록으로 DNAT 규칙을 만들어 ClusterIP를 파드 IP로 변환합니다. 파드가 Ready가 아니면 목록에서 빠지므로 트래픽을 받지 않습니다. 결국 라벨 셀렉터가 세 곳(Deployment 셀렉터, Pod 라벨, Service 셀렉터)에서 일치해야 동작합니다."

- **Q. Service의 로드밸런싱은 라운드로빈인가요?**
  - **A.** "기본은 L4 연결 단위 분산에 가깝고, HTTP keep-alive를 쓰는 클라이언트는 연결을 재사용해 같은 파드로 계속 갑니다. 그래서 파드별 부하가 고르지 않을 수 있습니다. 요청 단위 분산이 필요하면 Ingress 컨트롤러나 서비스 메시 쪽에서 다루고, 세션 유지가 필요하면 쿠키 기반 스티키니스나 세션 외부화(Redis)를 씁니다."

- **Q. CPU limit을 설정하지 않는 이유가 있나요?**
  - **A.** "CPU limit은 CFS quota로 100ms 주기 안에서 스로틀링을 걸어, 평균 사용률은 낮은데 p99 응답이 튀는 현상을 만듭니다. JVM처럼 GC·워밍업으로 순간 사용량이 몰리는 워크로드는 requests로 스케줄링 여유를 확보하고 limit은 두지 않는 선택을 하기도 합니다. 대신 노드 전체 자원 보호가 약해지므로 requests 산정을 정확히 해야 합니다. 반대로 메모리는 회수할 방법이 없어 limit을 반드시 둡니다."

- **Q. 메모리 limit을 768Mi로 줬는데 OOMKilled가 반복됩니다. 왜인가요?**
  - **A.** "컨테이너 메모리에는 힙 외에 Metaspace, 스레드 스택, Direct Buffer, JIT 코드 캐시가 포함됩니다. -Xmx를 limit과 같게 잡으면 그 나머지가 초과해 반드시 죽습니다. 컨테이너 인식 기반으로 MaxRAMPercentage를 65~70%로 잡고, 스레드 수와 Direct Buffer 사용량을 함께 봅니다."

- **Q. maxUnavailable을 0으로 두면 무중단인가요?**
  - **A.** "아닙니다. 그건 '기존 파드를 먼저 내리지 않는다'는 뜻일 뿐이고, 새 파드가 준비됐는지 판단하는 것은 Readiness Probe입니다. 프로브가 없거나 부정확하면 준비 안 된 파드가 트래픽을 받습니다. 또 노드 자원이 부족하면 새 파드가 Pending에 걸려 롤아웃 자체가 멈춥니다."

- **Q. HPA를 켰는데 트래픽이 늘어도 파드가 안 늘어납니다.**
  - **A.** "CPU 사용률 타깃은 requests.cpu 대비 비율이라 requests가 없으면 계산이 안 됩니다. metrics-server가 없어도 값이 unknown이 됩니다. 그 외에 maxReplicas 도달, 노드 자원 부족으로 Pending, 또는 애플리케이션이 CPU를 안 쓰고 DB에서 대기하는 경우도 확인합니다. 마지막 경우는 스케일 아웃이 아니라 쿼리·캐시 개선이 답입니다."

- **Q. 무중단 배포를 위해 팀에서 무엇을 합의해야 하나요?**
  - **A.** "세 가지입니다. 첫째, 스키마 변경은 확장-배포-수축 3단계로 하고 두 버전이 공존해도 동작해야 합니다. 둘째, 모든 서비스에 readiness/liveness와 graceful shutdown, preStop sleep을 기본 템플릿으로 넣습니다. 셋째, 배포 후 rollout status와 엔드포인트 개수, 에러율을 확인하는 절차를 정합니다. 도구보다 이 약속이 먼저입니다."

---

## 참고

- Kubernetes Docs — Deployments (RollingUpdate, maxSurge/maxUnavailable): https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes Docs — Pod Lifecycle (Container probes, Termination sequence): https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- Kubernetes Docs — Service와 EndpointSlice: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes Docs — Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
- Kubernetes Docs — Resource Management for Pods and Containers (requests/limits, QoS): https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- Kubernetes Docs — Horizontal Pod Autoscaling: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- Kubernetes Docs — Specifying a PodDisruptionBudget: https://kubernetes.io/docs/tasks/run-application/configure-pdb/
- Spring Boot Reference — Graceful Shutdown: https://docs.spring.io/spring-boot/reference/web/servlet.html#web.servlet.graceful-shutdown
- Spring Boot Reference — Kubernetes Probes (liveness/readiness 상태): https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes
- NGINX Ingress Controller — Retries and timeouts (`proxy-next-upstream`): https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
