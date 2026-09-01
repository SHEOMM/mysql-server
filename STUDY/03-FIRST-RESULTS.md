# 첫 실행 결과 — 환경이 실제로 동작한다는 증거

두 시나리오를 끝까지 돌려서 **코드에서 읽은 값과 optimizer trace 의 값이 같은지**를 확인했다.
같으면 "브레이크포인트에서 본 숫자 = 옵티마이저가 실제로 쓴 숫자"가 확정되고,
이후 모든 학습이 그 위에서 굴러간다.

## 1. 인덱스 선택 — 상수만 바꿔도 판단이 뒤집힌다

```
tools/dbg.sh recipes/index_selection.json sql/index_selection.sql idx1
```

테이블은 5만 행, `a` 는 0~99, `b` 는 `a` 와 무관하게 0~499. 인덱스는 `k_a(a)` 와 `k_ab(a,b)`.

### 케이스 A — `WHERE a = 7 AND b BETWEEN 100 AND 120`

| 지점 | 코드에서 읽은 값 |
|---|---|
| `records_in_range` keynr=1 | `n_rows=500`, `index->name = "k_a"` |
| `records_in_range` keynr=2 | `n_rows=21`, `index->name = "k_ab"` |
| range 플랜 확정 | `cost=2.367320644216691`, `type=INDEX_RANGE_SCAN` |
| `best_access_path` 확정 | `read_cost=2.36732`, `rows_fetched=21`, `pos->key=NULL` |

optimizer trace 쪽:

```
table_scan  rows=48877 cost=4930.05
k_a         rows=500   cost=175.26    chosen
k_ab        rows=21    cost=2.36732   chosen   <- 요약: k_ab
```

`cost` 가 소수점까지 같다. `pos->key` 가 NULL 인 것도 맞다. ref 접근이 아니라 range 접근이라
`POSITION::key`(Key_use\*)에는 아무것도 안 들어가고, 실제 계획은 `tab->range_scan()` 에 들어 있다.

### 케이스 B — `WHERE a < 90` (테이블의 90%)

여기서 재미있는 게 두 개 나온다.

| 지점 | 값 |
|---|---|
| `records_in_range` (k_a, k_ab 둘 다) | `n_rows=25067` |
| range 플랜 확정 | `cost=2513.07` |
| optimizer trace | `table_scan cost=5055.75` / `k_a 2513.07 index_only=True chosen` / `k_ab 2516.13 chosen=False cause=cost` |

**첫째, InnoDB 의 행 수 추정이 크게 빗나간다.** `a < 90` 은 실제로 5만 행 중 4만 5천 행인데
`records_in_range` 는 25,067 을 돌려준다. `btr_estimate_n_rows_in_range` 가 B-tree 를
전수 조사하지 않고 양 끝 페이지 사이를 샘플링해서 추정하기 때문이다. 옵티마이저는
이 값을 그대로 믿고 비용을 계산한다.

**둘째, 90% 를 훑는데도 풀스캔이 지는 이유는 커버링이다.** `SELECT COUNT(*)` 에 필요한 컬럼이
`a` 뿐이라 `k_a` 만 읽고 끝난다(trace 의 `index_only=True`). 레코드를 찾아 클러스터드 인덱스로
돌아가는 비용이 없으니 5,055 짜리 풀스캔보다 2,513 이 싸다.
`k_ab` 는 같은 행 수를 같은 방식으로 읽지만 인덱스가 더 넓어서 2,516 으로 근소하게 진다.

원래 "90% 면 풀스캔으로 갈 것"이라고 예상하고 만든 쿼리인데 예상이 틀렸다.
**커버링이 되는 순간 선택도 기준이 무의미해진다**는 걸 비용 숫자로 확인한 셈이다.

## 2. CREATE TABLE — 서버에서 InnoDB DDL 로그까지

```
tools/dbg.sh recipes/ddl_create_table.json sql/ddl_create_table.sql ddl1
```

```
mysql_execute_command  sql_parse.cc:2949   sql_command = SQLCOM_CREATE_TABLE
└ mysql_create_table   sql_table.cc:10037  db="study" table_name="t_ddl"
  └ create_table_impl  sql_table.cc:8743   path="./study/t_ddl"
                                           internal_tmp_table=false no_ha_table=false
                                           do_not_store_in_dd=false
    ├ mysql_prepare_create_table  sql_table.cc:7948
    └ ha_innobase::create         ha_innodb.cc:15105  name="./study/t_ddl"
      ├ Log_DDL::write_delete_space_log  space_id=7  file_path="./study/t_ddl.ibd"
      ├ Log_DDL::write_remove_cache_log
      ├ Log_DDL::write_free_tree_log     (PRIMARY)
      └ Log_DDL::write_free_tree_log     (k_ab)
```

**`write_free_tree_log` 가 두 번 불린 게 핵심이다.** 인덱스가 PRIMARY 와 `k_ab` 두 개라
각각에 대해 "이 B-tree 를 해제해야 한다"는 되감기 항목을 미리 남긴다.
`write_delete_space_log` 는 `.ibd` 파일 삭제 예약이다.

즉 InnoDB 는 **테이블을 만들기 전에 "실패하면 무엇을 지워야 하는지"를 먼저 기록한다.**
그래서 CREATE TABLE 도중에 서버가 죽어도 재기동 시 `Log_DDL::replay` 가 이 항목들을 되감아
반쯤 만들어진 테이블이 남지 않는다. 8.0 의 atomic DDL 이 이렇게 동작한다.

`ib_ddl_crash_during_create2` 같은 DBUG 키워드로 그 중간에 실제로 죽여 보면
복구 경로까지 눈으로 확인할 수 있다(`01-BREAKPOINTS.md` 참조). datadir 이 날아가도 되는 상태에서만.

## 얻은 교훈 — "확정 지점"은 반드시 소스에서 확인할 것

처음에 `best_access_path` 의 결정 지점을 `sql_planner.cc:1843` 으로 잡았는데 한 번도 안 걸렸다.
`pos->` 로 grep 해서 나온 첫 덩어리를 그대로 쓴 게 원인이었다. 그 라인은
`best_access_path`(void 반환)가 아니라 그 아래에 있는 LooseScan 헬퍼(bool 반환) 안이었다.

진짜 확정 지점은 **1218~1225** 이고, 전부 대입된 직후인 **1227** 에 걸어야 한다.

```
1217  best_read_cost += derived_mat_cost;
1218  pos->filter_effect = filter_effect;
1219  pos->rows_fetched  = rows_fetched;
1220  pos->read_cost     = best_read_cost;
1221  pos->key           = best_ref;
1222  pos->table         = tab;
...
1227  if (!best_ref && idx == join->const_tables && ...     <- 여기에 건다
```

확정 지점을 잡을 때는 함수의 **시작 라인부터 끝 라인까지 범위를 좁혀서** grep 해야 한다.
파일 전체에서 grep 하면 다른 함수의 비슷한 코드에 걸린다.
브레이크포인트가 `locations=1` 로 잡혔는데 `hit_count=0` 이면 이 실수를 의심하면 된다.
