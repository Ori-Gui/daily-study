# [2026-11-24] TLS 1.3 핸드셰이크와 QUIC·HTTP/3 — 1-RTT 키 스케줄·0-RTT 재전송 위험부터 Nginx·CloudFront 구성까지

> **한 줄 핵심 요약**: TLS 1.3(RFC 8446)은 ① 키 교환에 필요한 클라이언트 몫을 **ClientHello에 미리 실어 보내** 핸드셰이크를 2-RTT→1-RTT로 줄이고, ② `ServerHello` 이후 모든 핸드셰이크 메시지를 암호화하며, ③ 정적 RSA 키 교환을 없애 **모든 공개키 기반 키 교환이 전방향 비밀성(PFS)** 을 갖도록 바꿨습니다. QUIC(RFC 9000)은 여기에 **TCP를 UDP로 바꿔** 전송 계층의 HOL 블로킹을 없애고 TLS 1.3을 핸드셰이크에 내장해 **첫 연결 1-RTT, 재개 0-RTT** 를 만듭니다. 실무에서 진짜 위험한 지점은 성능이 아니라 **0-RTT 재전송(replay)** 과 **인증서·세션 티켓 운영**이며, 0-RTT는 `Early-Data: 1` + `425 Too Early`로 비멱등 요청을 막아야 합니다.

---

## 1. 개념 및 동작 원리

### 1.1 HTTPS가 실제로 보장하는 세 가지

| 보장 | 수단 | 없으면 |
| --- | --- | --- |
| 기밀성(Confidentiality) | AEAD 암호(AES-GCM, ChaCha20-Poly1305) | 경로상의 누구나 쿠키·토큰·본문 열람 |
| 무결성(Integrity) | AEAD 인증 태그, Finished MAC | 중간자가 응답 본문 변조(광고 삽입, 송금 계좌 변경) |
| 상대 인증(Authentication) | X.509 인증서 + 서명 검증(CA 체인) | `bank.com`으로 접속한 줄 알았는데 공격자 서버 |

- **TLS은 OSI 7계층 어디인가** — 엄밀히는 4계층(전송)과 7계층(응용) 사이의 **세션/표현 계층(5~6계층)** 에 놓이는 프로토콜입니다. 시험에서는 "전송 계층 위에서 동작하는 보안 프로토콜"로 서술하면 안전합니다. HTTPS는 `HTTP + TLS`이고 기본 포트는 **443**입니다(TLS 자체는 포트를 규정하지 않음).
- 암호의 역할 분담을 먼저 고정해야 전체가 보입니다. **비대칭키(RSA/ECDHE)는 키 교환·서명에만** 쓰고, 실제 데이터는 **대칭키**로 암복호화합니다. 대칭키가 빠르지만 키를 어떻게 안전하게 나눠 갖느냐가 문제이고, 그 문제를 푸는 것이 핸드셰이크입니다.
- TLS 1.3에서 **키 교환(DHE)과 인증서 서명은 서로 다른 키**를 씁니다. ECDHE로 세션키를 만들고, 그 핸드셰이크 내역(transcript)에 서버 인증서 개인키로 서명합니다(`CertificateVerify`). 그래서 **서버 인증서 개인키가 나중에 유출돼도 과거 트래픽은 복호화되지 않습니다**(PFS). TLS 1.2의 RSA 키 교환은 이 성질이 없어서 제거됐습니다.

### 1.2 TLS 1.2 vs TLS 1.3 — RTT가 줄어든 구조적 이유

```text
[TLS 1.2 전체 핸드셰이크]  TCP 1 RTT + TLS 2 RTT = 3 RTT
Client                                          Server
  |------ TCP SYN ------------------------------>|   1 RTT
  |<----- SYN-ACK -------------------------------|
  |------ ACK ---------------------------------->|
  |------ ClientHello (평문) ------------------->|
  |<----- ServerHello, Certificate, ... ---------|   2 RTT
  |------ ClientKeyExchange, CCS, Finished ----->|
  |<----- CCS, Finished -------------------------|
  |====== Application Data =====================>|

[TLS 1.3 전체 핸드셰이크]  TCP 1 RTT + TLS 1 RTT = 2 RTT
Client                                          Server
  |------ ClientHello + key_share ------------->|   ← 클라이언트 몫을 미리 계산해 전송
  |<----- ServerHello + key_share  ┐
  |<----- {EncryptedExtensions}    │ 암호화 구간
  |<----- {Certificate}            │
  |<----- {CertificateVerify}      │
  |<----- {Finished}              ┘
  |------ {Finished} --------------------------->|
  |====== Application Data =====================>|   ← 여기서 이미 서버 데이터 수신 가능
```

- 핵심은 **`key_share` 확장**입니다. TLS 1.2의 DHE/ECDHE는 "서버가 파라미터를 정하고 → 클라이언트가 자기 몫을 계산해 보내는" 2-RTT 구조였습니다. TLS 1.3은 클라이언트가 **쓸 것 같은 그룹(X25519 등)의 공개값을 ClientHello에 미리 넣어** 보내므로, 서버가 같은 그룹을 고르면 **첫 응답에서 바로 Finished까지** 보낼 수 있습니다.
- 서버가 클라이언트의 `key_share` 그룹을 받아들일 수 없으면 **HelloRetryRequest**로 한 번 되돌아가 **추가 1-RTT**가 붙습니다. 그래서 클라이언트는 지원 그룹 수만큼 `key_share`를 넣거나(증가된 패킷 크기) 가장 흔한 그룹(X25519)을 우선 넣는 튜닝을 합니다. 실무에서 "TLS 1.3인데 왜 2-RTT지?"의 대부분이 HRR입니다.
- **`ServerHello`까지만 평문**입니다(버전·난수·선택된 키 교환). `EncryptedExtensions`부터는 파생된 핸드셰이크 키로 암호화되므로 **인증서가 평문으로 노출되지 않습니다**. 그래서 SNI(`server_name`)만 남는데, 이 SNI까지 감추는 것이 **ECH(Encrypted Client Hello)** 이고 2026년 3월 **RFC 9849**로 표준화되었습니다(OpenSSL 4.0이 지원 예정).

### 1.3 1-RTT를 가능하게 만든 키 스케줄(HKDF)

TLS 1.3의 키 파생은 **HKDF(`HKDF-Extract` → `HKDF-Expand`)** 두 단계로 재설계되어, 각 단계 비밀이 서로 분리됩니다(RFC 8446 §7.1).

