# RDB 인덱스는 조회 성능에 얼마나 영향을 끼치는가

#### 실험 요약

1. 적절한 인덱스 유무만으로 낮게는 71%에서 93% 까지의 성능 개선을 이끌어 낼 수 있습니다.
2. 인덱스 컬럼의 데이터 분포에 따라서 인덱스의 효율은 큰 차이를 보입니다.
3. 적절한 쿼리 튜닝은 동일한 결과 집합을 보장하면서 더 나은 성능을 이끌어 내고, 추가적으로 나은 인덱스 세팅과 실행 계획을 이끌어 낼 수 있습니다.
4. 부적절한 인덱스 사용으로 인해 극심한 성능 저하가 발생할 수도 있습니다.

#### 실험환경

* Docker + Mysql 8.0
* **총 10만 건**이 저장된 회원 테이블
* **총 1,000만 건**이 저장된 2024\~2025년 범위의 주문 테이블
* **총 1,000만** 건이 저장된 2019\~2025년 범위의 주문 테이블

#### 실험방식

* 특정 기간 동안 구매 내역이 있는 회원 목록을 최종 접속일 기준 내림차순으로 조회하는 쿼리를 실행하며, 인덱스 유무, 인덱스 컬럼 데이터 분포 정도, 쿼리 튜닝 여부에 따른 소요 시간 산출 및 개선 정도 측정
* 매 쿼리 실행 시 마다 `docker run --rm --privileged --pid=host alpine sh -c "sync && echo 3 > /proc/sys/vm/drop_caches"` 명령어를 실행하여 Docker 내부 캐시를 초기화하여 실험 간 버퍼 캐시로 인한 영향을 최소화
* 실행 통계의 가장 상단의 `actual time` 기준 쿼리 성능 측정

#### 실험 통계 요약

| **주문 데이터 분포** | **Full Table Scan** | **Fit-index** | **Query Turning** | **Paging (Top-50)** | **Worst Index** |
| ------------- | ------------------- | ------------- | ----------------- | ------------------- | --------------- |
| 좁음            | 15,177 ms           | 4,348 ms      | 621 ms            | 20 ms               | 96,518 ms       |
| 넓음            | 8,828 ms            | 592 ms        | 210 ms            | 17 ms               |                 |

* **Query Turning**은 동일한 결과 집합을 보장하면서도 성능적으로 더 우수한 쿼리로 변경 이후, 적합한 인덱스를 새로 생성하여 측정
* \*\*Paging(Top-50)\*\*은 Query Turing 이후, 실무에 적합하게 페이징 기법을 적용하여 측정
* **Worst Index**는 Fit-index에서 조인 컬럼을 제외했을 때 성능을 측정
* 각 데이터 분포에서 Paging(Top-50) 방식은 Full Table Scan 대비, **99.86%, 99.8%** 의 성능 개선 효과를 보입니다.

***

#### 1. 최악의 상황을 기준으로 시작하는 성능 분석

대부분의 RDB 테이블은 PK, FK와 그에 따른 인덱스를 가지고 있습니다. 개발자 입장에서도 딱히 어색할 것이 없기 때문에 간과하지만 만약 조인 대상 테이블에 PK 인덱스가 없다면 어떻게 될까요?

```sql
EXPLAIN ANALYZE
SELECT m.*
FROM orders o
	JOIN member_no_pk m ON o.member_id = m.member_id
WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
ORDER BY m.last_login_at;

#해당 쿼리가 동작하는 member_no_pk 는 primary key를 포함한 어떠한 인덱스도 존재하지 않음
```

**요약**

members → orders 순으로 조인된 결과 극히 비효율적으로 쿼리가 수행되어 총 23분이 소요되었습니다.

**해석**

일반적인 PK, FK 인덱스가 구성된 테이블의 경우 옵티마이저는 **주문 → 상품 순으로 NL조인**을 수행합니다. 이유는 조건절에 포함된 주문일자 조건으로 인해 먼저 주문의 행을 줄이고 조인을 수행하는게 산술적으로 이득이 되기 때문입니다.

하지만 이 실험에서는 members 테이블에 PK 인덱스가 존재하지 않고, 그로인해 옵티마이저는 NL조인이 수행될때 members 테이블이 Inner table이 된다면, 극심한 성능 저하가 일어날 것이라 예상했습니다.

