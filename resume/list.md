# Object Storage 백엔드 개발 면접 예상 질문 리스트

본 문서는 JD(`jd.md`)와 이력서(`_data/*`) 기반으로 도메인/기술/행동 역량을 매핑하여 구성한 심층 예상 질문 리스트입니다. 각 질문은 (B:기본) (D:심화/설계) (E:확장/응용 꼬리질문) 태그를 포함합니다. 답변 준비시: 1) 문제상황/배경 2) 선택/설계 근거 3) 구현/튜닝 포인트 4) 결과/지표 5) 회고/개선 순 구조 추천.

---
 
## 1. Object Storage / 스토리지 엔진
 
- Object Storage 핵심 구성요소(API 서버, Metadata, Data Storage, Gateway) 역할과 상호작용을 설명해 주세요. (B)
	> 답변: API 서버는 클라이언트 요청을 인증/인가(HMAC/Signature 검증, 정책/ACL 평가)하고 파라미터 정규화(헤더/쿼리 Canonicalization) 후 메타 저장소에 "Intent(예약)"를 먼저 기록하여 중복/경쟁 업로드를 제어합니다. 실제 바디는 스트리밍으로 Chunk/N-Part(4~16MB 권장)로 분리되어 Data 노드(Blob/Chunk Server)에 쓰이고, 완료 후 메타 Commit(버전/ETag/Size 확정) 단계에서 원자적으로 가시화됩니다. Gateway/Front Layer는 다중 AZ Health 체크, Rate Limit, Connection Pool, Request Coalescing(동일 Key GET 중복 제거), TLS 종료를 수행해 내부 컴포넌트를 보호합니다. 비동기 파이프라인(Lifecycle, Replication, Event Publish)은 Commit 이벤트를 Consume 하며, 실패 시 재시도/Dead Letter 로직으로 격리됩니다. 이렇게 "정책/보안 ↔ 메타(작음/강한 일관) ↔ 데이터(크고 스트리밍)" 경계를 나누면 독립 Scale-out, 장애 Blast Radius 축소, 서로 다른 매체(NVMe 메타 / HDD 데이터 / Tape 아카이브) 선택이 용이합니다.
- Metadata 저장소 선정 기준(MongoDB vs RDB vs 자체 Key-Value)과 트레이드오프? (D)
	> 답변: 판단 축은 (1) Access 패턴(Random Lookup vs Range/Prefix Scan) (2) 일관성/트랜잭션 범위 (3) 스키마 진화 속도 (4) Secondary Index 다양성 (5) 운영 복잡도 & 팀 역량입니다. MongoDB는 빠른 모델링과 다중 인덱스, 복잡한 Document를 빠르게 담기에 유연하지만 Shard Key 설계 실패 시 Hot Chunk, Balancer Migration 비용이 급증합니다. RDB는 강한 Tx/조인/정합(Data Integrity) 장점이 있으나 Sharding/Schema Migration(Online DDL) 비용이 커지고 스키마 변화를 빈번히 요구하는 초기 단계엔 관성으로 작용합니다. 자체 LSM 기반 KV(Rocks/LevelDB 래핑)는 예측가능한 P99, 낮은 Storage Overhead, 단일 Key Path 집중 워크로드에서 최고 효율을 주지만 다중 인덱스/복합 조회/Ad-hoc 분석을 응용 계층이 재구성해야 합니다. 실무 전환 패턴은 "초기 빠른 기능 → Mongo", 트래픽 안정화 후 Hot Path(HEAD/GET 메타)만 KV로 Dual-Write & Shadow Read 검증, 이후 분석/검색은 별도 파이프라인(Outbox → ES/OpenSearch)으로 분리하는 다단 구조를 택합니다.
- Chunking 전략을 어떤 기준(크기, 업로드 패턴, 네트워크)으로 결정하나요? (D)
	> 답변: 워크로드 객체 크기 분포(P50/P90/P99), 동시 업로드 수, RTT·대역폭으로부터 BDP(=Bandwidth*RTT)를 구해 "전송 파이프를 비울지/채울지" 판단합니다. 목표는 (1) 재시도 시 낭비 대역폭 최소 (2) 인덱스/메타 오버헤드 최소 (3) 병렬화 효율 극대화 세 가지 균형입니다. 실무 경험상 4~16MB가 전송 효율 vs 실패 재전송 비용의 타협점이었고, 소형 객체 다수 패턴에서는 Small Object Pack(≤256KB 묶음 + Offset Index)으로 메타 폭증을 억제했습니다. 적응형은 초기 N개(예: 2~3) Part 시험 업로드로 실측 Throughput/RTT를 수집 → 목표 BDP 대비 Under-util이면 다음 Part Size 증가, 과다라면 축소하는 피드백 루프를 둡니다. 대용량은 Multipart 병렬 N-way(코어/네트워크 제한) + 부분 실패 Part만 재시도해 Tail을 줄입니다.
- Multipart Upload 설계 시 ETag 처리/무결성 검증 방식은? (D)
	> 답변: Part 업로드마다 스트림을 읽으며 CRC32C 또는 MD5 Rolling Hash를 계산·저장하고, Complete 시 (H1||H2||...||HN) 바이트열을 다시 MD5 → `combined-N` 형태 ETag 생성(호환성 목적)합니다. 무결성은 (1) 업로드 스트림 단계에서 조기 Fail-Fast (2) Complete 시 서버 재계산 vs 클라이언트 제공 해시 비교 2단 구조입니다. 실패 Part만 재전송해 전체 재시도 폭을 줄이고, 인증용 SHA256(HMAC)은 별도 경로(Signature V4)에서 처리하여 데이터 무결성과 요청 인증을 분리합니다. 나아가 대용량 병렬 업로드 시 서버 측 내구성(임시 Part 저장소)에 Checksummed Append-Log를 써 장애 시 부분 손상 검출을 용이하게 합니다.
- Consistent Hashing을 이용한 객체 파티셔닝 장단점? 직접 구현(cohashing) 경험에서의 이슈? (D,E)
	> 답변: 장점은 (1) 노드 증감 시 전체 재배치 비율이 낮아(O(k/n)) 재밸런싱 비용 절약 (2) 중앙 Directory 없이 결정적 매핑 (3) 캐시 키 안정성입니다. 단점은 (a) 분포 불균형(특히 Hot Prefix) (b) Virtual Node 증가 시 메모리/관리 오버헤드 (c) 재구성 순간 캐시/커넥션 Flush로 Tail Spike (d) 트래픽 편향 시 특정 노드 IO 큐 포화입니다. 개선: Jump Consistent Hash 도입으로 메모리 Footprint 축소, 짧은 슬라이딩 윈도우 QPS 표준편차 기반 Hot Key 탐지 후 임시 가중치 조정 또는 Shadow Replica 캐시, Write Throttle로 보호, 백그라운드 재밸런싱 진행 중 Read Path 영향 최소화를 위해 두 개 링(Old/New) 동시 조회 Grace Period를 두었습니다.
- Small Object 대량 저장 시 발생하는 성능 문제(메타데이터 폭증, I/O 증폭)와 대응 방안? (D)
	> 답변: 병목은 (1) 메타 엔트리 폭증 → 인덱스/Bloom/Cache 메모리 압박 (2) 작은 fsync/Flush/Compaction으로 Write Amplification 상승 (3) Syscall·Context Switch 비중 확대입니다. 대응: (a) Pack(여러 ≤256KB 객체를 하나 Blob + Offset Index) (b) Inline Threshold(≤4KB 본문 메타에 인라인) (c) Heat 기반 Tiered(Hot=Mem/Redis, Warm=LSM 상위, Cold=압축된 Pack) (d) Batch Put으로 인증/정책 검증 1회 후 Bulk Append (e) Lifecycle로 Cold Pack 재압축 및 중복 제거 (f) Background Merge 시 Rate Limit로 Foreground Tail 보호.
- LSM-Tree 기반 엔진(skv s/lsm-tree 구현)에서 Write Amplification 줄이는 방법? (D)
	> 답변: 목표는 불필요한 하위 레벨 Merge 감소와 Tombstone 체류 시간 단축입니다. (1) Level Size Ratio 튜닝(10→7 등)으로 상위 레벨 폭증 억제 (2) Key Range Hotness 통계 기반 Partial/Vertical Compaction만 수행 (3) Bloom FP율 하향 조정(1%→0.5% 등)으로 무의미한 Read 방지→Compaction 트리거 감소 (4) Tombstone Age/Count 임계 초과 시 GC Prioritize (5) Tiered+Leveled 혼합(초기 빠른 삽입 Tiered→핫 데이터 안정화 후 Leveled) (6) Flush/Compaction Rate Limit + IO 우선순위로 Tail 보호 (7) WAL Direct IO + 대용량 Sequential Segment, fsync Batching으로 쓰기 증폭 최소화.
- Bloom Filter를 적용한 Lookup 최적화 구조와 False Positive가 시스템에 미치는 영향? (D)
	> 답변: SSTable/Segment 단위 Bloom Filter로 부재(Key Not Present) 경로를 조기 차단해 Random IO를 감소시킵니다. False Positive는 "존재 X → 존재 가능" 오판으로 1회 디스크/캐시 탐색 추가가 전부이므로 Miss Path 비용과 메모리 사용량 Trade-off 최적점을 찾습니다. 경험상 p≈1% 이하에서 한계 체감(더 낮추면 메모리 증가 대비 IO 절감 소폭)이며, 공식 p≈(1-e^{-kn/m})^k 기반으로 n 추정치 편차 대비 10~15% 여유 m 비트를 할당합니다. FP율을 지속 수집→목표 초과 시 m 확장 또는 k 재조정 Rolling Rebuild 전략을 사용했습니다.
- Raft 합의 구현 경험에서 Log Compaction과 Snapshot 설계 포인트? (D,E)
	> 답변: Snapshot 설계 쟁점은 (1) 트리거(로그 크기/비율/시간) (2) 생성 비용 최소화 (3) 전송/설치 시 다운타임 최소화 (4) 안전성 메타(lastIncludedIndex/Term) 포함입니다. 전체 Snapshot은 간단하지만 대형 클러스터에서 네트워크/디스크 Burst 유발, Incremental은 Changed Range(Dirty Bitmap, Hash Diff)만 재전송해 비용 최적화합니다. 설치 시 Temp Path에 수신→Checksum 검증→원자적 Rename(Swap)→로그 Prefix Truncate로 공간 회수. Lagging Follower의 Snapshot Install Path를 최적화(Chunk 파이프라이닝, 압축)해 Election 지연을 줄였습니다.
- HDD / SSD / NVMe / Tape 매체 특성 차이를 Object Storage 설계에 반영하는 방식? (D)
	> 답변: NVMe(낮은 µs 단위 Latency, 높은 IOPS)는 메타/인덱스/Hot Small Object 캐시에, SSD(Sustained Read/Write 균형)는 Warm Tier, HDD(높은 용량/MB/s)는 Cold Large Object, Tape(매우 낮은 비용/높은 접근 지연)는 아카이브/규제 보존에 매핑합니다. Heat Score=f(최근 Access 빈도, Recency, Size, 변경 빈도)를 계산해 임계 초과 시 비동기 Migration Queue로 이동시키고, GB-month + Access Cost 모델(TCO) 지표를 대시보드화해 저장비 최적화를 지속 추적합니다.
- S3 호환 API 설계 시 필수 고려 헤더와 권한/서명(Auth) 처리 흐름? (D)
	> 답변: 필수 헤더/요소: Authorization(Signature V4), x-amz-date(or Date), x-amz-content-sha256, 필요 시 Content-MD5, 조건부 If-Match/If-None-Match, Range. 절차: (1) Canonical Request 생성(헤더 정렬·Lowercase·중복 공백 정규화, Query 정렬) (2) StringToSign(HASH(Date, Region, Service, SigningKey)) (3) HMAC-SHA256 계산 및 비교 (4) 키 권한/버킷 정책/ACL/Condition 평가 (5) 조건부 요청 선별(Return 304/412 등) (6) 본 동작 수행 + ETag/Version 반환 (7) Audit/Event Publish. 인증(HMAC)과 데이터 무결성(Content-MD5/Checksum)을 구분해 Layered Defense를 구성합니다.
- 대규모 삭제(Object Lifecycle / Expiration) Batch 처리 전략? (D)
	> 답변: 만료 예정 객체는 Min-Heap(Time, Key) 혹은 Time Wheel Slot에 스케줄합니다. Worker가 Batch 단위(예: 100~1000)로 Tombstone 마킹(메타 경량 기록)하고, 물리 블록 삭제는 Background GC/Compaction이 Rate Limit로 점진 수행해 Foreground Latency를 보호합니다. Idempotent 보장을 위해 Tombstone 존재 여부로 중복 제거, 지표는 Queue Lag, Purge Throughput, Orphan Chunk Ratio, Space Reclaim Latency를 추적합니다.
