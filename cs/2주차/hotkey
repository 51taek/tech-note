현재 외부 캐시를 사용하고 있음 (Redis)

#### Redis에 실제로 들어가는 것

- 현재가
    - market:current-price:<stockId>
    - TTL: 5분
- 현재가 갱신 시각
    - market:current-price-updated-at:<stockId>
    - TTL: 5분
- 유저별 상위 보유 종목
    - asset:top-holdings:<userId>
    - TTL: 2분
- 종목별 현재 watcher 목록
    - market:current-watcher:<stockId>
    - TTL: 10분
- 배치 가격 정보
    - market:batch-price:<stockId>
    - TTL: 24시간
- 예약 주문/거래 관련 정보
    - market:reservation:*
    - 일부 trade info TTL: 30일
- KIS 토큰/락
    - kis:accessToken:*
    - refresh lock TTL: 30초

우리는 많은 정보를 캐시에 저장하고 있고 캐시가 아주 중요한 도메인임. 

하나의 캐시에 너무 많은 부하가 가해지면 응답이 느려짐

왜?

레디스는 싱글 스레드기 때문에 한번에 하나의 요청만 처리함

순차적으로 요청을 처리하기 때문에 하나의 느린 응답 때문에 뒤에가 다 기다림

소켓 연결 대기열(backlog) → 이벤트 루프가 읽어온 요청은 각 클라이언트 버퍼에 쌓임 → Redis가 순차적으로 처리 → 응답도 클라이언트 output buffer에 쌓임

짧은 시간에 연결이 너무 많이 몰리면 OS의 connection queue와 Redis의 `tcp-backlog` 한계에 걸릴 수 있어. 그러면 아직 Redis가 명령을 처리하기도 전에 **연결 지연, 연결 실패, 재시도 폭증**이 생긴다. Redis 튜닝 문서도 많은 연결이 있으면 `tcp-backlog`와 OS의 `somaxconn`, `tcp_max_syn_backlog`를 같이 봐야 한다

https://redis.io/tutorials/operate/redis-at-scale/talking-to-redis/

많은 사람들이 입력 쪽만 보는데, spike 때는 **응답을 내보내는 쪽**도 문제야. Redis는 각 클라이언트에 대한 output buffer를 유지하는데, 클라이언트가 응답을 충분히 빨리 읽지 못하면 이 버퍼가 커진다. Redis 문서는 Pub/Sub 클라이언트, replica 등에 output buffer limit이 있고, 한계를 넘으면 연결을 끊는다고 설명한다. `INFO`에도 `client_output_buffer_limit_disconnections`가 따로 잡힌다. 즉, Redis가 안 죽어도 **느린 소비자(slow consumer)** 때문에 연결이 강제 종료될 수 있어.

https://redis.io/docs/latest/develop/reference/clients/

spike가 쓰기 폭증이거나 큰 응답을 만드는 읽기 폭증이면 buffer와 데이터 증가 때문에 메모리 압박이 온다. 이때 `maxmemory`와 eviction 정책이 적절하면 키 축출이 일어나고, eviction이 허용되지 않으면 Redis는 OOM 에러를 반환할 수 있다. 공식 문서도 메모리 한도를 넘었을 때 eviction 정책에 따라 키를 지우거나, 그렇지 않으면 OOM 메시지가 발생한다고 설명한다.

https://redis.io/docs/latest/develop/reference/eviction/

<aside>
💡

위와 같은 문제 때문에 캐시 서버를 분산시키는 방안을 선택했다.

</aside>

+k6로 단기적으로 큰 스파이크를 일으키는 테스트 진행. 보통 9시에 장이 시작할때 많은 사람들이 몰리는 상황을 가정.

하지만 캐시 서버를 scale-out 하는 것에도 고민이 필요하다.

첫번째로는 서버 A와 서버 B가 있을 때 어떤 키를 A에 두고 어떤 키를B에 둘 것인지에 대한 기준을 세워야 한다는 점이다.