```text
                   0
                   │
                   ▼
   PSK ───────► HKDF-Extract ──► Early Secret
                   │                ├─ Derive-Secret("ext binder"/"res binder") → binder_key
                   │                └─ Derive-Secret("c e traffic")             → client_early_traffic_secret  ← 0-RTT
                   ▼
  (EC)DHE ───► HKDF-Extract ──► Handshake Secret
                   │                ├─ "c hs traffic" → client_handshake_traffic_secret
                   │                └─ "s hs traffic" → server_handshake_traffic_secret
                   ▼
                   0 ──► HKDF-Extract ──► Master Secret
                                      ├─ "c ap traffic" → client_application_traffic_secret_0
                                      ├─ "s ap traffic" → server_application_traffic_secret_0
                                      ├─ "exp master"   → exporter_master_secret
                                      └─ "res master"   → resumption_master_secret → NewSessionTicket → 다음 연결의 PSK
```

- **`label` 문자열로 용도를 분리**합니다. 같은 비밀에서 방향(client/server)·용도(handshake/application/resumption)별 키를 파생하므로, 하나가 노출돼도 다른 키를 역산할 수 없습니다.
- 각 트래픽 비밀에서 `key`와 `iv`를 파생하고, **레코드마다 `iv XOR 시퀀스 번호`로 nonce**를 만듭니다. 그래서 TLS 1.2처럼 레코드마다 명시적 IV를 싣지 않아 오버헤드가 줄고, nonce 재사용으로 인한 AEAD 붕괴 위험도 구조적으로 막힙니다.
- `KeyUpdate` 메시지로 핸드셰이크 없이 애플리케이션 트래픽 키를 갱신할 수 있고, 갱신 횟수는 `traffic_secret_0 → _1 → _2 …`로 이어집니다.
- cipher suite의 의미가 **"암호 + 해시"로 단순화**되었습니다. 예: `TLS_AES_128_GCM_SHA256` = AEAD는 AES-128-GCM, HKDF 해시는 SHA-256. **키 교환·서명 알고리즘은 cipher suite가 아니라 별도 확장**(`supported_groups`, `signature_algorithms`)으로 협상합니다 — 이게 TLS 1.2와의 큰 차이점이고, "TLS 1.3 cipher suite는 5개뿐"인 이유입니다(`TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`, `TLS_AES_128_CCM_SHA256`, `TLS_AES_128_CCM_8_SHA256`).

### 1.4 TLS 1.3에서 사라진 것과 그 결과

| 제거/변경 | 이유 | 실무 파급 |
| --- | --- | --- |
| 정적 RSA 키 교환 | PFS 없음(서버 개인키 유출 = 과거 전체 복호화) | ECDHE 전용. RSA 인증서는 **서명용**으로 계속 사용 |
| CBC 모드·RC4·3DES·정적 DH | 암호학적 취약·구현 오류(Lucky13, BEAST 계열) | 구형 클라이언트는 TLS 1.2로 남아야 함 |
| 압축(compression) | CRIME 공격 | gzip 응답 압축(HTTP 계층)과 혼동 금지 |
| 재협상(renegotiation) | 다운그레이드·자원 고갈 공격 표면 | 재협상 기반 클라이언트 인증서 요구 로직은 재설계 필요 |
| DSA, 커스텀 DHE 파라미터, P-192 등 | 안전성·구현 복잡도 | DSA 인증서만 있는 서버는 1.3 불가 |
| `ChangeCipherSpec` | 상태 기계 단순화 | **middlebox 호환용 더미 CCS**는 여전히 전송됨(무시해야 함) |
| 세션 ID / 세션 티켓 이원화 | 단일 PSK 교환으로 통합 | `NewSessionTicket` 한 가지 경로만 이해하면 됨 |

- **하프 클로즈(half-close) 정책 차이**도 실무 함정입니다. TLS 1.3은 한쪽이 close_notify를 보내면 그 방향만 닫는 half-close이고, TLS 1.2까지는 양방향 종료(duplex-close) 관행이 있었습니다. 소켓 종료 순서에 의존하는 레거시 코드는 1.3 전환 후 `SocketException`·조기 EOF를 겪을 수 있습니다(JDK JEP 332가 명시한 호환성 위험).
- **버전 다운그레이드 보호**: 1.3을 지원하는 서버가 협상 결과 1.2가 되면 `ServerHello.random` 마지막 8바이트에 `44 4F 57 4E 47 52 44 01`("DOWNGRD\x01")을 넣습니다. 클라이언트는 이 값을 보고 중간자 다운그레이드를 탐지합니다.

### 1.5 인증서·SNI·ALPN — 핸드셰이크가 "누구와, 무슨 프로토콜로"를 정하는 방법

- **SNI**: 클라이언트가 `ClientHello`에 목적 도메인을 넣어 **한 IP에서 여러 인증서를 서비스**할 수 있게 합니다. 리버스 프록시의 가상호스트 라우팅이 이 값에 의존합니다. 대신 평문 노출이 프라이버시 문제라 ECH가 등장했습니다.
- **ALPN**: `h2`, `h3`, `http/1.1` 중 무엇으로 말할지 핸드셰이크 안에서 정합니다. HTTP/2가 "협상되는" 이유이고, ALPN이 없으면 서버는 HTTP/1.1로 떨어집니다.
- **인증서 체인**: 서버는 리프 인증서와 **중간 CA 인증서까지** 보내야 클라이언트가 루트까지 검증할 수 있습니다. 리프만 보내면 브라우저는 통과(중간 CA를 캐시·AIA로 보충)하지만 **Java·curl·모바일 SDK는 실패**하는 대표적 운영 사고입니다.
- **OCSP stapling**: 서버가 폐기 상태 응답을 미리 받아 붙여 클라이언트의 OCSP 서버 왕복과 프라이버시 노출을 줄입니다. TLS 1.3은 OCSP 확장 형식을 갱신했습니다(RFC 8446 §4.4.2.1). 스테이플링 갱신 실패가 조용히 누적되는지 모니터링해야 합니다.
- **CT(Signed Certificate Timestamp)**: 공인 CA가 발급 즉시 로그에 등록하게 해, 부정 발급을 탐지합니다. 브라우저는 SCT 없는 인증서를 거부합니다.

### 1.6 세션 재개와 0-RTT — 성능과 보안이 정면으로 교환되는 지점

- 전체 핸드셰이크 후 서버는 `NewSessionTicket`을 보냅니다. 클라이언트는 `resumption_master_secret`에서 파생한 **PSK**와 티켓을 저장하고, 다음 연결의 `ClientHello`에 `pre_shared_key`로 제시합니다. 서버가 수락하면 **Certificate/CertificateVerify 없이** 1-RTT로 끝납니다.
- **PSK + (EC)DHE** 조합이 일반적입니다. PSK만 쓰면 인증은 되지만 **PFS가 사라지므로**, 주요 구현은 재개 시에도 `key_share`를 함께 보내 PFS를 유지합니다.
- **0-RTT(early data)** 는 그 PSK로 **첫 비행에 바로 애플리케이션 데이터를 실어** 보냅니다. 서버가 수락하면 핸드셰이크 대기 시간이 0이 됩니다. 대신 RFC 8446 §2.3이 명시한 두 가지 손실이 있습니다.

