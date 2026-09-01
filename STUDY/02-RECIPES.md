# 실험하는 법

> 명령은 워크스페이스 루트 `/Users/eom/Documents/project/mysql-study` 에서 실행한다(`WS` 로 줄여 쓴다).
> 이 문서들은 upstream 에 없는 학습 자료다. `study-8.0.42` 브랜치에만 둔다.

## 준비 (최초 1회)

```
tools/server.sh init        # datadir 초기화. root 는 비밀번호 없음
```

## 계층 1~2 — 디버거 없이 (먼저 여기부터)

```
tools/server.sh start
tools/sql.sh sql/index_selection.sql
tools/sql.sh -e "EXPLAIN FORMAT=JSON SELECT COUNT(*) FROM study.t_idx WHERE a=7 AND b BETWEEN 100 AND 120"
tools/server.sh stop
```

인덱스 선택의 근거는 대부분 optimizer trace 에 이미 다 있다.
`rows_estimation` → `analyzing_range_alternatives` → `chosen_range_access_summary` 순서로 읽으면
어떤 인덱스가 후보였고 각각 비용이 얼마였고 왜 탈락했는지가 그대로 나온다.

## 계층 3 — 브레이크포인트 (하네스)

서버가 떠 있으면 안 된다. 하네스가 mysqld 를 직접 띄운다.

```
tools/server.sh stop
tools/dbg.sh recipes/index_selection.json sql/index_selection.sql idx1
```

결과는 `out/idx1/` 에 떨어진다.

| 파일 | 내용 |
|---|---|
| `calltree.md` | 사람이 읽는 콜트리. 스택 깊이로 들여쓰기 |
| `trace.jsonl` | 정지 1건 = 1줄. `jq` 로 파고들 때 |
| `client.out` | SQL 실행 결과 (optimizer trace 포함) |
| `mysqld.stderr` / `run/error.log` | 서버가 죽었을 때 |

```
jq -r 'select(.kind=="hit" and .bp=="best_access_path(확정)") | .vars' out/idx1/trace.jsonl
jq -r 'select(.kind=="arm") | "\(.statement): \(.query)"' out/idx1/trace.jsonl
```

## 계층 3 — 브레이크포인트 (CLion)

CMake 프로필에서 **Build directory 를 `build-debug` 로, Generator 를 Ninja 로** 맞춰야 한다.
다른 디렉터리를 쓰면 같은 빌드를 통째로 한 번 더 한다.

- CMake options: `-DWITH_DEBUG=1 -DCMAKE_BUILD_TYPE=Debug -DDOWNLOAD_BOOST=1 -DWITH_BOOST=<ws>/boost -DWITH_SSL=/opt/homebrew/opt/openssl@3 -DBISON_EXECUTABLE=/opt/homebrew/opt/bison/bin/bison -DWITH_UNIT_TESTS=OFF -DWITH_ROUTER=OFF -DWITH_FIDO=none -DMYSQL_MAINTAINER_MODE=OFF`
- Run/Debug 구성: target `mysqld`, program arguments `--defaults-file=<ws>/run/my.cnf`, working directory `<ws>/run`
- 브레이크포인트는 `01-BREAKPOINTS.md` 의 심볼을 Symbolic Breakpoint 로 넣는다
- 인덱싱이 무거우면 `mysql-test/`, `extra/` 를 Excluded 로 표시하고 IDE 힙을 8GB 로 올린다

## 레시피 만들기

`recipes/*.json` 하나가 실험 하나다.

```json
{
  "title": "설명",
  "arm": {
    "symbol": "dispatch_sql_command",
    "query_expr": "thd->m_query_string.str",
    "marker": "/* STUDY */"
  },
  "max_hits_per_bp": 30,
  "breakpoints": [
    { "name": "표시이름", "symbol": "함수명", "capture": ["표현식"], "backtrace": 4 },
    { "name": "표시이름", "file": "파일명.cc", "line": 1843, "capture": ["pos->read_cost"] }
  ]
}
```

- `symbol` 은 함수 진입, `file`+`line` 은 함수 중간의 특정 지점. **결론이 확정되는 라인**은 대개 후자로 잡아야 한다.
- `capture` 는 그 프레임에서 평가할 표현식. `a->b.c` 같은 단순 경로는 값 워킹으로 빠르게 처리되고,
  함수 호출이 섞이면 표현식 평가로 넘어가 느려진다. 되도록 필드 경로로 쓴다.
- 평가에 실패하면 값이 `<err: ...>` 로 남는다. 하네스가 죽지는 않으니 그걸 보고 표현식을 고치면 된다.
- `max_hits` 를 넘긴 히트는 조용히 버린다. 루프 안의 브레이크포인트가 로그를 덮지 않게 하는 장치다.

## 알아두면 덜 헤매는 것들

- **attach 는 안 된다.** 이 맥은 Developer mode 가 꺼져 있어서 실행 중인 프로세스에 붙으면 인증 창이 뜬다.
  하네스는 항상 mysqld 를 직접 띄운다. 굳이 붙어야 하면 `sudo DevToolsSecurity -enable` 을 한 번 실행한다.
- **`build-debug/` 를 옮기거나 지우지 말 것.** macOS 는 링크할 때 DWARF 를 실행파일에 넣지 않고
  `.o` 파일을 참조하는 debug map 을 쓴다. 빌드 디렉터리가 없어지면 심볼이 통째로 사라진다.
- **InnoDB 래치를 쥔 지점에 10분 이상 멈추지 말 것.** `innodb_semaphore_wait_timeout_debug` 의
  상한이 600초라 그걸 넘기면 서버가 스스로 죽는다. 하네스는 캡처 후 즉시 재개하므로 무관하고,
  CLion 에서 손으로 멈춰 놓고 자리를 비울 때만 문제가 된다.
- **3306 은 Docker 가 쓴다.** 이 인스턴스는 3307 과 `run/mysqld.sock` 을 쓴다.
- 디버그 빌드는 `-O0` 이라 느리다. 대량 INSERT 는 릴리스 대비 수 배 걸린다.
