# MySQL 8.0.42 소스 학습 환경 (세션 부트스트랩)

이 파일과 `STUDY/` 는 **upstream 에 없는 학습 자료**이고 `study-8.0.42` 브랜치에만 올라가 있다.
워크트리마다 따라가야 해서 추적 대상으로 두었다. `trunk` 에 올리거나 upstream 에 PR 을 보내지 않는다.

## 이게 뭔가

MySQL 8.0.42 를 디버그 빌드해서 **MySQL 을 DBA 수준으로 깊게 공부하는 환경**이다.
동작이 궁금하면 문서나 추측에 기대지 않고 소스에 브레이크포인트를 걸어 실제 값으로 확인한다.
사람은 CLion GUI 로, AI 는 셸에서 비대화형으로 같은 브레이크포인트 자산을 쓴다.

주제는 특정 영역에 한정하지 않는다. 옵티마이저·실행기, 스토리지 엔진과 버퍼풀, 트랜잭션과
락·MVCC, redo/undo 와 복구, 데이터 딕셔너리와 DDL, 복제와 binlog, 커넥션·스레드 관리,
통계와 실행계획까지 DBA 가 알아야 할 범위를 코드 레벨로 파고든다.
지금까지 다뤄 본 건 DDL 경로와 인덱스 선택이고, 그건 시작점이지 범위가 아니다.

- 레포: `SHEOMM/mysql-server` 포크 (`origin`). upstream 은 `mysql/mysql-server`
- 브랜치: **`study-8.0.42`** (태그 `mysql-8.0.42` 커밋 `6ba1fef58b0` 기준). 포크의 기본 브랜치도 이걸로 바꿔 놨다.
  `trunk` 은 MySQL 9.x 개발선이라 이 학습 환경과 무관하다. 워크트리를 뜰 때 trunk 로 뜨지 않게 주의
- 부분 클론(`--filter=blob:none`)이라 `git blame`·`git log -S` 를 쓰면 blob 을 그때그때 받아온다. 정상이다

## 경로

| | 경로 |
|---|---|
| 소스(여기) | `/Users/eom/Documents/project/mysql-study/mysql-server` |
| 빌드 | `/Users/eom/Documents/project/mysql-study/build-debug` |
| 도구·레시피·SQL·결과 | `/Users/eom/Documents/project/mysql-study/{tools,recipes,sql,out}` |
| 서버 datadir·my.cnf | `/Users/eom/Documents/project/mysql-study/run` |

도구는 상위 디렉터리에 있다. 세션을 이 레포에서 시작했다면
`/Users/eom/Documents/project` 를 작업 디렉터리로 추가해야 한다.
편의상 `WS=/Users/eom/Documents/project/mysql-study` 로 두고 절대경로로 부르면 된다.

**`build-debug/` 를 옮기거나 지우지 말 것.** macOS 는 링크 시 DWARF 를 실행파일에 넣지 않고
`.o` 를 참조하는 debug map 을 쓴다. 빌드 디렉터리가 사라지면 심볼이 통째로 날아간다.

## 워크트리에서 열었다면

Orca 처럼 세션마다 git worktree 를 만드는 도구로 열었다면, 지금 보는 소스는 메인 클론의 사본이다.
같은 커밋이라 내용은 같고 경로만 다르다.

- **빌드와 서버는 메인 클론 기준으로 하나뿐이다.** 워크트리마다 다시 빌드하지 않는다.
  위 경로표의 절대경로를 그대로 쓴다
- **lldb 는 컴파일 시점 경로로 소스를 찾는다.** 브레이크포인트에서 열리는 파일은 메인 클론 쪽이다.
  정상이다. 레시피는 파일명(basename)+라인으로 거니까 워크트리에서 읽은 라인 번호가 그대로 통한다
- **CLion 은 메인 클론에서 연다.** 워크트리에서 열면 CMake 가 새 빌드 디렉터리를 잡아 6GB 짜리 빌드를 한 번 더 한다
- 워크트리는 소스를 읽고 문서를 고치는 용도로 쓰고, 실행은 메인 클론 경로의 도구로 한다

## 바로 쓰는 명령

```
$WS/tools/server.sh start|stop|status      # 디버거 없이 서버 기동 (포트 3307, 소켓 run/mysqld.sock)
$WS/tools/sql.sh <파일|-e "SQL">            # SQL 실행
$WS/tools/trace.sh "<SELECT ...>"          # optimizer trace 를 원문 JSON 으로
$WS/tools/dbg.sh <레시피> <SQL> [이름]       # 브레이크포인트 실행 → out/<이름>/
```

