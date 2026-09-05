# MAIL Role

CentOS Stream 9의 `mailservers`에 Postfix·Dovecot을 설치하고 SMTP·IMAP 및 Maildir 저장 경로를 구성한다. 메일 사용자 계정과 DNS 레코드는 생성하지 않는다.

## 구성과 DNS 관계

[Task 진입점](tasks/main.yml)은 CentOS 검사·패키지 설치 → 두 서비스 기동·자동 시작 → 설정 → firewalld `smtp`·`imap` 등록 순서다.

[DNS Role](../dns/README.md)은 `mailservers` 그룹을 참조해 MX 레코드를 생성한다. 기본 Inventory의 메일 대상은 `ansible4.example.com`이고 주소는 `192.168.10.14`다.

메일 서버 설정 자체는 DNS Role 실행 없이도 적용할 수 있다. 도메인을 통한 메일 대상 탐색에는 DNS 레코드와 Resolver 설정이 필요하다. 전체 Playbook은 DNS 노드 구성 후 MAIL 노드를 구성하며, 클라이언트 Resolver는 변경하지 않는다. `meta/main.yml`의 Role dependency는 비어 있다.

## 관리하는 설정

[설정 Task](tasks/03_config.yml)는 메일 도메인이 비어 있으면 실패한다. Postfix `main.cf`와 Dovecot `dovecot.conf`는 `.OLD`로 한 번 보관하며, 기존 백업을 덮어쓰지 않는다.

| 파일 | 관리 항목 |
|---|---|
| `/etc/postfix/main.cf` | `myhostname`, `mydomain`, `myorigin = $mydomain`, `inet_interfaces = all`, `inet_protocols = ipv4`, `mydestination`, `home_mailbox` |
| `/etc/dovecot/dovecot.conf` | `protocols = imap` |
| `/etc/dovecot/conf.d/10-mail.conf` | `mail_location` |
| `/etc/dovecot/conf.d/10-auth.conf` | `disable_plaintext_auth = no`, `auth_mechanisms = plain login` |

파일 전체를 재생성하지 않고 `lineinfile`로 해당 항목을 관리한다. Postfix 설정 변경 시 Postfix만, Dovecot 관련 설정 변경 시 Dovecot만 재시작한다. 두 `conf.d` 파일에는 위의 `.OLD` 백업 처리가 없으며, 서비스 재시작 전에 설정 검사·자동 Rollback을 수행하는 Task도 없다.

## 변수

| [기본 변수](defaults/main.yml) | 기본값 | 의미 |
|---|---|---|
| `mail_domain` | Inventory 호스트명의 첫 점 뒤 부분 | 기본 `example.com`; 빈 값이면 실패 |
| `mail_hostname` | `{{ inventory_hostname }}` | Postfix 호스트명 |
| `mail_home_mailbox` | `Maildir/` | Postfix의 홈 디렉터리 기준 저장 경로 |
| `mail_location` | `maildir:~/Maildir` | Dovecot 조회 경로 |
| `mail_firewall_services` | `[smtp, imap]` | firewalld 등록 대상 |

[내부 변수](vars/main.yml)는 패키지·서비스 목록과 위 네 설정 파일의 경로를 정의한다.

Postfix 저장 경로와 Dovecot 조회 경로는 같은 Maildir을 가리켜야 한다. `mail_hostname`을 변경해도 DNS Role의 MX 대상은 바뀌지 않으므로 Inventory·DNS와 일치시켜야 한다. 방화벽 서비스 목록에 값을 추가하는 것만으로 TLS나 새로운 메일 리스너가 구성되지는 않는다.

## 실행과 확인