이 문제는 단순하게 생각했을 때 모듈러 연산을 통해 서버의 개수로 키 값을 나눈 나머지 대로 각 서버에 두면 된다.

예를 들면 서버가 3개일 때 37이라는 키값에 대해서 37%3 = 1 이고 1번째 서버에 들어가면 된다.

하지만 이 방식에는 문제가 있는데 서버를 추가로 증설하게 되면 기존에 있는 키들이 바뀌어야 한다는 것이다.

이 문제를 해결하기 위해서 이미 수많은 빅테크들이 좋은 해결방안들을 제시해놨다.

https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users

**요약**

> 특정 프로세스 하나가 수만 명의 통신을 감당하게 두지 않고, **메시지 전달 책임을 여러 노드로 분산시키는 'Manifold' 구조**를 통해 500만 명의 동시 접속 상황에서도 안정적인 채팅 환경을 구축할 수 있게 해줌.

⇒ 근데 이건 읽기에 대한 핫키 문제라기 보다 다른 문제 같긴 한데, 이것도 우리 프로젝트와 관련해서 해결할 만한 부분이 있을 것 같음. 지금 주식 업데이트 fanout 하면서 문제가 생겼는데 오히려 이거랑 관련이 있는듯?
> 

https://oliveyoung.tech/2025-07-23/redis-tips-for-developer/

**요약**

> 올리브영에서는 백그라운드에서 주기적으로 캐시를 갱신하거나 매 요청마다 일정 확률로 캐시를 갱신하는 PER 알고리즘을 적용하는 방식을 고려해보시길 권장.
> 

크게 보면 2가지의 방안 중에 고민했다.

| 항목 | Redis Cluster | Client-side Consistent Hashing |
| --- | --- | --- |
| 분산 방식 | Redis가 직접 샤딩 | 애플리케이션이 노드 선택 |
| 운영 난이도 | 중간~높음 | 중간~높음 |
| 애플리케이션 복잡도 | 낮음 | 높음 |
| 노드 추가/제거 시 영향 | 슬롯 재배치 필요 | 재배치량 적음 |
| 캐시 안정성 | 보통 | 좋음 |
| 장애 대응 | 비교적 표준화 | 직접 설계 필요 |
| 생태계/운영도구 | 풍부 | 직접 커스텀 많음 |
| Hot key 대응 | 별도 필요 | 별도 필요 |
| Spring/Java 운영 적합성 | 높음 | 커스텀 부담 큼 |

### 이 프로젝트에 맞춘 해석

#### 1) 지금 캐시 성격

현재 Redis 사용은 대부분:

- **현재가**: 짧은 TTL, 고빈도 조회
- **watcher**: 짧은 TTL, 실시간성
- **보유종목**: 사용자별 캐시
- **예약/배치 데이터**: 비교적 긴 TTL
- **토큰/락**: 분산 제어 성격

즉 이 시스템은 단순 “대용량 보관” 보다 **짧은 TTL + 빠른 조회 + 일부 실시간성**이 더 중요

---

#### 이 경우에 레디스 클러스터를 사용하는 것이 적합

- Redis를 표준 방식으로 확장하고 싶다
- 운영/장애대응을 검증된 방식으로 가고 싶다
- 앱 코드에 샤딩 로직을 많이 넣고 싶지 않다
- Pub/Sub, lock, token store까지 포함해서 한 체계로 관리하고 싶다
이 프로젝트에서 특히 좋은 점
- 현재 코드가 Redis를 직접 repository 단위로 사용하고 있어서,
앱에 별도 해시 라우팅 로직을 넣는 것보다
인프라 레벨에서 Cluster로 흡수하는 편이 관리가 쉽습니다.

---

#### 이 경우에 Client-side Consistent Hashing이 적합