| 0-RTT 데이터의 성질 | 의미 | 실무 결과 |
| --- | --- | --- |
| PFS 없음 | 오직 PSK로만 암호화 | PSK 유출 시 해당 early data 복호화 가능 |
| 재전송(replay) 보장 없음 | `ServerHello` 난수에 묶이지 않음 | **공격자가 같은 0-RTT 요청을 복제 전송해 서버가 두 번 처리 가능** |
| 연결 내 중복은 없음 | 한 연결에서 같은 데이터 두 번 처리 없음 | 재생은 "다른 연결"에서 발생 |

- 재생 방어는 서버 몫입니다. ① **티켓을 1회용**으로 만들고, ② `ticket_age_add`로 난독화된 클라이언트 보고 나이(`obfuscated_ticket_age`)를 **신선도 창(freshness window)** 과 대조해 오래된 티켓 재사용을 거부하며, ③ 허용 범위 안의 재사용은 **Bloom filter·캐시로 단일 사용 처리**합니다(RFC 8446 §8.1). ④ 애플리케이션은 **비멱등 요청에 early data를 허용하지 않는 것**이 최종 방어선입니다.
- HTTP 계층에는 이를 위한 표준 수단이 있습니다. TLS 종단 장치가 `Early-Data: 1` 헤더를 붙이고(RFC 8470 §5.1), 애플리케이션은 재시도가 안전하지 않은 요청에 **`425 Too Early`** 를 돌려줍니다. 이때 425를 받은 클라이언트는 **1-RTT 연결로 자동 재시도**합니다(11-19 노트의 Idempotency-Key와 같은 문제의식 — 재시도가 정상 동작이므로 서버가 흡수해야 함).

### 1.7 TCP 위에서는 왜 안 되는가 — HTTP/2의 구조적 한계

RFC 9114 §1.1이 정확히 이 지점을 지적합니다.

> HTTP/2는 바이너리 프레이밍과 멀티플렉싱을 도입했지만, **HTTP/2 멀티플렉싱의 병렬성은 TCP의 손실 복구 메커니즘에 보이지 않는다.** 패킷 하나가 유실되거나 재정렬되면, 그 패킷의 영향을 직접 받지 않은 트랜잭션까지 **모두 정지(stall)** 한다.

- HTTP/1.1은 연결 6개로 병렬화했지만 연결마다 혼잡 제어가 따로 돌아 비효율적입니다. HTTP/2는 한 연결에 스트림을 겹쳐 이 문제를 풀었지만, **TCP가 "바이트 스트림 하나"** 라서 3번 세그먼트가 유실되면 4·5번 세그먼트(다른 스트림)도 커널 버퍼에서 대기합니다 → **전송 계층 HOL 블로킹**.
- 애플리케이션 계층에서 아무리 스트림을 나눠도 **아래가 순서 보장 단일 파이프**면 소용이 없다는 것이 HTTP/2의 한계이고, 그래서 전송 계층 자체를 바꾼 것이 QUIC입니다.

### 1.8 QUIC — UDP 위에 전송 기능을 다시 올린 프로토콜

QUIC은 **UDP 데이터그램 위에** 신뢰성·혼잡 제어·멀티플렉싱·암호화를 **사용자 공간에서** 구현합니다. TCP를 고치는 대신 UDP를 고른 이유는 **인터넷의 미들박스·커널을 통과**하기 위해서입니다(TCP 옵션·새 프로토콜 번호는 장비가 막는 경우가 많음).

```text
HTTP/3 (RFC 9114)   : 요청/응답 의미론, QPACK 헤더 압축, 제어 스트림
──────────────────────────────────────────────────────────────
QUIC  (RFC 9000)    : 스트림 멀티플렉싱, 스트림별 흐름 제어, 연결 ID,
                      손실 복구(RFC 9002), 혼잡 제어, 연결 마이그레이션
                      └ TLS 1.3 핸드셰이크 내장 (RFC 9001)
──────────────────────────────────────────────────────────────
UDP (커널)          : 포트, 체크섬. 순서·재전송·혼잡 제어는 QUIC이 담당
```

**프레임과 패킷 단위 설계**

- QUIC 패킷은 **프레임**을 담고, 스트림 데이터는 `STREAM` 프레임에 **스트림 ID + 오프셋**으로 실립니다. 그래서 재전송이 **스트림 단위로 정확히** 일어나고 다른 스트림은 막히지 않습니다. 손실 복구는 **패킷 번호 공간**(Initial / Handshake / Application)별로 따로 관리합니다.
- **재전송 시 패킷 번호를 재사용하지 않습니다.** TCP는 재전송 패킷의 번호가 같아서 ACK가 원본인지 재전송인지 모호하지만(Karn 알고리즘 문제), QUIC은 **새 패킷 번호로 같은 프레임을 다시 보내** RTT 측정이 정확하고 **불필요한 재전송(spurious retransmission)** 이 줄어듭니다.
- **연결 ID(Connection ID)** 로 연결을 식별합니다. IP·포트가 4-튜플이 아니라 CID에 묶이므로, Wi-Fi→LTE 전환·NAT 리바인딩에도 **핸드셰이크 없이 연결이 이어집니다**(연결 마이그레이션). v1에서 **클라이언트만** 마이그레이션할 수 있고, CID 교체(`NEW_CONNECTION_ID`)로 경로 간 연결성을 링커빌리티 공격에서 보호합니다.
- TLS 1.3을 **레코드 계층 없이** 통합합니다(RFC 9001). 핸드셰이크 메시지는 `CRYPTO` 프레임으로 나르고, 각 암호화 레벨(Initial/Handshake/1-RTT)이 자체 패킷 보호 키를 가집니다. **QUIC에는 평문 모드가 없습니다**(항상 TLS 1.3).

**QUIC 패킷 보호의 특징**

- 헤더 필드(패킷 번호 등)는 **헤더 보호(header protection)** 로 일부를 가려, 관찰자가 패킷 번호·ACK 패턴으로 트래픽 특성을 읽기 어렵게 합니다(RFC 9001 §5.4).
- `Initial` 패킷은 고정된 솔트로 파생한 키를 쓰므로 실제 기밀성은 없습니다(핸드셰이크 부트스트랩용). 그래서 Initial 패킷 크기를 1200바이트 이상으로 강제해 **증폭 공격**을 제한하고, `Retry`·`NEW_TOKEN`으로 **주소 검증**을 합니다.