결과적으로 주문 필터링 포기하더라도 FK인덱스를 통해 수행되는게 낫다는 판단하에 Driving Table을 member\_no\_pk 테이블로 전환하여 쿼리가 수행되었습니다.

즉, 총 유저 수만큼 상품 테이블을 **`Index Range Scan`** 하였으며, 총 초요 시간은 1,370초 가량 으로 **하나의 쿼리를 실행하는데 약 23분이 소요되었습니다.**

일반적인 상황은 아니지만 인덱스 유무가 얼마나 큰 성능 차이를 만드는지 알 수 있습니다.

***

#### 2. members 테이블에 PK 인덱스가 존재한다면?

```sql
EXPLAIN ANALYZE
SELECT DISTINCT m.*
FROM orders_2024_2025 o FORCE INDEX (`PRIMARY`)
         JOIN members m ON m.member_id = o.member_id
WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
ORDER BY m.last_login_at;

```

```sql
-> Sort: m.last_login_at  (actual time=15170..15177 rows=99985 loops=1)
    -> Table scan on <temporary>  (cost=10.3e+6..10.3e+6 rows=1.08e+6) (actual time=15099..15124 rows=99985 loops=1)
        -> Temporary table with deduplication  (cost=10.3e+6..10.3e+6 rows=1.08e+6) (actual time=15099..15099 rows=99985 loops=1)
            -> Nested loop inner join  (cost=10.2e+6 rows=1.08e+6) (actual time=4.81..11310 rows=845825 loops=1)
                -> Filter: (o.order_date between '2024-12-01' and '2025-01-01')  (cost=9.82e+6 rows=1.08e+6) (actual time=4.79..9865 rows=845825 loops=1)
                    -> Table scan on o  (cost=9.82e+6 rows=9.73e+6) (actual time=4.78..7613 rows=10e+6 loops=1)
                -> Single-row index lookup on m using PRIMARY (member_id=o.member_id)  (cost=0.25 rows=1) (actual time=0.00156..0.00158 rows=1 loops=845825)

```

**요약**

조인 자체는 PK 인덱스로 빠르게 처리되었지만, 드라이빙 테이블에서 읽는 양이 커 후처리 비용이 증가하여 **약 15초**가 소요되었습니다. 인덱스가 전혀 없는 것보단 낫지만 실사용에는 부담스러운 성능입니다.

**해석**

* 병목은 **드라이빙 테이블의 대량 읽기**와 그에 따른 **임시 테이블·정렬**입니다.
* `FORCE INDEX(PRIMARY)`로 기간 조건에 적합한 인덱스를 활용하지 못해 초기 후보가 과도하게 커졌습니다.
* **15초**는 어드민 페이지에서 사용하기에도 너무 느리며, 반복 호출 시 DB 리소스 소모가 큽니다.

***

#### 3. 해당 쿼리에 적합한 인덱스를 구성한다면?

드라이빙 테이블에 기간 조건과 조인 키 순서에 맞춘 인덱스로 읽기 범위를 선별했습니다.

```sql
EXPLAIN ANALYZE
SELECT DISTINCT m.*
FROM orders_2024_2025 o FORCE INDEX (orders_narrow_order_date_member_id_index)
         JOIN members m ON m.member_id = o.member_id
WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
ORDER BY m.last_login_at;
```

```sql
-> Sort: m.last_login_at  (actual time=4341..4348 rows=99985 loops=1)
    -> Table scan on <temporary>  (cost=1.17e+6..1.19e+6 rows=1.79e+6) (actual time=4269..4294 rows=99985 loops=1)
        -> Temporary table with deduplication  (cost=1.17e+6..1.17e+6 rows=1.79e+6) (actual time=4269..4269 rows=99985 loops=1)
            -> Nested loop inner join  (cost=989932 rows=1.79e+6) (actual time=0.91..1614 rows=845825 loops=1)
                -> Filter: ((o.order_date >= TIMESTAMP'2024-12-01 00:00:00') and (o.order_date < TIMESTAMP'2025-01-01 00:00:00'))  (cost=363032 rows=1.79e+6) (actual time=0.896..334 rows=845825 loops=1)
                    -> Covering index range scan on o using orders_narrow_order_date_member_id_index over ('2025-01-01 00:00:00' < order_date <= '2024-12-01 00:00:00')  (cost=363032 rows=1.79e+6) (actual time=0.892..232 rows=845825 loops=1)
                -> Single-row index lookup on m using PRIMARY (member_id=o.member_id)  (cost=0.25 rows=1) (actual time=0.00138..0.0014 rows=1 loops=845825)

```