[공통 실행 조건](../../README.md#실행-조건)을 준비하고 저장소 루트에서 실행한다.

```bash
ansible-navigator run -m stdout playbook.yml --limit mailservers
ansible-navigator run -m stdout roles/mail/tests/test.yml
```

첫 명령은 MAIL 노드에 firewall과 mail을 적용하며 DNS 노드는 변경하지 않는다. 전체 구성과 DNS 갱신이 필요하면 루트 `playbook.yml`을 제한 없이 실행한다.

[테스트 Playbook](tests/test.yml)은 mail을 재적용한 뒤 MAIL 노드에서 `/usr/sbin/postfix check`와 `/usr/bin/doveconf -n`을 실행한다. 명령 실패는 Task 실패로 처리하지만 출력 내용에 대한 별도 판정은 없다. 실제 메일 송수신·사용자 인증 테스트는 포함하지 않는다.

MAIL 서버에서 점검하는 절차:

```bash
sudo postfix check
sudo doveconf -n
sudo systemctl status postfix
sudo systemctl status dovecot
sudo ss -lntup
```

기본 Inventory의 MX를 질의하는 절차:

```bash
dig +short @192.168.10.11 example.com MX
```

MX 기대값은 `10 ansible4.example.com.`이다. 설정 명령의 출력과 서비스 상태·리스너를 함께 확인하며, 이를 실제 사용자 메일 전달 성공으로 간주하지 않는다.

## Dovecot 2.4 호환성

이 Role이 쓰는 설정에는 Dovecot 2.4에서 변경된 항목이 있다. [공식 이행 문서](https://doc.dovecot.org/2.4.0/installation/upgrade/2.3-to-2.4.html)는 2.3 설정을 변경 없이 사용할 수 없다고 명시한다.

| 현재 코드의 항목 | 2.4의 변경 | 영향 |
|---|---|---|
| `disable_plaintext_auth = no` | `auth_allow_cleartext`로 대체. 기존 허용 의미는 `yes`에 대응 | 기존 `no` 값을 그대로 복사하지 않는다. TLS 없는 인증을 권장한다는 의미는 아니다 |
| `mail_location = maildir:~/Maildir` | `mail_driver = maildir`, `mail_path = ~/Maildir` 등으로 분리 | 저장 형식·경로 대응이며, 전체 설정 이행과 실행 검증은 별도 |
| 기존 기본 설정 파일에 항목 추가 | `dovecot_config_version`, `dovecot_storage_version` 등 필수 설정과 구문 변경 | 배포판 기본 파일까지 함께 검토해야 함 |

대체 항목의 의미는 [Dovecot 2.4 설정 목록](https://doc.dovecot.org/2.4.0/core/summaries/settings.html)의 `auth_allow_cleartext`, `mail_driver`, `mail_path`를 기준으로 한다.

확인일은 2026-09-05다. 실제 설치된 Dovecot 버전은 저장소만으로 확정할 수 없으며, CentOS 패키지가 곧 2.4라는 의미는 아니다. 기존 구현을 유지한 상태로 2.4 이식 시 필요한 검토 범위를 설명한다.

## 범위와 적용 조건

- OS 검사는 CentOS만 허용하며, firewalld는 사전에 설치·기동되어 있어야 한다.
- 사용자 계정 생성, SMTP 인증 세부 구성, TLS 인증서, 외부 Relay, 스팸·바이러스 필터는 포함하지 않는다.
- `disable_plaintext_auth = no`와 `plain login`을 설정한다. 인증정보를 보호하는 TLS 구성을 제공하지 않으므로 인터넷 공개용 메일 구성으로 사용하기 위한 조건을 충족한 것으로 볼 수 없다.
- SELinux 모드·Context는 변경하지 않는다. 기존 실습 환경은 permissive이며, enforcing에서의 사용자 Maildir 접근은 별도 확인 대상이다.
- Postfix·Dovecot 패키지 버전은 고정되어 있지 않다. 배포판 패키지의 설정 형식과 기본값이 달라지는 환경은 별도 호환성 확인이 필요하다.

작성자: tjung03 · [MIT License](LICENSE)