`dbg.sh` 는 mysqld 를 직접 띄운다. **먼저 `server.sh stop` 으로 내려야 한다.**

## 관측 4계층 (비용 낮은 순서로 쓴다)

1. **L0 optimizer trace / EXPLAIN** — 인덱스 선택 근거는 대부분 여기 다 있다. `trace.sh`
2. **L1 DBUG 트레이스** — `SET SESSION debug='d:f,<함수>:t:i:o,<파일>'` 로 호출 트리를 텍스트 덤프
3. **L2 DEBUG_SYNC** — `SET DEBUG_SYNC='<지점> SIGNAL x WAIT_FOR y'` 로 디버거 없이 내부 지점에서 정지
4. **L3 LLDB 하네스** — `dbg.sh`. 변수·스택을 실제로 봐야 할 때만

## AI 가 브레이크포인트 찍는 법

대화형 lldb 는 툴 호출 사이에 세션이 안 남아서 못 쓴다. 대신 레시피 JSON 을 만들어 `dbg.sh` 로 돌린다.

```json
{
  "arm": { "symbol": "dispatch_sql_command", "query_expr": "thd->m_query_string.str", "marker": "/* STUDY */" },
  "breakpoints": [
    { "name": "표시이름", "symbol": "함수명", "capture": ["표현식"], "backtrace": 4 },
    { "name": "확정지점", "file": "sql_planner.cc", "line": 1227, "capture": ["pos->read_cost"] }
  ]
}
```

- SQL 문장에 `/* STUDY */` 를 넣은 것만 잡힌다(arming). 안 그러면 부팅·내부 쿼리까지 다 걸려 못 쓴다
- 결과: `out/<이름>/calltree.md`(사람용), `trace.jsonl`(jq 용), `client.out`(SQL 결과)
- 자세한 건 `STUDY/02-RECIPES.md`

## 함정 (전부 실제로 밟아봄)

- **`mysql` 클라이언트는 주석을 뗀다.** `--comments` 없으면 `/* STUDY */` 마커가 서버에 안 온다. 도구엔 이미 붙여 놨다
- **DEBUG_SYNC 는 `--debug-sync-timeout` 이 있어야 켜진다.** `my.cnf` 에 넣어 뒀다. 확인은 `SELECT @@debug_sync`
- **"확정 지점"은 함수 라인 범위를 좁혀서 찾아야 한다.** 파일 전체 grep 으로 잡으면 다른 함수의 비슷한 코드에 걸린다.
  `locations=1` 인데 `hit_count=0` 이면 이 실수다. 실제 사례는 `STUDY/03-FIRST-RESULTS.md` 마지막 절
- **attach 불가.** 이 맥은 Developer mode 가 꺼져 있어 실행 중 프로세스에 붙으면 인증 창이 뜬다.
  하네스는 항상 직접 launch 한다. 굳이 필요하면 `sudo DevToolsSecurity -enable` 한 번
- **InnoDB 래치 구간에 10분 이상 멈추지 말 것.** `innodb_semaphore_wait_timeout_debug` 상한이 600초다
- **포트 3306 은 Docker 가 쓴다.** 이 인스턴스는 3307

## 현재까지 검증된 것

`STUDY/03-FIRST-RESULTS.md` 에 실제 숫자가 있다. 요약하면

- 인덱스 선택: 브레이크포인트에서 읽은 `read_cost`·`rows_fetched` 가 optimizer trace 의 값과 소수점까지 일치
- CREATE TABLE: `mysql_execute_command → mysql_create_table → create_table_impl → ha_innobase::create → Log_DDL::write_*` 체인 확인.
  인덱스 2개에 대해 `write_free_tree_log` 가 2번 불리는 것까지 관찰됨(atomic DDL 되감기 예약)

## 다음에 해볼 만한 것

- ALTER TABLE INPLACE vs COPY 경로 비교 (`alter_table_inplace_*` DEBUG_SYNC 지점 활용)
- `ib_ddl_crash_during_create2` 로 DDL 중간에 죽이고 `Log_DDL::replay` 복구 경로 관찰 (datadir 날아가도 되는 상태에서)
- 조인 순서 결정(`choose_table_order` → `greedy_search`) 레시피 추가
- `dict_stats` 통계 수집이 `records_in_range` 추정에 어떻게 반영되는지