**요약**

초기 후보를 인덱스로 선별하여 실행 시간은 약 **4.3초**로 단축되었습니다. 2번 대비 **약 71%** 개선되었습니다.

**해석**

* 드라이빙 테이블에서 **불필요한 테이블 접근이 제거**되며 I/O·CPU가 크게 줄었습니다.
* 반복이 빈번치 않은 어드민성 조회로는 수용 가능하나, B2C 트래픽에는 추가 최적화 여지가 있습니다.

***

#### 4. 만약 주문 데이터의 생성일자 분포가 다르다면? - Table Full Scan

동일한 1,000만 건을 유지하되 주문일자 분포를 **2019년 \~ 2025년 9월 3일**로 확장한 뒤, 동일한 조회 기간(2024‑12‑01 \~ 2025‑01‑01)에 대해 실행한 시나리오입니다. 아래 실행 계획에서는 드라이빙 테이블이 **전체 테이블 스캔**을 수행했습니다.

```sql
EXPLAIN ANALYZE
SELECT DISTINCT m.*
FROM orders_2024_2025 o FORCE INDEX (orders_narrow_order_date_member_id_index)
         JOIN members m ON m.member_id = o.member_id
WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
ORDER BY m.last_login_at;
```

```sql
-> Sort: m.last_login_at  (actual time=8823..8828 rows=71688 loops=1)
    -> Table scan on <temporary>  (cost=11e+6..11.1e+6 rows=1.08e+6) (actual time=8768..8787 rows=71688 loops=1)
        -> Temporary table with deduplication  (cost=11e+6..11e+6 rows=1.08e+6) (actual time=8768..8768 rows=71688 loops=1)
            -> Nested loop inner join  (cost=10.9e+6 rows=1.08e+6) (actual time=4.49..8462 rows=126720 loops=1)
                -> Filter: ((o.order_date >= TIMESTAMP'2024-12-01 00:00:00') and (o.order_date < TIMESTAMP'2025-01-01 00:00:00'))  (cost=10.6e+6 rows=1.08e+6) (actual time=4.48..8242 rows=126720 loops=1)
                    -> Table scan on o  (cost=10.6e+6 rows=9.72e+6) (actual time=4.46..7692 rows=10e+6 loops=1)
                -> Single-row index lookup on m using PRIMARY (member_id=o.member_id)  (cost=0.25 rows=1) (actual time=0.00159..0.00161 rows=1 loops=126720)

```

**요약**

후보 행 수는 845,825건 → 126,720건 (약 1/7)으로 줄었고, 전체 실행 시간은 **약 8.8초**로 단축되었습니다. 다만 **풀스캔 I/O 비용**이 커 **#2 대비 대략 절반 수준**의 개선에 그쳤습니다.

**해석**

* 분포 확장으로 기간 선택도는 높아졌으나, **풀스캔의 초기 I/O**가 지배적이어서 시간 감소가 후보 수 감소에 비례하지 않았습니다.
* 스캔 방식(풀스캔 vs 범위 스캔)이 전체 지연을 좌우합니다.

***

#### 5. 만약 주문 데이터의 생성일자 분포가 다르다면? - Index Range Scan

```sql
EXPLAIN ANALYZE
SELECT DISTINCT m.*
FROM orders_2019_2025 o FORCE INDEX (orders_2019_2025_order_date_member_id_index)
         JOIN members m ON m.member_id = o.member_id
WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
ORDER BY m.last_login_at;
```