- 데이터 일관성(Strong vs Eventual) 요구가 설계/성능에 주는 영향? (D,E)
	> 답변: 글로벌 Strong Read-after-Write를 강제하면 모든 Read가 Leader(또는 Quorum) 경로로 몰려 Latency·비용 상승, Failover 시 가용성 일시 저하가 발생합니다. 반면 Eventual은 빠르지만 짧은 시간 Stale 가능성이 있습니다. 실무 절충: 메타(버전/이름/권한)는 Strong(Raft Commit 인덱스 기반 Read Index)으로 정확성 확보, 대용량 바디는 Eventual Replica에서 Serving + ETag/Checksum 검증 및 배경 Repair(Read Repair/Merkle Tree)로 최종 일관성 수렴을 보장하는 하이브리드입니다.
- SOS 프로젝트의 Chunk 저장 구조(LevelDB)에서 GC 혹은 Compaction 정책은 어떻게? (D)
	> 답변: LevelDB Leveled Compaction 기본 정책 위에 외부 Live Reference Table을 유지(객체 메타→Chunk 리스트)하여 Tombstone 이후 참조 끊긴 Chunk를 Batch 식별→Purge Queue에 넣습니다. 정기 Compaction 후 Space Amplification 측정(Used/Logical)하여 임계(예: 1.4x) 초과 시 Key-Range 강제 Compaction을 트리거, Long Tail IO 방지를 위해 Rate Limit와 스케줄 윈도우(저부하 시간대)를 둡니다.
