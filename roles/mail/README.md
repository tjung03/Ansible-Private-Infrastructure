# MAIL Role

CentOS Stream 9의 `mailservers`에 Postfix·Dovecot을 설치하고 SMTP·IMAP 및 Maildir 저장 경로를 구성한다.

## 구성과 DNS 관계

[Task 진입점](tasks/main.yml)은 CentOS 검사·패키지 설치 → 두 서비스 기동·자동 시작 → 설정 → firewalld `smtp`·`imap` 등록 순서다.

[DNS Role](../dns/README.md)은 `mailservers` 그룹을 참조해 MX 레코드를 생성한다. 기본 Inventory의 메일 대상은 `ansible4.example.com`이고 주소는 `192.168.10.14`다.

전체 Playbook은 DNS 노드 구성 후 MAIL 노드를 구성한다. 도메인의 MX 레코드는 메일을 전달할 서버를 가리키며, DNS 조회에는 Resolver 설정이 필요하다.

## 관리하는 설정

[설정 Task](tasks/03_config.yml)는 메일 도메인이 비어 있으면 실패한다. Postfix `main.cf`와 Dovecot `dovecot.conf`는 `.OLD`로 한 번 보관한다.

| 파일 | 관리 항목 |
|---|---|
| `/etc/postfix/main.cf` | `myhostname`, `mydomain`, `myorigin = $mydomain`, `inet_interfaces = all`, `inet_protocols = ipv4`, `mydestination`, `home_mailbox` |
| `/etc/dovecot/dovecot.conf` | `protocols = imap` |
| `/etc/dovecot/conf.d/10-mail.conf` | `mail_location` |
| `/etc/dovecot/conf.d/10-auth.conf` | `disable_plaintext_auth = no`, `auth_mechanisms = plain login` |

`lineinfile`로 해당 설정 항목을 관리한다. Postfix 설정 변경 시 Postfix만, Dovecot 관련 설정 변경 시 Dovecot만 재시작한다.

## 변수

| [기본 변수](defaults/main.yml) | 기본값 | 의미 |
|---|---|---|
| `mail_domain` | Inventory 호스트명의 첫 점 뒤 부분 | 기본 `example.com`; 빈 값이면 실패 |
| `mail_hostname` | `{{ inventory_hostname }}` | Postfix 호스트명 |
| `mail_home_mailbox` | `Maildir/` | Postfix의 홈 디렉터리 기준 저장 경로 |
| `mail_location` | `maildir:~/Maildir` | Dovecot 조회 경로 |
| `mail_firewall_services` | `[smtp, imap]` | firewalld 등록 대상 |

[내부 변수](vars/main.yml)는 패키지·서비스 목록과 위 네 설정 파일의 경로를 정의한다.

Postfix는 사용자 홈의 `Maildir/`에 저장하고 Dovecot은 같은 경로를 조회한다. `mail_hostname`은 Inventory·DNS의 MX 대상과 일치시킨다. 인증 설정은 사설 실습망에서 평문 인증을 허용하는 `disable_plaintext_auth = no`, `plain login`이다.

## 실행과 확인

[공통 실행 조건](../../README.md#실행-조건)을 준비하고 저장소 루트에서 실행한다.

```bash
ansible-navigator run -m stdout playbook.yml --limit mailservers
ansible-navigator run -m stdout roles/mail/tests/test.yml
```

첫 명령은 MAIL 노드에 firewall과 mail을 적용한다. 전체 구성과 DNS 갱신이 필요하면 루트 `playbook.yml`을 제한 없이 실행한다.

[테스트 Playbook](tests/test.yml)은 mail을 재적용한 뒤 MAIL 노드에서 `/usr/sbin/postfix check`와 `/usr/bin/doveconf -n`을 실행한다. Postfix 설정을 점검하고 Dovecot의 적용 설정을 출력한다.

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

MX 기대값은 `10 ansible4.example.com.`이다. 설정 명령의 출력과 서비스 상태·리스너를 함께 확인한다.

## Dovecot 2.4 호환성

이 Role의 Dovecot 설정은 2.3 계열 형식이다. 2.4에서는 다음 항목과 설정 구문이 변경되었다. 공식 문서 확인일은 2026-09-05다.

| 저장소의 설정 | Dovecot 2.4 대응 항목 |
|---|---|
| `disable_plaintext_auth = no` | `auth_allow_cleartext = yes` — 평문 인증 허용 |
| `mail_location = maildir:~/Maildir` | `mail_driver = maildir`, `mail_path = ~/Maildir` — 저장 형식과 경로 분리 |

2.4의 전체 설정 이행에는 `dovecot_config_version`, `dovecot_storage_version` 등 필수 항목과 구문 변경도 적용된다. 세부 내용은 [공식 2.3 → 2.4 이행 문서](https://doc.dovecot.org/2.4.0/installation/upgrade/2.3-to-2.4.html)와 [2.4 설정 목록](https://doc.dovecot.org/2.4.0/core/summaries/settings.html)에서 확인할 수 있다.

작성자: tjung03 · [MIT License](LICENSE)