- Redis Cluster를 쓰기 어렵고
- 단순 캐시 서버 여러 대를 애플리케이션이 직접 분산해야 하고
- 노드 추가/제거 시 캐시 미스 충격을 최소화하는 게 최우선일 때
하지만 이 프로젝트에서는 단점이 큼
- 애플리케이션이 키 라우팅 책임을 가져감
- 토큰/락/PubSub 같은 기능은 설계가 더 복잡해짐
- 팀 운영 난이도 상승
- 장기적으로 “왜 이렇게 분산했는지” 유지보수 비용이 커짐

---

### 결론

우리는 레디스 클러스터가 더 적합하다

이유

1. 표준적
2. 운영 도구/장애 대응이 상대적으로 명확
3. 앱 코드에 커스텀 샤딩 책임을 덜 실음
4. 현재 Redis 사용 범위가 캐시뿐 아니라
    - 토큰 저장
    - 분산 락
    - Pub/Sub
    까지 있어서, client-side consistent hashing은 더 복잡해짐
5. 특별한 로직이 들어가는 핫 키 분산 알고리즘이 필요하다면 그렇게 하겠지만 우리에게 딱히 필요할 만한 가정이 안보임. 

---

#### 최종 의사결정 문장

> Redis Cluster를 선택해야 하는 경우
“우리는 Redis를 캐시 + 토큰 + 락 + 이벤트 인프라로 함께 쓰고 있고, 애플리케이션 샤딩 복잡도를 늘리고 싶지 않다.”
Consistent Hashing을 선택해야 하는 경우
“우리는 Redis를 거의 캐시 전용으로 쓰며, 노드 증감 시 캐시 재배치 최소화가 가장 중요하다.”
> 

---

Consistent Hashing은 “반드시 직접 샤딩해야 하는 사유”가 있을 때만 검토하는 게 좋습니다.

주의해야 할 핵심 포인트

1. 락은 결국 “하나의 키” 기준.
락 키 하나는 특정 shard에 저장.
즉 락 자체는 가능하지만, 그 락의 안정성은 그 shard의 상태에 영향을 받음.
2. failover 때 락 손실 가능성
Redis replication은 기본적으로 강한 동기식이 아니어서, master에 락이 써졌는데 replica에 완전히 반영되기 전에 failover가 일어나면 락이 사라진 것처럼 보일 수 있습니다.
즉, “절대 중복 실행되면 안 되는 비즈니스 락” 에는 Redis 락만 믿는 건 위험할 수 있습니다.
3. TTL 설정이 중요 락 TTL이 너무 짧으면 작업 중 만료될 수 있고, 너무 길면 장애 시 오래 잡혀 있을 수 있습니다. 
그래서 보통:
    - lease time
    - watchdog/auto-renew
    - unlock 시 소유자 검증 이 중요합니다.
4. 멀티키/멀티리소스 락은 더 조심 여러 키를 동시에 묶는 락은 Cluster에서 더 복잡해집니다. 단일 리소스 락이 가장 안전하고 단순합니다.

---

#### 이 프로젝트 관점

현재처럼

- KIS 토큰 refresh lock
- 짧은 작업 중복 방지 같은 용도라면 Cluster에서도 충분히 현실적인 선택입니다.

반면 아래는 위험도가 올라갑니다:

- 주문 체결 같은 금전/정합성 핵심 로직
- 중복되면 큰 사고 나는 작업
- 락 유실이 곧 데이터 오류로 이어지는 경우 그런 경우는 Redis 락만으로 끝내지 말고:
    - DB unique constraint
    - idempotency key
    - optimistic/pessimistic locking
    - 상태 전이 검증 같은 2차 방어선이 필요합니다.

---

실무 권장
써도 좋은 경우

- 캐시 재생성 중복 방지
- 토큰 재발급 중복 방지
- 짧은 배치/스케줄러 중복 실행 방지
Redis 락만 믿으면 위험한 경우
- 결제
- 주문
- 자산 수량 변경
- 정확히 1회만 실행되어야 하는 핵심 트랜잭션

### 🚨 주의: 하지만 Hot key는 해결 못함.

예를 들어 인기 종목 하나에 조회가 몰리면:

- Cluster여도 한 샤드가 뜨거워질 수 있고
- Consistent Hashing이어도 해당 키는 한 노드로 감. 즉 진짜 병목이 market:current-price:<stockId>
- 특정 종목 watcher 같은 소수 키 집중이면, 핵심 대책은 해시 방식보다:

그럼 핫 키를 해결할 수 있는 방법은 뭐가 있냐

- TTL 재설계 로컬 캐시/L1 + Redis/L2 이중화
- 인기 종목 pre-warming
- read replica 활용
- key 설계 개선 입니다.

### 현재 핫키 위험도

| 키 패턴 | Hot key 위험도 | 이유 |
| --- | --- | --- |
| market:current-price:<stockId> | 매우 높음 | 종목별 단일 키 + 조회 집중 |
| market:current-price-updated-at:<stockId> | 높음 | 현재가와 함께 인기 종목 쏠림 |
| market:current-watcher:<stockId> | 높음 | 인기 종목 watcher 집중 |
| market:batch-price:<stockId> | 중간 | 종목별이지만 실시간성 낮음 |
| asset:top-holdings:<userId> | 낮음~중간 | userId 기준으로 분산됨 |
| kis:accessToken:<appKey> | 중간 | appKey 수가 적으면 집중 |
| kis:accessToken:refreshLock:<appKey> | 중간 | 갱신 시 경쟁 가능 |
| market:reservation:* | 낮음~중간 | 일반적으로 분산되지만 일부 종목 집중 가능 |

### 핫 키 문제를 해결하는 방법

https://matthewtejo.substack.com/p/handling-hotkeys-in-timeline-storage

**요약**

> 우리의 상황과 거의 동일? 

특정 유명인(Celebrity)의 데이터를 수백만 명의 팔로워가 동시에 조회할 때 발생하는 문제를 해결하는 아키텍처를 설명.

여기서 제시한 방식
1. 하이브리드 팬아웃 (잘 모르겠음..)

2. 키 복제 (Key Replication)
- 데이터 `K`가 핫 키로 판단되면, 이를 `K_1, K_2, ..., K_N`의 형태로 여러 샤드(Shard)나 노드에 분산 저장
- 수천 명의 사용자가 동시에 같은 정보를 요청해도, 시스템은 이들을 서로 다른 노드로 분산시켜 응답할 수 있습니다. 이는 특정 노드의 CPU나 네트워크 대역폭이 고갈되는 현상을 직접적으로 방지합니다.

3. 요청 병합 (Request Coalescing / Collapsing)
- 동일한 핫 키에 대한 요청이 찰나의 순간에 수만 건이 들어올 때, 이를 하나로 묶어서 처리하는 방식
- **Wait Group 방식:** 1,000개의 요청이 동시에 같은 키를 조회하러 오면, 첫 번째 요청만 저장소(DB/Redis)로 보내고 나머지 999개는 대기(Wait)합니다. 첫 번째 요청이 결과를 받아오면 기다리던 모든 요청에 그 결과를 한꺼번에 반환
- 실제 저장소에 가해지는 부하는 1,000배에서 1배로 줄어들며, 이를 통해 캐시 스탬피드(Cache Stampede) 현상을 방지할 수 있습니다.

4. 적응형 캐싱 (Adaptive Caching)
- 실시간으로 어떤 키가 '핫'해지는지 감지하여 동적으로 처리 방식을 바꿈
- 글에서는 특정 임계치(Threshold) 이상의 트래픽이 발생하는 키를 실시간 모니터링하여, 해당 키만 별도의 고성능 메모리 레이어(L1 캐시)로 올리거나 위에서 언급한 '복제 전략'을 즉시 적용하는 메커니즘을 설명
⇒ But, 이러면 인스턴스가 여러개일때 각 인스턴스마다 같은 값이지만 캐시에는 서로 다른 값으로 저장되어있을 수 있음. 일관성이 깨짐. 이 문제를 해결해야함.
>