- 대량 Put 시 Hot Partition을 줄이기 위한 Key 설계 기법? (D)
	> 답변: Key Prefix에 Uniform Hash(또는 Shard Code) 삽입, 짧은 주기 Time Salt(분 단위) + Random Suffix로 Burst를 분산합니다. Range Scan 요구가 있으면 주키(시간/정렬 가능)와 분산 Prefix(해시)를 분리(Composite Key: hash#timestamp)하고, 조회 시 Secondary Index/Manifest를 통해 순차 재구성합니다. Hot Key 탐지(짧은 윈도우 QPS 편차) 시 Adaptive Rehash(추가 Prefix 비트) 또는 Write Throttle, 인기 객체는 Edge Cache/Replica 확장으로 흡수합니다.

## 2. 분산 시스템 / 합의 / 신뢰성

- CAP 정리와 Object Storage에 실제로 요구되는 선택 조합 예시? (B,D)
	> 답변: CAP은 네트워크 Partition 상황에서 Consistency vs Availability 중 어떤 속성을 유지할지 선택입니다. Object Storage에서는 "메타데이터(이름/버전/권한)" 충돌 비용이 크므로 Leader 기반 Strong(사실상 CP 경향)으로 유지, "대용량 객체 바디"는 약간 지연/재조합 허용(Eventual)해 AP 특성을 활용해 비용과 지연을 낮춥니다. 즉 Metadata=CP, Data=AP 하이브리드로 SLA(정확한 명명/버전)와 비용(복제 대역폭/쓰기 지연)을 균형화합니다.
- Raft vs Paxos vs Gossip 기반 멤버십 선택 기준? (D)
	> 답변: Raft는 Leader Append → Majority Commit 구조가 직관적이라 로그 일관/State Machine 복제에 적합하고, Paxos는 이론적 범용성이 높지만 구현·디버깅 복잡성이 실무 비용을 높입니다. Gossip은 헬스/멤버십/로드 메트릭 전파에 Eventually Consistent + O(log n)에 가까운 수렴 특성으로 가볍습니다. 따라서 "강한 순서/트랜잭션 경로=Raft", "상태/헬스/로드 전파=Gossip" 이원화가 운영 단순성과 확장성 균형을 줍니다.
- Leader 선출 지연이 클라이언트 가용성에 미치는 영향과 완화책? (D,E)
	> 답변: 선출 지연 동안 쓰기 중단 및 일부 읽기(Strong Read) Fail/지연 증가가 발생합니다. 완화: (1) Pre-Vote로 불필요 Term 증가 및 Split Vote 억제 (2) Election Timeout Random Jitter (3) Snapshot/Log Install 최적화로 Lagging Follower 빠른 Catch-up (4) Lease Read(ReadIndex)로 안전한 빠른 선형화 읽기 (5) 빠른 장애 감지(Heartbeat Timeout 단축 + 적절한 Quorum) (6) 모니터링: Election Duration, Failed Vote, Write Unavailable Seconds, Leadership Stability 지표.
- 장애 시 Read Repair / Anti-Entropy 전략을 어떻게 구성할지? (D)
	> 답변: 주기적으로 Shard/Partition 단위 Merkle(또는 Segment Hash) Tree를 교환해 Divergence Key 목록만 추려 네트워크 사용을 최소화합니다. 클라이언트 Read 시 Version/ETag 차이 감지되면 최신 블록을 비동기 Push(Read Repair)하고, Partition 발생 중 수신 못한 업데이트는 Hinted Handoff Queue에 기록 후 재연결 시 우선 적용합니다. 지표: Divergence Count, 95p Repair Latency, Repair Bandwidth, Hinted Queue Lag.
- 네트워크 Partition 발생 시 Write 처리 정책? (D)
	> 답변: Quorum 상실 시 옵션: (1) 즉시 Reject(Consistency 우선) (2) 임시 수락 후 Fencing Token/버전 Vector로 충돌 해결 (3) 로컬 Append Log 버퍼 후 재합류 시 합병입니다. 메타데이터는 충돌 비용이 커 Reject/Fail-Fast, 데이터 Chunk는 Append 특성상 임시 수용 + 백그라운드 Reconcile 허용이 경제적입니다.
- 재시도/중복 업로드(Idempotency)를 어떻게 구현? (D)
	> 답변: 클라이언트 Idempotency-Key(or UploadId)를 수신하면 최초 성공 결과(상태/ETag/버전)를 KV/TTL 캐시에 저장, 재시도는 저장된 결과 반환으로 부작용 제거합니다. Multipart는 (UploadId, PartNumber) PK + Completed Set(비트맵) 추적으로 중복 Part 빠르게 단락시킵니다. Put 시 조건부 헤더(If-Match/If-None-Match)와 결합해 경쟁 업데이트도 안전하게 처리합니다.
- Kafka를 이용한 이벤트 동기화(myStream) 아키텍처 구조 설명? (B,D)
	> 답변: 메타 Commit 이벤트를 Topic에 Publish(Partition Key=Bucket or Object Hash → 순서 보존). Consumer Group은 기능별(Replication, Lifecycle, Index) 분리하고 각 Partition 오프셋 관리로 독립 확장합니다. Lag 임계 초과 시 Publish Backpressure(프로듀서 배치 크기/압축 조정)와 Consumer 병렬/Fetch Size 튜닝으로 흡수, Idempotency는 (Partition, Offset, ObjectVersion) 조합으로 처리해 재처리 시 중복 Side-effect 방지합니다.
- Exactly-Once가 어려운 이유와 At-Least-Once 보정 패턴? (D)
	> 답변: 소비 처리와 오프셋 Commit/ACK 사이 경계에서 장애가 나면 "실행 O, Commit X" 또는 반대 상황이 생겨 중복/유실이 발생합니다. 이것을 절대 배제하려면 End-to-End 트랜잭션(2PC/원자 로그)이 필요해 복잡·지연 비용이 큽니다. 따라서 실무는 At-Least-Once + Idempotent Sink(Version 조건 Put), Outbox(로컬 Tx 내 기록→Relay Publish), Dedup Cache(Bloom+TTL)로 최종 상태 정확성을 보장합니다. 재시도는 지수 백오프 + 최대 시도 제한, Poison 메시지는 DLQ로 격리합니다.
- 분산 트랜잭션(2PC / Outbox) 적용 여부 판단 근거? (D)
	> 답변: 판단 질문: (1) Cross-Object 원자성이 진짜 비즈니스 필수인가? (2) 실패 시 보상 처리 비용은? (3) Latency Budget 허용치는? 단일 Key 위주 Object Storage 메타 경로에 2PC는 Coordinator 복잡성·락 유지·지연 증가를 초래해 과도합니다. 대신 Metadata Commit + Outbox 레코드 동일 로컬 Tx 기록 → 별도 Relay가 Kafka Publish 하는 Outbox 패턴으로 Eventually Consistent 파이프라인을 구성, 재처리/중복은 Idempotent 소비로 흡수합니다.

## 3. 성능 / 최적화 / 모니터링

- Throughput vs Latency 목표 설정 시 우선순위 결정 방식? (B)
	> 답변: 첫 단계로 워크로드 프로파일을 계측(요청 크기 분포, 동시성, Read/Write 비율, 사용자 체감 지연 한계 SLO)하여 지표 간 상관을 확보합니다. 일반적으로 p99 Latency SLO 위반이 재시도 폭증→Throughput 하락을 유발하므로 우선순위는 (1) 안정적 p99 (2) 그 다음 집계 Throughput입니다. 의사결정 프레임: Amdahl/Gustafson 관점으로 병목 레이어(네트워크, 디스크, 메타 저장소) 별 지연 기여율을 분해하고, Latency 개선 1ms가 오류/재시도율 감소→QPS 증가로 환산되는 경제적 가치(추가 처리량 * 단위 가치)를 모델링합니다. 결과적으로 버스트/변동 큰 서비스 초기에는 Tail Latency(Percentile Spread)를 줄여 재시도 폭발을 막고, 안정화 후 배치/대량 업로드 경로에서 Throughput 개선(멀티 파이프라인, 배치, 압축)을 다룹니다. 지표 트리: User p95 → Internal p99 → Component Queue Wait → CPU Run Queue / IO Wait.
- Chromium 기반 브라우저 CPU 사용량 최적화(Tile 조정, GPU Raster) 경험을 스토리지 I/O 최적화에 전이한다면? (E)
	> 답변: 브라우저에서 '필요한 작업만 먼저, 비가시 영역 지연, 파이프라인 병렬화' 원칙을 적용했듯이 스토리지에서는 (1) 요청 분류(Hot Meta, Large Sequential, Small Random) (2) 중요 경로(Latency Critical) 우선 스케줄링 (3) 비핵심 비동기화(Checksum, Async Replication)로 전이합니다. Tile 크기 조정 경험은 Chunk/Part Size 튜닝과 유사: 너무 작으면 Overhead 증가, 크면 재전송 비용 커짐. GPU Raster 병렬화는 업로드 다중 Part 병렬 / 다중 큐(Submit, Compute, Flush) 분리를 떠올리게 해 CPU→IO 파이프라인을 Stage 분리 후 각 Stage의 병렬도와 큐 길이를 모니터링합니다. 또한 브라우저 Paint Skipping처럼 동일 GET 키 동시 요청 Coalescing, Backpressure(Queue Depth 기반 동적 Concurrency 조절) 전략으로 자원 효율을 극대화합니다.
- GPU 인코딩 도입으로 30% CPU 개선 측정 방법과 실험 설계? (D)
	> 답변: 가설: GPU Offload가 CPU 바운드 구간(압축/인코딩) 점유를 낮춰 동일 하드웨어 QPS가 증가한다. 설계: (1) 대조군: 기존 CPU 경로 (2) 실험군: GPU 경로; 동일 데이터 셋(크기/포맷 혼합) 재생성 불가 편향 방지를 위해 고정 Seed 샘플링. (3) 워밍업 후 15~30분 안정 구간 측정. 수집 지표: CPU Sys/User %, GPU Util, p50/p95/p99 Latency, Throughput(QPS or MB/s), 에너지(선택), Fail/Requeue Rate. 분석: Throughput 대비 Latency Degradation 없는지, Tail 확대 여부, Context Switch 감소 여부(strace/perf). 30% CPU 절감이 실제 코어 해방→추가 워커 확장으로 이어져 총 처리량 % 상승을 2차로 검증합니다. 회귀 위험: GPU 큐 대기 증가→Tail 악화. Mitigation: 배치 크기/동시 offload 제한 튜닝 실험(DoE) 실시.
- Profiling 도구/지표(IOPS, P99 Latency, Write Amp, Cache Hit) 수집 파이프라인 설계? (D)
	> 답변: 계층화 아키텍처: (1) 에이전트(노드별 Exporter: eBPF/kprobe로 sys_read/write, block layer latency 히스토그램, RocksDB Stats, Go runtime metrics) (2) 수집(옵션: Prometheus + PushGateway for Batch) (3) 스트림 처리(Kafka → Flink/Spark for Percentile Rollup) (4) 장기 보관(Columnar TSDB, e.g. M3/Thanos) (5) 대시보드/알람(Grafana). P99는 Raw Histogram(HDR or Circllhist) 기반 서버 측 집계(중앙에서 단순 평균 금지), Write Amplification은 (Physical Bytes Written) / (Logical Bytes) 계산을 위해 WAL+SST I/O 카운터를 수집합니다. Correlation 탐색 위해 Trace(OpenTelemetry)에서 특정 느린 요청 Span ID를 Metrics Tag와 조인, 샘플링(Adaptive: Tail-heavy)을 적용합니다. 장애시 Last N Minutes 고해상(1s) 버퍼 + 장기(1m) 다운샘플 2계층 보관을 둡니다.
- 백엔드 리팩토링 성능 개선 사례(지표 전/후) 설명? (B,D)
	> 답변: 예시: 메타데이터 조회 경로에서 JSON Unmarshal 다중 단계 + 반복 해시 계산으로 p99=120ms. 최적화: (1) 구조체 사전 컴파일(코드 생성) (2) Canonical Key 해시 캐시 (3) Batch GET 도입 (4) Goroutine Pool 재사용. 결과: p99 120→55ms (-54%), CPU User 30%→18%, GC Pause p95 12ms→6ms. 회고: 병목은 알고리즘이 아니라 중복 변환/할당이었으며, 개선 후 Disk IO 대기 비중이 커져 다음 단계는 Block Cache Hit 향상을 위한 Bloom 튜닝으로 이동.
- Gstreamer 파이프라인 튜닝 논리와 유사한 데이터 처리 파이프라인 병목 진단 절차? (E)
	> 답변: Gstreamer에서 Element 간 큐/시계 동기/Backpressure를 본 것처럼, 스토리지 파이프라인(Ingress → Auth → Meta → Data Write → Async Event) 각 Stage를 노드로 모델링하여 (Queue Depth, 처리율, 평균/최대 체류시간)을 시계열로 수집합니다. 병목 진단 순서: (1) 전체 Latency 분해(Wall Clock Trace) (2) 가장 긴 Queue 체류 Stage 식별 (3) CPU vs IO vs Lock 대기 분류(perf + pprof + eBPF) (4) 상위 1~2 Stage에 대해 병렬도 조정/Batch 크기 실험 (5) Tail 영향 측정(Var, p99 Spread). 파이프라인 튜닝 원칙: Stage 균형(Throughput 비슷하게), 필요시 Bypass(저가치 처리 지연) 또는 Fuse(연속 CPU Bound Stage 통합) 적용.

## 4. 데이터 구조 / 스토리지 내부

- Skiplist 인덱스 구조 동작 원리와 B-Tree 대비 장단점? (B,D)
	> 답변: Skiplist는 다층 레벨(L1=Full, Ln=확률 p^n 샘플) Forward Pointer로 로그 시간 기대 검색을 제공합니다. 장점: (1) 단순 구현 (2) 순차/Range Scan O(k)로 연속 메모리 접근 패턴 양호 (3) Lock-free/Optimistic 동시성 패턴 적용 용이 (4) 삽입 시 국소적 재배치(균형 트리 회전 없음). 단점: (a) 포인터 Overhead (메모리 단편 + 캐시 미스) (b) Worst Case O(n) 가능성(확률적) (c) B-Tree 대비 디스크 친화성 낮음(노드 압축/페이지화 덜 효율). 메모리 내 LSM MemTable엔 Skiplist가 적합하지만, 디스크 구조엔 B-Tree/B+Tree가 페이지 IO 효율이 좋아 적합합니다.
- LSM-Tree 단계별(Flush, Compaction) 비용과 Write Path 상세? (D)
	> 답변: Write Path: WAL Append(fsync 정책) → MemTable Insert(Skiplist or Hash) → MemTable Full 시 Immutable 전환→Flush(SST 생성) → 다수 SST 축적 시 Compaction(상위→하위 Merge + Tombstone 적용). 비용: Flush는 순차 쓰기(저렴)지만, Compaction은 다중 파일 Merge로 (Read Amp + Write Amp) 발생. Level Size Ratio가 크면 하위 레벨 비대→Merge Cost 급증, 작으면 레벨 수 증가→Lookup Depth 증가. 최적화: (1) 압축 전략(상위 레벨 빠른 LZ4, 하위 ZSTD) (2) Bloom Filter 조기 미스 단축 (3) Compaction Picker: Hot Range 우선, Cold Range 지연 (4) Rate Limit/IO Priority로 Foreground Tail 보호 (5) Partial/Key Range Compaction.
- Bloom Filter False Positive Rate 계산 요소(k, m, n) 조정 경험? (D)
	> 답변: 목표 FP p_target 주어지면 m = -(n * ln p_target)/(ln2^2), k = (m/n)*ln2. 실제 n 추정이 빗나가면 p 상승→Random IO 증가. 경험적으로 초기 n 예측에 10~20% 여유를 둬 m을 크게 할당, 런타임 Telemetry(요청 수 대비 Miss 후 디스크 Hit 비율)로 실제 FP 역추정합니다. p가 목표를 2배 초과하면 해당 SST Bloom 재생성(Backfill) 또는 차기 세대 SST에 새 파라미터 적용. k 과다 시 CPU Hash 비용↑, 과소 시 p↑. Murmur/XXHash 조합 + SIMD Bitset Probe로 CPU 영향 최소화.
- Key 설계 시 Prefix Locality와 Range Query Trade-off? (D)
	> 답변: Prefix(Locality)=동일 Prefix 데이터가 연속 저장→Range Scan/압축 효율↑/Cache Hit↑. 그러나 Hot Prefix 집중→Hot Partition. Trade-off 해결: (1) Write 시 Hash Prefix + 자연키 분리(hash#natural) 저장, (2) Range Scan 필요 시 Manifest/Secondary Index에서 자연 순서 키 목록 획득 후 Batch Random Read 병렬화. 또는 Time-bucket + Hash Salt(시간 단위 변경)로 단기 분산 + 장기 순차성 유지. 의사결정은 Range Scan 빈도 * 길이 vs Hot Key QPS 편차를 비용 함수로 모델링.
- LevelDB / RocksDB류 엔진에서 Block Cache와 OS Page Cache 상호작용? (E)
	> 답변: Block Cache(유저 공간 LRU/Clock)는 압축 해제된 Block을 저장, OS Page Cache는 파일 시스템 레벨 Raw Page(압축된 바이트)를 보유. 중복 캐싱 오버헤드가 있으나 (1) 압축 해제 비용 절감 (2) 사용자 키 접근 패턴 기반 세밀한 Eviction 정책 구현 때문에 Block Cache가 필요합니다. 조정 포인트: (a) 총 메모리 예산을 Page Cache vs Block Cache 비율로 나누어 파일 시스템 재탐색을 줄이되, 너무 큰 Block Cache는 커밋 지연(WAL flush) 시 OS Dirty Ratio 압박 유발 (b) Direct IO + 자체 캐시 전략으로 OS Page Cache 우회를 선택 가능 (c) Bloom/Index Block은 Pin, Data Block은 LRU. 관측 지표: Cache Hit, Compressed Miss→Decompress Latency, Page Cache Read Hit, Write Stall.

## 5. 네트워크 / 프로토콜 / 미디어 경험 전이

- HTTP/1.1 vs HTTP/2 선택이 Object Storage 대량 업로드에 주는 영향? (D)
	> 답변: 핵심 축은 연결 수 관리와 HOL(Head-of-line) Blocking 완화입니다. HTTP/1.1은 대량 병렬 업로드 시 다수 TCP 커넥션(커넥션 폭발→핸드쉐이크/커널 자원/컨텍스트 스위치 비용) 필요, 파이프라이닝은 실사용 제약(HOL, 프록시)으로 비활성화되는 경우 많습니다. HTTP/2는 단일 TCP 위 다중 Stream(Multiplex)으로 커넥션 수를 줄이고 서버 측 커넥션 재활용/Flow Control로 공정성을 확보할 수 있으나, 하나의 TCP 손실이 모든 Stream RTT 지연을 유발하는 HOL at TCP Layer 문제는 남습니다. 대용량 Multipart 업로드에서는 (1) Stream 우선순위 부여(메타/작은 Part 우선) (2) 윈도우 동적 조절(BDP 기반) (3) TLS 세션 재활용 + 0-RTT(HTTP/3 고려)로 초기 지연 감소 전략을 결합합니다. 실측 관점: HTTP/2 전환 후 커넥션 수 60% 감소, p95 Handshake Latency 개선, 단 패킷 손실률 높은 구간에서 단일 커넥션 혼잡 윈도우 축소→Throughput 하락 위험을 대비해 업로드 도메인 분리(다수 커넥션 Shard) 혹은 HTTP/3(QUIC) 실험을 병행합니다.
- RTMP -> HLS Transmuxing 파이프라인 단계와 유사한 대용량 업로드 처리 파이프라인 설계? (E)
	> 답변: RTMP→HLS는 (수신 → Demux → Segmenter → Transcode → Playlist 업데이트) 단계로 분절/버퍼링 제어합니다. 이를 업로드 파이프라인에 대응시키면 (Ingress TLS Termination → Auth/RateLimit → Upload Session Init → Part Stream Ingest → Chunk Assembly/Checksum → Async Replication/Event Fanout) 으로 Stage화합니다. Segmenter 개념은 Multipart Part Size 결정과 유사하고, Playlist 갱신은 메타 Commit(Version & ETag 가시화)에 대응합니다. 병목 완화는 각 Stage Queue Depth 모니터링 + Backpressure(Upstream Read Slow) 적용, Transcode 지연에 해당하는 Checksumming/Compression을 비동기 처리하거나 CPU Pinning으로 Tail을 줄입니다. 장애 복구는 RTMP 재연결과 유사하게 UploadId 기반 Resume로 구현, 마지막 Commit된 Part Number 이후만 재전송하도록 하여 Partial 실패 비용을 최소화합니다.
- MQTT/Kafka/WebSocket을 사용해 본 관점에서 Control Plane과 Data Plane 분리 설계? (D)
	> 답변: Control Plane은 저용량/고중요 이벤트(세션 협상, 업로드 Intent, 권한 토큰 갱신, 상태 Heartbeat) 위주로 신뢰성과 순서를 우선, Data Plane은 고대역폭(객체 바디, Part 스트림)으로 Throughput·지연 튜닝을 우선합니다. 구현: Control Plane은 gRPC/HTTP2 + 재시도/Idempotency-Key + 작은 메시지 압축, Data Plane은 HTTP(S)/QUIC 직스트리밍 또는 S3 PutObject/UploadPart API 직경로. Kafka는 비동기 후처리(Replication/Lifecycle) 이벤트 버스, MQTT/WebSocket은 실시간 업로드 프로그레스/세션 KeepAlive 채널로 사용. 분리는 (1) 장애 격리 (2) 리소스 QoS 차등 (3) 독립 Scale-out 이점. 공통 상관ID(Correlation ID)로 Trace 결합, Control → Data 승인 토큰에 짧은 TTL & Scope 최소화로 보안 강화.
- TCP 튜닝(윈도우, Nagle, KeepAlive)으로 업로드 성능 개선 사례 가정? (E)
	> 답변: 병목: 고 RTT(>80ms) 환경에서 작은 Part 전송 시 BDP 미활용 + Nagle 지연으로 p95 업로드 시간이 늘어남. 조치: (1) Part Size 조정(BDP≈Bandwidth*RTT에 근접) (2) Nagle 비활성(TCP_NODELAY)로 대기 제거 (3) Initial Congestion Window 10→16 패킷 상향(커널 파라미터, 빠른 Slow Start) (4) TCP autotuning 수신 버퍼 여유 확보 (5) KeepAlive 간격 최적화로 장시간 Multipart idle 동안 커넥션 Re-establish 비용 감소. 결과 가정: 평균 Throughput +18%, p95 Part ACK Latency -22%. Side Effect: 작은 메시지 폭주로 패킷 수 증가→CPU per packet 비용 상승, 이를 Offload(GRO/TSO) 및 배치 API(Writev)로 완화.

## 6. Go / C++ / 언어 심화

- Go GC 동작(Generational 아님) 특성과 메모리 관리 최적화 패턴? (D)
	> 답변: Go GC는 tri-color mark and sweep, concurrent, non-generational 구조(점진적 write barrier)라 대량 단명 객체가 많으면 Minor GC 이점이 있는 언어 대비 부담이 큽니다. 최적화: (1) Escape Analysis 친화 코드(heap 탈출 억제, value receiver, stack 할당) (2) sync.Pool로 재사용(단, Hot Path에서 contention 관측되면 Sharded Pool) (3) Large Buffer Arena(미리 할당 후 slice subslicing) (4) 구조체 Zero-allocation Builder 패턴 (5) 프로파일(go tool pprof alloc_space) 기반 상위 타입 집중 개선. GC 목표 비율(GOGC) 조정으로 메모리 vs CPU 트레이드오프 관리; Latency 민감 경로는 allocation-free fast path 확보.
- Go에서 sync.Pool / Worker 패턴(개인 worker 패키지) 사용 기준? (D)
	> 답변: sync.Pool은 GC 직후 버려질 수 있는 비결정성 특성을 가지므로 (1) 생성 비용 높고 (2) 재사용 직전 초기화가 간단하며 (3) 사이즈 일정한 객체 (예: encoder buffer, small struct)에 적합. Worker Pool은 (a) 처리량 향상 목적이 아니라 외부 리소스 동시성 제한(IO, DB Connection) (b) 큐잉으로 Backpressure (c) Task 배치/집계 필요 시 선택. 기준: 초당 생성 100k 이상, 평균 생존 매우 짧은 버퍼는 Pool, 외부 API 호출 concurrency 500→50 제한 필요 시 Worker Pool. 주의: Pool overuse는 캐시/false sharing 유발 및 Latency 증가. 측정: 히스토그램(할당 크기), STDDEV, p95 Latency.
- Goroutine Leak 탐지/예방 전략? (D)
	> 답변: Leak 패턴: 채널 송신 대기, context 미취소, for { select { ... default: } } Busy Loop. 예방: (1) context.WithCancel/Timeout 전파 (2) 생산자-소비자 채널 close 규약 문서화 (3) goroutine 수 계측(Metrics + pprof goroutine dump 주기 분석) (4) static vet/linters(channel misuse) (5) Select 누락 case에 time.After leak 방지 타임아웃 추가. 탐지: 테스트에서 goroutine snapshot 전/후 diff, 의도된 장수 루틴 화이트리스트. 운영: OOM 전 메모리/스레드 급증 그래프 경보.
- Chromium Task 기반 비동기 모델과 Go Scheduler 비교? (E)
	> 답변: Chromium은 전용 Thread Pool + Task Queue(특정 시멘틱: UI, IO, Compositor)로 우선순위/실행 순서를 명시 제어, Go는 M:N 스케줄러가 Work Stealing으로 자동 분배. 장점 비교: Go는 개발 생산성(명시 큐 생성 필요↓), Chromium 모델은 우선순위 역전/Latency SLO 관리 용이. 스토리지 서버에서 혼합 워크로드(짧은 메타 요청 vs 대형 업로드 처리) 시 Go 환경에서는 (1) 별도 우선순위 큐 구현 (2) runtime.GOMAXPROCS 및 세션별 semaphore로 IO heavy 고루틴 억제 (3) pprof + trace로 STW/스케줄 지연 모니터링. 필요시 특정 경로를 별도 Worker Pool/Rate Limit로 격리해 Chromium Task Queue 유사 제어 달성.
- Zero-Copy I/O를 구현하거나 흉내낸 경험/아이디어? (E)
	> 답변: Linux에서 sendfile/splice 이용 시 유저 공간 복사 없이 Page Cache→Socket 전송 가능, 단 암호화/압축 삽입 지점 제약. 아이디어: (1) 업로드 수신은 mmap + copy-less hashing(Chunk 경계 해시 rolling) (2) 객체 GET은 sendfile 경로 기본, 조건부 Range 병합 시 io.CopyBuffer 최소화 (3) TLS 경로는 kTLS(가능한 커널 버전)로 암호화 offload (4) 압축/암호화 필요한 경우 double-buffer 대신 ring buffer + scatter-gather writev 사용 (5) Checksumming은 SIMD + zero-copy slice walk. 측정: syscount(context switches), CPU cycles per byte, cache miss, NIC throughput.

## 7. Linux / 시스템 / 디버깅

- strace / perf / eBPF 활용한 병목 분석 절차? (D)
- > 답변: 단계화된 접근을 고수합니다. (1) 1차 증상 수집: dmesg, sar(iostat, vmstat), pidstat, ss -s, top/BPF 기반 run queue 길이로 리소스( CPU / IO / 메모리 / 네트워크 ) 중 어떤 축이 주범인지 가설을 세웁니다. (2) syscalls 레벨: strace -f -tt -T -e trace=read,write,open,fsync 등 상위 몇 초 샘플링으로 블로킹 구간(Time delta)과 호출 패턴(짧은 read 폭주, fsync 빈도)을 확인; Overhead 최소화를 위해 -c 통계 모드와 조건부(-p 특정 PID) 사용. (3) CPU 바운드 의심 시 perf top / perf record + perf report 로 hottest symbol (user vs kernel)과 branch miss, LLC miss 비중, off-CPU(스케줄 아웃) 여부(perf sched record) 확인. (4) IO / 커널 경합이면 eBPF(bcc/bpftrace) 툴: biolatency, biosnoop, tcplife, runqlat, offcputime, filetop 로 latency distribution과 자주 기다리는 lock/파일을 추출. (5) Correlate: 동일 타임스탬프 축으로 애플리케이션 로그(traceID)와 커널 이벤트(join). (6) 가설 검증: ex) 짧은 read 폭주 → read-ahead / larger batch 개선 실험, fsync 과다 → group commit or O_DSYNC→WAL batch 전환. 성공 기준은 p99 지연 ∆, syscall rate 변화, CPU cycles per request 감소. 회고 단계에서 재발 방지를 위해 해당 지표를 지속 모니터링 대시보드에 편입합니다.
- 파일시스템(Page Cache, Dirty Page Flush) 이해가 Write Latency에 미치는 영향? (D)
- > 답변: 애플리케이션 write()는 대부분 Page Cache에 흡수되어 즉시 반환되지만 Dirty Page 비율이 vm.dirty_background_ratio / vm.dirty_ratio 임계에 접근하면 kswapd / flush thread가 대량 writeback을 발생시켜 Flush Burst → Tail Latency 상승을 초래합니다. 또한 small sync write 패턴이 많으면 Journal + Data double write(EXT4 Ordered)로 Write Amplification이 증가. 대응: (1) WAL + 배치 fsync (2) Direct IO 선택(캐시 혜택 적고 재차 읽기 적은 순차 대형 쓰기) (3) Dirty Throttling이 발동하기 전 flush 간격 고르게: fadvise(DONTNEED), Background flush thread (4) NUMA 환경에서는 페이지 로컬리티(첫 touch 정책)로 cross-node 메모리 접근 줄임. 관측 지표: Dirty ratio trend, writeback latency histogram, blk io queue depth. 결과적으로 write path 지연 편차 감소(p99→p95 수렴)와 장기 tail 안정성을 확보합니다.
- 컨테이너(Docker) 환경에서 스토리지 성능 편차 원인(Cgroup, fs driver)? (D)
- > 답변: 편차 주요 원인은 (1) Cgroup blkio / io.max 설정 및 Weight 경쟁 (2) OverlayFS (upper / lower layer copy-up)로 인한 작은 파일 쓰기 성능 손실 (3) Host 커널 I/O 스케줄러(mq-deadline vs none) 및 큐 공유 (4) Container ↔ Host page cache 공유로 인한 예상치 못한 eviction. 해결: 대용량 순차 IO 워크로드는 전용 hostPath + XFS/EXT4 direct mount, Read-most layer 이미지는 squashfs + 데이터 레이어 분리, blkio Weight 명시 & 중요한 워크로드 전용 디바이스(namespace). 관측: cgroup_io_stat, iolatency BPF, container별 cache miss / throttled time. 재현: 동일 베이스 이미지/노드에서 fio 재실험 후 차이 분석. 최종적으로 프로덕션에서는 'Hot Path 컨테이너 = Direct Mount + QoS Weight 고정' 정책 문서화합니다.
- Wayland 연동/디버깅 경험을 커널/유저 공간 인터페이스 이해로 확장 설명? (E)
- > 답변: Wayland는 최소화된 코어 프로토콜 + 컴포지터 중심 아키텍처로 명령 버퍼(이벤트/요청)를 비동기 교환하고, SHM / DMA-BUF로 zero-copy buffer 공유를 합니다. 이는 커널/유저 공간 경계에서 불필요한 round-trip과 copy를 제거하는 설계 사고를 촉진: 스토리지 I/O에서도 (1) syscalls 수 감소(sendfile/splice) (2) 비동기 submission(io_uring) (3) 공유 메모리 ring (lock-free queue) 로 커널 전환 비용을 줄입니다. 디버깅 시 Wayland protocol snoop처럼 eBPF uprobes/kprobes로 특정 I/O 함수 호출 흐름을 캡쳐해 user intent → kernel action 매핑을 시각화, latency outlier 구간(예: bio dispatch→completion) 식별합니다. 이런 mental model이 고성능 데이터 경로 설계에서 '필요 최소 인터페이스 + zero-copy + latency tracing hook' 원칙으로 직결됩니다.
- 고 IOPS 상황에서 IO 스케줄러 선택 영향? (E)
- > 답변: NVMe 다중 큐 환경에서는 none(noop) 또는 mq-deadline이 일반적으로 가장 낮은 CPU 오버헤드와 예측 가능한 지연을 제공합니다. cfq/kyber은 혼합 워크로드 QoS에 장점 있으나 순수 랜덤 읽기/쓰기 혼합 고 QD(Queue Depth)에서 추가 계층 비용으로 tail 증가. 측정 접근: fio (iodepth 1→64, rw=randrw, mix=70/30) + blktrace + biolatency BPF; 선택 기준은 (1) p99 latency (2) IOPS 안정성 표준편차 (3) CPU cycles per IO. 결과: mq-deadline에서 write starvation 방지 + 읽기 p99 5~7% 낮음 → 채택. 운영 중 queue depth spikes 관측 시 autotune(ionice or cgroup io)로 보호.
- IOPS 병목 분석?
- > 답변: 절차: (1) fio synthetic 대비 실제 IOPS 비율 산출(기대치 70% 미만이면 병목 의심) (2) iostat -x로 svctm, %util, await, avgqu-sz 확인; %util 100% 지속이면 디바이스 포화 → 상위 레이어 확인 (3) blktrace/ebpf(biosnoop)로 IO size distribution, merge ratio, 재정렬 여부 (4) perf 혹은 offcputime으로 block layer 락 경합(예: q->mq) (5) 파일시스템 Lock(inode, extent tree) 또는 LSM compaction IO 폭주 여부를 app metrics와 상관. 개선 옵션: IO size up(batching), 병렬 shard 분리, 압축/Checksum 비동기화, write combining. 목표: 동일 하드웨어 대비 p99 latency 상승 없이 IOPS Utilization 85~90% 달성.

## 8. 아키텍처 / 설계 / MSA / 이벤트

- myStream DDD 적용 과정(Event Storming, Context Map)에서 얻은 인사이트? (B,D)
- > 답변: Event Storming에서 '업로드 시작', '세그먼트 인코딩 완료', '라이프사이클 만료' 등 도메인 이벤트를 타임라인으로 배치하자 Aggregate 경계가 자연스럽게 'Session', 'MediaObject', 'TranscodeJob'으로 수렴했습니다. 이는 Object Storage에서도 'UploadSession', 'ObjectMeta', 'LifecyclePolicy'와 1:1 유사 매핑되어, 변경 책임(불변/상태 머신 단계) 단위로 마이크로서비스를 나누면 트랜잭션 경계가 단순해짐을 재확인. Context Map으로 Upstream(인증) → Storage Core → Downstream(Analytics/Index) 의 의존 흐름을 시각화해 Outbox 이벤트 발행 지점을 표준화했고, ACL/권한은 Core 앞단 Anti-corruption Layer에 집중 배치하여 하위 서비스 단순화를 달성했습니다.
- 서비스 경계(Bounded Context) 식별 기준과 팀 구조 매핑? (D)
- > 답변: 기준: (1) 서로 다른 Ubiquitous Language (2) 트랜잭션 일관성 필요 범위 (3) 변경 빈도의 상관관계 (4) 독립적 Scale 패턴 (5) 보안/규제 경계. Object Storage 예: Auth(토큰/정책), Metadata(이름/버전), Data IO(바디 스트림), Lifecycle(정책 실행), Analytics(Index/Search). 팀 매핑은 변경율 높은 Metadata+DataIO를 Core 팀, 규칙/스케줄 중심 Lifecycle을 Platform, 인덱싱/검색을 Data 팀으로 구성. KPI: 팀별 배포 빈도, 장애 Blast Radius, 의존 지연(교차 PR) 감소. 경계 설계 후 교차 호출을 이벤트/비동기 조회 패턴으로 전환해 결합도 저감.
- 이벤트 기반 동기화(Kafka)에서 멱등 처리/재처리 설계? (D)
- > 답변: 메시지 Idempotency Key=(TopicPartition, Offset) + ObjectVersion 조합을 Sink 테이블에 기록, 적용 전 존재하면 Skip. Outbox → Kafka Publish 시 로컬 Tx 원자성 확보. 재처리: Consumer Lag 회복 시 Offset rewind, DLQ 재주입 시 동일 키 중복 가능 → Idempotent Upsert(조건부 Version 비교). 중복 Side-effect(이중 인덱싱) 방지를 위해 인덱스 테이블에 (ObjectID, Version) Unique Key. 상태 머신형 이벤트(JobStarted→JobCompleted) 는 허용 전이만 검증해 역행(Completed 후 Started) discard. 모니터링: Duplicate Skip Rate, DLQ Reprocess Success, Event Age p95.
- API Gateway / Edge 레이어에서 Auth, Rate Limit, QoS 적용 순서? (D)
- > 답변: (1) Very Early Connection Guard(TCP Handshake/TLS 완료 후 IP/Basic SYN flood 보호) (2) Authentication/Signature 검증(HMAC, Token 만료) (3) Authorization(정책/ACL) (4) Request Normalization(Canonicalization, Header 정리) (5) Rate Limit(Token Bucket / User, IP, Action tier) (6) QoS Routing/Weighted Fair Queue (7) Backend Dispatch. 이유: 비싼 정책 평가 이전에 위조/만료/과도 요청 차단→자원 절약. Rate Limit 전에 권한 평가를 두어 잘못된 Key로 리밋 소모 방지. QoS는 이미 인증된 정상 트래픽 중 우선순위(메타 vs 대형 body) 구분 적용. 지표: Early Drop %, Auth Fail %, Limit Hit %, Priority Queue Latency.
- Backpressure 처리 패턴(Channel, Queue, Token Bucket)? (D)
- > 답변: 계층별: 클라이언트(HTTP 429 + Retry-After), Ingress(Queue Depth 기반 Concurrency 동적 축소), 내부 워커(Channel 버퍼 한계 도달 시 생산자 Block), 토큰 버킷(전역 QPS / 바이트 레이트), Circuit Breaker(하류 p95 급증 시 임시 Fail-Fast). 선택 기준: Burst 흡수 필요→Queue, 평준화→Token Bucket, 즉시 신호→Channel Block, 실패 전 차단→Breaker. Implementation: 고루틴 풀 앞 bounded channel, 토큰 재충전 ticker, 지수 백오프 재시도. 메트릭: Dropped Tasks, Queue Wait p95, Refill Jitter, Backpressure Activation Duration.
- Object Lifecycle / Retention 정책 설계 시 이벤트 설계? (E)
- > 답변: 정책 상태 머신(Create→Active→PendingDeletion→Deleted) 과 타임 기반 Trigger(ExpirationTime) 이벤트를 분리. 스케줄러는 Min-Heap 혹은 Wheel에 정책/객체 만료를 enqueue, Worker는 Batch Tombstone 후 PolicyApplied / ObjectExpired 이벤트 발행. Audit/Compliance를 위해 Immutable Log(Append-only) 레코드를 남기고, Downstream(검색 인덱스, Billing)은 이벤트를 구독해 즉시 반영. 멱등: Tombstone 존재 시 재처리 skip. 지표: Queue Lag, Expired Objects/s, Policy SLA(만료 시각 대비 처리 지연). 확장성: 샤딩 기준=BucketHash, 재밸런싱은 파티션 reassignment + lag catch-up 후 전환.

## 9. 운영 / 배포 / 신뢰성

- 롤링 업데이트 vs 블루그린 vs 카나리 선택 기준? (B,D)
- > 답변: 기준 축: 위험 감내도, 트래픽 전환 속도, 인프라 비용, 롤백 시간. 롤링: 비용 최소, 부분 실패 감지 용이하지만 schema 불일치/상태 공유 위험 존재. 블루그린: 즉시 전환/즉시 롤백 장점, 2배 자원 비용. 카나리: 점진 전환 + 메트릭 게이팅으로 안전, 구현 복잡/전환 시간 길다. Object Storage 코어: 메타데이터 스키마 변경 포함 시 카나리(소수 AZ → 퍼센트 증분) + 메트릭(에러율, p95, GC, WAL fsync latency) 게이팅; 단순 stateless API는 롤링. 대규모 API Behavior 변경은 블루그린으로 일관 트래픽 비교 가능. 결정 프레임: (변경 영향 반경 * 실패 비용) / (추가 비용). 롤백 MTTR 목표 <5분이면 블루그린 혹은 빠른 카나리 게이트를 선호.
- 장애 사후분석(Postmortem) 템플릿 핵심 항목? (B)
- > 답변: (1) 요약(영향 범위: 시간, 사용자, 데이터) (2) 타임라인(탐지→완화→해결) (3) 근본 원인(RCA: 기술적 + 조직적) (4) 탐지/모니터링 갭 (5) 완화/복구 조치 (6) 재발 방지 액션(Owner/기한) (7) 남은 리스크/Follow-up (8) 첨부: 로그/그래프/다이어그램. 원칙: 비책임 추궁(blameless), 정량화된 영향(Minutes of SLO error budget consumed), 실행 가능한 액션만 포함.
- SLA / SLO / SLI 정의 예시(Object Storage) 제시? (D)
- > 답변: SLI: (1) Availability=(성공 요청 수)/(총 요청 수) (2) Latency=p95 PUT Init < X ms, p99 GET < Y ms (3) Durability=연간 데이터 손실 비율 < 10^-11 (4) Integrity=Checksum mismatch rate. SLO: Availability 99.9%, p99 GET 400ms, p95 PUT Init 150ms, Durability 11 9's. SLA: SLO 위반 누적 월 Z 시간 초과 시 크레딧. Error Budget=1 - SLO, 소진율 지표로 변경 속도 결정(예: 남은 예산 50% 이하→배포 Freeze). 내부 세분: Metadata API vs Data API 별도 SLO로 정확성과 대용량 경로 특성 반영.
- Observability (Logs / Metrics / Traces) 수집 파이프라인 간 상호 활용? (D)
- > 답변: Metrics는 지속 추세/알람(CPU, p99, QPS), Traces는 개별 느린 경로 세분화(Span breakdown), Logs는 비정형 예외/ rare event 근거. 상호 연결: TraceID를 로그 MDC 및 메트릭 라벨 샘플링에 삽입 → 특정 p99 spike 시 해당 구간 Trace 샘플 밀도 증가(Adaptive Tail Sampling) → Root Cause(Slow DB, IO Wait) 추출 후 관련 메트릭 대시보드 drill-down. 파이프라인: Agent(OpenTelemetry) → Collector(Batch + Tail Sampler) → Metrics TSDB / Trace Store(Tempo/Jaeger) / Log Store(Elastic). SLO 위반 알람은 자동으로 최근 5분 느린 Trace Bundle 링크를 Incident 티켓에 첨부. 회고 시 Noise 로그(중복 WARN)를 카디널리티/중복 비율로 분석하여 억제.
- Rate Limit, QoS, Traffic Shaping 적용 위치와 구현 패턴? (D)
- > 답변: Edge(L4/7 프록시)에서 1차 글로벌/공격성 한도(IP, ASN), API Gateway에서 사용자/버킷/Action 별 Token Bucket, 내부 서비스 Hop 간에는 Priority Queue + Leaky Bucket shaping. 구현: Redis Cluster + Lua(원자 버킷) 혹은 In-memory + BPF maps for ultra-fast path. QoS: 메타 경로 High, 대형 업로드 Low 공유 자원(CPU, Disk IO) 분리. Shaping은 eBPF tc 계층 혹은 Envoy Filter로 초과 패킷 pace. 지표: Throttled Requests, Queue Delay, Priority Starvation Rate.
- 장애 재현 환경 구성 전략(프로덕션 패리티)과 데이터 마스킹? (E)
- > 답변: 목표는 (1) 토폴로지 유사(AZ 수, 노드 수 축소 비율 유지) (2) 대표 워크로드 재생(실제 트래픽 샘플 리플레이) (3) 민감 데이터 노출 방지. 방법: 생산 로그에서 Sampling + PII 필드 토큰화(Hash + Format Preserving), Object Body는 사이즈 분포만 반영하는 synthetic generator. Infra: IaC(Terraform) + Chaos(네트워크 패킷 손실, 디스크 지연) 인젝션. 검증: 주요 SLI(p95, 오류율) 차이 허용 오차 정의(예 10%). 마스킹 영향: 해시 재현성으로 캐시 키/분포 유지. 재현 성공 판단: 원인 가설을 트리거하는 동일 지표 패턴(p99 상승 시점, 큐 길이) 확인.

## 10. 보안 / DRM 경험 전이

- DRM (Widevine / PlayReady / CoreTrust) 연동 경험이 저장 객체 암호화(KMS, Envelope)에 주는 시사점? (E)
- > 답변: DRM은 (콘텐츠 키 분리, 라이선스 서버 통한 짧은 수명 세션 키 발급, 복호화 경로 최소 노출) 세 축을 중시합니다. 이를 Object Storage 암호화에 적용하면 (1) Master Key는 KMS(HSM 보호)에서만 관리, (2) Object 별 Data Key(DEK)는 랜덤 생성 후 KMS로 Envelope(KEK 이용) 암호화 → 메타데이터에 암호문 저장, (3) GET 시 권한 검증 후 KMS에 DEK 복호화 요청(정책/사용자 스코프 포함), (4) 서버 측 스트리밍 중 chunk 단위 AEAD(GCM) 검증으로 조기 무결성 실패 차단. DRM의 단기 라이선스 개념을 Presigned URL / 세션 토큰 TTL 단축 및 Key Rotation 주기 설정에 활용: Hot 객체는 접근 패턴 기반 DEK 재사용 기간을 짧게(예 24h) 유지해 키 유출 위험을 낮춥니다. 측정 지표: KMS 호출 p95, 암호화 CPU cycles/MB, 키 로테이션 성공률.
- 전송 중 암호화(TLS) 외 저장 시 암호화(At-Rest) 키 회전 전략? (D)
- > 답변: 회전 모델 두 가지: (1) Lazy(신규 Write만 신규 KEK/DEK) (2) Active(백그라운드 Re-encrypt). 대규모 객체 전수 재암호화는 I/O 폭발 위험이 있으므로 우선 메타에 New Key Version 기록 후 Read 경로에서 구 Key 발견 시 on-the-fly 재암호화 + Write-back(Rewrite)로 점진 전환(Write-back Rate Limit). KEK 회전은 DEK 재암호화만 수행해 데이터 바디 재쓰기 회피. 실패 시 재시도 큐(Idempotent: ObjectID, OldKeyVer→NewKeyVer). 모니터링: Remaining Old Key Ratio, Re-encrypt Throughput, Failure Rate. 보안 vs 비용 Trade-off를 수치화(평균 잔존 OldKey 시간)를 내 SLO로 관리.
- Signed URL / Presigned URL 만료/권한 처리 설계? (D)
- > 답변: Presigned URL 구성 요소: Method, Path, Expiry, Canonical Query, Signature(HMAC(k, canonical string)). 설계 포인트: (1) 만료 짧게(분~시간) + 서버 검증 시 Expiry > now? 체크 (2) 권한 Scope(READ/WRITE)와 객체 Key Prefix 제한을 쿼리에 인코딩 후 서명 포함 (3) 재사용 방지: 단건 업로드 PUT은 Content-Length 혹은 PartNumber 포함해 다른 바디 재사용 차단 (4) Revocation: 긴 만료 필요 시 서버 측 Blocklist(Nonce or TokenID) 캐시 운영 (5) Clock Skew 허용 범위(±N분) 정의. 오탐/남용 지표: Expired Access Attempt Count, Blocklist Hit Rate, Invalid Signature Rate.

## 11. 품질 / 코드 / 리팩토링

- 레거시 재설계(브라우저 미디어 스택, 프록시 서버) 시 의사결정 기준? (B,D)
- > 답변: 평가 축: (1) 변경 비용(로직 복잡도, 테스트 부재) (2) 결함/장애 빈도 (3) 성능 병목 기여도 (4) 전략적 정렬(향후 기능과의 적합성) (5) Bus Factor. 기존 브라우저 미디어 경로에서 단계적 분리(Decoder → Buffer Queue → Renderer)로 리팩토링 한 경험을 객체 스토리지 메타 경로에도 적용: 모놀리식 Handler를 Auth, Validate, Meta Mutation, Event Publish 미들웨어 체인으로 쪼개 단위 테스트/측정 가능성을 확보. 의사결정 시 Value/Cost 점수화(예: (장애시간 절감*가치)+(성능 개선 잠재치) / 예상 공수) 후 상위 항목 순으로 스프린트 배치. 결과 측정: 변경 후 p99, 사이드 이펙트 버그, MTTR 감소.
- 성능 개선과 가독성/추상화 사이 트레이드오프 사례? (D)
- > 답변: 고 빈도 Hot Path에서 Generic Interface 추상화가 인라이닝 불가 + escape 증가로 p99 15% 악화. 해결: Hot Path 전용 구체 타입/함수 도입, 주변은 기존 추상화 유지한 하이브리드. 기준: (1) Hot Path CPU 비중>30% (2) 변경 빈도 낮음 (3) 프로파일 근거 확실. 이후 리팩터 후 코드 복잡도/중복 위험을 문서화하고 테스트 커버리지(>90%) 확보로 회귀 리스크 최소화. 최종 평가: 유지보수 난이도 증가 < 성능 이득(KPI)일 때만 적용.
- Secure Coding(MISRA, CERT) 경험을 Go 백엔드 품질 게이트로 옮기는 방안? (E)
- > 답변: 원칙 추출: 입력 검증, 명시적 에러 처리, 정의된 자원 수명. 구현: (1) Linter 확장(golangci-lint + custom rule: 금지 패턴 eval, unchecked error) (2) Threat Model 문서 템플릿 PR 체크리스트 포함 (3) Unsafe/reflect 사용 경로 별 별도 리뷰 Required Label (4) 에러 분류(Error Type taxonomy)로 로깅 수준 통일 (5) fuzz 테스트(Property-based) CI 통합. 측정: 미해결 High Severity 취약점 수, unchecked error 감소 추세, fuzz coverage.
- 테스트 전략(Unit, Integration, Property-based) 적용 우선순위? (D)
- > 답변: 피라미드: (1) Unit(순수 로직, fast) 최대 커버리지로 회귀 가드 (2) Integration(스토리지, 네트워크) 경로 happy + failure case 최소 세트 (3) Property-based: Key Invariant(정렬, idempotency, 해시 충돌 처리) 검증 (4) Load/Chaos: SLO 영향 요소. 초기 스프린트엔 실패 비용 큰 경로(API 인증, 메타 원자 Commit, 업로드 재시도 로직)부터 Unit/Integration 작성, 이후 성능 민감 LSM 컴포넌트에는 Property fuzz 적용. 메트릭: Test Runtime Budget, Flaky Rate, Mean Time to Detect Regression.

## 12. CS 기초 심화

- 메모리 계층(Locality, Cache Line) 이해가 Hash/Skiplist 최적화에 주는 영향? (D)
- > 답변: CPU Cache Line(보통 64B) 경계에 구조체를 맞추고 false sharing 필드를 패딩으로 분리하면 Lock 경합/캐시 invalidation 감소. Hash Table: Open Addressing + linear probing (혹은 quadratic) 으로 연속 메모리 스캔시 prefetch 도움 → Random Pointer chasing 줄여 p99 개선. Skiplist는 레벨 포인터 다수 → 캐시 미스 비용 높아 Hot Path read가 많은 경우 레벨 승격 확률(p) 조정(높은 노드 수 감소) + Arena 할당(연속 메모리)으로 spatial locality 확보. 측정: LLC miss %, cycles/op, branch misprediction. 튜닝 후 Tail latency ∆ 확인.
- 락 경합 줄이기 위한 Sharding / Lock-free 기법 적용 경험 또는 설계? (D)
- > 답변: Sharding: 키 해시 기반 N-way mutex 배열로 단일 글로벌 락을 대체 → contention ratio 하락. Lock-free: atomic CAS 기반 single-producer ring buffer로 로그 큐 처리 지연 감소. 선택 기준: (1) 경합률(락 대기 / 총 시간) >20% (2) 쓰기 비율 높음 (3) 구조 단순. ABA 문제는 generation counter 또는 hazard pointer 대체. 메트릭: Throughput vs CPU, 실패 CAS 재시도 횟수. 적용 결과 p99 enqueue latency 40% 감소.
- 멀티스레드 vs 이벤트루프 모델 선택 기준? (B,D)
- > 답변: 축: (1) 동시 IO 연결 수 (2) CPU 바운드 비율 (3) 지연 민감도 (4) 언어/런타임 지원. 수만 장기 연결 + 낮은 per-request CPU → 이벤트루프(netpoll + 비동기) 유리. 고 CPU 연산/멀티코어 활용 필요 → 멀티스레드(워크 stealing) 또는 hybrid(이벤트 루프 + worker pool). Object Storage 프론트: 네트워크/디스크 IO 혼합이므로 Go 런타임(goroutine 스케줄러) 모델 적합; 내부 암호화/압축 CPU Hot Path는 별도 풀(offload)로 분리. 선택 검증: 프로파일링으로 Run Queue Length, Syscall blocking, Context Switch rate.
- 네트워크 지연 구성 요소(TCP Handshake, TLS, DNS)와 최적화 아이디어? (B,D)
- > 답변: 구성: DNS Lookup, TCP 3-way, TLS Handshake(1-RTT/2-RTT), Request/Response 전송. 최적화: (1) Connection Reuse/Pooling (2) TLS 1.3 + 0-RTT 재연결 (재전송/Replay 위험 검증) (3) DNS Prefetch / 캐시 TTL 최적화 (4) Early Data 이전에는 idempotent 요청만 (5) BDP 기반 윈도우 / 초기 cwnd 확장 (6) QUIC 고려(암호화+전송 통합, 패킷 손실 영향 감소). 측정: handshake time p95, new connection ratio, retransmission rate.

## 13. 행동 / 협업 / 문화

- 신규 팀에서 온보딩 시 학습 로드맵을 스스로 구성한 방법? (B)
- > 답변: 30/60/90일 프레임: 0~30일(지도: 아키텍처 다이어그램, 주요 SLO, 장애 히스토리 읽기) → 31~60일(기여: 작은 버그/테스트 추가, 메트릭 개선 PR) → 61~90일(개선: 병목 하나 선정 개선 실험). 학습 소스 우선순위: 설계 문서 > 코드 > 지표/대시보드 > 과거 Postmortem. 매주 러닝 노트(Log)로 개념/질문/액션 정리, 멘토와 주간 검증. 성공 지표: 첫 PR merge 시점, 독립 배포 기여, 온보딩 문서 수정 제안 수.
- 타 부서/디자이너/플랫폼 팀과의 충돌 해결 경험과 원칙? (B,D)
- > 답변: 원칙: (1) 공통 목표(사용자 가치/ SLO) 재정렬 (2) 데이터 우선(정량 지표) (3) 의사결정 로그 투명화. 사례: 플랫폼 팀 API 변경 일정 vs 릴리즈 마감 충돌 → 영향 범위 정량화(추가 지연 시 에러 버짓 소진률 상승), 대안(Feature Flag 통한 점진적 적용) 제시, 합의된 타임라인 문서화. 지표: 조정 회의 수 축소, 결정 후 재논쟁 재발률.
- 성능 개선 과제를 우선순위화할 때 의사소통한 방식? (B)
- > 답변: ICE(Impact, Confidence, Effort) 또는 RICE(Reach 포함) 스코어링 표 사용, 각 후보 개선 예상치(예: p99 -20% → 재시도율 -X% → 비용 절감 Y) 모델링. 워킹 세션에서 가정/근거 문서 공유 후 합의, 스프린트 목표에 명시. 진행 중 매주 Burn-down + 실제 측정치 vs 추정치 차이 리뷰.
- 품질 문화(코드리뷰, 린트, 테스트) 정착을 위해 시도한 활동? (B,D)
- > 답변: (1) PR 템플릿: 변경 목적/리스크/테스트 증거/롤백 플랜 (2) 린트 실패 시 머지 금지 CI 게이트 (3) 주간 '성능/품질 하이라이트' 공유 슬랙 스레드 (4) 커버리지 레포트 + 신규/핫 경로 미달 시 알람 (5) 작은 실험: 리뷰 회전 시간 SLA(예 24h 이내) 설정. 지표: 평균 리뷰 시간, 커버리지 추세, 린트 실패 재발률, 회귀 버그 건수.
- 실패/이슈 사례 1~2개: 원인, 대응, 재발 방지? (B,D)
- > 답변: 사례: Multipart Upload 메타 Race → 중복 Commit 발생. 원인: Commit 전 상태 확인 Lock 누락. 대응: 재현 테스트 작성 → 원자 Compare-and-Swap + Idempotent Commit 레코드. 영향: 중복 ETag 노출 0.02%. 재발 방지: PR 체크리스트에 '경쟁 상태 여부' 항목 추가, Race Test(go run -race) CI. 지표: 관련 Class 오류 재발 0, Race Detector 경고 0 유지.

## 14. 프로젝트 별 구체 꼬리질문 (Portfolio Deep Dive)

### SOS (소규모 Object Storage)

- 아키텍처 다이어그램을 그린다면 핵심 컴포넌트와 데이터 흐름? (B)
- > 답변: 구성: (Client) → HTTP API → Auth/Validation → Metadata(Store: MongoDB 초기) → Chunk Manager(LevelDB 인덱스) → Blob Storage(Local FS). 흐름: PUT Init 시 메타 Intent(업로드 세션, 예상 크기, 사용자) 기록 → 클라이언트 Multipart(Part N) 업로드 → 각 Part는 임시 tmp/에 저장 후 해시/크기 검증 → 모든 Part 수신 후 Assemble + 최종 LevelDB에 Part Offset Manifest(객체ID→[offset,length,checksum]) 기록하고 MongoDB Document 상태 COMMITTED로 전환. 비동기 이벤트(Replication 모사)는 Commit Hook에서 Outbox에 Append (후속 처리: 로그 추적). 간단히 Intent/Assemble/Commit 3단계로 나눠 재시도 안전성과 중복 방지(idempotent commit)를 확보했습니다.
- Chunk 사이즈 선택 근거와 실험? (D)
- > 답변: 초기 분포 수집(샘플 파일 크기 P50=600KB, P95=4MB) 후 1MB / 4MB / 8MB 실험. 지표: (1) 업로드 실패 재시도당 재전송 바이트 (2) LevelDB 메타 엔트리 수 (3) Throughput(MB/s). 결과: 1MB는 메타 엔트리 과다, 8MB는 실패 시 낭비 과다; 4MB에서 메타 오버헤드와 재시도 비용 균형. 대형(>1GB)은 클라이언트 병렬 파트 수↑로 네트워크 BDP 활용, 소형(<256KB)은 Small Object Pack 경로(Inline 또는 pack 파일)로 분기해 tail latency 개선.
- MongoDB 인덱스 전략과 쿼리 패턴? (D)
- > 답변: CRUD 주 패턴: (Bucket, ObjectKey) 단건 조회, Prefix(List Objects), Version(최신 vs 이전) 조회. 인덱스: 1) {Bucket:1, ObjectKey:1} Unique (2) {Bucket:1, ObjectKey:1, Version:-1} 복합(최신 우선) (3) {Bucket:1, PrefixKey:1} Prefix 검색용(엔트리 생성 시 별도 저장된 Normalized PrefixKey). List 성능 문제(페이지당 많은 skip) 완화 위해 "검색 커서 + 마지막 ObjectKey" 기반 Seek After 방식 채택. 핫 키 집중 발생 시 PrefixKey 해시 도입하여 샤드 스캔 분산하는 실험 설계.
- LevelDB Compaction 튜닝 포인트? (D)
- > 답변: 작은 쓰기 다량 → L0 파일 증가 → Read Amp/Pause. 튜닝: (1) L0→L1 트리거 파일 수 낮춤(4→2) (2) write_buffer_size 조정으로 flush 빈도 제어 (3) bloom filter bits/key 10→12로 FP 감소 → 불필요 디스크 탐색 절감 (4) 동시 Compaction Thread 1→2로 백로그 감소. 모니터링: Compaction Pending, Stall Time, Write Amp, L0 File Count p95. 결과: Stall Time 40% 감소, p99 Read 18% 개선.
- 장애(데이터 손상/Partial Write) 대비 전략? (D,E)
- > 답변: Part 저장은 append-only temp 파일 + part meta(hash,size) Journal. Assemble 단계에서 모든 Part hash 재검증 및 최종 combined checksum 계산 후 원자 rename. Partial Write 탐지는 part meta missing 혹은 size mismatch로 Fail-Fast. 프로세스 crash 이후 restart 시 Journal replay → 미완료 세션 정리. 추가로 LevelDB manifest 스냅샷 주기 백업(손상 시 최근 스냅샷 + WAL 재적용). 손상 주입 테스트(고의 kill -9, 디스크 sync 지연)로 복구 MTTR 측정.