```sql
-> Sort: m.last_login_at  (actual time=587..592 rows=71688 loops=1)
    -> Table scan on <temporary>  (cost=176807..180196 rows=270892) (actual time=529..548 rows=71688 loops=1)
        -> Temporary table with deduplication  (cost=176807..176807 rows=270892) (actual time=529..529 rows=71688 loops=1)
            -> Nested loop inner join  (cost=149718 rows=270892) (actual time=0.639..263 rows=126720 loops=1)
                -> Filter: ((o.order_date >= TIMESTAMP'2024-12-01 00:00:00') and (o.order_date < TIMESTAMP'2025-01-01 00:00:00'))  (cost=54906 rows=270892) (actual time=0.629..98.5 rows=126720 loops=1)
                    -> Covering index range scan on o using orders_2019_2025_order_date_member_id_index over ('2025-01-01 00:00:00' < order_date <= '2024-12-01 00:00:00')  (cost=54906 rows=270892) (actual time=0.627..84.6 rows=126720 loops=1)
                -> Single-row index lookup on m using PRIMARY (member_id=o.member_id)  (cost=0.25 rows=1) (actual time=0.00118..0.0012 rows=1 loops=126720)

```

**요약**

**범위 스캔+커버링 인덱스**가 제대로 작동하여 **약 0.6초**로 단축되었습니다. 동일한 데이터 분포의 #4 대비 **86%** 개선되었으며, #2번 대비 **96%** 빠르게 동작합니다.

**해석**

* 3번 대비 `order_date` 조건 충족 레코드가 **약 84만 → 약 12만**으로 줄어 **드라이빙 테이블 크기**가 크게 감소했습니다.
* 그 결과 `members` 접근 횟수·랜덤 I/O가 함께 줄어 전체 지연이 급감했습니다.
* 결론적으로, 기존 **약 9초**에서 **0.6초 내외**로 단축되었습니다.

***

#### 6. 비효율적인 쿼리를 수정한다면?

앞선 테스트 쿼리들은 모두 `DISTINCT m.*`로 중복을 제거했는데, 이는 조인 중복을 임시로 처리하기 위함이었습니다. 동일 결과 집합을 보장하면서 중복 제거 비용을 줄이기 위해 **IN** 또는 **EXISTS**를 사용할 수 있습니다.

```sql
# IN 조건을 사용한 방식
EXPLAIN ANALYZE
SELECT m.*
FROM members m
WHERE m.member_id IN
      (SELECT member_id
       FROM orders_2019_2025 o
       WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01')
ORDER BY m.last_login_at;

# EXISTS 조건을 사용한 방식
EXPLAIN ANALYZE
SELECT m.*
FROM members m
WHERE EXISTS(SELECT 1
             FROM orders_2019_2025 o
             WHERE m.member_id = o.member_id
               AND o.order_date BETWEEN '2024-12-01' AND '2025-01-01')
ORDER BY m.last_login_at;
```

```sql
# IN 조건을 사용한 실행 통계
-> Nested loop inner join  (cost=2.71e+9 rows=27.1e+9) (actual time=149..210 rows=71688 loops=1)
    -> Sort: m.last_login_at  (cost=10093 rows=99889) (actual time=79.5..87.4 rows=100000 loops=1)
        -> Table scan on m  (cost=10093 rows=99889) (actual time=0.0354..28.8 rows=100000 loops=1)
    -> Single-row index lookup on <subquery2> using <auto_distinct_key> (member_id=m.member_id)  (cost=81995..81995 rows=1) (actual time=0.00109..0.00113 rows=0.717 loops=100000)
        -> Materialize with deduplication  (cost=81995..81995 rows=270892) (actual time=69.4..69.4 rows=71688 loops=1)
            -> Filter: ((o.order_date >= TIMESTAMP'2024-12-01 00:00:00') and (o.order_date < TIMESTAMP'2025-01-01 00:00:00'))  (cost=54906 rows=270892) (actual time=0.652..37.1 rows=126720 loops=1)
                -> Covering index range scan on o using orders_2019_2025_order_date_member_id_index over ('2025-01-01 00:00:00' < order_date <= '2024-12-01 00:00:00')  (cost=54906 rows=270892) (actual time=0.649..25.7 rows=126720 loops=1)

# EXISTS 조건을 사용한 실행 통계
-> Nested loop inner join  (cost=2.71e+9 rows=27.1e+9) (actual time=144..204 rows=71688 loops=1)
    -> Sort: m.last_login_at  (cost=10093 rows=99889) (actual time=75.7..83.8 rows=100000 loops=1)
        -> Table scan on m  (cost=10093 rows=99889) (actual time=0.0296..27.8 rows=100000 loops=1)
    -> Single-row index lookup on <subquery2> using <auto_distinct_key> (member_id=m.member_id)  (cost=81995..81995 rows=1) (actual time=0.00107..0.00111 rows=0.717 loops=100000)
        -> Materialize with deduplication  (cost=81995..81995 rows=270892) (actual time=67.9..67.9 rows=71688 loops=1)
            -> Filter: ((o.order_date >= TIMESTAMP'2024-12-01 00:00:00') and (o.order_date < TIMESTAMP'2025-01-01 00:00:00'))  (cost=54906 rows=270892) (actual time=0.626..36.8 rows=126720 loops=1)
                -> Covering index range scan on o using orders_2019_2025_order_date_member_id_index over ('2025-01-01 00:00:00' < order_date <= '2024-12-01 00:00:00')  (cost=54906 rows=270892) (actual time=0.622..25.8 rows=126720 loops=1)

```

