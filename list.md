# Object Storage 백엔드 개발 면접 예상 질문 리스트

본 문서는 JD(`jd.md`)와 이력서(`_data/*`) 기반으로 도메인/기술/행동 역량을 매핑하여 구성한 심층 예상 질문 리스트입니다. 각 질문은 (B:기본) (D:심화/설계) (E:확장/응용 꼬리질문) 태그를 포함합니다. 답변 준비시: 1) 문제상황/배경 2) 선택/설계 근거 3) 구현/튜닝 포인트 4) 결과/지표 5) 회고/개선 순 구조 추천.

---
## 1. Object Storage / 스토리지 엔진

- Object Storage 핵심 구성요소(API 서버, Metadata, Data Storage, Gateway) 역할과 상호작용을 설명해 주세요. (B)
- Metadata 저장소 선정 기준(MongoDB vs RDB vs 자체 Key-Value)과 트레이드오프? (D)
- Chunking 전략을 어떤 기준(크기, 업로드 패턴, 네트워크)으로 결정하나요? (D)
- Multipart Upload 설계 시 ETag 처리/무결성 검증 방식은? (D)
- Consistent Hashing을 이용한 객체 파티셔닝 장단점? 직접 구현(cohashing) 경험에서의 이슈? (D,E)
- Small Object 대량 저장 시 발생하는 성능 문제(메타데이터 폭증, I/O 증폭)와 대응 방안? (D)
- LSM-Tree 기반 엔진(skv s/lsm-tree 구현)에서 Write Amplification 줄이는 방법? (D)
- Bloom Filter를 적용한 Lookup 최적화 구조와 False Positive가 시스템에 미치는 영향? (D)
- Raft 합의 구현 경험에서 Log Compaction과 Snapshot 설계 포인트? (D,E)
- HDD / SSD / NVMe / Tape 매체 특성 차이를 Object Storage 설계에 반영하는 방식? (D)
- S3 호환 API 설계 시 필수 고려 헤더와 권한/서명(Auth) 처리 흐름? (D)
- 대규모 삭제(Object Lifecycle / Expiration) Batch 처리 전략? (D)
- 데이터 일관성(Strong vs Eventual) 요구가 설계/성능에 주는 영향? (D,E)
- SOS 프로젝트의 Chunk 저장 구조(LevelDB)에서 GC 혹은 Compaction 정책은 어떻게? (D)
- 대량 Put 시 Hot Partition을 줄이기 위한 Key 설계 기법? (D)

## 2. 분산 시스템 / 합의 / 신뢰성

- CAP 정리와 Object Storage에 실제로 요구되는 선택 조합 예시? (B,D)
- Raft vs Paxos vs Gossip 기반 멤버십 선택 기준? (D)
- Leader 선출 지연이 클라이언트 가용성에 미치는 영향과 완화책? (D,E)
- 장애 시 Read Repair / Anti-Entropy 전략을 어떻게 구성할지? (D)
- 네트워크 Partition 발생 시 Write 처리 정책? (D)
- 재시도/중복 업로드(Idempotency)를 어떻게 구현? (D)
- Kafka를 이용한 이벤트 동기화(myStream) 아키텍처 구조 설명? (B,D)
- Exactly-Once가 어려운 이유와 At-Least-Once 보정 패턴? (D)
- 분산 트랜잭션(2PC / Outbox) 적용 여부 판단 근거? (D)

## 3. 성능 / 최적화 / 모니터링

- Throughput vs Latency 목표 설정 시 우선순위 결정 방식? (B)
- Chromium 기반 브라우저 CPU 사용량 최적화(Tile 조정, GPU Raster) 경험을 스토리지 I/O 최적화에 전이한다면? (E)
- GPU 인코딩 도입으로 30% CPU 개선 측정 방법과 실험 설계? (D)
- Profiling 도구/지표(IOPS, P99 Latency, Write Amp, Cache Hit) 수집 파이프라인 설계? (D)
- 백엔드 리팩토링 성능 개선 사례(지표 전/후) 설명? (B,D)
- Gstreamer 파이프라인 튜닝 논리와 유사한 데이터 처리 파이프라인 병목 진단 절차? (E)

## 4. 데이터 구조 / 스토리지 내부

- Skiplist 인덱스 구조 동작 원리와 B-Tree 대비 장단점? (B,D)
- LSM-Tree 단계별(Flush, Compaction) 비용과 Write Path 상세? (D)
- Bloom Filter False Positive Rate 계산 요소(k, m, n) 조정 경험? (D)
- Key 설계 시 Prefix Locality와 Range Query Trade-off? (D)
- LevelDB / RocksDB류 엔진에서 Block Cache와 OS Page Cache 상호작용? (E)