### 1.9 HTTP/3 — 스트림 하나가 요청 하나

- HTTP/3는 **요청/응답 하나당 양방향 스트림 하나**를 씁니다(HTTP/2는 한 TCP 연결에 프레임을 인터리빙). 그래서 스트림 하나가 막혀도 나머지는 진행합니다.
- 헤더 압축은 **QPACK(RFC 9204)** 입니다. HPACK은 순서에 의존하는 동적 테이블을 모든 스트림이 공유해 **머리 블로킹**을 만들었지만, QPACK은 **인코더/디코더 전용 단방향 스트림**으로 테이블 상태를 동기화해 스트림 독립성을 유지합니다. 대신 "헤더 블록이 참조하는 동적 테이블 항목이 아직 도착하지 않은" 경우를 다루는 **blocked stream** 개념이 생겼습니다.
- **프로토콜 발견**은 UDP 443으로 바로 쏘지 않습니다. 첫 연결은 TCP+TLS(HTTP/2)로 맺고, 응답 헤더의 **`Alt-Svc: h3=":443"; ma=86400`** 을 보고 다음 연결부터 QUIC을 시도합니다. 브라우저는 QUIC 실패 시 **즉시 TCP로 폴백**하므로, UDP가 막힌 망에서도 서비스가 유지됩니다.
- 실무에서 이 구조는 배포 위치를 결정합니다. **엣지(CDN·브라우저 앞단)에서 HTTP/3를 종료하고, 오리진까지는 HTTP/1.1·HTTP/2로 가는 구성**이 일반적입니다. CloudFront도 뷰어 구간만 HTTP/3이고 오리진 fetches는 HTTP/1.1입니다.

### 1.10 RTT 총합 비교 — 시험에 그대로 나오는 표

| 구간 | TCP + TLS 1.2 | TCP + TLS 1.3 | QUIC (HTTP/3) |
| --- | --- | --- | --- |
| 전송 연결 수립 | 1 RTT (3-way) | 1 RTT | 0 (핸드셰이크에 통합) |
| 암호 핸드셰이크 | 2 RTT | 1 RTT | 1 RTT |
| **최초 연결 합계** | **3 RTT** | **2 RTT** | **1 RTT** |
| 재개(resumption) | 2 RTT | 2 RTT (PSK 1-RTT) | **1 RTT**, early data 시 **0 RTT** |
| HOL 블로킹 | 있음 | 있음(전송 계층) | **없음(스트림 단위)** |
| 연결 마이그레이션 | 불가(4-튜플) | 불가 | **가능(CID)** |
| 전송 계층 암호화 | 없음(헤더 평문) | 없음 | **패킷 단위 암호화+헤더 보호** |

- "**1-RTT면 데이터를 바로 보낼 수 있나?**" — TLS 1.3에서 클라이언트는 자기 `Finished`를 보낸 **직후** 애플리케이션 데이터를 보낼 수 있고, 서버는 첫 비행에서 이미 앱 데이터를 보낼 수 있습니다. 즉 1-RTT 핸드셰이크는 **왕복 1회 뒤 양방향 데이터**가 가능한 구조입니다.
- 숫자를 외울 때는 **"TCP 1 + TLS 1.2의 2 = 3"** 과 **"TCP 1 + TLS 1.3의 1 = 2, QUIC은 TCP를 없애 1"** 두 줄로 정리하면 헷갈리지 않습니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 Spring Boot에서 TLS 1.3 명시하기

```yaml
# src/main/resources/application.yml
server:
  port: 8443
  ssl:
    enabled: true
    enabled-protocols: TLSv1.3          # 구형 클라이언트 지원이 필요하면 TLSv1.2,TLSv1.3
    ciphers:                             # TLS 1.3 cipher suite는 '암호+해시' 조합만 지정
      - TLS_AES_128_GCM_SHA256
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
    client-auth: none                    # mTLS는 need(=필수) / want(=선택)
  http2:
    enabled: true                        # ALPN으로 h2 협상
```

- JDK 11(JEP 332)부터 TLS 1.3이 **기본 활성**이고, JDK 21에서는 `TLS_CHACHA20_POLY1305_SHA256`·X25519·EdDSA까지 기본 제공됩니다. 별도 라이브러리가 필요 없습니다.
- **JDK JSSE는 0-RTT(early data)를 구현하지 않았습니다**(JEP 332의 비목표). 즉 Spring Boot(내장 Tomcat/Jetty) 앞단에서 0-RTT를 켜도 애플리케이션까지 early data가 전달되지 않으므로, 0-RTT는 **엣지(Nginx·CDN)에서 종료하고 오리진은 일반 요청으로 받는 구조**가 자연스럽습니다.
- 인증서를 파일로 관리한다면 SSL Bundle을 쓰는 편이 낫습니다(회전·번들 분리).

```yaml
# PEM 인증서를 SSL Bundle로 (Spring Boot 3.1+)
spring:
  ssl:
    bundle:
      pem:
        api:
          keystore:
            certificate: file:/etc/tls/tls.crt      # 리프 + 중간 CA 체인 포함
            private-key: file:/etc/tls/tls.key
          truststore:
            certificate: file:/etc/tls/ca.crt
server:
  ssl:
    bundle: api
```

> `enabled-protocols`에 `TLSv1.3`만 남기면 **구형 Android·사내 프록시·일부 결제 단말이 접속하지 못합니다.** 실제로는 1.2·1.3을 열고, 취약 cipher를 `jdk.tls.disabledAlgorithms`로 막는 쪽이 안전합니다.

### 2.2 Nginx에서 TLS 1.3 + HTTP/3 종료

Nginx의 HTTP/3는 **1.25.0부터 실험적**으로 제공되며 `--with-http_v3_module`로 빌드해야 합니다. QUIC은 TLS 1.3이 필수입니다.

```nginx
# /etc/nginx/conf.d/api.conf
server {
    # 같은 포트에서 TCP(TLS)와 UDP(QUIC)를 함께 연다
    listen 443 ssl;
    listen 443 quic reuseport;

    server_name api.example.com;
    ssl_certificate     /etc/tls/fullchain.pem;   # 리프 + 중간 CA
    ssl_certificate_key /etc/tls/privkey.pem;

    ssl_protocols       TLSv1.3;                  # QUIC은 1.3 필수
    ssl_session_tickets on;
    ssl_session_timeout 1d;

    quic_retry on;                # 주소 검증(Retry) → 증폭 공격·위조 Initial 차단
    quic_gso on;                  # Linux UDP_SEGMENT 지원 시 배치 전송
    ssl_early_data on;            # 0-RTT (아래 주의사항 필수)

    add_header Alt-Svc 'h3=":443"; ma=86400' always;   # HTTP/3 광고
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://spring_app;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Early-Data $ssl_early_data;   # 0-RTT 여부를 앱에 전달
    }
}
```

