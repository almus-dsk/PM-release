# Payroll Manager 설치 및 PostgreSQL 서버 DB 구성 가이드

기준 버전은 2026-11-19 정식 배포된 **v0.94.6**입니다. Windows 사용자는 1단계에서 프로그램을 설치합니다. 사내 서버를 처음 구성하는 관리자는 서버 준비부터 진행하고, 이미 운영 중인 서버에 PC를 추가할 때는 관리자가 알려준 기존 DB명과 업무 스키마로 연결을 등록합니다.

## 1. Windows 프로그램 설치

1. [v0.94.6 배포 페이지](https://github.com/almus-dsk/PM-release/releases/tag/v0.94.6)에서 전체 패키지 **PayrollManager.0.94.6.20261119-win64.zip**을 받습니다. 이름에 `patch`가 들어간 ZIP은 기존 설치본의 자동 업데이트용입니다.
2. Windows 10 또는 11의 64비트 PC에서 쓰기 가능한 폴더에 ZIP 전체를 풉니다. `PayrollManager.exe`와 `_internal`, `app`, `resources` 폴더를 함께 둡니다. Python과 PostgreSQL 클라이언트 런타임은 패키지에 포함됩니다.
3. `PayrollManager.exe`를 실행해 DB 선택 화면을 확인합니다. 로컬 사용자는 **로컬 DB → 새 DB 생성**, 서버 사용자는 **서버 DB**를 선택합니다. 기존 설치를 수동으로 교체할 때는 먼저 프로그램을 종료하고 기존 회사 DB와 `data`, `user`, `credentials` 폴더를 백업합니다.

## 2. 서버와 업무 DB 준비

아래 예시는 Linux Docker의 PostgreSQL 17 컨테이너 `postgresql`, 대상 DB `dev_PM`, 앱 계정 `devpm`을 사용합니다. 실제 환경의 컨테이너명·DB명·계정명으로 바꾸세요. DB 데이터 디렉터리는 영구 볼륨에 두고, 서버 주소에는 서버 인증서의 DNS 이름을 사용합니다.

서버 관리자가 새 DB를 준비하는 경우 다음 명령으로 관리 콘솔을 엽니다.

```bash
docker exec -it postgresql psql -U postgres -d postgres
```

새 전용 계정과 DB를 만드는 예시입니다. 기존 DB를 사용하는 경우에는 기존 계정의 접속 권한과 업무 스키마 접근 권한을 확인합니다.

```sql
CREATE ROLE devpm LOGIN;
\password devpm
CREATE DATABASE "dev_PM" OWNER devpm;
```

PostgreSQL이 컨테이너의 접속 인터페이스에서 수신하는지, Docker 포트 매핑과 방화벽에서 허용된 클라이언트가 접근할 수 있는지 확인합니다. 외부 포트 15432를 내부 5432로 전달한다면 Windows 앱에는 15432를 입력합니다. 빈 DB에 처음 업무 테이블을 구성하는 절차는 아래의 **최초 이관**을 따릅니다.

Windows의 `sslmode=verify-full`은 클라이언트가 서버를 검증하는 설정이다. PostgreSQL은 클라이언트가 선택한 SSL 모드를 강제할 수 없다. 서버에서 인증서 없는 클라이언트를 거부하려면 서버도 클라이언트 인증서를 검증하는 상호 TLS를 구성한다. TLS 암호화만 필수로 할 때는 `hostssl`과 `hostnossl ... reject`로 충분하다.

## 3. 인증서 준비

- 클라이언트 인증서를 발급할 CA를 준비하고 공개 인증서를 컨테이너의 `/certs/client-ca.crt`에 배치한다. CA 개인키는 발급 환경에 보관한다.
- 그 CA로 `CN=devpm`, 용도 `clientAuth`의 클라이언트 인증서를 발급한다. 인증서와 해당 클라이언트 개인키를 Windows 앱 실행 계정에 전달하고 개인키 접근을 제한한다.
- Windows는 기존 서버 공개 인증서 `server.crt`로 서버를 검증한다. PostgreSQL은 별도로 `client-ca.crt`를 사용해 Windows가 제출하는 인증서를 검증한다.

## 4. 서버 TLS와 접속 규칙 설정

Docker 호스트에서 실제 설정 파일을 먼저 찾는다. 복구에 사용할 컨테이너 내부 관리자 소켓 접속을 확인하고 설정 파일 원본을 보관한다.

```bash
docker exec postgresql psql -U postgres -d postgres \
  -c 'SHOW config_file' -c 'SHOW hba_file' -c 'SHOW ssl_ca_file'
```

컨테이너 설정을 관리하는 호스트 파일 또는 Compose 설정에서 `ssl_ca_file`을 추가한다. 아래 경로는 컨테이너 내부 기준이다.

```conf
ssl = on
ssl_cert_file = '/certs/server.crt'
ssl_key_file = '/certs/server.key'
ssl_ca_file = '/certs/client-ca.crt'
```

`pg_hba.conf`의 기존 광범위한 TCP 허용 규칙보다 앞에 다음 테스트 전용 규칙을 둔다. `192.0.2.10/32`는 예시이므로 PostgreSQL 서버에서 실제로 보이는 Windows 클라이언트의 IP 또는 허용 CIDR로 바꾼다. IPv6를 허용하려면 같은 형식의 `hostssl` 허용 행을 해당 IPv6 거부 행보다 앞에 추가한다.

```conf
# TYPE      DATABASE USER   ADDRESS         METHOD
hostssl     dev_PM   devpm  192.0.2.10/32   scram-sha-256 clientcert=verify-full
host        dev_PM   devpm  0.0.0.0/0       reject
host        dev_PM   devpm  ::/0            reject
```

첫 규칙은 TLS, 신뢰된 클라이언트 인증서, 인증서 CN과 DB 계정 이름 일치, SCRAM 비밀번호를 요구한다. 뒤의 두 규칙은 이 DB·계정 조합의 나머지 TCP 접속을 차단한다. PostgreSQL은 첫 일치 규칙을 사용하므로 앞에 있는 `host all all ...` 허용 규칙도 점검한다. 이 예시는 테스트 DB의 앱 계정에만 적용한다. 전체 원격 계정에 적용할 때는 앱, 관리자, 백업, 모니터링의 필요한 접속 규칙을 먼저 작성하고 남은 TCP 허용 규칙을 점검한다. Unix 소켓 백업은 기존 `local` 규칙을 사용한다.

관리자 소켓 연결에서 오류를 확인한 뒤 설정을 다시 읽고 서버 로그를 확인한다.

```sql
SELECT sourcefile, sourceline, name, error FROM pg_file_settings WHERE error IS NOT NULL;
SELECT rule_number, file_name, line_number, error FROM pg_hba_file_rules WHERE error IS NOT NULL;
SELECT pg_reload_conf();
```

다시 읽기 요청의 반환값은 규칙 적용 성공을 보장하지 않는다. 새 연결로 아래 검증을 수행한다. 적용에 실패하거나 필요한 연결이 끊기면 관리자 소켓 연결로 원래 파일을 복원하고 다시 읽는다.

## 5. Windows 앱에서 연결 등록 및 최초 이관

1. **서버 DB → 서버 연결 등록…**을 누릅니다. 연결 이름, 서버 주소, 포트, 데이터베이스, DB 사용자를 입력합니다. **서버 신뢰 인증서**에는 서버 인증서를 검증할 신뢰 파일을 지정합니다. 상호 TLS를 구성했다면 **클라이언트 인증서**와 **클라이언트 개인키**에 해당 DB 사용자의 파일을 지정합니다. 관리자 PC에서는 **DB 사용자 인증서 발급…**도 사용할 수 있습니다.
2. **기존 업무 스키마 연결**은 관리자가 제공한 스키마 이름을 입력합니다. **최초 이관**은 **기존 로컬 DB를 새 업무 스키마로 이관**을 체크하고 원본 로컬 DB 파일을 선택합니다. 빈 업무 환경으로 시작한다면 먼저 앱에서 로컬 DB를 생성한 뒤 이관할 수 있습니다.
3. **서버 로그인 비밀번호**에는 PostgreSQL 계정 비밀번호를, **이관 원본 DB 비밀번호**에는 해당 업무 자료의 원본 로컬 DB 비밀번호를 입력합니다. 후자는 이관한 개인정보 필드를 여는 데도 사용합니다.
4. **연결 테스트**를 실행합니다. 이관 모드에서는 원본 구조와 대상 DB의 스키마 생성 권한을 확인합니다. 기존 업무 스키마 연결에서는 서버 인증과 업무 스키마 조회를 확인합니다.
5. **연결 설정 저장**을 누르고 이 PC에서 연결을 열 때 사용할 저장 비밀번호를 지정합니다. 최초 이관은 확인 후 새 업무 스키마를 생성합니다. 표시되는 스키마 이름을 기록해 두고, 다른 PC에서는 같은 이름으로 기존 스키마에 연결합니다.
6. DB 선택 화면에서 저장한 연결을 선택하고 **서버 DB 열기**를 누릅니다. 저장 비밀번호로 로그인한 뒤 인사정보에서 알려진 사원을 조회해 대상 DB가 맞는지 확인합니다.

## 6. 기존 연결 수정과 장애 확인

- 서버 주소·포트·계정·인증서가 달라졌다면 **서버 DB → 기존 연결 선택 → 선택한 연결 수정…**을 누릅니다. PC의 저장 비밀번호로 기존 값을 불러온 뒤 수정하고 **연결 테스트 → 연결 설정 저장**을 누릅니다. 기존 서버가 응답하지 않아도 설정을 열 수 있습니다.
- PC의 저장 비밀번호는 로그인 후 **시스템 → DB 비밀번호 변경**에서 변경합니다. PostgreSQL 계정 비밀번호를 서버에서 바꿨다면 각 PC의 **선택한 연결 수정…**에서 **서버 로그인 비밀번호**를 갱신합니다.
- 접속 실패 시 Windows PowerShell에서 아래 예시의 주소와 포트를 실제 값으로 바꿔 실행합니다. `TcpTestSucceeded`가 `False`이면 VPN·포트포워딩·방화벽·서버 수신 상태부터 확인합니다. `True`이면 앱의 연결 테스트 오류를 기준으로 인증서 이름과 유효기간, DB 계정, 업무 스키마 순으로 확인합니다.

```powershell
Test-NetConnection -ComputerName "db.example.com" -Port 15432
```

참고: [PostgreSQL 17 pg_hba.conf](https://www.postgresql.org/docs/17/auth-pg-hba-conf.html), [서버 TLS 설정](https://www.postgresql.org/docs/17/ssl-tcp.html), [libpq 인증서 검증](https://www.postgresql.org/docs/17/libpq-ssl.html).