## 5. 네트워크 / 프로토콜 / 미디어 경험 전이

- HTTP/1.1 vs HTTP/2 선택이 Object Storage 대량 업로드에 주는 영향? (D)
- RTMP -> HLS Transmuxing 파이프라인 단계와 유사한 대용량 업로드 처리 파이프라인 설계? (E)
- MQTT/Kafka/WebSocket을 사용해 본 관점에서 Control Plane과 Data Plane 분리 설계? (D)
- TCP 튜닝(윈도우, Nagle, KeepAlive)으로 업로드 성능 개선 사례 가정? (E)

## 6. Go / C++ / 언어 심화

- Go GC 동작(Generational 아님) 특성과 메모리 관리 최적화 패턴? (D)
- Go에서 sync.Pool / Worker 패턴(개인 worker 패키지) 사용 기준? (D)
- Goroutine Leak 탐지/예방 전략? (D)
- C++11~17 주요 기능(Concurrency, Move Semantics)을 실제 프로젝트에서 활용한 사례? (B,D)
- Chromium Task 기반 비동기 모델과 Go Scheduler 비교? (E)
- Zero-Copy I/O를 구현하거나 흉내낸 경험/아이디어? (E)

## 7. Linux / 시스템 / 디버깅

- strace / perf / eBPF 활용한 병목 분석 절차? (D)
- 파일시스템(Page Cache, Dirty Page Flush) 이해가 Write Latency에 미치는 영향? (D)
- 컨테이너(Docker) 환경에서 스토리지 성능 편차 원인(Cgroup, fs driver)? (D)
- Wayland 연동/디버깅 경험을 커널/유저 공간 인터페이스 이해로 확장 설명? (E)
- 고 IOPS 상황에서 IO 스케줄러 선택 영향? (E)

## 8. 아키텍처 / 설계 / MSA / 이벤트

- myStream DDD 적용 과정(Event Storming, Context Map)에서 얻은 인사이트? (B,D)
- 서비스 경계(Bounded Context) 식별 기준과 팀 구조 매핑? (D)
- 이벤트 기반 동기화(Kafka)에서 멱등 처리/재처리 설계? (D)
- API Gateway / Edge 레이어에서 Auth, Rate Limit, QoS 적용 순서? (D)
- Backpressure 처리 패턴(Channel, Queue, Token Bucket)? (D)
- Object Lifecycle / Retention 정책 설계 시 이벤트 설계? (E)

## 9. 운영 / 배포 / 신뢰성

- 롤링 업데이트 vs 블루그린 vs 카나리 선택 기준? (B,D)
- 장애 사후분석(Postmortem) 템플릿 핵심 항목? (B)
- SLA / SLO / SLI 정의 예시(Object Storage) 제시? (D)
- Observability (Logs / Metrics / Traces) 수집 파이프라인 간 상호 활용? (D)
- Rate Limit, QoS, Traffic Shaping 적용 위치와 구현 패턴? (D)
- 장애 재현 환경 구성 전략(프로덕션 패리티)과 데이터 마스킹? (E)

## 10. 보안 / DRM 경험 전이

- DRM (Widevine / PlayReady / CoreTrust) 연동 경험이 저장 객체 암호화(KMS, Envelope)에 주는 시사점? (E)
- 전송 중 암호화(TLS) 외 저장 시 암호화(At-Rest) 키 회전 전략? (D)
- Signed URL / Presigned URL 만료/권한 처리 설계? (D)

## 11. 품질 / 코드 / 리팩토링

- 레거시 재설계(브라우저 미디어 스택, 프록시 서버) 시 의사결정 기준? (B,D)
- 성능 개선과 가독성/추상화 사이 트레이드오프 사례? (D)
- Secure Coding(MISRA, CERT) 경험을 Go 백엔드 품질 게이트로 옮기는 방안? (E)
- 테스트 전략(Unit, Integration, Property-based) 적용 우선순위? (D)

## 12. CS 기초 심화

- 메모리 계층(Locality, Cache Line) 이해가 Hash/Skiplist 최적화에 주는 영향? (D)
- 락 경합 줄이기 위한 Sharding / Lock-free 기법 적용 경험 또는 설계? (D)
- 멀티스레드 vs 이벤트루프 모델 선택 기준? (B,D)
- 네트워크 지연 구성 요소(TCP Handshake, TLS, DNS)와 최적화 아이디어? (B,D)