### skvs (분산 KV Storage)

- Raft 구현에서 로그 재전송 최적화? (D)
- > 답변: 느린 Follower Catch-up 시 전체 스냅샷 전송 대신 nextIndex / matchIndex 기반 누락 구간만 Segment 전송, 다수 연속 항목은 Batch AppendEntries. 재전송 실패 반복 시 지수 백오프 대신 최근 ack 지점 heuristic(마지막 성공 인덱스 ± 탐색폭)으로 binary search 감축. 압축된 스냅샷은 Chunk Streaming + 파이프라인 적용해 install 지연 감소, 전송 중 CRC 검증으로 조기 오류 감지.
- Snapshot 시점 결정 기준? (D)
- > 답변: (1) 로그 크기 임계(예: 256MB) (2) 마지막 스냅샷 이후 엔트리 수 (3) 가용 메모리(복구 시간 목표) (4) Low traffic 윈도우. Snapshot 비용 vs 복구 시간 함수로 최적 지점 탐색. Incremental snapshot(Dirty Key bitmap) 사용하여 full copy 비용 감소. 메트릭: Snapshot Duration, Install Time, Log Truncation Lag.
- LSM Tree 레이어 구분 및 SST 파일 관리 전략? (D)
- > 답변: MemTable → L0(무정렬 다수 파일) → L1..Ln(증가 용량). Hot Key 재압축 빈도 줄이기 위해 키 범위 히트율 기반 Partial Compaction(Hot Range 우선) 적용. SST 메타(Index, Bloom, Tombstone 비율) 수집 → compaction picker가 Tombstone aged ratio > threshold 시 GC 우선. Cold SST는 압축 강도↑(ZSTD), Hot SST는 LZ4로 Latency 절충. 지표: Read Amp, Write Amp, Compaction Bytes Moved, Bloom FP.
- Consistent Hashing과 노드 스케일 아웃 시 Rebalancing 절차? (D,E)
- > 답변: 절차: (1) 새 노드 가상 노드(토큰) 릴리즈 → 링 삽입 (2) 이동 대상 키 범위 계산 (3) Source 노드 백그라운드 스캔 + Key/Version 전송(스트리밍 + 체크섬) (4) Dual Read Phase: Old+New 둘 다 조회 후 결과 일치 검사 (5) Cutover 시 Old 범위 write redirect to new. Throttle: 재배치 중 QPS 기반 동적 전송 Rate. 검증: Missing Key Ratio, Divergence Count, Migration ETA 추적.

