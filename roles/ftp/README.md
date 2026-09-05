# FTP Role

CentOS Stream 9의 `ftpservers`에 vsftpd를 설치하고 익명 다운로드용 디렉터리와 기본 파일을 배포한다.

## 적용 동작

[Task 진입점](tasks/main.yml)은 CentOS 검사·패키지 설치 → 서비스 기동·자동 시작 → 설정·파일 배포 → firewalld `ftp` 등록 순서다.

[설정 Task](tasks/03_config.yml)는 기존 `/etc/vsftpd/vsftpd.conf`를 `.OLD`로 한 번 보관하고 `lineinfile`로 다음 항목을 관리한다.

| 설정 | 값 | 역할 |
|---|---|---|
| `anonymous_enable` | `YES` | 익명 로그인 허용 |
| `local_enable` | `NO` | 로컬 사용자 로그인 차단 |
| `write_enable`, `anon_upload_enable`, `anon_mkdir_write_enable` | `NO` | 쓰기·업로드·디렉터리 생성 차단 |
| `dirmessage_enable`, `xferlog_enable` | `YES` | 디렉터리 메시지·전송 로그 |
| `connect_from_port_20` | `YES` | Active 모드 데이터 연결의 출발 포트 설정 |
| `listen`, `listen_ipv6` | `NO`, `YES` | 독립 실행 리스너 설정 |

공개 디렉터리는 root 소유 `0755`, 기본 파일은 `0644`로 생성한다. 기본 파일 변경 시 백업을 생성하며, 설정 또는 기본 파일이 변경되면 vsftpd를 한 번 재시작한다. Handler 대신 등록한 Task 결과의 `changed`를 사용한다.

## 변수

| [기본 변수](defaults/main.yml) | 기본값 | 의미 |
|---|---|---|
| `ftp_root_dir` | `/var/ftp` | 공개 디렉터리 기본 경로 계산에 사용 |
| `ftp_public_dir` | `{{ ftp_root_dir }}/pub` | 생성할 디렉터리 |
| `ftp_sample_file` | `README.txt` | 배포할 파일명 |
| `ftp_sample_content` | 호스트·IP·OS 정보 | 파일 내용 |
| `ftp_firewall_service` | `ftp` | firewalld 서비스 |
| `ftp_test_url` | `ftp://127.0.0.1/pub/{{ ftp_sample_file }}` | 서버 내부 점검 URL |
| `ftp_test_dest` | `/tmp/ansible_ftp_test_{{ ftp_sample_file }}` | 다운로드 저장 경로 |

[내부 변수](vars/main.yml)는 패키지·서비스 `vsftpd`와 설정 경로 `/etc/vsftpd/vsftpd.conf`를 정의한다.

`ftp_root_dir`·`ftp_public_dir`는 생성할 경로에만 반영된다. vsftpd의 `anon_root`나 FTP 계정 홈 경로를 수정하지 않는다. 다른 경로를 사용할 때는 서버의 실제 익명 루트와 SELinux Context를 별도로 맞춰야 한다. 테스트 URL의 `/pub/`도 디렉터리 변수에 따라 자동 변경되지 않는다.

## 실행과 검증

[공통 실행 조건](../../README.md#실행-조건)을 준비하고 저장소 루트에서 실행한다.

```bash
ansible-navigator run -m stdout playbook.yml --limit ftpservers
ansible-navigator run -m stdout roles/ftp/tests/test.yml
```

첫 명령은 FTP 노드에 firewall과 ftp를 적용한다. [테스트](tests/test.yml)는 ftp를 재적용한 뒤 **FTP 서버 자신에서** `get_url`로 파일을 내려받는다. 파일 내용에 대한 별도 Assertion이나 외부 클라이언트 접근 검사는 없다.

서버 내부와 별도 클라이언트에서 실행할 절차를 구분한다.

```bash
# FTP 서버에서
curl ftp://127.0.0.1/pub/README.txt
```

```bash
# 별도 클라이언트에서: 기본 Inventory 주소
curl ftp://192.168.10.13/pub/README.txt
```

기본 파일의 호스트·IP·OS 내용이 응답 판단 기준이다. 연결 실패 시 서버에서 다음을 확인한다.

```bash
sudo systemctl status vsftpd
sudo ss -lntup
sudo firewall-cmd --query-service=ftp
```

## 범위

firewalld 설치·기동은 사전 조건이며, OS 검사는 CentOS만 허용한다. 업로드, 인증 사용자 구성, FTPS/TLS, Passive 포트 범위와 NAT 설정, SELinux 변경은 포함하지 않는다. 서버 내부 다운로드 성공만으로 외부 FTP 데이터 연결까지 검증된 것으로 볼 수 없다.

작성자: tjung03 · [MIT License](LICENSE)
