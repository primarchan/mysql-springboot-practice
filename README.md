# 대용량 처리를 위한 MySQL 이론 및 실습

## SNS 모델링으로 배우는 정규화 / 비정규화

### 실무에서의 정규화 / 비정규화에 대한 고민들
- 실무에서는 어떤 고민들을 할까?
  - 중복된 데이터면 반드시 정규화를 해야하는 걸까?
    - 사실 실무에서 중복 데이터면 기계적으로 정규화하는 분들이 종종 있다.
    - <b>정규화도 비용이다.</b> 읽기 비용을 지불하고 쓰기 비용을 줄이는 것
  - 정규화 시 고려해야 하는 것
    - 얼마나 빠르게 데이터의 최신성을 보장해야 하는가?
    - 히스토리성 데이터는 오히려 정규화를 하지 않아야 한다.
    - 데이터 변경 주기와 조회 주기는 어떻게 되는가?
    - 객체(테이블) 탐색 깊이가 얼마나 깊은가?
  - 정규화를 하기로 했다면 읽기 시 데이터를 어떻게 가져올 것인가?
    - 테이블 조인을 많이 활용하는데, 이건 사실 고민해볼 문제다.
    - 테이블 조인은 서로 다른 테이블의 결합도를 엄청나게 높인다.
    - 조회시에는 성능이 좋은 별도 데이터베이스나 캐싱 등 다양한 최적화 기법을 이용할 수 있다.
    - 조인을 상요하게 되면, 이런 기법들을 사용하는데 제한이 있거나 더 많은 리소스가 들 수 있다.
    - 읽기 퀄리 한번 더 발생되는 것은 그렇게 큰 부담이 아닐 수도 있다.

<hr>

## 조회 최적화를 위한 인덱스 이해하기

### 데이터베이스 성능의 핵심
- 메모리 / 디스크 비교

|  구분   | 메모리              | 디스크      |
|:-----:|:-----------------|:---------|
|  속도   | 빠름               | 느림       |
|  영속성  | 전원이 공급되지 않으면 휘발  | 영속성이 있음  |
|  가격   | 비쌈               | 저렴함      |

- 데이터베이스의 데이터는 결국 디스크에 저장된다.
  - 결국 데이터베이스 성능의 핵심은 디스크 I/O(접근)을 최소화하는 것
  - 따라서 메모리에 올라온 데이터로 최대한 요청을 처리하는 것 (메모리 캐시 히트율을 높이는 것)
  - 심지어 쓰기도 곧바로 디스크에 저장하지 않고 메모리에 우선 저장한다.