## 13. 행동 / 협업 / 문화

- 신규 팀에서 온보딩 시 학습 로드맵을 스스로 구성한 방법? (B)
- 타 부서/디자이너/플랫폼 팀과의 충돌 해결 경험과 원칙? (B,D)
- 성능 개선 과제를 우선순위화할 때 의사소통한 방식? (B)
- 품질 문화(코드리뷰, 린트, 테스트) 정착을 위해 시도한 활동? (B,D)
- 실패/이슈 사례 1~2개: 원인, 대응, 재발 방지? (B,D)

## 14. 프로젝트 별 구체 꼬리질문 (Portfolio Deep Dive)

### SOS (소규모 Object Storage)

- 아키텍처 다이어그램을 그린다면 핵심 컴포넌트와 데이터 흐름? (B)
- Chunk 사이즈 선택 근거와 실험? (D)
- MongoDB 인덱스 전략과 쿼리 패턴? (D)
- LevelDB Compaction 튜닝 포인트? (D)
- 장애(데이터 손상/Partial Write) 대비 전략? (D,E)

### skvs (분산 KV Storage)

- Raft 구현에서 로그 재전송 최적화? (D)
- Snapshot 시점 결정 기준? (D)
- LSM Tree 레이어 구분 및 SST 파일 관리 전략? (D)
- Consistent Hashing과 노드 스케일 아웃 시 Rebalancing 절차? (D,E)

### myStream (실시간 방송 플랫폼)

- DDD 적용 시 도출한 주요 Aggregate와 경계? (D)
- 이벤트 순서 보장 문제와 해결 패턴(Kafka Partition, Key 전략)? (D)
- 미디어 Transcoding 파이프라인 장애 복구 전략? (D)
- RTMP -> HLS 변환에서 Latency vs 화질 트레이드오프? (D)

### Chromium 기반 브라우저 / DRM 연동

- EME(Encrypted Media Extensions) 흐름과 DRM Plugin 연동 구조? (D)
- Widevine vs PlayReady 연동 차이? (D)
- Gstreamer 파이프라인 디버깅 핵심 도구와 지표? (B,D)
- GPU Raster 적용 전후 성능 측정 지표 정의? (D)

### 자율주행 센서 데이터 플랫폼

- 다양한 센서(카메라/LiDAR 등) 동시 처리 시 병목/동기화 전략? (D)
- Kafka 기반 데이터 파이프라인 Backpressure 처리? (D)
- ROS 메시지 전송 구조와 네트워크 최적화? (D)

## 15. 예상 실무 시나리오 문제

- 신규 Region 확장 시 메타데이터 분산/복제 전략을 설계해 보세요. (Case)
- 1TB/day Small Object 폭증 상황: 인덱스/스토리지 비용 급증 대응 방안? (Case)
- 업로드 지연(P99 2s -> 5s 상승) 원인 추적 절차를 단계별로 설명. (Case)
- 다중 AZ 장애에서 RPO/RTO 목표 달성을 위한 설계? (Case)
- Tape Tier 도입으로 비용 최적화를 추진할 때 정책/워크플로? (Case)

## 16. 역질문(면접관에게 할 질문 예시)

- Object Storage 현재 아키텍처와 향후 12개월 로드맵(Scale/Feature)은?
- 성능/안정성 관련 가장 큰 Pain Point과 최근 해결 시도는?
- 온보딩 트레이닝 커리큘럼과 기술 부채 처리 방식?
- 운영 중 대표적 장애 패턴과 사후분석(Postmortem) 문화는?
- 신규 저장 매체(HDD/SSD/NVMe/Tape) 도입 의사결정 프로세스?
- 인프라/플랫폼/서비스 팀 간 협업 구조와 의사결정 단계?

---

### 준비 가이드 요약

- 각 Deep Dive 프로젝트별 STAR + 아키텍처 다이어그램 준비
- 핵심 수치(성능 개선 %, Latency, Throughput, 자원 사용) 메모라이즈
- 실패/장애 2~3개 케이스 재현 로그/지표 흐름 정리
- 스토리지 핵심 개념(Consistent Hashing, LSM, Bloom, Raft, Lifecycle, Multipart) 정의 1~2문장 암기
- Go/C++ 병행 경험 차별화 포인트(메모리, 동시성, 디버깅) 정리