- **`ssl_early_data on`은 재생 위험을 애플리케이션에 넘기는 스위치**입니다. Nginx는 early data를 그냥 통과시키므로, 앱이 `Early-Data` 헤더를 보고 비멱등 요청을 425로 거절하지 않으면 **중복 결제·중복 발송이 열립니다.** Nginx 공식 문서도 `proxy_set_header Early-Data $ssl_early_data;`를 명시적으로 권고합니다.
- Nginx 문서 기준 0-RTT는 **OpenSSL 3.5.1 이상(또는 BoringSSL/QuicTLS)** 에서 지원되고, 1.29.1 이전에는 OpenSSL로는 0-RTT를 켤 수 없었습니다. 실무 결론: **0-RTT는 기본 OFF**로 두고, 정적 GET 트래픽 비중이 큰 서비스에서만 검증 후 켭니다.
- Nginx는 **OpenSSL 내장 재생 방어를 기본으로 끕니다**(`ssl_early_data` 문서: 세션 재개와 충돌). 필요하면 `ssl_conf_command Options AntiReplay;`로 되살릴 수 있지만, 인스턴스마다 상태를 두는 방식이라 앞단 로드밸런서가 여러 대면 완전하지 않습니다. 결국 **애플리케이션 계층 방어(2.5절)가 최종선**입니다. 0-RTT 구간에서 `GET` 외 메서드를 아예 막는 `limit_except GET { deny all; }`도 함께 쓰는 구현이 많습니다.
- `quic_bpf on`은 Linux 5.7+에서만 동작하며 **연결 마이그레이션 라우팅**을 가능하게 합니다. 이걸 켜지 않으면 클라이언트 IP가 바뀔 때 연결이 끊길 수 있습니다.
- 멀티 인스턴스에서 `ssl_session_tickets`를 쓰면 **티켓 암호화 키가 인스턴스마다 달라** 재개율이 급감합니다. `ssl_session_ticket_key` 파일을 공유하거나(정기 로테이션), 앞단 로드밸런서에서 TLS를 종료하세요.

### 2.3 AWS에서의 위치 선정 — ALB인가 CloudFront인가

| 구성 | HTTP/3 종료 | 비고 |
| --- | --- | --- |
| **CloudFront** | ✅ (뷰어 구간) | Alt-Svc 자동 삽입, 추가 비용 없음. **오리진 구간은 HTTP/1.1** |
| **ALB** | ❌ | 클라이언트 구간 HTTP/1.1·HTTP/2만. HTTPS 리스너에서 자동 협상 |
| **NLB** | ✅ (QUIC/TCP_QUIC 리스너) | L4라 HTTP 의미론·헤더 라우팅 없음. L7 기능 필요하면 부적합 |

- 정석은 **CloudFront(HTTP/3, TLS 종료, WAF) → ALB(HTTP/2) → 앱**입니다. ALB는 HTTP/3를 종료하지 않으므로 "ALB에 HTTP/3를 켜려고" 시간을 쓰면 안 됩니다.
- CloudFront에서 HTTP/3를 켜면 **보안 정책과 무관하게 뷰어 구간은 TLS 1.3만** 사용됩니다(QUIC 요구사항). 구형 클라이언트를 지원해야 한다면 HTTP/3를 끄거나, 폴백(HTTP/2)이 실제로 동작하는지 확인해야 합니다.

### 2.4 애플리케이션 서버에서 HTTP/3를 직접 서비스할 때

