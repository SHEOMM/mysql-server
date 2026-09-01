# 셋업 (macOS 26.6 / M1 Pro / Apple clang 21 에서 실제로 통과한 절차)

> 명령은 워크스페이스 루트 `/Users/eom/Documents/project/mysql-study` 에서 실행한다(`WS` 로 줄여 쓴다).
> 이 문서들은 upstream 에 없는 학습 자료다. `study-8.0.42` 브랜치에만 둔다.

이 문서에는 **실제로 돌려서 통과한 명령만** 적는다.

## 환경

| 항목 | 값 |
|---|---|
| macOS | 26.6 (Darwin 25.6), M1 Pro 10코어, RAM 32GB |
| 컴파일러 | Apple clang 21.0.0 (Command Line Tools, Xcode 없음) |
| cmake / ninja / bison / ccache | 4.4.3 / 1.13.2 / 3.8.2 / 4.14 (전부 brew) |
| OpenSSL | brew openssl@3 3.6.2 |
| lldb | `/usr/bin/lldb` (lldb-2100), 임베디드 Python 3.9.6 |
| MySQL | 8.0.42 (`git tag mysql-8.0.42`, HEAD `6ba1fef58b0`) |

## 1. 도구 설치

```
brew install cmake ninja bison ccache
```

시스템 bison 은 2.3 이라 MySQL 이 요구하는 3.x 에 못 미친다. brew 것을 명시적으로 지정해야 한다.

## 2. 소스

```
cd ~/Documents/project/mysql-study
git clone --filter=blob:none https://github.com/mysql/mysql-server.git
cd mysql-server && git checkout mysql-8.0.42
```

`--filter=blob:none` 부분 클론이라 1.8GB 로 끝난다(전체 클론은 4.7GB). `git log`·`git blame` 은
그대로 되고 필요한 blob 만 그때그때 받아온다. 커밋 메시지에 WL#/BUG# 번호가 있어서
"이 코드가 왜 이렇게 생겼나"를 추적할 때 쓸모가 크다.

## 3. configure

```
cmake -S mysql-server -B build-debug -G Ninja -DWITH_DEBUG=1 -DCMAKE_BUILD_TYPE=Debug -DDOWNLOAD_BOOST=1 -DWITH_BOOST=$PWD/boost -DWITH_SSL=/opt/homebrew/opt/openssl@3 -DBISON_EXECUTABLE=/opt/homebrew/opt/bison/bin/bison -DWITH_UNIT_TESTS=OFF -DWITH_ROUTER=OFF -DWITH_FIDO=none -DMYSQL_MAINTAINER_MODE=OFF -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

172초 걸렸다(boost 1.77 다운로드 포함). 결과 플래그:

```
CMAKE_CXX_FLAGS_DEBUG: -DSAFE_MUTEX -DENABLED_DEBUG_SYNC -g
```

`ENABLED_DEBUG_SYNC` 가 붙었으니 DEBUG_SYNC 를 SQL 로 쓸 수 있고, `-g -O0` 이라 인라이닝이 없어
모든 함수에 브레이크포인트가 걸린다.

미리 걱정했던 것들은 실제로는 문제가 아니었다.

- **CMake 4.4.3** — MySQL 루트가 `CMAKE_MINIMUM_REQUIRED(3.11.2)` 라 4.x 의 하드 컷(3.5)에 안 걸린다.
  `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` 폴백은 필요 없었다.
- **Apple clang 21** — `MYSQL_MAINTAINER_MODE` 는 GCC 일 때만 기본 ON 이다. clang 에서는 OFF 라
  `-Werror` 가 안 붙고, 신형 컴파일러의 새 경고가 빌드를 깨지 않는다.
- **번들 protobuf 24.4** — 그대로 컴파일된다.

옵션을 이렇게 고른 이유:

| 옵션 | 이유 |
|---|---|
| `-DWITH_DEBUG=1` | DBUG 트레이스, DEBUG_SYNC, `*_debug` 시스템 변수, 디버그 전용 assert |
| `-DWITH_ROUTER=OFF -DWITH_UNIT_TESTS=OFF` | 학습에 안 쓰는데 빌드 시간의 상당 부분을 먹는다 |
| `-DWITH_FIDO=none` | 번들 libfido2 빌드를 통째로 건너뛴다 |
| `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` | clangd·CLion 인덱싱용 `compile_commands.json` |
| ccache launcher | 코드에 `DBUG_PRINT` 를 넣고 재빌드할 때 크게 절약된다 |

## 4. 빌드

```
ninja -C build-debug mysqld mysql mysqladmin
```

2,908 타깃, **7분 58초**(10코어 전부 사용). 결과는 `build-debug` 6.4GB, `mysqld` 174MB.

## 5. 인스턴스 초기화

```
tools/server.sh init
```

`run/my.cnf` 를 쓴다. 3306 은 Docker 가 점유 중이라 이 인스턴스는 **3307** 과 `run/mysqld.sock` 이다.
디버거에 오래 멈춰도 세션이 끊기지 않게 네트워크·락 타임아웃을 크게 잡아 놨고,
`--gdb`(시그널 핸들러 미설치)를 켜 뒀다.

## 6. 실제로 걸렸던 함정 두 가지

**DEBUG_SYNC 는 기동 옵션이 있어야 켜진다.** 빌드에 `-DENABLED_DEBUG_SYNC` 가 들어가도
`--debug-sync-timeout` 없이 띄우면 `@@debug_sync` 가 `OFF` 다. `my.cnf` 에
`debug-sync-timeout = 600` 을 넣어야 `ON - signals: ''` 이 된다.

**mysql 클라이언트는 주석을 떼고 보낸다.** `--comments` 없이 보내면
`SELECT /* STUDY */ 1` 이 서버에는 `SELECT 1` 로 도착한다. 마커 기반 arming 이 통째로
안 먹는다. `tools/sql.sh` 와 하네스 둘 다 `--comments` 를 붙여 놨다.

## 7. 동작 확인

```
tools/server.sh start && tools/sql.sh -e "SELECT VERSION()" && tools/server.sh stop
tools/dbg.sh recipes/index_selection.json sql/index_selection.sql idx1
```
