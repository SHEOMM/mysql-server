# 브레이크포인트 카탈로그 (MySQL 8.0.42)

라인 번호는 태그 `mysql-8.0.42` 기준으로 실제 확인한 값이다. **심볼 이름이 기준**이고 라인은 참고용이다.

CLion 에서는 `Run > View Breakpoints > + > Symbolic Breakpoint` 로 아래 심볼을 그대로 넣으면
AI 하네스(`recipes/*.json`)와 정확히 같은 지점에 걸린다.

## 공통 진입

| 심볼 | 위치 | 보는 것 |
|---|---|---|
| `dispatch_command` | `sql/sql_parse.cc:1688` | 프로토콜 커맨드(COM_QUERY 등) 진입 |
| `dispatch_sql_command` | `sql/sql_parse.cc:5254` | 파싱 후 실행 진입. `thd->m_query_string.str` 에 원문 SQL이 들어있다 |
| `mysql_execute_command` | `sql/sql_parse.cc:2948` | `thd->lex->sql_command` 로 갈라지는 거대 switch |

하네스는 `dispatch_sql_command` 를 **arming 지점**으로 쓴다. 여기서 쿼리 문자열을 읽어
`/* STUDY */` 마커가 있을 때만 나머지 브레이크포인트를 켠다.

## DDL — CREATE TABLE

```
mysql_execute_command
└ mysql_create_table            sql/sql_table.cc:10035
  └ create_table_impl           sql/sql_table.cc:8733
    ├ mysql_prepare_create_table  sql/sql_table.cc:7940   컬럼/키 정의를 KEY 배열로 확정
    ├ (dd::) 테이블 정의를 데이터 딕셔너리 객체로 채움    sql/dd/dd_table.cc
    └ ha_innobase::create       storage/innobase/handler/ha_innodb.cc:15103
      └ Log_DDL::write_*        storage/innobase/log/log0ddl.cc
```

`create_table_impl` 의 파라미터가 특히 볼 만하다. `no_ha_table`, `do_not_store_in_dd` 는
ALTER 의 중간 단계나 임시 테이블에서 "엔진에는 만들되 딕셔너리에는 안 넣는" 식으로 쓰인다.

**Atomic DDL 의 핵심** — `log0ddl.cc` 의 DDL 로그. DDL 이 중간에 죽어도 되감을 수 있게,
InnoDB 는 작업 전에 "무엇을 되돌려야 하는지"를 먼저 기록한다.

| 심볼 | 라인 | 의미 |
|---|---|---|
| `Log_DDL::write_free_tree_log` | `log0ddl.cc:853` | 인덱스 B-tree 해제 예약 |
| `Log_DDL::write_delete_space_log` | `log0ddl.cc:961` | 테이블스페이스 파일 삭제 예약 |
| `Log_DDL::write_drop_log` | `log0ddl.cc:1258` | 딕셔너리 엔트리 삭제 예약 |
| `Log_DDL::write_rename_table_log` | `log0ddl.cc:1313` | 이름 되돌리기 예약 |
| `Log_DDL::replay` | `log0ddl.cc:1576` | 크래시 복구 시 실제 되감기 |

## 인덱스 판단

```
JOIN::optimize                      sql/sql_optimizer.cc:337
├ estimate_rowcount                 sql/sql_optimizer.cc:5904
│ └ test_quick_select               sql/range_optimizer/range_optimizer.cc:524   range 후보 생성·비용
│   └ ha_innobase::records_in_range storage/innobase/handler/ha_innodb.cc:16695  실제 행수 추정
└ make_join_plan                    sql/sql_optimizer.cc:5308
  └ Optimize_table_order::choose_table_order  sql/sql_planner.cc:1951
    └ Optimize_table_order::best_access_path  sql/sql_planner.cc:981            ★ 여기서 결정
```

진입점만 걸면 입력만 보이고 결론이 안 보인다. **결론이 확정되는 라인**을 따로 잡아야 한다.

| 이름 | 위치 | 그 시점에 확정된 값 |
|---|---|---|
| best_access_path 확정 | `sql/sql_planner.cc:1843` | `pos->read_cost`, `pos->rows_fetched`, `pos->key`(null 이면 인덱스 미사용) |
| range 플랜 확정 | `sql/range_optimizer/range_optimizer.cc:808` | `best_path->cost`, `best_path->type` |
| records_in_range 결과 | `storage/innobase/handler/ha_innodb.cc:16808` | `n_rows` — InnoDB 가 옵티마이저에 돌려주는 추정치 |
| 통계 전달 | `ha_innobase::info_low` (`ha_innodb.cc:17201`) | `stats.records`, 인덱스별 카디널리티 |

`pos->key` 는 `Key_use*` 다. null 이면 ref 접근을 안 쓴다는 뜻이고, 인덱스 번호는 `pos->key->key` 에 있다.

## 디버거 없이 같은 지점을 보는 법

브레이크포인트보다 먼저 시도할 것들이다. 전부 SQL 로 켜고 끈다.

**optimizer trace** — 인덱스 선택의 비용 계산이 JSON 으로 그대로 나온다.
```sql
SET SESSION optimizer_trace='enabled=on';
SELECT ...;
SELECT TRACE FROM information_schema.OPTIMIZER_TRACE;
```

**DBUG 트레이스** — 디버그 빌드 전용. 함수 호출 트리를 텍스트로 덤프한다.
```sql
SET SESSION debug='d:t:i:o,/Users/eom/Documents/project/mysql-study/run/dbug.trace';
SELECT ...;
SET SESSION debug='';
```
전체를 뜨면 수백 MB 가 되므로 함수 필터를 건다: `d:f,best_access_path,test_quick_select:t:i:o,<파일>`

**DEBUG_SYNC** — 디버거 없이 서버 내부 지점에서 결정적으로 멈춘다. 동시성·DDL 원자성 학습용.
DDL 경로에 이미 박혀 있는 지점들(소스에서 추출):
`locked_table_name`, `alter_table_before_open_tables`, `alter_table_before_create_table_no_lock`,
`alter_table_copy_after_lock_upgrade`, `alter_table_inplace_after_lock_upgrade`,
`alter_table_inplace_before_commit`, `alter_table_before_main_binlog`,
`action_after_write_bin_log`, `rm_table_no_locks_before_binlog`,
`mysql_rm_table_after_lock_table_names`, `copy_data_between_tables_before`

```sql
-- 세션 A
SET DEBUG_SYNC='alter_table_inplace_before_commit SIGNAL reached WAIT_FOR go';
ALTER TABLE t ADD COLUMN x INT;   -- 여기서 멈춘다
-- 세션 B
SET DEBUG_SYNC='now WAIT_FOR reached';
-- ... 이 사이에 다른 트랜잭션을 걸어 관찰 ...
SET DEBUG_SYNC='now SIGNAL go';
```

**DBUG 주입 지점** — 디버그 빌드에서 `SET SESSION debug='+d,<키워드>'` 로 켜는 코드 경로.
크래시 시뮬레이션이 많아 atomic DDL 복구 학습에 쓸 만하다:
`ib_ddl_crash_during_create2`, `create_table_update_dd_fail`,
`ib_truncate_crash_after_create_new_table`, `ib_truncate_crash_after_rename`,
`ib_truncate_rollback_test`, `crash_records_in_range`, `prefer_ordering_index_check`

주의: `ib_*_crash_*` 계열은 서버를 실제로 죽인다. datadir 이 날아가도 되는 상태에서만 쓴다.