### myStream (실시간 방송 플랫폼)

- DDD 적용 시 도출한 주요 Aggregate와 경계? (D)
- > 답변: Aggregate: StreamSession(상태: INIT→LIVE→ENDED), MediaSegment(불변), TranscodeJob(상태 머신), ViewerStat(집계). 경계: 실시간 상호작용(채팅/상태) vs 미디어 배포 분리해 트래픽 급증이 미디어 처리 영향 최소화. StreamSession은 단일 쓰기 책임(상태 전이 검증), Segment는 Append-only 특성으로 확장. 이러한 경계 덕에 Object Storage 업로드 세션 모델로 전이 시 상태 머신 설계(유효 전이, Idempotent 종료)가 즉시 재활용.
- 이벤트 순서 보장 문제와 해결 패턴(Kafka Partition, Key 전략)? (D)
- > 답변: 순서 요구되는 엔티티(StreamSessionId, ObjectKey)는 Partition Key로 고정해 단일 파티션 내 Total Order 확보. 순서 혼합 이벤트(Transcode 완료 vs 라이프사이클)는 타입별 Topic 분리 + 소비자 사이 Timestamp/Version 비교 재정렬(Buffer with TTL). Out-of-order 허용 범위 정의(SLO: 재정렬 지연 < 3s) 후 초과 시 보정 이벤트(STATE REQUERY) 발행.
- 미디어 Transcoding 파이프라인 장애 복구 전략? (D)
- > 답변: 파이프라인: Ingest → Segmenter → Encoder Profiles → Packager → CDN Publish. 장애 유형: Encoder crash, Segment 손상. 복구: (1) 워크 큐 재배치(작업 원자성: Segment 단위) (2) 손상 Segment 검증 체크섬 실패 시 재인코딩 요청 (3) 상태 저장은 Outbox(DB) + Idempotent 작업 재수행. 재시도 지수 백오프 + Max Attempt, 실패 초과 시 Degraded Profile(낮은 비트레이트) fallback. SLA: 라이브 End-to-End 지연 Budget 내 유지(재인코딩 허용 횟수 제한).
- RTMP -> HLS 변환에서 Latency vs 화질 트레이드오프? (D)
- > 답변: Latency↓: 짧은 Segment(2s), Low GOP, 빠른 키프레임 → Encoding 효율 저하, Bitrate 증가. 화질↑: 긴 Segment(6s+), 높은 GOP → 초기 지연/재생 위치 공백 증가. 타협: 핵심 Profile Low-latency(short segment) + Background 고품질 Profile. 모니터링: Rebuffer Ratio, Start-up Latency, Bitrate Efficiency(bitrate/SSIM). 적용 후 Latency 30% 개선 with 5% bandwidth 증가 허용.

