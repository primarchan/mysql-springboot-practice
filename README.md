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