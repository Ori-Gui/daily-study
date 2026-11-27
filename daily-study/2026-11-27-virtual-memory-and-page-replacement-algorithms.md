# [2026-11-27] 가상 메모리와 페이지 교체 알고리즘 — OPT·FIFO·LRU·Clock 비교와 Belady의 이상현상, 스래싱·워킹셋, Redis·InnoDB LRU 실무

> **한 줄 핵심 요약**: 가상 메모리는 "프로세스마다 0번지부터 시작하는 넓은 주소 공간을 주고, 실제 물리 메모리는 필요한 페이지만 그때그때 붙여주는" 추상화이고, 그 대가로 **주소 변환(MMU·페이지 테이블·TLB)** 과 **페이지 폴트 처리**라는 비용이 붙습니다. 페이지 교체 알고리즘은 그 비용을 어떻게 줄일지의 문제인데, 공기업 전산직·정보처리기사에서는 **참조열(reference string)로 폴트 횟수를 세는 계산 문제**와 **Belady의 이상현상**으로, 실무에서는 **Redis `maxmemory-policy`·InnoDB buffer pool LRU·컨테이너 메모리 limit에서의 reclaim 지연**으로 같은 개념이 등장합니다. 정확한 LRU는 이론의 기준선일 뿐 OS가 그대로 쓰지 않고, **참조 비트 하나로 근사한 Clock(2차 기회)이 실무 표준**이라는 점이 두 영역을 잇는 핵심입니다.

---

## 1. 개념 및 동작 원리

### 1.1 가상 메모리가 해결하는 세 가지 문제

가상 메모리는 "물리 메모리보다 큰 프로그램을 돌린다"는 한 줄로 요약되지만, 실제로는 서로 다른 세 문제를 한 번에 해결합니다.

| 문제 | 가상 메모리 없는 세계 | 가상 메모리가 주는 것 |
| --- | --- | --- |
| 물리 메모리 한계 | 8GB RAM에서 12GB 프로그램 불가 | 실행 중 필요한 페이지만 적재(demand paging), 나머지는 디스크·파일 |
| 프로세스 격리 | 모든 프로세스가 물리 주소를 직접 공유 → 한 프로세스가 남의 메모리 덮어쓰기 가능 | 프로세스별 독립 주소 공간 + 페이지 단위 보호 비트 |
| 재배치·링킹 | 실행 시 물리 주소를 미리 확정해야 함 | 링크 시점엔 가상 주소만 확정, 실제 배치는 커널이 결정 |

- **핵심은 "지연"입니다.** 메모리 할당을 프로그램 로드 시점이 아니라 **첫 접근 시점**으로 미루고, 그 지연을 감당할 근거는 **지역성(locality)** 입니다. 시간적 지역성(방금 쓴 것을 또 쓴다)과 공간적 지역성(인접한 것을 곧 쓴다)이 성립하지 않는 프로그램에서는 가상 메모리의 성능 이점이 사라집니다.
- **가상 메모리 ≠ 스왑.** 가상 메모리는 주소 변환 추상화이고, 스왑은 그 추상화를 유지하기 위한 백킹 스토어 중 하나입니다. 파일에 매핑된 페이지(mmap, 실행 파일 코드)는 스왑 없이도 파일 자체가 백킹 스토어입니다.

### 1.2 주소 변환 — MMU, 페이지 테이블, TLB

CPU가 내는 주소는 **가상 주소**이고, 메모리 컨트롤러가 받는 주소는 **물리 주소**입니다. 사이에서 변환하는 하드웨어가 **MMU**입니다.

```text
가상 주소 (48비트, x86-64 4단계 페이징)
┌────────┬────────┬────────┬────────┬────────────┐
│ PML4   │ PDPT   │ PD     │ PT     │ offset     │
│ 9비트  │ 9비트  │ 9비트  │ 9비트  │ 12비트     │
└────────┴────────┴────────┴────────┴────────────┘
   ↓        ↓        ↓        ↓        4KB 페이지 내 위치
 각 단계마다 페이지 테이블의 엔트리(PTE, 8바이트) 하나를 읽는다
```