> 필요 시 카테고리 확장이나 모의 답변 구조 템플릿도 추가 가능합니다.

## 17. 지원서(request.md) 기반 추가 맞춤 질문

### 동기 / 커리어 방향
- 임베디드/클라이언트 개발 → 백엔드/스토리지로 전환할 때 가장 어려웠던 기술적 갭과 메운 방법은? (B)
- Object Storage 직무가 본인 장기 목표(고성능/확장성/안정성 전문가) 달성에 어떻게 기여한다고 보는가? (B)
- "NCP Object Storage를 선택하게 만들 핵심 역할"을 수치화한다면 어떤 KPI를 정의하겠는가? (D)

### 학습 / 온보딩 전략
- 입사 후 90일 온보딩 플랜을 3단계(탐색/기여/개선)로 나누어 구체적 활동과 측정 지표를 제시해 보라. (D)
- 기존 개인 프로젝트(sos, skvs, myStream) 중 어떤 부분을 바로 사내 코드베이스 개선에 적용할 수 있는가? (D)
- 레거시 파악 시 코드/러닝 노트/지표 중 어떤 순서로 Mapping을 진행할 것인가? (D)

### 데이터/AI 트렌드 연계
- AI 워크로드(대규모 모델 학습/추론) 특성을 고려한 Object Storage 기능 개선 아이디어 2~3가지? (E)
- Vector DB / 분산 캐시 / Tiered Storage 조합 설계 시 Object Storage가 제공해야 할 최소 기능은? (D)
- 데이터 로컬리티 최적화를 위한 Pre-Fetch / Hint API를 설계한다면 인터페이스 초안을 말해보라. (E)

### 성과지표 / 측정
- 서비스 성능 병목 개선 과제 선정 시 사용하고 싶은 우선순위 평가 매트릭스 예시? (D)
- 안정성 관련 SLO를 정의할 때 Read/Write/Metadata API를 구분해야 하는 이유와 지표 후보? (D)
- 사용자 경험 개선을 위한 Non-Functional KPI(에러 재시도율, P95 Upload Init Latency 등) 선정 근거? (D)

### 조직/협업 / 문화 기여
- 기술 공유 문화 활성화를 위해 첫 6개월 내 실행할 작은 실험 2개? (B)
- 갈등 조정 사례(일정/성능) 학습을 스토리지 장애 RCA에도 적용한다면 어떤 프로세스가 되는가? (D)
- 객관적 검증 문화를 팀에 스며들게 하기 위한 PR 템플릿 혹은 Check-list 초안? (D)

### 문제 해결 / 사례 확장
- DRM 파이프라인 POC 일정 재조정 경험을 Object Storage 대규모 마이그레이션 프로젝트에 적용하는 시나리오? (E)
- 동적 파이프라인 제안→데이터 기반 반박 경험을 통해 얻은 '가설-실험-결론' 루프를 스토리지 성능 튜닝에 적용한 예를 들어보라(가정 가능). (E)
- 리소스 제약이 있는 상황에서 어떤 개선 작업을 먼저 Drop하거나 Defer할지 판단 기준? (D)

### 장기 성장 / 리더십
- 기술 리더로 성장하기 위해 향후 2년간 분기별 학습/실천 로드맵을 카테고리(스토리지 심화, 분산합의, Observability, 운영 자동화)로 나누어 개략 제시. (E)
- '확장성·안정성·고성능' 세 축이 충돌할 때 본인의 의사결정 프레임워크? (D)
- Bus Factor 감소와 지식 확산을 동시에 달성하기 위한 문서화/리뷰 전략? (D)

### 추가 기술 심화 (AI 시대 맥락)
- Tiered Storage + Object Tagging + Lifecycle Policy를 AI 데이터 파이프라인에 최적화하려면 어떤 메타데이터가 추가로 필요? (E)
- 대규모 학습 Job이 반복적으로 동일 데이터 세트를 읽는 상황에서 Read Amplification 감소 전략 2가지? (D)
- Server-Side Encryption + Client-Side Caching 조합 시 키 관리 및 무결성 검증 흐름을 단순 다이어그램으로 설명(구두). (E)

### 역질문 파생
- 팀에서 최근 가장 큰 기술 부채는 무엇이며 제거 로드맵이 있는가? (역질문 파생)
- 성능 개선 성공을 어떻게 Celebrating / Knowledge Base화 하는가? (역질문 파생)
- 신규 기능 출시 전 실험(Experiment) 또는 Canary 절차는 표준화되어 있는가? (역질문 파생)