### 자율주행 센서 데이터 플랫폼

- 다양한 센서(카메라/LiDAR 등) 동시 처리 시 병목/동기화 전략? (D)
- > 답변: 센서별 Frame Rate/데이터량 상이(카메라 30FPS, LiDAR 10Hz). Timestamp 기반 버퍼(슬라이딩 윈도우)에서 근접 시간 범위(+/- tolerance) 매칭 후 Fuse. 병목: 디코딩/압축 해제 CPU, 디스크 순차 기록 대역폭. 최적화: Zero-copy mmap ingest, GPU 가속 디코딩, LiDAR 포인트 클라우드 압축(Octree) 비동기. 동기화 실패 시 Partial Set 처리 정책(후처리 보정 또는 placeholder). 지표: 매칭 성공률, Frame Processing Latency p95, Dropped Frame Ratio.
- Kafka 기반 데이터 파이프라인 Backpressure 처리? (D)
- > 답변: Producer: acks=all + linger.ms 튜닝(배치 효율 vs 지연), 배압 시 RecordAccumulator 메모리 임계 도달하면 전송 스레드 flush 우선. Consumer: poll loop latency 모니터링 + max.poll.interval.ms 임계 전 경고. Downstream 처리 느릴 때 Consumer Concurrency 증가 대신 처리 단계 Queue Depth 기반 Dynamic Throttle(일시 commit 지연). DLQ 분리로 Poison 데이터 격리. 지표: Lag Growth Rate, Rebalance Frequency, Commit Latency.

