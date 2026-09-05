# Ansible로 구성하는 사설 서버 인프라

**서버마다 반복하는 설치·설정 작업을 하나의 Playbook으로 묶은 프로젝트다.**
준비된 CentOS Stream 9 서버 4대에 DNS, 웹, 파일 다운로드, 메일 서비스를 각각 구성한다.

관리자는 [Inventory](inventory)에 대상 서버를 정의하고 [Playbook](playbook.yml)을 실행한다.
Ansible은 SSH로 접속해 패키지 설치부터 서비스 기동, 설정 파일 배포, 방화벽 등록까지 처리한다.

## 무엇을 구성하는가

![Ansible 제어 노드가 SSH와 sudo로 DNS·WEB·FTP·MAIL 서버 4대를 구성하는 구조](docs/images/infrastructure.png)

| 서버 | 구성되는 기능 | 구현 기술 |
|---|---|---|
| DNS | 서버 이름을 IP로 조회하고 메일 대상 서버를 찾는 A·MX 레코드 생성 | BIND · Jinja2 |
| WEB | 호스트·IP·OS 정보를 표시하는 기본 웹 페이지 배포 | Apache HTTP Server |
| FTP | 공개 디렉터리의 파일을 익명으로 다운로드하는 서비스 구성 | vsftpd |
| MAIL | 메일 전달·조회 서비스와 사용자 홈 기준 Maildir 경로 설정 | Postfix · Dovecot |

VM 생성·OS 설치·SSH 접속 준비는 사전 작업이다. 메일 사용자 생성과 실제 송수신 검증은 포함하지 않는다.

## 주요 구현

- **공통 기반과 서비스 구성 분리:** `firewall`이 firewalld 설치·기동을 담당하고, 각 서비스 Role이 필요한 firewalld 서비스를 영구 설정과 실행 중 설정에 등록한다.
- **Inventory 기반 DNS 구성:** Jinja2로 Zone·Include·named 설정을 생성하고, Zone과 named 설정은 배포 시 각각 `named-checkzone`, `named-checkconf`로 검사한다.
- **변경 결과에 따른 재시작:** DNS·FTP·MAIL은 Task의 `changed` 결과를 모아 해당 서비스의 재시작 여부를 결정한다. WEB은 설정 파일을 수정하지 않고 기본 페이지를 배포한다.
- **기본 변수와 내부 변수 분리:** `defaults/main.yml`에는 환경별 조정값을, `vars/main.yml`에는 패키지·서비스명 등 Role 내부 값을 정의한다.

## 자동화가 실행되는 순서

1. **공통 준비:** 모든 대상 서버에 `firewall` Role을 적용해 firewalld를 설치·기동한다.
2. **서비스 구성:** `dns → web → ftp → mail` 순서로 각 Inventory 그룹에 해당 Role을 적용한다.
3. **동작 확인:** 구성 후 필요한 Role별 테스트를 별도로 실행한다. 전체 Playbook이 테스트까지 자동 실행하지는 않는다.

DNS Role은 Inventory의 주소와 `mailservers` 그룹으로 레코드를 만들고, MAIL Role은 메일 서버 설정을 담당한다.
서로 다른 서버의 작업 순서는 상위 Playbook이 관리한다. 클라이언트 DNS 설정은 별도 준비가 필요하다.

## 실행 조건

기존 실습 환경은 Windows 10·VMware VMnet8 NAT, `192.168.10.0/24`, Control Node `192.168.10.10`, SELinux permissive로 기록되어 있다. VM 생성·OS 설치·SSH 계정 준비·SELinux 설정은 이 Playbook의 범위 밖이다.

- 대상 노드: CentOS Stream 9. 코드의 OS 검사는 `ansible_distribution == "CentOS"`이며, RHEL·Rocky·AlmaLinux까지 허용하지 않는다.
- Control Node: Ansible·ansible-navigator 설치 및 대상 노드로의 SSH 연결이 필요하다.
- 접속·권한: [ansible.cfg](ansible.cfg)의 기본 계정은 `ansible`, 권한 상승은 sudo/root이다. SSH 인증과 sudo 사용 조건을 사전에 준비한다.
- 의존성: 서비스 Role은 `ansible.posix.firewalld`를 사용한다. 실제 Ansible 실행 환경에서 `ansible.posix`를 사용할 수 있어야 한다.
- 버전: Role metadata의 최소 Ansible 표기는 `2.14`다. 실행 버전·Collection·패키지 버전을 고정하는 파일은 없으며, 모든 최신 조합의 호환성을 보장하지 않는다.