**요약**

두 방식 모두 **동일 결과 집합**을 보장하면서, 조인 중복 제거 부담을 낮춰 **#5 대비 약 50% 내외**의 추가 단축을 확인했습니다.

**해석**

* 조인 단계에서의 **불필요한 중복 제거 작업을 회피**하여 실행 시간이 감소합니다.
* 데이터 분포·캐시 상황에 따라 두 방식의 체감 차이는 미소할 수 있습니다.
* 다만 데이터 분포 상황에 따라 옵티마이저에 의해 **두 쿼리의 실행 계획이 달라질 수 있으므로** 확인이 필요할 수 있습니다.

***

#### 7. 페이징 방식을 사용한 실용적 쿼리 튜닝

```sql
# IN 조건을 사용한 방식
EXPLAIN ANALYZE
SELECT m.*
FROM members m
WHERE m.member_id IN
      (SELECT member_id
       FROM orders_2019_2025 o FORCE INDEX (orders_members_member_id_fk)
       WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
	       AND o.member_id = m.member_id)
ORDER BY m.last_login_at
**LIMIT 0, 50**;

**CREATE INDEX members_last_login_at_index ON members (last_login_at DESC);**

```

2\~4번에서는 드라이빙 테이블이 `orders`였지만, 본 쿼리는 `members`**를 드라이빙**으로 하여 **정렬 인덱스+부분 범위 처리**를 결합합니다.

1. 정렬/페이징에 맞는 인덱스로 `members`를 최신순(내림차순)으로 순회합니다.
2. 각 회원에 대해 `orders`에서 조건 충족 레코드 존재 여부만 확인합니다.
3. 조건을 만족하는 회원이 **50명** 모이면 즉시 종료합니다.

```sql
-> Limit: 50 row(s)  (cost=8.2e+6 rows=41.9) (actual time=0.386..17.1 rows=50 loops=1)
    -> Nested loop semijoin  (cost=8.2e+6 rows=41.9) (actual time=0.385..17.1 rows=50 loops=1)
        -> Index scan on m using members_last_login_at_index (reverse)  (cost=0.00419 rows=4) (actual time=0.0628..0.329 rows=60 loops=1)
        -> Filter: (o.order_date between '2024-12-01' and '2025-01-01')  (cost=861 rows=10.5) (actual time=0.279..0.279 rows=0.833 loops=60)
            -> Index lookup on o using orders_members_member_id_fk (member_id=m.member_id)  (cost=861 rows=94.3) (actual time=0.261..0.268 rows=46.8 loops=60)

```

**요약**

평균 응답이 **약 0.017초**로 단축되어 API용으로 충분한 수준이 되었습니다. #4 대비 **99.8%**, #5 대비 **97%**, #6 대비 **90%** 개선입니다.

**해석**

* 정렬 인덱스와 제한된 **부분 범위 처리**가 결합되면서, 필요한 상위 페이지만 읽고 **조기 종료**가 가능합니다.
* 실사용 패턴(최신순·페이지 단위)에 맞춘 **접근 순서 재설계**가 대규모 데이터에서도 체감 성능을 좌우합니다.

동일 조건을 단기간 데이터(2024‑2025)로도 검증했습니다.

```sql
EXPLAIN ANALYZE
SELECT m.*
FROM members m
WHERE m.member_id IN
      (SELECT member_id
       FROM orders_2024_2025 o FORCE INDEX (orders_2024_2025_members_member_id_fk)
       WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
	       AND o.member_id = m.member_id)
ORDER BY m.last_login_at
LIMIT 0, 50;
```