## 15. 예상 실무 시나리오 문제

- 신규 Region 확장 시 메타데이터 분산/복제 전략을 설계해 보세요. (Case)
- > 답변: 목표: 낮은 추가 지연 + 강한 메타 정합성 + 운영 단순성. 선택: Primary Region Leader(메타 Raft) + 다수 Follower + 신규 Region Read-Replica(비동기) 단계 도입 → Traffic 성장/지연 요구 충족 시 Local Write Shard 승격. 단계: (1) 글로벌 네임스페이스: ObjectID = `RegionPrefix`#`Key` (2) Bootstrap: 스냅샷 전송 + Incremental Log Apply (3) 지연 모니터링(p95 remote GET meta) SLO 초과 시 Partial Partition(버킷 subset) Ownership 이동, 이동 절차: Freeze Writes → Snapshot → Ship → Catch-up → DNS/Router Switch → Unfreeze. 다중 Region 간 Conflict 방지를 위해 Ownership Manifest(버킷→Leader Region) 강한 합의 관리. DR: Async Log Ship + RPO 분 단위, 재해 시 Promote Follower with Log Gap ≤ 목표. 지표: Cross-region RTT, Meta Apply Lag, Ownership Switch Mean Time.
- 1TB/day Small Object 폭증 상황: 인덱스/스토리지 비용 급증 대응 방안? (Case)
- > 답변: 병목: 메타 엔트리 수 증가→Bloom/Index 메모리·Compaction 폭발. 전략: (1) Small Object Pack(≤256KB) 묶음 + Sparse Index (2) Inline Threshold 확대(≤4KB 메타 inlining) (3) Cold Pack 압축(ZSTD) + Dedup(Chunk Hash Table) (4) Tiered GC: Hot Pack(메모리/SSD) → Aging → HDD 대형 파일 (5) Access 빈도 낮은 Object TTL/Lifecycle로 자동 만료. 실행 순서: 분포 측정→임계(P95 <=256KB 비율) 확인→POC(파일 수 vs 패킹 후 공간 절감)→롤아웃. 목표 지표: Metadata RAM/Logical Size Ratio, Write Amplification, Cost/GB-month. 위험: Pack 내부 일부 갱신 diff 비용↑ → Immutable Pack + Append New Pack + GC.
- 업로드 지연(P99 2s -> 5s 상승) 원인 추적 절차를 단계별로 설명. (Case)
- > 답변: 단계: (1) 메트릭 변동 교차 확인: p99 상승 시간대 QPS/에러/자원(CPU, IO) (2) 분해: Upload = Auth + Meta Intent + Part Stream + Commit Trace Span 분석(어느 구간 증가?) (3) 시스템 지표: run queue, io wait, network retransmission, GC pause 상관 (4) 샘플 요청 히스토그램 비교(평균 vs p99 변위) (5) 원인 분류 가설: a) 메타 DB 슬로우 쿼리 b) 네트워크 패킷 손실 c) Compaction Stall d) Rate Limit throttle (6) 각각 증거 수집: DB slow log, tcplife, compaction pending, limiter 로그 (7) 재현(소규모 트래픽 리플레이) (8) 해결: 예: L0 파일 폭증 → 임시 write throttle + emergency compaction, 이후 기본 파라미터 조정. 회고: 탐지 지표 부족 시 새 알람 추가(p95 Intent Latency).
- 다중 AZ 장애에서 RPO/RTO 목표 달성을 위한 설계? (Case)
- > 답변: 가정: RPO ≤ 5분, RTO ≤ 15분. 설계: (1) Metadata: Raft 5노드(3 AZ), 2 AZ 손실 대비 5노드→Quorum 유지 불가 → Async Shadow Cluster(원격 Region) 로그 복제 + 주기 Snapshot 전송 (2) Data: 객체 바디 3-way cross AZ + 원격 Region 비동기 Copy Queue (3) Failover Runbook: 장애 감지(Health quorum 실패) → Freeze Writes → Promote Shadow Metadata Cluster → DNS/Endpoint 전환 → Data Replica Lag 확인 후 점진 unfreeze. 테스트: GameDay 혼합 실패 주기적 주입. 지표: Snapshot Age, Replica Lag, Failover Drill Duration. 비용 vs 위험 Trade-off 문서화.
- Tape Tier 도입으로 비용 최적화를 추진할 때 정책/워크플로? (Case)
- > 답변: 대상: Access 빈도 낮고(>90일 미접근), Compliance 보존 필요 데이터. 워크플로: (1) Heat Score 계산(최근 Access, 사이즈) (2) Threshold 하락 시 Lifecycle Transition 이벤트 생성 (3) Tape Write Staging: HDD → Staging Buffer(집계/압축/암호화) → Batch Tape Append(대역폭 효율) (4) Catalog 메타에 Location=TAPE, Recall ETA 필드 추가 (5) GET 요청 시 Recall Job Queue → 사용자 SLA(예 6시간) 안내 + Partial Retrieve(헤더/메타 먼저) (6) 만료/삭제 시 Tape Catalog Tombstone + 비동기 Garbage Pass. 지표: Cost Saved(GB-month), Recall Success Latency, Staging Backlog. 리스크: Recall 폭주 → 우선순위 큐 + Rate Limit.

## 16. 지원서(request.md) 기반 추가 맞춤 질문

### 동기 / 커리어 방향

- 임베디드/클라이언트 개발 → 백엔드/스토리지로 전환할 때 가장 어려웠던 기술적 갭과 메운 방법은? (B)
- > 답변: 가장 큰 갭은 '하드웨어 자원 직접 제어'에서 '분산 환경 일관성/장애 모델'로 사고 전환. 임베디드에선 메모리/실시간성 세밀 튜닝 경험이 있었지만, 스토리지에서는 네트워크 파티션/복제 지연/확률적 장애 패턴 이해가 핵심. 메움 전략: (1) Raft/Paxos 논문+실습(소규모 구현) (2) AWS S3/ES 설계 사례 읽기 및 복제/정합성 패턴 정리 (3) 개인 프로젝트(skv s)로 합의·LSM·Consistent Hash end-to-end 체험 (4) 장애 주입(네트워크 지연, 크래시) 실험 통해 관찰→회고 문서화. 측정: 처음 3개월 내 주요 개념(Quorum, Read Index, Bloom, Compaction) 1~2문장 정의 가능 여부, 토이 구현 성능 지표(Throughput, p99) 스스로 최적화.
- Object Storage 직무가 본인 장기 목표(고성능/확장성/안정성 전문가) 달성에 어떻게 기여한다고 보는가? (B)
- > 답변: Object Storage는 대량 데이터 수명주기(수집→저장→정책→아카이브) 전 구간의 확장성 문제(Hot/Cold tier, Consistency trade-off), 고성능 IO 경로(멀티파트, zero-copy), 안정성(복제, 일관성 수렴)을 종합적으로 다룹니다. 즉 성능·확장·신뢰성 세 축 교차 최적화 경험이 축적되어 장기 목표 역량(시스템 병목 진단, 비용/효율 모델링, 장애 내성 설계)을 가장 빠르게 성장시킬 도메인입니다.
- "NCP Object Storage를 선택하게 만들 핵심 역할"을 수치화한다면 어떤 KPI를 정의하겠는가? (D)
- > 답변: (1) p99 Upload Init Latency < 150ms, (2) Metadata Availability 99.95%+, (3) Storage Cost/GB-month 연 단위 -X%(목표 10% 절감), (4) Lifecycle 자동화로 Cold Data 전환 비율 ≥ Y%(예 35%), (5) 운영 효율: 장애 탐지 MTTA < 2분, Postmortem Action 이행률 95%+. Vanity Metric 대신 사용자 영향/비용/안정성 직접 반영 지표로 선택.

### 학습 / 온보딩 전략

- 입사 후 90일 온보딩 플랜을 3단계(탐색/기여/개선)로 나누어 구체적 활동과 측정 지표를 제시해 보라. (D)
- > 답변: 0~30 탐색: 아키텍처, SLO, 주요 서비스 다이어그램 재작성, p99 지연 상위 3 경로 추출(지표). 31~60 기여: 작은 버그/테스트/린트 규칙 3건 이상 병합, 관측 공백(미수집 메트릭) 1개 추가. 61~90 개선: 선정된 병목 개선 실험(예: 메타 DB 캐시 Hit 5%p 상승), 문서화(Onboarding 보강 PR). KPI: 첫 PR Merge Day, 신규 메트릭 채택, p99 개선 ∆, 온보딩 문서 diff 라인 수.
- 기존 개인 프로젝트(sos, skvs, myStream) 중 어떤 부분을 바로 사내 코드베이스 개선에 적용할 수 있는가? (D)
- > 답변: sos: Multipart Intent/Assemble 패턴 → 재시도 안전성 강화. skvs: Raft Snapshot 증분/Compaction Hot Range 우선 정책 → 메타 저장소 성능 개선. myStream: 이벤트 기반 Outbox + Segment 상태 머신 → 비동기 라이프사이클 처리 안정화. 우선순위: (1) 고빈도 메타 경로(캐시/Compaction) (2) 재시도 패턴 향상 (3) 비동기 이벤트 순서/멱등 강화.
- 레거시 파악 시 코드/러닝 노트/지표 중 어떤 순서로 Mapping을 진행할 것인가? (D)
- > 답변: (1) 지표로 현재 병목/이상 패턴 인지(p99, 오류율) (2) 해당 경로 코드 리딩(핵심 Handler→Storage Layer) (3) 설계 문서/과거 Postmortem 크로스 체크(의도 vs 구현 편차) (4) 러닝 노트로 개념/의문 기록 (5) 주간 리뷰로 갭 해소. 순서를 지표 우선으로 둬 조기 최적화 함정 회피.

### 데이터/AI 트렌드 연계