- 메모리에 저장된 데이터 유실을 고려해 WAL(Write Ahead Log)를 사용
  - 대부분의 트랜잭션은 무작위하게 Write 가 발생
  - 이를 지연시켜 랜덤 I/O 횟수를 줄이는 대신 순차적 I/O 를 발생시켜 정합성 유지
  - [참고자료](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
- 이를 종합했을 때, 결국 데이터베이스 성능의 핵심은 <b>디스크의 랜덤 I/O를 취소화</b> 하는 것이다.

### 인덱스의 기본동작
- 인덱스는 정렬된 자료구조, 이를 통해 탐색범위를 최소화
- 인덱스도 테이블, 인덱스의 핵심은 탐색(검색) 범위를 최소화하는 것

### 인덱스의 자료구조
- 검색이 빠른 자료구조
  - `HashMap`, `List`, `Tree`, `Binary Search Tree`...
- `HashMap`
  - 단 건 검색 속도 O(1)
  - but, 범위 탐색은 O(N)
  - 전방 일치 탐색 불가
    - `ex) like 'AB%'`
- `List`
  - 정렬되지 않은 리스트의 탐색은 O(N)
  - 정렬된 리스트의 탐색은 O(logN)
  - 정렬되지 않은 리스트의 정렬 시간 복잡도는 O(N) ~ O(N*logN)
  - 삽입 / 삭제 비용이 매우 높음
- `Tree`
  - 트리 높이에 따라 시간 복잡도가 결정됨
  - 트리의 높이를 초소화하는 것이 중요!
  - 한쪽으로 노드가 치우치지 않도록 균형을 잡아주는 트리 사용
    `ex) Red-Black Tree, B+ Tree`
- `B+ Tree`
  - 삽입 /삭제 시 항상 균현을 이룸
  - 하나의 노드가 여러 개의 자식 노드를 가질 수 있음
  - 리프 노드에만 데이터 존재
    - 연속적인 데이터 접근 시 유리
  - [B+ Tree 실습 링크](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)

### 클러스터 인덱스
- 클러스터 인덱스는 데이터 위치를 결정하는 키 값이다.
  - 클러스터 키 순서에 따라서 데이터 저장 위치가 변경된다. (클러스터 키 삽입/갱신 시에 성능 이슈 발생)
- MySQL 의 PK 는 클러스터 인덱스다.
  - PK 로 Auto Increment vs UUID
- MySQL 에서 PK 를 제외한 모든 인덱스는 PK 를 가지고 있다.
  - PK 의 사이즈가 인덱스의 사이즈를 결정
- 클러스터 인덱스의 장점
  - PK 를 활용한 검색이 빠름 (특히 범위 검색)
  - Secondary Index 들이 PK 를 가지고 있어 커버링에 유리

### SNS 어플리케이션 API 모델링 및 구현을 통한 인덱스 실습
- 요구사항
  - 사용자가 작성한 글 캘린더 구현
    - 작성일자와 일자별 회원의 작성한 게시물 갯수를 반환한다.
  - 데이터의 분포도에 따라 인덱스에 의한 성능 향상 정도가 달라진다. (데이터 분포도를 고려하지 않은 인덱스는 오히려 성능 저하를 발생)
  - 실행 계획이 포함된 쿼리 예시
```sql
explain SELECT createdDate, memberId, count(id)
FROM POST use index (POST__index_member_id_created_date)
WHERE memberId = 1 AND createdDate BETWEEN '1900-01-01' AND '2024-01-01'
GROUP BY memberId, createdDate;
```
- 인덱스를 다룰 때 주의해야할 점
  - 인덱스 필드 가공
    - 인덱스 필드의 값 or 타입을 변경할 경우, 인덱스를 사용할 수 없음
  - 복합 인덱스
    - 두 개 이상의 컬럼으로 구성된 인덱스
    - 복합 인덱스 생성 시 선두 컬럼 선정이 굉장히 중요
  - 하나의 쿼리에는 하나의 인덱스만 탐
    - 여러 인덱스 테이블을 동시에 탐색하지 않음
    - WHERE, ORDER BY, GROUP BY 혼합해서 사용할 때에는 인덱스를 잘 고려
  - 의도대로 인덱스가 동작하지 않을 수 있음 -> explain 으로 확인
  - 인덱스도 비용이다. 쓰기를 희생하고 조회 성능 향상을 얻는 것
  - 꼭 인덱스로만 해결할 수 있는 문제인가에 대해서 고민
  - 데이터의 식별 정도가 높은 컬럼으로 인덱스를 지정

<hr>

## 페이지네이션 최적화

### 페이지네이션이란
- 많은 양의 데이터를 어떻게 노출시킬 것인가
- 다음 페이지 vs 스크롤
- 마지막 페이지를 구하기 위해 전체 갯수를 알아야 함

### 오프셋 기반 페이징 구현 실습
- 대용량 데이터의 경우 offset 값이 커질 수록 성능이 저하됨 -> 커서 기반 페이징이 대안으로 사용됨 
```sql
SELECT *
FROM POST
WHERE memberId = 1
ORDER BY createdDate desc
LIMIT 2
OFFSET 0;
```
- 커서 기반 페이징
  - 키를 기준으로 데이터 탐색범위를 최소화
  - 전체 데이터를 조회하지 않기 때문에 게시판 형태(다음 페이지)의 UI 구현 난이도가 높음

### 커버링 인덱스
- 검색조건이 인덱스에 부합하다면, 테이블에 바로 접근 하는 것보다 인덱스를 통해 접근하는 것이 매우 빠르다.
- 그렇다면 테이블에 접근하지 않고 인덱스로만 데이터 응답을 내려줄 수는 없을까? -> 커버링 인덱스
- MySQL 에서는 PK가 클러스터 인덱스이기 때문에 커버링 인덱스에 유리
- 커버링 인덱스로 페이지네이션을 최적화를 어떻게 할 수 있을까? 
  - order by, offset, limit 절로 인한 불필요한 데이터블록 접근을 커버링 인덱스를 통해 최소화

<hr>

## 타임라인 최적화

### 타임라인이란?
- SNS (트위터, 페이스북, 인스타그램 등) 에서 팔로워들의 게시물들을 보여주는 피드
- 회원 ID 를 받아, 해당 회원의 팔로워들의 게시물을 시간순으로 조회

### 타임라인 구현
- 커서 기반 페이징 방식으로 구현
- 타임라인 구현 Flow
  1. 회원의 팔로 목록 조회
  2. 1번의 팔로우 회원 id 로 게시물 조회

### 타임라인 최적화 - 서비스가 커질 수록 느려지는 타임라인
- Follow 테이블 예시

|  id   |  from  |  to   |
|:-----:|:------:|:-----:|
|   1   |   1    |   3   |
|   2   |   1    |   2   |
|   3   |   3    |   1   |
|   4   |   1    |   4   |
|   5   |   2    |   5   |

- Post 테이블 예시

|  id   | memberId |
|:-----:|:--------:|
|   1   |    3     |
|   2   |    1     |
|   3   |    2     |
|   4   |    3     |
|   5   |    2     |

- 시간복잡도
  - `log(Follow 전체 레코드) + 해당 회원의 Following * log(Post 전체 레코드)`
  - Fan Out On Read (Pull Model) -> 사용자가 매번 홈에 접속할 때마다 부하가 발생
- 서비스가 커질 수록 성능이 저하되는 이슈 발생

### 타임라인 최적화 - Fan On Write 방식의 타임라인 (Fan out 타임라인 이론)
- Fan Out On Read (Pull Model)
- Fan Out On Write (Push Model)
  - 게시물 작성 시, 해당 회원을 팔로우하는 회원들에게 데이터를 배달한다. (게시물 작성 시 Timeline 테이블에 적재)
  - 즉, 타임라인 조회 시에는 Timeline 테이블을 조회하여 게시물들을 조회
  - Pull Model 에서의 조회 시점의 부하를 쓰기 시점 부하로 치환

### 타임라인 최적화 - 정합성과 성능의 트레이드 오프
- 정합성과 성능
  - Push Model 은 공간복잡도를 희생, Pull Model 은 시간 복잡도를 희생
- Push Model vs Pull Model 중 어떤 것이 정합성을 보장하기 쉬울까?
  - Pull Model 은 원본 데이터를 직접 참조하므로, 정합성 보장에 유리
  - 하지만, Follow 가 많은 회원일 수록 처리속도가 느리다.
  - [Facebook 관련 링크](https://www.facebook.com/help/211926158839933?helpref=related_articles)
  - [Twitter 관련 링크](https://help.twitter.com/ko/using-twitter/twitter-follow-limit)
  - Push Model 에서는 게시물 작성과 타임라인 배달의 정합성 보장에 대한 고민이 필요하다.
- 모든 회원의 타임라인에 배달되기 전까지 게시물 작성의 트랜잭션을 유지하는 것이 맞을까?
  - CAP 이론
  - Push Model 은 Pull Model 에 비해 시스템 복잡도가 높다.
  - 하지만 그만큼 비지니스, 기술 측면에서 유연성을 확보시켜준다.
  - 결국 은총알은 없다. 상황, 자원 등 여러가지를 고려해 트레이드 오프 해야한다.

<hr>

## 데이터 정합성 보장을 위한 트랜잭션

### 01. 트랜잭션이 없는 세상은
- 처리 중인 데이터를 다른 곳에서 조회하게 되면 문제가 발생
- 여러 SQL 문을 마치 하나의 오퍼레이션으로 묶을 수 있어야 한다!
- 트랜잭션은 기본적으로 DB 커넥션 풀을 점유하고 있기 때문에 트랜잭션 범위는 최대한 짧게 가져가는 것이 성능적으로 유리
- Spring 에서 지원하는 선언적 트랜잭션 관리 (@Transactional) 의 경우, Proxy 로 동작하기 때문에 inner 함수에서는 정상적으로 동작하지 않음에 주의

### 02. 트랜잭션 ACID
- [ACID 의 사전적 정의](https://ko.wikipedia.org/wiki/ACID)
- ATOMICITY
  - 원자적 연산을 보장해야 한다. (All or Nothing)
  - MySQL 에서는 MVCC 를 통해 지원 (이전 데이터를 Undo Log 에 보관하여 버전 관리)
  - 트랜잭션이 Atomicity 한 단위가 된다.
- CONSISTENCY
  - 트랜잭션이 종료되었을 때, 데이터 무결성이 보장된다.
  - 제약조건을 통해 보장 ex) 유니크 제약, 외래키 제약 등
- ISOLATION
  - 트랜잭션은 서로 간섭하지 않고 독립적으로 동작
  - 하지만, 많은 성능을 포기해야하므로 개발자가 제어가 가능 -> 트랜잭션 격리레벨을 통해 via MVCC
- DURABILITY
  - 완료된 트랜잭션은 유실되지 않는다. -> via WAL

### 03. 게시물과 타임라인 정합성 보장

### 04. 트랜잭션 격리레벨 (MySQL)
- 동시에 여러 트랜잭션이 처리될 때, 트랜잭션끼리 얼마나 서로 고립되어 있는지를 나타내는 것
- 즉, 특정 트랜잭션이 다른 트랜잭션에 변경한 데이터를 볼 수 있도록 하는 허용 여부를 결정하는 것
- 격리 레벨 구분 기준
  - Dirty Read, Non-Repeatable Read, Phantom READ
- 격리레벨 종류 (아래로 내려갈 수록, 트랜잭션 간 고립 정도가 높아지며, 성능이 떨어지는 것이 일반적)
  - READ UNCOMMITTED
    - 어떤 트랜잭션의 변경내용이 COMMIT 이나 ROLLBACK 과 상관없이 다른 트랜잭션에서 보여진다.
    - 데이터 정합성에 문제가 많으므로, RDBMS 표준에서는 격리수준으로 인정하지 않는다.
  - READ COMMITTED
    - 어떤 트랜잭션의 변경 내용이 COMMIT 되어야만 다른 트랜잭션에서 조회할 수 있다.
    - 오라클 DBMS 에서 기본으로 사용하고 있고, 온라인 서비스에서 가장 많이 선택되는 격리수준이다.
  - REPEATABLE READ
    - 트랜잭션이 시작되기 전에 커밋된 내용에 대해서만 조회할 수 있는 격리수준
    - MySQL DBMS 에서 기본으로 사용하고 있고, 이 격리수준에서는 NON-REPEATABLE READ 부정합이 발생하지 않는다.
    - 자신의 트랜잭션 번호보다 낮은 트랜잭션 번호에서 변경된(+커밋된) 것만 보게 되는 것이다.
    - 모든 InnoDB의 트랜잭션은 고유한 트랜잭션 번호(순차적으로 증가하는)를 가지고 있으며,
      Undo 영역에 백업된 모든 레코드는 변경을 발생시킨 트랜잭션의 번호가 포함되어 있다.
  - SERIALIZABLE READ
    - 가장 단순하고 가장 엄격한 격리수준이다.
    - 동시처리 능력이 다른 격리수준보다 떨어지고, 성능저하가 발생하게 된다.

<hr>

## 동시성 제어하기

### 01. 멀티 스레드 환경에 대한 이해
- 대부분 하나의 웹 서버는 여러 개의 요청을 동시에 수행할 수 있다.
- 하나의 자원을 두고 여러 개의 연산들이 경합 -> 데이터 정합성을 깨뜨릴 수 있다.
- 데이터베이스에서 동시성 이슈가 발생하는 일반적인 패턴 -> 동시성 이슈를 제어하기 위해서 공유 자원에 대한 잠금을 획득하여 줄 세우기
  1. 공유자원 조회 -> 다른 오페레이션 수행 -> 공유자원 갱신
- 동시성 이슈가 어려운 이유
  - 로컬에서는 대부분 하나의 스레드로 테스트
  - 이슈가 발생하더라도 오류가 발생하지 않는다.
  - 코드에서 잘 보이지 않는다.
  - 항상 발생하지 않고 비결정적으로 발생한다.
- 간단하게 표현하면, 작성한 코드 한 줄은 동시에 수행될 수 있다 -> 즉, 동시에 수행되어도 이슈가 없는지 확인 필요

### 02. 쓰기락과 읽기락

### 03. 좋아요 기능 구현

### 04. 낙관적 락

### 05. 낙관적 락으로 좋아요 구현

### 06. 좋아요 수 집계 테이블 분리

### 07. 읽기와 쓰기의 트레이드 오프