ansible-navigator의 Execution Environment 이미지·활성화 여부는 저장소에서 지정하지 않는다. 별도 Execution Environment를 사용한다면 그 안의 Collection과 SSH 접근 조건도 준비해야 한다.

## 실행

**최신 환경에서 재현하기 전에는 아래 [버전과 호환성](#버전과-호환성)을 확인한다.**

저장소 루트에서 실행한다. 먼저 Inventory의 주소·FQDN과 접속 계정을 실제 실습 환경에 맞춘다. 기본 DNS·MAIL 도메인은 Inventory 호스트명의 도메인 부분에서 계산한다.

```bash
ansible-galaxy collection install ansible.posix
ansible-navigator inventory -m stdout --list
ansible-navigator run -m stdout playbook.yml --syntax-check
ansible-navigator run -m stdout playbook.yml
```

위 명령은 구성 절차다. `--syntax-check`만으로 동적 Role 내부의 모든 실행 경로나 원격 서비스 동작이 검증되지는 않는다.

특정 그룹에만 적용하는 예:

```bash
ansible-navigator run -m stdout playbook.yml --limit webservers
```

이 경우 WEB 노드에 `firewall`과 `web`이 적용된다. `--limit mailservers`는 DNS 노드의 레코드를 갱신하지 않는다.

## 검증 범위

저장소에는 아래 테스트 Playbook이 있다. **모두 해당 Role을 다시 적용하므로 읽기 전용 점검이 아니다.** 서비스별 테스트는 firewalld가 이미 설치·기동되어 있어야 한다.

| 테스트 | 코드에 포함된 확인 | 확인하지 않는 범위 |
|---|---|---|
| [firewall](roles/firewall/tests/test.yml) | Role 재적용 | 별도 상태 검증 Task 없음 |
| [dns](roles/dns/tests/test.yml) | DNS 노드에서 A 응답에 Inventory IP가 포함되는지 검사 | MX 자동 검사·클라이언트 DNS 설정 |
| [web](roles/web/tests/test.yml) | WEB 노드에서 HTTP 200 검사 | 외부 클라이언트 접근·페이지 내용 검증 |
| [ftp](roles/ftp/tests/test.yml) | FTP 노드에서 기본 파일 다운로드 | 외부 경로·업로드 |
| [mail](roles/mail/tests/test.yml) | `postfix check`, `doveconf -n` 실행 | 실제 송수신·사용자 인증·인터넷 전달 |

예를 들어 전체 구성 후 DNS 테스트는 다음과 같이 실행한다.

```bash
ansible-navigator run -m stdout roles/dns/tests/test.yml
```

테스트 코드와 실행 절차는 존재하지만, 저장소에는 성공을 입증하는 실행 로그나 정량 측정 결과가 포함되어 있지 않다. 서비스 동작과 재실행 결과는 대상 환경에서 별도로 확인해야 한다.

## 버전과 호환성

공식 문서 확인일: **2026-09-05**. 아래는 기존 코드의 재현·이식에 영향을 주는 사항이며, 최신 버전으로 수정하거나 실행 검증한 결과는 아니다.

| 저장소에서 사용하는 항목 | 현재 공식 기준·대체 항목 | 적용 판단 |
|---|---|---|
| metadata의 최소 Ansible `2.14` | Core 2.14는 2024-05-20 지원 종료. 재현 시 유지보수 중인 Core와 지원 Python 조합 확인 | 실제 설치 버전을 뜻하지 않으며, 새 실행 환경을 2.14로 맞추라는 의미도 아니다. |
| 버전을 지정하지 않은 `ansible.posix` | 확인한 공식 문서의 `2.2.2`는 Core `2.16+` 요구 | 최소 요구사항 충족과 현재 지원 여부는 별개다. Core 2.14와 최신 Collection을 그대로 조합하지 않는다. |
| `ansible.builtin.systemd` | 현재 구현 모듈은 `ansible.builtin.systemd_service`; 기존 이름도 유효한 별칭 | 삭제·실행 불가로 분류하지 않는다. |
| Dovecot `disable_plaintext_auth` | 2.4의 `auth_allow_cleartext` | 단순 이름 교체 시 허용·금지 의미가 뒤집히지 않도록 확인한다. |
| Dovecot `mail_location` | 2.4의 `mail_driver`·`mail_path` 등 | MAIL Role의 설정 이식이 필요하다. [설정별 대응](roles/mail/README.md#dovecot-24-호환성)을 확인한다. |

근거: [Ansible 유지보수 정책](https://docs.ansible.com/projects/ansible/latest/reference_appendices/release_and_maintenance.html), [ansible.posix 요구 버전](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/index.html), [Dovecot 2.3 → 2.4](https://doc.dovecot.org/2.4.0/installation/upgrade/2.3-to-2.4.html), [systemd 별칭](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/systemd_module.html).

서비스 패키지는 `state: present`로 설치하며 버전을 고정하지 않는다. 따라서 실제 배포판 패키지 버전과 벤더의 유지보수 정책을 함께 확인해야 한다. 위 항목은 확인된 주요 차이이며, 전체 의존성에 대한 최신 버전 호환성 인증은 아니다.

## 적용 범위와 제한

이 구성은 **사설 실습망의 기본 서비스 구성**을 대상으로 한다. HA, VM Provisioning, TLS 인증서 관리, 메일 사용자 생성, 인터넷 메일 전달 구성은 포함하지 않는다.

DNS 템플릿은 `listen-on`·`allow-query`를 `any`로 설정하고 재귀 질의를 활성화한다. FTP는 익명 다운로드를 허용하며 MAIL은 `disable_plaintext_auth = no`를 설정한다. 인터넷 공개용 보안 구성을 제공하는 저장소로 해석해서는 안 된다.

WEB·FTP의 디렉터리 변수만 변경해도 서버의 DocumentRoot·익명 FTP 루트까지 바뀌는 것은 아니다. 경로 변경과 SELinux 적용 조건은 각 Role 문서에 설명한다.

## 상세 문서와 코드

공통 실행 조건은 이 README에서, 변수·적용 파일·점검 방법은 기존 Role 문서에서 다룬다.

| Role 문서 | 상세 내용 |
|---|---|
| [FIREWALL](roles/firewall/README.md) | 공통 서비스 상태와 서비스별 방화벽 등록의 경계 |
| [DNS](roles/dns/README.md) | Zone 생성·검사·Serial·Inventory 제약 |
| [WEB](roles/web/README.md) | 기본 페이지 관리·문서 경로·HTTP 점검 |
| [FTP](roles/ftp/README.md) | 익명 다운로드 설정·파일 배포·테스트 범위 |
| [MAIL](roles/mail/README.md) | Postfix·Dovecot 설정·DNS 관계·메일 검증 범위 |

### 저장소 구조

주요 파일과 디렉터리만 표시했다. 각 Role의 `tasks/main.yml`이 세부 Task를 불러온다.

```text
.
├── README.md
├── inventory                 # 서비스별 대상 서버
├── playbook.yml              # 전체 적용 순서
├── ansible.cfg               # 접속 계정·sudo·Role 경로
├── ansible-navigator.yml     # 실행 결과 파일 옵션
├── roles/
│   ├── firewall/             # 공통 firewalld 설치·기동
│   ├── dns/                  # BIND 구성, templates/ 포함
│   ├── web/                  # Apache 기본 페이지
│   ├── ftp/                  # 익명 다운로드
│   └── mail/                 # Postfix·Dovecot
└── docs/images/              # README 구성도
```

각 Role은 `defaults/`(조정값), `vars/`(내부 값), `tasks/`(적용), `tests/`(재적용·점검), `README.md`(상세 설명)를 중심으로 구성된다.

작성자: tjung03. 각 Role에는 MIT [LICENSE](roles/dns/LICENSE)가 포함되어 있다.