| 요소 | 역할 | 비용 |
| --- | --- | --- |
| 페이지 테이블(PTE) | VPN → PFN 매핑, 유효(valid) 비트, 보호 비트, dirty/accessed 비트 | 메모리 접근 1회당 최대 단계 수만큼 추가 접근 |
| TLB | 최근 변환 결과를 담는 소형 연관 캐시(수십~수천 엔트리) | 접근당 1사이클 수준, 미스 시 페이지 워크 |
| MMU | 변환 + 권한 검사, 위반 시 예외(#PF) | 하드웨어, 무료에 가까움 |

**단일 페이지 테이블이 불가능한 이유(계산).** 48비트 가상 주소, 4KB 페이지, 8바이트 PTE라면 페이지 수는 2^36개, 테이블 크기는 **512GB**입니다. 그래서 단계를 나눠 실제로 쓰는 부분만 테이블을 둡니다(4단계 × 2^9 엔트리 = 각 테이블 4KB로 한 페이지에 딱 맞음).

**TLB가 왜 결정적인가(EAT 계산).** 메모리 접근 100ns, TLB 조회 10ns, 1단계 페이지 테이블, 적중률 h라고 하면

```text
적중 시: 10 + 100                = 110ns
미스 시: 10 + 100(PTE) + 100(데이터) = 210ns
EAT = h·110 + (1−h)·210
h = 0.98  → 107.8 + 4.2  = 112.0ns     (TLB 없을 때 100ns 대비 +12%)
h = 0.999 → 109.89 + 0.21 = 110.1ns    (+10%)

4단계 페이징에서 미스 시: 10 + 4×100 + 100 = 510ns
h = 0.999 → 0.999·110 + 0.001·510 = 110.4ns  ← 적중률이 높으면 단계 수는 거의 무의미
```

- **결론: TLB 적중률이 99%대에서 무너지면 다단계 페이징 비용이 곧바로 드러납니다.** 그래서 페이지 크기를 키우거나(2MB huge page) 대형 페이지를 쓰면 같은 TLB 엔트리로 커버하는 범위(**TLB reach**)가 커집니다. 엔트리 1,536개 × 4KB ≈ 6MB가 2MB 페이지에서는 ≈ 3GB가 됩니다.
- TLB는 **문맥 교체 시 무효화(ASID 미사용 시 flush)** 되고, 페이지 테이블을 고치면 해당 엔트리를 무효화해야 합니다(`invlpg`). 커널이 페이지 테이블을 자주 고치는 워크로드에서 TLB shootdown이 비용이 되는 이유입니다.

### 1.3 페이지 폴트 — 정상 동작이지 예외 상황이 아니다

페이지 폴트는 "메모리에 없는 페이지를 건드렸다"는 뜻이고, **정상적인 요구 페이징의 일부**입니다. 치명적인 것은 접근 자체가 잘못된 경우(SIGSEGV)입니다.

```text
① CPU 접근 → MMU가 PTE 확인 → valid=0 (또는 권한 위반)
② #PF 예외 → 커널 do_page_fault()
③ 주소가 프로세스의 VMA 안인가?
   └ 아니면 SIGSEGV (segmentation fault)
④ 권한이 맞는가? (읽기 전용 페이지에 쓰기 → COW 처리 또는 SIGSEGV)
⑤ 빈 프레임 확보 (free list / page cache / swap-in)
   └ 프레임이 없으면 → 페이지 교체 알고리즘으로 victim 선택
       └ victim이 dirty면 디스크로 write-back (이게 진짜 느린 부분)
⑥ 디스크/파일에서 페이지 읽기, PTE 갱신, TLB 무효화
⑦ 폴트를 일으킨 명령어를 재실행 (restart)
```

| 구분 | 의미 | 비용 |
| --- | --- | --- |
| minor fault | 페이지가 이미 메모리에 있음(공유 페이지, page cache, zero page, COW) | 수 µs (I/O 없음) |
| major fault | 디스크·스왑에서 읽어야 함 | 수백 µs ~ 수십 ms |
| protection fault | 권한 위반. COW면 복사 후 재실행, 아니면 SIGSEGV | COW는 minor, 나머지는 프로세스 종료 |

- **COW(Copy-on-Write)** 는 `fork()` 직후 부모·자식이 같은 물리 페이지를 읽기 전용으로 공유하다가, 쓰기 시점에 protection fault로 복사하는 기법입니다. "fork는 가볍다"는 말의 근거가 이것이고, 반대로 자식이 곧바로 대량 쓰기를 하면 minor fault가 폭발합니다(Redis `BGSAVE` 포크 후 부하가 튀는 전형적 원인).
- **요구 페이징 + 오버커밋**으로 "할당은 성공했는데 나중에 터지는" 상황이 생깁니다. 리눅스의 `vm.overcommit_memory` 정책과 OOM killer, 그리고 자바에서 `OutOfMemoryError` 대신 프로세스가 죽는 경우가 여기에 해당합니다.

### 1.4 페이지 교체 알고리즘 — 무엇을 근거로 victim을 고르는가

빈 프레임이 없을 때, 어떤 페이지를 내보낼지 결정하는 규칙입니다. 평가 기준은 **폴트율**과 **구현 비용**입니다.

| 알고리즘 | victim 선택 근거 | 구현 비용 | Belady 이상현상 | 스택 알고리즘 |
| --- | --- | --- | --- | --- |
| **OPT / MIN**(Belady) | 앞으로 가장 늦게 참조될 페이지 | 불가능(미래를 알아야 함) — 이론적 하한 | 없음 | 예 |
| **FIFO** | 가장 먼저 적재된 페이지 | 매우 낮음(큐 하나) | **발생** | 아니오 |
| **LRU** | 가장 오래 참조되지 않은 페이지 | 높음(참조마다 순서 갱신, 락 필요) | 없음 | 예 |
| **Clock / 2차 기회** | 참조 비트 0인 첫 페이지(1이면 0으로 지우고 통과) | 낮음(하드웨어 참조 비트 + hand 포인터) | 드물게 발생 가능 | 아니오 |
| **Enhanced 2차 기회** | (참조, dirty) 4등급 중 최하위부터 | 낮음 | 가능 | 아니오 |
| **LFU** | 참조 횟수가 가장 적은 페이지 | 중간(카운터) | 가능 | 아니오 |
| **Random** | 무작위 | 매우 낮음 | — | 아니오 |

#### (1) OPT — 채점의 기준선, 실무의 벤치마크

미래 참조열을 알고 있다고 가정하고 **가장 멀리 있는 페이지를 축출**합니다. 실현 불가능하지만, "이 워크로드에서 교체 정책이 가질 수 있는 최선"을 알려주는 **하한선**으로 씁니다. 실무에서는 실제 트레이스(예: Redis `redis-cli --lru-test`, InnoDB 접근 패턴 로그)에 정책별 시뮬레이션을 돌려 OPT 대비 폴트율을 비교하는 식으로 사용합니다.

#### (2) FIFO — 구현은 쉽지만 순서를 오해한다

적재 시각만 봅니다. 참조가 일어나도 큐 위치가 바뀌지 않으므로 **자주 쓰는 페이지가 오래 있었다는 이유로 축출**됩니다. 그리고 프레임을 늘렸는데 폴트가 늘어나는 **Belady의 이상현상**이 발생합니다.

```text
참조열 1,2,3,4,1,2,5,1,2,3,4,5  (FIFO, 페이지 = 프레임, 왼쪽이 가장 오래된 것)

[프레임 3개] 총 9회 폴트
 ①1 미스 [1]        ②2 미스 [1,2]      ③3 미스 [1,2,3]
 ④4 미스 1 축출 [2,3,4]   ⑤1 미스 2 축출 [3,4,1]   ⑥2 미스 3 축출 [4,1,2]
 ⑦5 미스 4 축출 [1,2,5]   ⑧1 히트   ⑨2 히트
 ⑩3 미스 1 축출 [2,5,3]   ⑪4 미스 2 축출 [5,3,4]   ⑫5 히트

[프레임 4개] 총 10회 폴트   ← 프레임을 늘렸는데 더 나빠졌다
 ①②③④ 1,2,3,4 적재(4회)
 ⑤1 히트  ⑥2 히트
 ⑦5 미스 1 축출 [2,3,4,5]    ⑧1 미스 2 축출 [3,4,5,1]
 ⑨2 미스 3 축출 [4,5,1,2]    ⑩3 미스 4 축출 [5,1,2,3]
 ⑪4 미스 5 축출 [1,2,3,4]    ⑫5 미스 1 축출 [2,3,4,5]
```

프레임이 늘어난 쪽이 오히려 뒤쪽 참조를 전부 미스로 만듭니다. FIFO 큐의 "나이"만 보고 고르므로, 새로 들어온 페이지의 재사용 가능성을 전혀 고려하지 못하는 것이 원인입니다.

#### (3) LRU — 이론적으로 좋고, Belady가 없다

"가장 오래 참조되지 않은 것"을 축출합니다. LRU는 **스택 알고리즘**입니다: 프레임 n개일 때의 메모리 내용이 항상 프레임 n+1개일 때의 부분집합이므로, 프레임을 늘리면 폴트가 **절대 늘지 않습니다**. 실측 예시(위 참조열 대신 고전 20개 참조열)에서 FIFO 15, LRU 12, OPT 9로 나옵니다.

```text
참조열 7,0,1,2,0,3,0,4,2,3,0,3,2,1,2,0,1,7,0,1  (프레임 3개)
FIFO 15회 / LRU 12회 / OPT 9회
```

LRU의 문제는 **"참조할 때마다" 순서를 갱신해야 한다**는 점입니다. 소프트웨어로는 해시맵 + 이중 연결 리스트로 O(1)에 구현되지만, 매 메모리 접근마다 리스트를 만지는 것은 락 경합과 캐시 오염을 만듭니다. 하드웨어가 "마지막 접근 시각"을 자동으로 기록해주지 않기 때문에 OS는 정확한 LRU를 쓸 수 없습니다.

#### (4) Clock(2차 기회) — 실무에서 실제로 쓰는 LRU 근사

각 PTE의 **참조 비트(reference bit)** 만 하드웨어가 자동으로 세팅합니다. 페이지들을 원형 리스트로 두고 hand를 돌립니다.

```text
hand가 가리키는 페이지의 ref bit 검사
  ref = 1 → 0으로 지우고 다음으로 (한 번의 기회를 준다)
  ref = 0 → 이 페이지를 축출

예: [A(ref1), B(ref1), C(ref0)]  hand→A
1회전: A(1→0) → B(1→0) → C(0) → C 축출
```

- 최악의 경우 한 번의 폴트에 리스트 전체를 훑지만, **분할 상환하면 페이지당 O(1)** 입니다.
- **Enhanced 2차 기회**는 (ref, dirty)로 4등급을 나눠 **(0,0) → (0,1) → (1,0) → (1,1)** 순으로 축출합니다. dirty 페이지를 내보내면 write-back이 필요하므로, 같은 조건이면 clean을 먼저 내보내는 것이 이득입니다. 이것이 리눅스 페이지 회수(active/inactive 리스트)의 기본 사고방식입니다.

#### (5) LFU가 실무 캐시에서 자주 실패하는 이유

LFU는 "인기 있는 페이지"를 남기지만, **인기가 식은 페이지가 카운터 때문에 영원히 남습니다**(cache pollution). SNS 타임라인의 옛 화제글, 한 번 터진 검색어가 대표적입니다. 그래서 Redis의 LFU는 8비트 카운터에 **로그 증가 + 주기적 감쇠(decay)** 를 붙여 "최근 인기"를 근사합니다.

### 1.5 스래싱과 워킹셋 — 폴트가 폴트를 부르는 구간

```text
다중 프로그래밍 정도 ↑ → 각 프로세스의 프레임 ↓ → 폴트율 ↑
→ 폴트 처리에 CPU를 쓰느라 실제 작업 ↓ → CPU 이용률 ↓
→ OS가 "CPU가 놀고 있다"고 판단 → 다중 프로그래밍 정도 ↑↑  (악순환)
```

이 구간에 들어가면 디스크가 100%이고 CPU는 20~30%인 상태가 지속됩니다. 고전적 대응은 **다중 프로그래밍 정도를 낮추는 것**이고, 현대적 대응은 **프로세스별 워킹셋을 보장**하는 것입니다.

| 기법 | 정의 | 판단/동작 |
| --- | --- | --- |
| 워킹셋(WS) | 최근 Δ 시간 동안 참조된 페이지 집합 | WSS_i를 주기적으로 측정 |
| 워킹셋 총합 D | D = Σ WSS_i | **D > 물리 프레임 수 → 스래싱**, 일부 프로세스 suspend |
| PFF(페이지 폴트 빈도) | 폴트 간격이 짧으면 프레임 추가, 길면 회수 | 상·하한 임계값으로 제어 |
| 지역성 강화 | 페이지 크기·배치 조정, 접근 패턴 개선 | 애플리케이션 레벨 대응 |

**EAT로 보는 폴트의 위력.** 메모리 접근 200ns, 폴트 서비스 8ms(스왑 SSD), 폴트율 p라면

```text
EAT = (1−p)·200 + p·8,000,000
p = 0.001(0.1%) → 199.8 + 8,000 ≈ 8.2µs   → 약 41배 느려짐
p = 0.00001     → 200 + 80 ≈ 280ns        → 1.4배

성능 저하를 10% 이내로 막으려면:
p ≤ 0.1·m / (s − m) = 20 / 7,999,800 ≈ 2.5 × 10⁻⁶   → 100만 번 중 2.5회
```

**이 계산이 실무 감각의 근거입니다.** "폴트율 1%면 괜찮겠지"는 성립하지 않습니다. 폴트율은 **0.001% 수준**을 목표로 봐야 하고, 그래서 캐시·DB의 hit ratio를 99%대에서 관리합니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 계산 문제에서 실제로 틀리는 지점

| 함정 | 정확한 처리 |
| --- | --- |
| 빈 프레임에 처음 채우는 것 | **폴트로 센다** (페이지가 없었으므로 당연히 폴트) |
| 히트 시 순서 갱신 | **LRU는 갱신, FIFO는 갱신 안 함** — 여기서 답이 갈린다 |
| OPT의 동점(tie) | 미래에 아예 안 쓰이는 페이지가 여러 개면 축출 순서에 따라 이후가 달라짐 → 채점 기준 확인, 서술형에서는 "동점 처리 가정"을 명시 |
| 페이지 교체 vs 폴트 | 프레임이 남아 있으면 교체 없이 적재. "교체 횟수"를 물으면 폴트에서 초기 적재분을 뺀다 |
| Belady 이상현상 | **FIFO에서 발생**, LRU·OPT는 스택 알고리즘이라 불가 |
| 참조 비트 문제(Clock) | hand 위치와 ref bit 초기값을 문제에서 준 대로 시작, 1회전 시 지운 비트를 되살리지 않는다 |
| 프레임 1개 극단값 | 참조열이 A,B,A,B…면 폴트 = 참조 횟수 |
| 페이지 크기 변경 | 폴트 수는 줄지만 **내부 단편화와 전송 시간**이 늘어남 (아래 2.6) |

**손으로 세는 것보다 시뮬레이션으로 검산하는 습관.** 위 본문 수치는 아래 코드로 검증했습니다.

```python
# page_replacement.py — 참조열 폴트 횟수 검산 (FIFO / LRU / OPT)
def fifo(ref, n):
    frames, queue, faults = [], [], 0
    for page in ref:
        if page in frames:
            continue
        faults += 1
        if len(frames) < n:
            frames.append(page); queue.append(page)
        else:
            victim = queue.pop(0); frames.remove(victim)
            frames.append(page); queue.append(page)
    return faults

def lru(ref, n):
    frames, faults = [], 0
    for page in ref:                       # frames[-1] = 가장 최근 참조
        if page in frames:
            frames.remove(page); frames.append(page)
            continue
        faults += 1
        if len(frames) < n:
            frames.append(page)
        else:
            frames.pop(0); frames.append(page)
    return faults

def opt(ref, n):
    frames, faults = [], 0
    for i, page in enumerate(ref):
        if page in frames:
            continue
        faults += 1
        if len(frames) < n:
            frames.append(page); continue
        nxt = {p: (ref.index(p, i + 1) if p in ref[i + 1:] else float("inf")) for p in frames}
        frames.remove(max(frames, key=lambda p: nxt[p])); frames.append(page)
    return faults

ref = [7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1]
print(fifo(ref, 3), lru(ref, 3), opt(ref, 3))      # 15 12 9
belady = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]
print(fifo(belady, 3), fifo(belady, 4))            # 9 10  ← Belady 이상현상
```

### 2.2 "정확한 LRU"를 쓰지 않는 이유와 대체재

- **OS**: 참조 비트만 하드웨어가 기록 → Clock/2차 기회, 리눅스는 active/inactive 리스트 2단 구조로 회수합니다.
- **애플리케이션 캐시**: 정확한 LRU는 매 접근마다 전역 자료구조를 잠가야 해서, 다중 스레드에서 **캐시 자체가 병목**이 됩니다. Java에서 LRU를 직접 구현할 때는 다음이 가장 간단합니다.

```java
// accessOrder=true → 접근 순서 유지 (LRU)
Map<String, byte[]> cache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, byte[]> eldest) {
        return size() > MAX_ENTRIES;      // 스레드 안전하지 않음 → 외부 동기화 필요
    }
};
// 실무에서는 Caffeine: W-TinyLFU(빈도 + 최근성) + 비동기 만료, 세그먼트 단위 동시성
```

- **W-TinyLFU**가 LRU보다 나은 이유: 100% 히트를 반복하는 오래된 인기 항목이 새로 유입된 항목을 밀어내지 못하게, **빈도 스케치(작은 카운터 테이블)** 로 신규 항목에 최소한의 기회를 줍니다(admission window).
- **순차 스캔이 섞인 워크로드**에서는 LRU가 최악입니다. 대량 full scan이 캐시를 한 번에 밀어내기 때문이고, 이때 **MRU·2Q·midpoint 삽입**이 답입니다(다음 절).

### 2.3 Redis `maxmemory-policy` — 이름이 정확한 LRU가 아니다

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru     # 또는 volatile-lru / allkeys-lfu / volatile-ttl / noeviction
maxmemory-samples 5              # 근사 정확도 ↔ CPU 트레이드오프
```

| 정책 | 축출 대상 | 주의 |
| --- | --- | --- |
| `noeviction`(기본) | 축출 없음 | 한도 도달 시 **쓰기 명령이 OOM 에러**, 읽기는 계속 동작 |
| `allkeys-lru` | 모든 키 | TTL 없는 키도 축출 — 캐시 용도로는 이것이 안전 |
| `volatile-lru` | **TTL이 설정된 키만** | TTL 없는 키만 있으면 축출 후보가 없어 OOM 에러. "TTL 걸어뒀는데 왜 OOM?"의 정체 |
| `allkeys-lfu` / `volatile-lfu` | 접근 빈도 | 8비트 카운터 + 로그 증가 + `lfu-decay-time` 감쇠 |
| `allkeys-random` | 무작위 | 히트율이 워크로드에 좌우, 예측 어려움 |

- **Redis의 LRU는 근사**입니다: 키마다 24비트 시계만 기록하고, 축출 시 **`maxmemory-samples`개를 무작위 표본 추출해 그중 가장 오래된 것**을 버립니다. 표본 수를 10으로 올리면 정확 LRU에 가까워지지만 축출 시 CPU를 더 씁니다.
- **복제본에서는 축출이 일어나지 않습니다.** 마스터가 결정해 DEL을 전파하고, 복제본은 `maxmemory`를 넘겨도 그대로 보관합니다(축출 여유분이 없으면 복제 지연·재연결 실패). 클러스터에서는 **`maxmemory`가 노드(샤드)별 값**이라 키 분포가 치우치면 특정 노드만 먼저 OOM이 납니다.
- **축출 폭풍 + 스탬피드**가 겹치면 DB가 죽습니다. 대량 축출 직후 같은 키에 대한 요청이 몰리므로, TTL 지터, single-flight(캐시 미스 1건만 DB 조회), 미스 시 짧은 백오프를 함께 설계해야 합니다.

### 2.4 InnoDB buffer pool — LRU에 midpoint를 넣은 이유

InnoDB는 전통 LRU를 그대로 쓰지 않고 **new sublist(5/8) / old sublist(3/8)** 로 나눕니다.

| 파라미터 | 기본값 | 의미 |
| --- | --- | --- |
| `innodb_buffer_pool_size` | — | 전체 메모리의 60~80% 수준이 일반적(단일 인스턴스 기준) |
| `innodb_buffer_pool_instances` | 8 (버퍼풀이 1GB 미만이면 1, 최대 64) | 버퍼풀 mutex 분할 |
| `innodb_old_blocks_pct` | 37 | old sublist 비율(%) |
| `innodb_old_blocks_time` | 1000(ms) | 새로 적재된 페이지가 이 시간 안에 다시 접근되면 new로 승격하지 않음 |

- **메커니즘**: 새 페이지는 old sublist의 **head(midpoint)** 에 삽입되고, 일정 시간이 지난 뒤 다시 접근돼야 new sublist로 승격됩니다. 그래서 **한 번만 훑고 버려질 페이지(full table scan)** 가 new sublist를 오염시키지 못합니다.
- **전형적 실패**: `innodb_old_blocks_time=0`으로 두고 대량 배치를 돌리면 hit ratio가 급락하고 디스크 읽기가 폭증합니다. 반대로 시간이 너무 길면 실제로 자주 쓰는 페이지의 승격이 늦어져 초기 워밍업이 느려집니다.
- hit ratio는 `SHOW ENGINE INNODB STATUS`의 `Buffer pool hit rate`(OLTP 기준 99% 이상) 또는 `performance_schema`의 `innodb_buffer_pool_read_requests` 대비 `innodb_buffer_pool_reads`로 봅니다.
- **버퍼풀과 OS page cache의 이중 캐싱**은 메모리를 두 번 쓰는 구조입니다. 대부분 기본 설정에서는 OS 캐시를 우회(O_DIRECT 계열)하거나, 버퍼풀을 충분히 크게 잡아 OS 캐시 역할을 줄입니다.

### 2.5 스왑·메모리 limit 환경의 엣지 케이스

| 상황 | 현상 | 대응 |
| --- | --- | --- |
| DB·캐시 서버의 swap 사용 | p99 지연이 수십 ms~수백 ms로 튐(스왑인은 major fault) | `vm.swappiness` 낮게(1~10), 힙·버퍼풀을 물리 메모리 안에 확보. **0으로 두면 회수할 게 없을 때 OOM이 급격해짐** |
| JVM 힙이 swap으로 나감 | GC pause가 수십 배, STW가 초 단위 | `-Xms`=`-Xmx`, `-XX:+AlwaysPreTouch`, 컨테이너 limit 내로 힙 상한 설정(`MaxRAMPercentage`) |
| 컨테이너 memory limit | 한도 근처에서 **direct reclaim**으로 지연 스파이크, 초과 시 OOM kill | `memory.stat`의 `pgscan`/`pgsteal`, `workingset_refault` 확인. limit은 실사용 working set의 1.5배 이상 |
| JVM 컨테이너 메모리 산정 | 힙 + Metaspace + 스레드 스택 + 코드 캐시 + direct buffer가 limit을 넘으면 kill | limit = 힙 + 25~30% 여유. `container_memory_working_set_bytes`를 기준 지표로 사용 |
| 대용량 파일 I/O | page cache가 늘어 limit 근처 | cgroup v2는 page cache를 회수 대상으로 계산 — "메모리 사용량이 limit에 붙어 있다"가 곧 위험은 아님(회수 가능분 확인) |

**진단 명령.**

```bash
vmstat 1 5                    # si/so(스왑 I/O), cs, us/sy 비율
pidstat -r -p <pid> 1         # minflt/s, majflt/s 분리 확인
perf stat -e page-faults,minor-faults,major-faults -p <pid> sleep 10
grep -E 'pgmajfault|pswpin|pswpout' /proc/vmstat
cat /proc/<pid>/smaps_rollup  # Rss, Pss, Swap — 프로세스가 실제로 스왑에 내려갔는지
sar -B 1 5                    # pgpgin/pgpgout, majflt/s
```

`majflt/s`가 0에 가까운데 지연이 튀면 페이지 폴트가 아니라 **GC·락·네트워크**를 봐야 합니다. 폴트는 원인 후보 중 하나일 뿐입니다.

### 2.6 페이지 크기를 키우면 좋아지는 것과 나빠지는 것

| 항목 | 4KB → 2MB(대형 페이지) |
| --- | --- |
| TLB reach | 크게 증가 (엔트리 수 × 페이지 크기) |
| 페이지 테이블 크기 | 감소(단계 수 감소) |
| 폴트 횟수 | 감소 |
| 내부 단편화 | **증가** — 2MB보다 작은 할당이 메모리를 낭비 |
| 스왑·I/O 단위 | 커져서 작은 랜덤 읽기에 불리 |
| 스캔·대형 배열 | 유리 (순차 접근, TLB 미스 감소) |

- 리눅스 THP(Transparent Huge Page)는 `always`/`madvise`/`never`로 제어합니다. `always`는 `khugepaged`의 압축 작업으로 지연 스파이크와 메모리 팽창이 생길 수 있어, DB·JVM 서버에서는 `madvise`(명시적으로 요청한 영역만) 또는 `never`를 권하는 경우가 많습니다. 판단 기준은 "TLB 미스가 병목인가"입니다.
- **페이징 vs 세그멘테이션**(필기 단골 비교).

| 구분 | 페이징 | 세그멘테이션 |
| --- | --- | --- |
| 분할 단위 | 고정 크기(보통 4KB) | 논리 단위(코드·데이터·스택), 가변 |
| 주소 형식 | 페이지 번호 + offset | 세그먼트 번호 + offset (한계 길이 검사 필요) |
| 단편화 | **내부 단편화** | **외부 단편화** |
| 사용자 인식 | 없음(투명) | 있음(논리 구조 노출) |
| 공유·보호 | 페이지 단위, 공유 시 주소 정렬 필요 | 논리 단위로 자연스러움 |
| 하드웨어 | 페이지 테이블, TLB | 세그먼트 테이블(기준·한계 레지스터), TLB |

### 2.7 공기업 전산직·정보처리기사 대비 정리

- 필기에서 반복되는 형태는 **(1) 참조열 폴트 횟수 계산, (2) Belady 이상현상 판별, (3) EAT 계산, (4) 워킹셋·스래싱 원인 서술, (5) 페이징 vs 세그멘테이션 비교** 다섯 가지입니다.
- 암기 우선순위: **OPT(미래) → FIFO(적재 순) → LRU(참조 순) → LFU(빈도) → Clock(참조 비트)**. 각 알고리즘의 "근거가 되는 정보"만 정확히 붙여두면 계산 문제는 시뮬레이션으로 풀립니다.
- 서술형 포인트: "가상 메모리의 목적 2가지 이상 + 페이지 폴트 처리 절차 5단계 + 스래싱의 악순환과 워킹셋 해결"은 통째로 쓸 수 있게 준비해 두는 것이 효율적입니다.

---

## 3. 기술 면접 / 실전 예상 질문 & 답변 팁

- **Q. 가상 메모리를 쓰는 이유를 설명해 보세요.**
  - **A.** "세 가지입니다. 첫째, 물리 메모리보다 큰 주소 공간을 프로세스에 주고 실제로 쓰는 페이지만 적재해 메모리 한계를 넘습니다. 둘째, 프로세스마다 독립된 주소 공간을 주므로 한 프로세스가 다른 프로세스 메모리를 침범할 수 없고 페이지 단위 권한 검사가 가능합니다. 셋째, 링크 시점에 물리 주소를 확정하지 않아도 되어 재배치와 공유가 쉬워집니다. 이 모든 이점은 지역성이 성립한다는 전제 위에 있고, 지역성이 깨지면 페이지 폴트가 폭발해 오히려 느려집니다."

- **Q. 페이지 폴트가 발생하면 커널에서 무슨 일이 일어나나요?**
  - **A.** "MMU가 PTE의 valid 비트를 보고 예외를 올리면 커널이 폴트 주소가 해당 프로세스의 VMA 안에 있는지 확인합니다. 밖이면 SIGSEGV입니다. 안이면 권한을 확인하고, COW 대상이면 복사 후 재실행합니다. 아니면 빈 프레임을 확보하고, 없으면 교체 알고리즘으로 victim을 고릅니다. victim이 dirty면 디스크에 먼저 쓰고, 요청 페이지를 읽어 PTE를 갱신한 뒤 명령어를 재실행합니다. 핵심은 폴트가 예외가 아니라 정상 경로이고, 비용의 대부분은 디스크 I/O와 write-back이라는 점입니다."

- **Q. LRU가 최적인데 OS는 왜 정확한 LRU를 쓰지 않나요?**
  - **A.** "매 메모리 접근마다 순서를 갱신해야 하는데, 하드웨어가 접근 시각을 자동으로 남겨주지 않습니다. 소프트웨어로 유지하면 접근마다 자료구조를 만져야 해서 폴트 처리 비용보다 관리 비용이 커집니다. 대신 PTE의 참조 비트를 하드웨어가 공짜로 세팅해 주므로, 이를 이용해 LRU를 근사하는 Clock(2차 기회)을 씁니다. 성능은 LRU에 근접하고 비용은 페이지당 O(1)입니다. 리눅스는 active/inactive 리스트로 한 단계 더 근사합니다."

- **Q. Belady의 이상현상이 뭔가요?**
  - **A.** "프레임 수를 늘렸는데 페이지 폴트가 오히려 증가하는 현상으로 FIFO에서 발생합니다. 참조열 1,2,3,4,1,2,5,1,2,3,4,5에서 3프레임은 9회, 4프레임은 10회입니다. 원인은 FIFO가 프레임 수와 무관하게 적재 순서만 보기 때문입니다. LRU와 OPT는 스택 알고리즘이라 n프레임의 페이지 집합이 항상 n+1프레임의 부분집합이므로 이 현상이 발생하지 않습니다."

- **Q. 그럼 실무에서 LRU와 가까운 정책을 쓰는 곳을 예로 들면?**
  - **A.** "Redis의 `allkeys-lru`입니다. 다만 정확한 LRU가 아니라 키에 24비트 시계만 남기고, 축출 시 `maxmemory-samples`개를 표본 추출해 가장 오래된 것을 버리는 근사입니다. MySQL InnoDB는 여기에 한 가지를 더해서, 새 페이지를 LRU의 중간(midpoint)에 넣고 일정 시간 뒤 재접근돼야 승격시킵니다. full table scan이 버퍼풀을 한 번에 밀어내는 것을 막는 장치입니다."

- **Q. Redis가 TTL을 걸었는데 OOM 에러가 납니다. 왜죠?**
  - **A.** "`maxmemory-policy`가 `volatile-lru` 계열이면 축출 대상이 TTL이 있는 키로 한정됩니다. TTL 없는 키가 메모리의 대부분을 차지하면 축출 후보가 없어 쓰기가 실패합니다. 확인은 `INFO memory`의 `used_memory`와 `db0:expires` 비율, `INFO stats`의 `evicted_keys`가 0인지 보면 됩니다. 캐시 용도면 `allkeys-lru`로 바꾸는 것이 맞고, 영속 데이터가 섞여 있으면 키 공간을 인스턴스로 분리해야 합니다."

- **Q. EAT 계산 문제를 빠르게 푸는 법은?**
  - **A.** "두 공식을 구분해서 씁니다. TLB는 `h(t+m) + (1−h)(t+k·m+m)`이고 k는 페이지 테이블 단계 수입니다. 페이지 폴트는 `(1−p)m + p·s`입니다. 예를 들어 m=200ns, s=8ms, p=0.1%면 8.2µs로 41배가 됩니다. 여기서 얻을 결론은 폴트율을 체감으로 판단하면 안 되고, 10% 이내 저하를 목표로 하면 100만 회당 2~3회 수준이어야 한다는 것입니다."

- **Q. 스래싱은 어떻게 진단하고 대응하나요?**
  - **A.** "CPU 이용률은 낮은데 디스크 I/O와 폴트가 높고, 프로세스들이 조금씩만 진행하는 상태입니다. `vmstat`의 si/so와 `pgmajfault`, `sar -B`의 majflt/s로 확인합니다. 원인은 다중 프로그래밍 정도가 보장 가능한 프레임보다 큰 것이므로, 워킹셋 총합을 측정해 초과하면 일부 프로세스를 중단하거나 동시 요청 수를 줄입니다. 애플리케이션 레벨에서는 페이지 크기·접근 패턴·캐시 히트율을 조정합니다. 서버라면 스왑을 줄이고 working set을 메모리 안에 넣는 것이 우선입니다."

- **Q. 컨테이너 메모리 limit에 붙어 있는데 왜 죽지 않나요?**
  - **A.** "cgroup v2의 memory 사용량에는 page cache가 포함되지만 이것은 회수 가능분입니다. 한도에 가까우면 커널이 reclaim을 돌려 회수하고, 회수할 anon 메모리가 부족할 때 OOM kill이 일어납니다. 판단은 `container_memory_working_set_bytes`(usage − inactive_file)와 `memory.stat`의 `pgscan`·`workingset_refault`로 합니다. refault가 늘면 실제로 필요한 페이지가 계속 밀려나는 중이라 limit 상향이 필요하다는 신호입니다."

- **Q. 페이지 크기를 2MB로 키우면 무조건 좋은가요?**
  - **A.** "TLB reach와 폴트 수 측면에서는 유리하고, 큰 배열을 순차 접근하는 워크로드에서 효과가 큽니다. 반대로 작은 객체를 많이 할당하는 워크로드에서는 내부 단편화와 메모리 팽창이 생기고, 스왑·I/O 단위가 커져 랜덤 읽기에 불리합니다. THP의 `always`는 `khugepaged` 압축으로 지연 스파이크를 만들 수 있어 DB나 지연 민감 서비스에서는 `madvise`로 두고 필요한 영역만 명시하는 편이 안전합니다."

- **Q. 내부 단편화와 외부 단편화를 페이징·세그멘테이션과 연결해 설명해 보세요.**
  - **A.** "페이징은 고정 크기로 잘라 마지막 페이지에 남는 공간이 생기므로 내부 단편화가 발생합니다. 세그멘테이션은 필요한 크기만큼 가변 할당하므로 내부 단편화는 없지만, 반납과 할당이 반복되면 총량은 충분한데 연속된 공간이 없어 실패하는 외부 단편화가 생깁니다. 실제 시스템은 둘을 섞어, 세그멘트를 다시 페이지로 나누는 방식을 씁니다."

- **Q. `LinkedHashMap`으로 LRU 캐시를 만들면 실무에서 충분한가요?**
  - **A.** "단일 스레드나 외부에서 락을 건 경우에는 충분합니다. 다만 모든 접근이 하나의 맵 락을 지나므로 동시성이 높은 서버에서는 캐시가 병목이 됩니다. 실무에서는 Caffeine처럼 세그먼트 단위 동시성과 W-TinyLFU 정책을 쓰는 라이브러리를 사용합니다. W-TinyLFU는 빈도 스케치로 신규 항목에 최소한의 기회를 줘서, 오래된 인기 항목이 새 항목을 계속 밀어내는 문제를 완화합니다."

---

## 참고

- OSTEP — Chapter 15~22 (Address Translation, Paging, TLB, Swapping, Policy): https://pages.cs.wisc.edu/~remzi/OSTEP/
- Silberschatz, *Operating System Concepts* — Virtual Memory / Page Replacement (FIFO·OPT·LRU·Clock, Belady anomaly)
- Linux Kernel Docs — Page Table Management: https://docs.kernel.org/mm/page_tables.html
- Linux Kernel Docs — Transparent Hugepage Support: https://docs.kernel.org/admin-guide/mm/transhuge.html
- Linux Kernel Docs — Memory Management / Swap: https://docs.kernel.org/admin-guide/mm/index.html
- Linux man — `vmstat(8)`, `proc(5)` (`/proc/vmstat`), `proc_pid_smaps(5)`: https://man7.org/linux/man-pages/man5/proc.5.html
- Redis Docs — Key eviction (`maxmemory-policy`, `maxmemory-samples`): https://redis.io/docs/latest/develop/reference/eviction/
- Redis Docs — `CONFIG SET` (`maxmemory-samples`, `lfu-log-factor`, `lfu-decay-time`): https://redis.io/docs/latest/commands/config-set/
- MySQL Docs — InnoDB Buffer Pool (`innodb_old_blocks_pct`, `innodb_old_blocks_time`): https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html
- MySQL Docs — Making the Buffer Pool Scan Resistant: https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-midpoint_insertion.html
- Caffeine — Efficiency / W-TinyLFU: https://github.com/ben-manes/caffeine/wiki/Efficiency
- Kubernetes Docs — Meaning of memory (working set): https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory
- Java SE API — `LinkedHashMap` (accessOrder): https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashMap.html
