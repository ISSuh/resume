# 예상 질문 리스트

### 기술적 내용

전체 시스템에서 성능병목을 어떻게 분석할 것인가?

전체 시스템의 성능분석을 어떤 과정으로 진행할 것인가?

Event-Driven Architecture

MSA

Object Storage 시스템 디자인

대규모 요청에 대한 아키텍처 설계

Application 레벨의 traffic shaping, QOS

OSI 7 layer, TCP/IP 지식

리눅스 파일 시스템 관련 내용

운영체제/network 관련 내용

kafka 데이터 스트리밍

GoLang 에 대한 내용 - groutine, GC



### 경력상 내용

본인이 주도적으로 진행했던 프로젝트는 무엇인지

가장 자신있는 프로젝트에 대한 설명 및 관련된 딥한 기술질문

프로젝트 실패경험

프로젝트 성공 경험

가장 어려웠던 트러블 슈팅

가장 자신있는 개발 영역

대규모 시스템 배포



### 서비스적 내용

타 클라우드와 비교했을때, 네이버 클라우드의 장점은 무엇인지



### 기타 내용

최근 스터디 어떤걸로?

요즘 공부하고 있는 기술이 있는가?



### 이력서, 개인프로트 기반 내용

skip-list

WAL

#### LSM-tree 
- LSM-tree 구현시 도전적이면서 어려웠던점
  - 
- Write Amplification
  - Write Amplification은 실제로 디스크에 쓰여지는 데이터 양이 애플리케이션에서 요청한 쓰기 데이터 양보다 몇 배 더 많아지는 현상을 말합니다.
  - LSM-tree는 데이터를 여러 레벨(L0, L1, L2...)에 저장하는데, 각 레벨간 병합(compaction) 과정에서 동일한 데이터가 여러 번 쓰여지게 됩니다.
  - 예를 들어:
    - 사용자 쓰기: 1KB
    - 실제 디스크 쓰기 과정:
    - 1. MemTable → SSTable(L0): 1KB 쓰기
    - 2. L0 → L1 compaction: 1KB + 기존 데이터 10KB = 11KB 쓰기  
    - 3. L1 → L2 compaction: 11KB + 기존 데이터 100KB = 111KB 쓰기
    - Write Amplification = 122KB / 1KB = 122배
  - 최적화 전략
    - Compaction 전략 개선
      - Leveled Compaction → Tiered Compaction 전환
      - Size-tiered 방식으로 같은 크기 SSTable끼리만 병합
    - Write Buffer 크기 조정
      - MemTable 크기 증가 → L0 SSTable 수 감소
      - 배치 처리로 compaction 빈도 줄이기
    - Bloom Filter 최적화
      - False Positive Rate 감소 → 불필요한 읽기/쓰기 방지

Key/Value Storage

ObjectStorage

RAFT,

Consistency hasing

왜 이러한 기술을 사용했는지????