- Spring Boot은 아직 HTTP/3를 1급으로 지원하지 않습니다(spring-boot 이슈 #47163 open). **Reactor Netty 1.3 계열과 Jetty 12.1 계열**은 HTTP/3 서버를 제공하지만 **Tomcat·Undertow는 미지원**입니다.
- Reactor Netty로 직접 띄우면 **TCP 리스너와 UDP 리스너를 따로** 띄워야 합니다(HTTP/3 전용이 아니라 HTTP/2·1.1 폴백을 함께 서비스해야 하므로). 이는 운영 복잡도를 크게 올리므로, **1차 선택은 엣지 종료**입니다.
- 직접 서비스가 정당한 경우는 ① 사내에서 엣지 CDN을 못 쓰는 환경, ② 모바일 SDK처럼 지연에 민감한 API, ③ 단일 리전·단일 홉 구조입니다. 이때도 **지표(핸드셰이크 실패율, 재개율, UDP 차단 비율)를 먼저 계측**한 뒤 켭니다.

### 2.5 0-RTT를 애플리케이션에서 막는 코드

```java
// src/main/java/com/example/web/EarlyDataGuardFilter.java
package com.example.web;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.Set;

/**
 * TLS 종단 장치(Nginx·CDN)가 0-RTT 요청에 붙이는 Early-Data: 1(RFC 8470 §5.1)을 보고
 * 재생되어서는 안 되는 요청을 425 Too Early로 되돌린다. 클라이언트는 1-RTT로 재시도한다.
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)   // 인증·본문 처리보다 앞
public class EarlyDataGuardFilter extends OncePerRequestFilter {

    /** 425 Too Early — Servlet API에 상수가 없어 리터럴을 쓴다(RFC 8470). */
    private static final int SC_TOO_EARLY = 425;

    private static final Set<String> SAFE_METHODS = Set.of("GET", "HEAD", "OPTIONS");

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        boolean earlyData = "1".equals(request.getHeader("Early-Data"));
        boolean safe = SAFE_METHODS.contains(request.getMethod());

        if (earlyData && !safe) {
            response.setStatus(SC_TOO_EARLY);
            response.setHeader("Content-Type", "application/json");
            response.getWriter().write(
                    "{\"code\":\"EARLY_DATA_REJECTED\",\"message\":\"0-RTT 요청은 재시도 후 처리됩니다.\"}");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

- **안전한 메서드만 통과**시키는 것이 핵심입니다. `GET`도 부수효과가 있는 설계(예: `GET /coupon/issue`)라면 화이트리스트를 경로 단위로 좁혀야 합니다.
- 425를 돌려줄 때 **본문과 코드를 명시**해야 클라이언트가 "재시도 가능"을 오해 없이 판단합니다. `Retry-After`를 함께 주는 구현도 있지만, 0-RTT 실패는 **즉시 1-RTT 재시도**가 정답이라 대기 시간은 의미가 없습니다.
- 여기에 11-19 노트의 **Idempotency-Key**를 겹쳐 두면 0-RTT 재생·프록시 재시도·사용자 더블클릭을 한 구조로 흡수할 수 있습니다.

### 2.6 세션 재개·티켓 운영에서 조용히 새는 것들

| 증상 | 원인 | 대응 |
| --- | --- | --- |
| 재개율이 낮다(항상 전체 핸드셰이크) | 인스턴스마다 티켓 키가 다름 / `ssl_session_tickets off` | 티켓 키 공유·로테이션, TLS는 엣지에서 종료 |
| 0-RTT가 안 된다 | Nginx 빌드 OpenSSL 버전 미달, `ssl_early_data` 미설정, 클라이언트가 티켓 없음 | `nginx -V`로 SSL 라이브러리 확인, 첫 방문은 원래 0-RTT 불가 |
| 재개 연결에서 인증서 교체가 반영 안 됨 | PSK 재개는 인증서를 다시 보내지 않음 | 인증서 교체 시 **티켓 키를 함께 로테이션** |
| 간헐적 `bad record mac` | 키 로테이션 중 구·신 키 혼재, L4 로드밸런서의 세션 고정 문제 | 로테이션에 겹침 구간을 두고, 진단은 `SSLKEYLOGFILE`로 |
| 앱은 정상인데 특정 고객만 TLS 실패 | 그 고객망의 TLS 검사 프록시가 1.3 미지원·SNI 차단 | 1.2 허용 정책 검토, SNI 없는 접속 대비(기본 인증서) |

- **재개는 인증서 검증을 건너뜁니다.** 폐기(revocation)된 인증서라도 유효한 PSK가 남아 있으면 연결이 이어질 수 있으므로, 인증서 사고 시 **티켓 키 폐기 + 세션 캐시 초기화**를 사고 대응 절차에 넣어야 합니다.
- 세션 **캐시**(서버 메모리)와 **티켓**(클라이언트 보관)의 트레이드오프: 캐시는 서버 상태가 필요하지만 폐기가 즉각적이고, 티켓은 무상태지만 키 유출 시 위조 재개가 가능합니다. 다중 인스턴스면 대개 티켓 + 키 공유를 택합니다.

### 2.7 mTLS와 프록시 뒤에서의 함정

```yaml
# 클라이언트 인증서를 요구하는 내부 API
server:
  ssl:
    client-auth: need
    trust-store: file:/etc/tls/client-ca.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12
```

- **TLS 종료 지점과 인증 지점이 다르면 인증서가 사라집니다.** 엣지에서 TLS를 끊고 오리진에 HTTP로 넘기면 `request.getAttribute("jakarta.servlet.request.X509Certificate")`는 null입니다. 이때는 엣지가 검증 후 **서명된 헤더(JWT 등)로 신원을 전달**하고, 오리진은 **그 헤더를 외부에서 위조할 수 없도록** 네트워크를 격리해야 합니다(헤더 스푸핑이 곧 인증 우회).
- 서버가 클라이언트 인증서를 요구하는 순간 **핸드셰이크가 1-RTT 추가**될 수 있습니다(서버가 `CertificateRequest` 후 클라이언트 응답을 기다림). 내부 서비스 메시에서는 흔하지만, 브라우저 대상 서비스에는 쓰지 않습니다.
- `X-Forwarded-Proto`를 신뢰하지 않으면 HTTPS 요청을 HTTP로 오인해 **무한 리다이렉트**가 납니다. 반대로 아무 프록시나 신뢰하면 `http://`로 들어온 요청을 https로 위장당합니다. **신뢰 프록시 대역을 명시**하는 설정(Spring의 `ForwardedHeaderFilter` + 신뢰 경계)이 필요합니다.
- 헬스체크가 TLS를 타지 않거나, 반대로 인증서 검증을 끄고 도는 경우가 많습니다. **프로브는 TLS로, 검증은 켠 채로** 두고, 인증서 만료가 프로브 실패로 이어지도록 감시합니다.

### 2.8 성능·비용 관점의 판단

- TLS 1.3 핸드셰이크는 **CPU가 더 듭니다**(ECDHE 연산 + AEAD + 인증서 서명 검증). 세션 재개율이 낮으면 지연과 CPU가 동시에 나빠지므로, **재개율을 지표로 잡는 것**이 실질적 최적화입니다.
- **ECDSA(P-256) 인증서가 RSA 2048보다 서명·검증이 가볍고 체인도 짧습니다.** 다만 일부 레거시 클라이언트 호환 때문에 RSA를 병행(EcDSA 우선, RSA 폴백)하기도 합니다.
- QUIC은 **패킷당 처리 비용이 TCP+TLS보다 큰 경향**이 있고(UDP GSO/GRO로 완화), 커널 최적화가 없어 고pps 환경에서 CPU가 먼저 병목이 됩니다. 대역폭이 큰 정적 파일 전송보다 **지연 민감·모바일·손실 많은 망**에서 이득이 큽니다.
- UDP 443이 차단된 망(사내 방화벽, 일부 ISP, 구형 프록시)에서는 QUIC이 실패하고 TCP로 폴백합니다. **폴백이 정상 동작하는지**를 배포 전에 확인해야 하며, 폴백이 안 되면 해당 사용자층이 전면 장애를 겪습니다.
- HTTP/3를 켜도 **오리진 구간이 HTTP/1.1이면 TTFB 개선은 제한적**입니다. 이득은 핸드셰이크·HOL 제거에서 오므로, **정적 자산·다중 요청 페이지**에서 먼저 효과를 봅니다.

### 2.9 진단 도구와 명령

```bash
# 1) 협상된 프로토콜·cipher·체인 확인 (TLS 1.3 강제)
openssl s_client -connect api.example.com:443 -servername api.example.com \
  -tls1_3 -alpn h2 -showcerts </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# 2) ALPN 협상 결과와 세션 재개 여부
openssl s_client -connect api.example.com:443 -servername api.example.com \
  -tls1_3 -reconnect </dev/null 2>/dev/null | grep -E "Protocol|Reused|Cipher"

# 3) HTTP/3로 실제 요청 (curl 8.x, HTTP/3 빌드)
curl --http3 -sS -o /dev/null -w "%{http_version} %{time_appconnect} %{time_total}\n" \
  https://api.example.com/health

# 4) Nginx가 어떤 SSL 라이브러리로 빌드됐는지 (0-RTT 지원 판단)
nginx -V 2>&1 | tr ' ' '\n' | grep -E "http_v3|openssl|quic"

# 5) TLS 1.3 복호화(디버깅 전용) — 브라우저/클라이언트가 키 로그를 남기게
export SSLKEYLOGFILE=/tmp/sslkeys.log     # Wireshark: (Pre)-Master-Secret log filename
```

- `-reconnect`로 `Reused : ...`가 찍히면 세션 재개가 동작하는 것입니다. 재개가 안 되면 티켓 키·세션 캐시부터 확인합니다.
- `curl --http3`이 실패하면 `curl -V`에 `HTTP3` 기능이 있는지 확인합니다(배포판 기본 빌드는 대개 미포함).
- `SSLKEYLOGFILE`은 **민감 정보**입니다. 운영 환경에서 절대 켜지 말고, 재현 환경에서만 사용합니다.

### 2.10 체크리스트

- [ ] `openssl s_client`로 **중간 CA가 체인에 포함**되는지 확인했는가 (Java 클라이언트로 검증)
- [ ] TLS 1.2를 끊을 때 **고객사의 레거시 클라이언트**를 확인했는가
- [ ] 인증서 만료 **자동 갱신 + 만료 30일 전 알림**이 있는가 (갱신 후 티켓 키 로테이션 포함)
- [ ] `Alt-Svc`를 **HTTPS 응답에만** 내보내는가 (HTTP로 내보내면 다운그레이드 유도)
- [ ] 0-RTT가 **켜져 있다면** `Early-Data` 가드가 비멱등 요청을 425로 막는가
- [ ] 세션 티켓 키가 **인스턴스 간 공유**되는가, 로테이션 절차가 있는가
- [ ] 엣지에서 TLS 종료 시 **원 클라이언트 IP·프로토콜**을 신뢰 경계 안에서만 전달하는가
- [ ] mTLS 사용 시 **TLS 종료 지점과 인증 지점이 일치**하는가
- [ ] QUIC 배포 시 **TCP 폴백**이 실제로 동작하는가 (UDP 차단망 시뮬레이션)
- [ ] HSTS `max-age`를 늘리기 전에 **서브도메인 전체 HTTPS 준비**가 끝났는가(`includeSubDomains`는 되돌리기 어려움)

---

## 3. 기술 면접 / 실전 예상 질문 & 답변 팁

- **Q. TLS 1.3이 TLS 1.2보다 1-RTT 빠른 이유를 구조로 설명해 보세요.**
  - **A.** "TLS 1.2의 (EC)DHE는 서버가 파라미터를 먼저 정하고 클라이언트가 자기 몫을 계산해 보내는 2-RTT 구조였습니다. TLS 1.3은 클라이언트가 `key_share` 확장으로 **자기 키 교환 공개값을 ClientHello에 미리 실어** 보내고, 서버가 그 그룹을 수락하면 `ServerHello`부터 `Finished`까지 첫 비행에 모두 보냅니다. 그래서 핸드셰이크가 1-RTT이고, TCP 연결까지 합치면 3-RTT가 2-RTT가 됩니다. 서버가 그룹을 못 받아들이면 HelloRetryRequest로 1-RTT가 더 붙습니다."

- **Q. TLS 1.3에서 정적 RSA 키 교환을 없앤 이유는?**
  - **A.** "정적 RSA 키 교환은 클라이언트가 만든 세션키를 서버 공개키로 암호화해 보내는 방식이라, **서버 개인키가 나중에 유출되면 과거 트래픽이 전부 복호화**됩니다. 전방향 비밀성(PFS)이 없습니다. TLS 1.3은 (EC)DHE만 허용해 세션키가 **일회성 키 교환 결과**로만 만들어지게 했고, RSA는 인증서 서명(RSASSA-PSS) 용도로만 남겼습니다."

- **Q. 전방향 비밀성이 무엇이고, 어떻게 보장되나요?**
  - **A.** "세션키가 유출되거나 서버 장기 개인키가 유출돼도 **과거 세션의 트래픽은 복호화되지 않는** 성질입니다. ECDHE로 매 연결마다 임시 키 쌍을 만들고, 핸드셰이크가 끝나면 그 임시 개인키를 파기하기 때문에 성립합니다. 세션 재개에서도 `key_share`를 함께 보내 PSK만 쓰는 것을 피합니다."

- **Q. TLS 1.3의 키 스케줄을 설명해 보세요.**
  - **A.** "HKDF의 Extract와 Expand 두 단계로, Early Secret → Handshake Secret → Master Secret 순으로 비밀을 쌓습니다. 각 단계에서 `Derive-Secret`에 label을 붙여 용도별 비밀을 파생합니다 — `c hs traffic`, `s ap traffic`, `res master` 같은 식입니다. 트래픽 키는 이 비밀에서 key와 iv를 뽑고, 레코드마다 `iv XOR 시퀀스 번호`를 nonce로 씁니다. label로 용도를 분리하기 때문에 하나가 노출돼도 다른 키를 역산할 수 없습니다."

- **Q. TLS 1.3 cipher suite가 5개뿐인 이유는?**
  - **A.** "TLS 1.2의 cipher suite 이름에는 키 교환·인증·암호·MAC이 모두 들어 있었습니다(`ECDHE-RSA-AES128-GCM-SHA256`). TLS 1.3은 키 교환과 서명을 `supported_groups`·`signature_algorithms` 확장으로 분리하고, cipher suite는 **AEAD 암호와 HKDF 해시 조합**만 표현합니다. CBC·RC4 같은 취약 모드가 빠져 경우의 수가 줄었습니다."

- **Q. 0-RTT는 왜 위험하고, 어떻게 방어하나요?**
  - **A.** "0-RTT early data는 이전 세션의 PSK로만 암호화되어 **PFS가 없고, ServerHello 난수에 묶이지 않아 재전송(replay)이 가능**합니다. 공격자가 같은 0-RTT 요청을 복제하면 서버가 두 번 처리할 수 있어 중복 결제가 열립니다. 방어는 서버가 티켓을 1회용으로 쓰고, `obfuscated_ticket_age`로 신선도를 검사하고, 재사용을 캐시로 거부하는 것이고, 애플리케이션은 `Early-Data: 1` 헤더를 보고 **비멱등 요청에 425 Too Early**를 돌려줘 1-RTT로 재시도하게 합니다."

- **Q. HTTP/2의 HOL 블로킹과 HTTP/3의 차이를 설명해 보세요.**
  - **A.** "HTTP/2는 스트림을 겹쳐 애플리케이션 계층 HOL은 풀었지만, 아래가 **순서 보장 단일 TCP 바이트 스트림**이라 세그먼트 하나가 유실되면 다른 스트림 데이터도 커널 버퍼에서 대기합니다. RFC 9114도 이 점을 명시합니다. QUIC은 스트림별로 독립적인 순서·재전송을 제공해 **손실된 스트림만** 막히고, 혼잡 제어만 연결 전체에 공유합니다."

- **Q. QUIC이 TCP 대신 UDP를 쓰는 이유는?**
  - **A.** "TCP를 고치려면 커널과 **인터넷 미들박스**를 모두 바꿔야 합니다. 새 전송 프로토콜 번호나 TCP 옵션은 방화벽·NAT 장비가 막거나 변형하는 경우가 많습니다. UDP는 이미 통과되고, 그 위에 신뢰성·혼잡 제어·암호화를 **사용자 공간에서** 구현하면 배포와 진화가 빠릅니다. 대신 UDP 차단망을 위한 TCP 폴백이 필수입니다."

- **Q. QUIC의 연결 마이그레이션이 무엇이고 왜 가능한가요?**
  - **A.** "연결을 IP·포트 4-튜플이 아니라 **Connection ID**로 식별하기 때문에, Wi-Fi에서 LTE로 바뀌어 주소가 달라져도 핸드셰이크 없이 연결이 이어집니다. NAT 리바인딩에도 견디고, 경로 변경 시 `PATH_CHALLENGE`로 새 경로를 검증합니다. CID를 교체(`NEW_CONNECTION_ID`)해 경로 간 연결성을 추적당하지 않게 합니다."

- **Q. QUIC의 재전송이 TCP와 다른 점은?**
  - **A.** "TCP는 재전송 패킷의 번호가 원본과 같아 ACK가 어느 것에 대한 것인지 모호합니다. QUIC은 **패킷 번호를 절대 재사용하지 않고**, 같은 스트림 데이터를 **새 번호의 패킷**으로 다시 보냅니다. 그래서 RTT 측정이 정확하고 불필요한 재전송이 줄며, ACK 프레임에 지연 정보를 실어 보내 정밀한 손실 복구가 가능합니다."

- **Q. HTTPS에서 대칭키와 비대칭키는 각각 어디에 쓰이나요?**
  - **A.** "비대칭키는 **키 교환(ECDHE)과 신원 증명(인증서 서명)** 에, 대칭키는 **실제 데이터 암복호화**에 씁니다. 대칭키가 수백 배 빠르지만 당사자끼리 키를 안전하게 공유할 방법이 없어서, 그 키를 비대칭키로 안전하게 합의하는 구조입니다. 해시는 무결성, HMAC은 무결성+인증, 전자서명은 인증+부인방지에 씁니다."

- **Q. TLS 종료를 엣지에서 하는 것과 애플리케이션에서 하는 것의 트레이드오프는?**
  - **A.** "엣지 종료는 인증서 관리·HTTP/3·WAF·캐싱을 한곳에서 처리하고 오리진 부하를 줄이는 대신, **엣지-오리진 구간이 평문**이 되어 네트워크 격리나 재암호화가 필요합니다. 애플리케이션 종료는 종단 간 암호화와 mTLS가 쉽지만, 인스턴스마다 인증서·세션 티켓을 관리해야 하고 HTTP/3 같은 신규 전송은 엣지보다 지원이 늦습니다. 실무에서는 엣지에서 종료하고 오리진은 사설망으로 격리하는 구성이 일반적입니다."

- **Q. (공기업 정보통신 단골) SSL/TLS는 어느 계층이고, HTTPS와의 관계는?**
  - **A.** "TLS은 전송 계층(TCP) 위, 응용 계층(HTTP) 아래에서 동작하는 보안 프로토콜로, OSI 7계층으로는 5~6계층(세션·표현)에 위치시킵니다. HTTPS는 HTTP를 TLS 위에서 동작시킨 것이고 기본 포트는 443입니다. TLS은 기밀성·무결성·상호 인증을 제공하며, 인증서는 X.509 형식으로 CA가 서명하고, 폐기 여부는 CRL·OCSP로 확인합니다."

- **Q. PKI 구성 요소를 정리해 보세요.**
  - **A.** "인증서 발급·서명을 하는 **CA**, CA의 인증서를 서명하는 **Root CA**, 이용자를 등록·관리하는 **RA**, 인증서 저장소인 **디렉터리**, 폐기 목록 **CRL/OCSP**, 그리고 인증서와 개인키를 보관하는 **저장소**로 구성됩니다. 인증서에는 주체·공개키·유효기간·발급자·용도 제한이 담기고, CA의 개인키로 서명되어 신뢰 사슬을 만듭니다."

- **Q. 서비스에 HTTP/3를 도입한다면 무엇부터 확인하시겠습니까?**
  - **A.** "먼저 **종료 지점**입니다. ALB는 HTTP/3를 종료하지 않으므로 CloudFront나 Nginx에서 종료해야 합니다. 다음으로 **UDP 443 차단 비율과 TCP 폴백 동작**, 그리고 **지표**입니다 — QUIC 핸드셰이크 실패율, 재개율, CPU 사용률, 오리진 구간 프로토콜. 성능 이득은 핸드셰이크와 HOL 제거에서 오므로 다중 요청·정적 자산이 많은 경로부터 적용하고, 0-RTT는 재생 방어가 준비된 뒤에 켭니다."

---

## 참고

- RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3: https://www.rfc-editor.org/rfc/rfc8446
- RFC 8446 §2.3 (0-RTT Data), §7.1 (Key Schedule), §8.1 (Anti-Replay): https://www.rfc-editor.org/rfc/rfc8446#section-2.3
- RFC 8470 — Using Early Data in HTTP (425 Too Early, `Early-Data`): https://www.rfc-editor.org/rfc/rfc8470
- RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport: https://www.rfc-editor.org/rfc/rfc9000
- RFC 9001 — Using TLS to Secure QUIC: https://www.rfc-editor.org/rfc/rfc9001
- RFC 9002 — QUIC Loss Detection and Congestion Control: https://www.rfc-editor.org/rfc/rfc9002
- RFC 9114 — HTTP/3: https://www.rfc-editor.org/rfc/rfc9114
- RFC 9204 — QPACK: Field Compression for HTTP/3: https://www.rfc-editor.org/rfc/rfc9204
- RFC 9849 — TLS Encrypted Client Hello (2026-03): https://www.rfc-editor.org/rfc/rfc9849
- JEP 332 — Transport Layer Security (TLS) 1.3 (JDK 11, 0-RTT 비목표): https://openjdk.org/jeps/332
- nginx — Support for QUIC and HTTP/3: https://nginx.org/en/docs/quic.html
- nginx — Module ngx_http_v3_module: https://nginx.org/en/docs/http/ngx_http_v3_module.html
- AWS — CloudFront FAQ (HTTP/3 지원 범위): https://aws.amazon.com/cloudfront/faqs/
- spring-boot issue #47163 — Consider supporting HTTP/3 servers: https://github.com/spring-projects/spring-boot/issues/47163