- AI 워크로드(대규모 모델 학습/추론) 특성을 고려한 Object Storage 기능 개선 아이디어 2~3가지? (E)
- > 답변: (1) Manifest 기반 Bulk Prefetch API: 학습 Job이 Epoch 시작 전 필요 샤드 목록 제출 → 병렬 staging(SSD 캐시)에 pre-warm (2) Sparse Range Fetch(텐서 체크포인트 일부 레이어만 읽기) 최적화: 다중 Range 병합 + 서버 측 압축 블록 색인 (3) Training Artifact TTL/Lifecycle 태그 자동 추천(접근 패턴 ML)로 비용 절감. 지표: Cache Warm Hit%, Prefetch Lead Time, 학습 시작 지연 감소.
- Vector DB / 분산 캐시 / Tiered Storage 조합 설계 시 Object Storage가 제공해야 할 최소 기능은? (D)
- > 답변: (1) Range / Batch Get API (2) Object Tag 기반 Tier Hint (Hot/Warm/Cold) (3) Consistent ETag/Version for cache invalidation (4) Pre-signed Bulk Manifest (5) Event Stream(ObjectCreated/Expired) for index sync. 필수 비기능: p99 Get 안정성, Metadata Strong Read-after-Write.
- 데이터 로컬리티 최적화를 위한 Pre-Fetch / Hint API를 설계한다면 인터페이스 초안을 말해보라. (E)
- > 답변: POST /prefetch {manifest:[{key,offsets:[{start,end}] , priority, deadline}], cache_tier:"ssd"}. 응답: job_id, accepted_count, eta. GET /prefetch/{job_id} 상태(pending, warming, ready). 정책: deadline 기반 스케줄 우선순위, 중복 요청 coalescing(job merge). 모니터링: Warm Success %, Avg Warm Latency, Eviction Before Use Ratio.

### 성과지표 / 측정

- 서비스 성능 병목 개선 과제 선정 시 사용하고 싶은 우선순위 평가 매트릭스 예시? (D)
- > 답변: Score 계산식은 `Score = (Impact*Reach*Confidence)/Effort`. Impact는 p99 감소→Retry 감소→비용 절감 금액 환산, Reach는 영향 사용자/요청 비중, Confidence는 과거 측정/프로파일 근거, Effort는 이상/최악 범위 추정. 상위 N만 착수, 나머지 대기열. 주기적 재평가(새 지표 반영). 시각화: Bubble Chart(Impact vs Effort, 크기=Reach, 색=Confidence).
- 안정성 관련 SLO를 정의할 때 Read/Write/Metadata API를 구분해야 하는 이유와 지표 후보? (D)
- > 답변: 워크로드 특성(크기/빈도/지연 민감도) 상이 → 단일 SLO는 특정 경로 리스크 은닉. 분리로 정밀한 에러 버짓 정책(Write 느려질 때 배포 중단 vs Read는 허용 범위) 가능. 지표: Metadata p95 Latency, Write p99 Throughput 안정성(±X% 범위), GET Error Rate, Commit Success Rate, Durability(Checksum mismatch/Lost object), Availability.
- 사용자 경험 개선을 위한 Non-Functional KPI(에러 재시도율, P95 Upload Init Latency 등) 선정 근거? (D)
- > 답변: 직접 기능 결과 아닌 체감 품질 지표가 재시도 폭발→비용 상승 선행 신호로 작동. 선택 기준: (1) 사용자 행동 변화와 상관계수 (2) 조기 경보(문제 확대 전 탐지) (3) 튜닝 액션 명확. 예: Retry Rate↑ → Auth 지연/네트워크 손실 조사, P95 Init 상승 → 메타 경로 병목. KPI 선정 후 지표→가설→실험 루프(EDA Dash→Hypothesis→Change→Result) 문서화.

### 조직/협업 / 문화 기여

- 기술 공유 문화 활성화를 위해 첫 6개월 내 실행할 작은 실험 2개? (B)
- > 답변: (1) 주간 10분 '스토리지 메트릭 리딩' 세션(한 지표 선정 구조/변동 설명) (2) 분기별 Mini Postmortem 리뷰(과거 1건 선정하여 개선 학습 공유). 측정: 참석률, 신규 문서 PR, 제안 채택 수.
- 갈등 조정 사례(일정/성능) 학습을 스토리지 장애 RCA에도 적용한다면 어떤 프로세스가 되는가? (D)
- > 답변: 기존 갈등 해결 프레임(공통 목표 재정립, 사실/데이터 우선, 결정 로그)을 Incident RCA에 그대로 이식: 1) 영향 범위 정량 2) 사실 타임라인(Sources 명시) 3) 가설별 증거 체크리스트 4) 결정/액션 로그 5) 후속 추적 오너/기한. 측정: RCA 완료까지 리드타임, Action 완료율.
- 객관적 검증 문화를 팀에 스며들게 하기 위한 PR 템플릿 혹은 Check-list 초안? (D)
- > 답변: 섹션: Purpose, Risk(성능/안정성), Metrics Impact(지표 전/후 예상), Test Evidence(스크린샷/벤치), Rollback Plan, Security/Privacy 고려, Observability(추가 메트릭/로그), Reviewer Guide(집중 지점). Merge 전 자동 체크: 커버리지, 린트, 성능 회귀 벤치(threshold). 효과 지표: 미인가 변경 회귀율↓, 리뷰 리드타임↓.

### 문제 해결 / 사례 확장

- DRM 파이프라인 POC 일정 재조정 경험을 Object Storage 대규모 마이그레이션 프로젝트에 적용하는 시나리오? (E)
- > 답변: DRM 일정 지연 원인(암호화 라이브러리 성능 미측정) → 사전 벤치마크 게이트 추가 교훈을 마이그레이션에 적용: 마이그레이션 단계별(스냅샷 추출, 검증, 증분 동기화, 컷오버) 각단계 예행연습(Load+시간 측정) 후 Go/No-Go 체크. Scope Freeze, 변경 요청 Change Board 운영. Risk Register 유지(성능, 데이터 손상, 시간 초과) + 확률/영향 점수화. 지표: 단계별 예측 vs 실제 소요 시간 편차.
- 동적 파이프라인 제안→데이터 기반 반박 경험을 통해 얻은 '가설-실험-결론' 루프를 스토리지 성능 튜닝에 적용한 예를 들어보라(가정 가능). (E)
- > 답변: 가설: LSM Bloom FP율 1%→0.5%로 낮추면 디스크 read 8% 감소→p99 5% 개선. 실험: Stage 환경에서 bits/key 조정 후 fio+워크로드 재생, 측정: read ops, cpu/hash overhead, p95/p99. 결과: read ops 6% 감소, CPU 2%p 증가, p99 3% 개선(목표 미달) → 결론: Bloom 대신 Hot Range Compaction 우선순위 조정이 ROI 높음. 루프 문서화로 향후 유사 제안 근거 강화.
- 리소스 제약이 있는 상황에서 어떤 개선 작업을 먼저 Drop하거나 Defer할지 판단 기준? (D)
- > 답변: 프레임: `Priority = (Impact/Cost)*Urgency*Reversibility/StrategicAlignment`. Drop: Impact 낮고 Reversibility 높으며 긴 Effort. Defer: Impact 크지만 긴 준비(데이터 수집 필요). 즉각 수행: Impact 중~높고 Reversibility 높고 긴급(에러 버짓 소진 임박). 주간 포트폴리오 리뷰로 재평가.

### 장기 성장 / 리더십

- 기술 리더로 성장하기 위해 향후 2년간 분기별 학습/실천 로드맵을 카테고리(스토리지 심화, 분산합의, Observability, 운영 자동화)로 나누어 개략 제시. (E)
- > 답변: Q1~Q2: LSM 내부 소스 리딩/컴팩션 실험, Raft 로그 압축 최적화 POC, eBPF 트레이싱 도입. Q3~Q4: 멀티 Region 메타 설계 리뷰 참여, OpenTelemetry 샘플링 정책 개선, 운영 스크립트 → 오케스트레이션 전환. Q5~Q6: 크로스리전 복제 설계 주도, SLO 에러 버짓 정책 고도화, 자동화(릴리즈/롤백) 구현, 멘토링 1~2명. Q7~Q8: 대규모 스토리지 비용 최적화 이니셔티브 리드, 사내 디자인 가이드 정립, 커뮤니티 발표/문서화. 측정: 주도 디자인 문서 수, 성능/비용 개선 누적 %, 멘토링 피드백.
- '확장성·안정성·고성능' 세 축이 충돌할 때 본인의 의사결정 프레임워크? (D)
- > 답변: 1) 사용자 영향(가용성>정확성>성능 순) 안전망 확보 2) 성능 이득 vs 비용/복잡도 곡선(한계 체감 구간) 3) 실패 모드(장애시 축 영향 확산?) 4) 가역성(롤백 쉬운가) 5) 측정 가능성(빠른 피드백 루프) 평가. 의사결정 로그에 대안/가정/리스크 기록, 30일 후 성과 리뷰로 가정 검증.
- Bus Factor 감소와 지식 확산을 동시에 달성하기 위한 문서화/리뷰 전략? (D)
- > 답변: (1) Design Doc 표준 템플릿(배경/문제/대안/결정/리스크) (2) 코드 변경 전 RFC 리뷰(비동기 코멘트→미팅 최소화) (3) 핵심 모듈 On-call 교차 로테이션 → 실전 지식 이전 (4) 주간 Tech Sync에서 신규 컴포넌트 5분 Flash Talk (5) 문서 품질 KPI(최근 90일 조회/갱신 비율) 추적. Bus Factor 지표: 핵심 기능 수정자 상위 1인 비중 감소.

### 추가 기술 심화 (AI 시대 맥락)

- Tiered Storage + Object Tagging + Lifecycle Policy를 AI 데이터 파이프라인에 최적화하려면 어떤 메타데이터가 추가로 필요? (E)
- > 답변: (1) Access Pattern Stats(last N read timestamps, sequentiality score) (2) Training Epoch Count(참조 빈도) (3) Data Quality Tag(검증 통과 여부) (4) Cost Class(예산 그룹) (5) Prefetch Priority. 이를 태그/확장 메타로 저장 → Lifecycle Engine이 Rule(Access 최근성+Epoch 미사용+Cost Class) 결합해 Tier 전환 결정, ML 추천 모델 학습 데이터로 사용.
- 대규모 학습 Job이 반복적으로 동일 데이터 세트를 읽는 상황에서 Read Amplification 감소 전략 2가지? (D)
- > 답변: (1) Epoch Prefetch & Local SSD Warm Cache(Manifest 기반) + Byte-range Coalescing (2) Columnar/Shard 재구성: 빈번히 사용되는 Feature Subset 분리 저장 → 불필요 데이터 스캔 감소. 보조: 체크포인트 Delta 저장, 압축 블록 색인 캐시. 측정: Remote Bytes Read vs Logical Bytes, Cache Hit%, Epoch Start Latency.
- Server-Side Encryption + Client-Side Caching 조합 시 키 관리 및 무결성 검증 흐름을 단순 다이어그램으로 설명(구두). (E)
- > 답변: 흐름: Client GET → Cache Miss → Signed Request → Server: (Auth→Meta Lookup→DEK(KMS) 복호화→Ciphertext 읽기→AEAD Decrypt & Chunk Checksum) → Plaintext → Client Cache(Store with ETag/Version + MAC) → 재요청: If-None-Match로 재검증. Key Rotation: Meta Key Version 변경 시 Cache Entry ETag mismatch 발생 → 재검증 후 새 암호문 경로. 무결성: 서버 AEAD + 클라이언트 캐시 MAC 이중 방어.

## 18. 역질문(면접관에게 할 질문 예시)

- Object Storage 현재 아키텍처와 향후 12개월 로드맵(Scale/Feature)은?
- 성능/안정성 관련 가장 큰 Pain Point과 최근 해결 시도는?
- 온보딩 트레이닝 커리큘럼과 기술 부채 처리 방식?
- 운영 중 대표적 장애 패턴과 사후분석(Postmortem) 문화는?
- 신규 저장 매체(HDD/SSD/NVMe/Tape) 도입 의사결정 프로세스?
- 인프라/플랫폼/서비스 팀 간 협업 구조와 의사결정 단계?
- 팀에서 최근 가장 큰 기술 부채는 무엇이며 제거 로드맵이 있는가? (역질문 파생)
- 성능 개선 성공을 어떻게 Celebrating / Knowledge Base화 하는가? (역질문 파생)
- 신규 기능 출시 전 실험(Experiment) 또는 Canary 절차는 표준화되어 있는가? (역질문 파생)

---

### 준비 가이드 요약

- 각 Deep Dive 프로젝트별 STAR + 아키텍처 다이어그램 준비
- 핵심 수치(성능 개선 %, Latency, Throughput, 자원 사용) 메모라이즈
- 실패/장애 2~3개 케이스 재현 로그/지표 흐름 정리
- 스토리지 핵심 개념(Consistent Hashing, LSM, Bloom, Raft, Lifecycle, Multipart) 정의 1~2문장 암기
- Go/C++ 병행 경험 차별화 포인트(메모리, 동시성, 디버깅) 정리

> 필요 시 카테고리 확장이나 모의 답변 구조 템플릿도 추가 가능합니다.

---

참고: 현재 번호 체계는 16 다음 18(역질문)로 이어지며 17번 섹션은 정의되어 있지 않습니다. 추후 17번을 추가한다면 예: "Observability 고급 / 비용 최적화" 혹은 "실시간 분석/AI 통합" 같은 주제를 배치할 수 있습니다.