```sql
-> Limit: 50 row(s)  (cost=8.56e+6 rows=44.6) (actual time=0.397..20.1 rows=50 loops=1)
    -> Nested loop semijoin  (cost=8.56e+6 rows=44.6) (actual time=0.396..20.1 rows=50 loops=1)
        -> Index scan on m using members_last_login_at_index (reverse)  (cost=0.00419 rows=4) (actual time=0.0309..0.144 rows=50 loops=1)
        -> Filter: (o.order_date between '2024-12-01' and '2025-01-01')  (cost=956 rows=11.2) (actual time=0.399..0.399 rows=1 loops=50)
            -> Index lookup on o using orders_2024_2025_members_member_id_fk (member_id=m.member_id)  (cost=956 rows=100) (actual time=0.393..0.395 rows=14.7 loops=50)

```

**요약(동일 분포 단기간 데이터)**

총 **20ms**로, 동일 데이터셋의 3번(4,348ms) 대비 **약 99.5% 개선**을 확인했습니다.

***

#### 8. 비효율적인 인덱스로 인한 성능 저하 사례

```sql
EXPLAIN ANALYZE
SELECT DISTINCT m.*
FROM orders_2024_2025 o FORCE INDEX (orders_2024_2025_order_date_index)
         JOIN members m ON m.member_id = o.member_id
WHERE o.order_date BETWEEN '2024-12-01' AND '2025-01-01'
ORDER BY m.last_login_at;
```

```sql
-> Sort: m.last_login_at  (actual time=96511..96518 rows=99985 loops=1)
    -> Table scan on <temporary>  (cost=3.98e+6..4e+6 rows=1.76e+6) (actual time=96438..96463 rows=99985 loops=1)
        -> Temporary table with deduplication  (cost=3.98e+6..3.98e+6 rows=1.76e+6) (actual time=96438..96438 rows=99985 loops=1)
            -> Nested loop inner join  (cost=3.8e+6 rows=1.76e+6) (actual time=61.7..91666 rows=845825 loops=1)
                -> Index range scan on o using orders_2024_2025_order_date_index over ('2025-01-01 00:00:00' <= order_date <= '2024-12-01 00:00:00'), with index condition: (o.order_date between '2024-12-01' and '2025-01-01')  (cost=1.87e+6 rows=1.76e+6) (actual time=60.2..74816 rows=845825 loops=1)
                -> Single-row index lookup on m using PRIMARY (member_id=o.member_id)  (cost=0.998 rows=1) (actual time=0.0195..0.0196 rows=1 loops=845825)

```

**요약**

조건절에 포함된 `order_date` **단일 인덱스만** 사용하면, 조인 키(`member_id`)를 선별하지 못해 **약 96초**까지 지연이 악화될 수 있습니다.

**해석**

* 범위 스캔 자체는 인덱스로 가능하지만, 조인 키가 인덱스에 없는 관계로 **조인 단계에서 약 85만 회**의 테이블 접근이 발생합니다.
* 이 경우는 **인덱스 사용 = 성능 개선**이 아님을 보여줍니다. **조건 + 조인 키**를 함께 고려한 **복합 인덱스 설계**가 필요합니다.

***

#### 실험 후기

그간 인덱스를 통한 성능 개선 효과는 서버 개발자들 사이에서 중요하게는 여겨졌지만, 얼마나 효용이 있는지는 미지수인 기능이었습니다. 마침 SQLP를 준비하면서 쿼리 튜닝에 대한 숙련도가 크게 증가하여 실제적으로 쿼리 튜닝이 얼마나 성능 격차를 만들어 낼 수 있는지 확인하고자 이번 실험을 진행했습니다.

결론적으로 쿼리 튜닝은 대규모 데이터를 기준으로 제가 생각했던 것보다 훨씬 더 유의미한 정도의 성능 차이를 만들어 낼 수 있다는 것을 확인하였으며, 적합한 인덱스와 쿼리 튜닝의 효용, 페이징으로 이어지는 인덱스 설계의 중요성 그리고 적합하지 않은 인덱스 설계가 미치는 파급력을 정형화된 수치로 확인할 수 있는 유의미한 시간이었습니다.
