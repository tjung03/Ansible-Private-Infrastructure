# FIREWALL Role

CentOS Stream 9 / RHEL 계열 9 환경에서 firewalld를 설치하고 기동하는 Ansible role이다.

이 role은 전체 managed node에 공통으로 적용되는 기반 role이다. 각 서비스 role이 firewalld service를 등록하기 전에, firewalld 패키지와 서비스 상태를 먼저 보장한다.

주요 작업 흐름은 다음과 같다.

```text
1) 패키지 설치
2) 서비스 기동
```

## Scope

이 role이 처리하는 범위는 다음과 같다.

```text
firewalld 패키지 설치
firewalld 서비스 기동
firewalld 서비스 부팅 시 자동 시작 설정
```

이 role이 처리하지 않는 범위는 다음과 같다.

```text
DNS 포트 등록
HTTP 포트 등록
FTP 포트 등록
SMTP 포트 등록
IMAP 포트 등록
개별 서비스별 방화벽 정책 관리
firewalld zone 세부 구성
rich rule 구성
port 직접 개방
SELinux 모드 변경
```

개별 서비스 포트 등록은 각 서비스 role에서 처리한다.

```text
dns role   -> firewalld dns 서비스 등록
web role   -> firewalld http 서비스 등록
ftp role   -> firewalld ftp 서비스 등록
mail role  -> firewalld smtp, imap 서비스 등록
```

## Requirements

대상 OS는 다음을 기준으로 한다.

```text
CentOS Stream 9
RHEL 계열 9
```

이 role은 Ansible built-in 모듈만 사용한다.

```text
ansible.builtin.dnf
ansible.builtin.systemd
ansible.builtin.fail
ansible.builtin.include_tasks
```

`ansible.posix.firewalld` 모듈은 이 role에서 사용하지 않는다.

`ansible.posix.firewalld` 모듈은 DNS, WEB, FTP, MAIL role에서 각 서비스 포트를 등록할 때 사용한다.

## Dependencies

이 role은 다른 role dependency를 선언하지 않는다.

```yaml
dependencies: []
```

이 role은 전체 인프라 playbook에서 가장 먼저 실행되는 것을 권장한다.

권장 실행 순서는 다음과 같다.

```text
1. firewall role
2. dns role
3. web role
4. ftp role
5. mail role
```

이렇게 구성하면 각 서비스 role이 firewalld service를 등록하기 전에 firewalld 설치와 기동이 먼저 보장된다.

## Quick Start

프로젝트 루트로 이동한다.

```bash
cd $HOME/private_infrastructure
```

inventory 예시는 다음과 같다.

```ini
[dnsservers]
ansible1.example.com ansible_host=192.168.10.11

[webservers]
ansible2.example.com ansible_host=192.168.10.12

[ftpservers]
ansible3.example.com ansible_host=192.168.10.13

[mailservers]
ansible4.example.com ansible_host=192.168.10.14
```

firewall role은 전체 managed node에 적용하는 것을 기본으로 한다.

playbook 예시는 다음과 같다.

```yaml
---
- name: FIREWALL 기본 구성
  hosts: all
  gather_facts: true

  tasks:
    - name: FIREWALL role 실행
      ansible.builtin.include_role:
        name: firewall
```

실행한다.

```bash
ansible-navigator run -m stdout playbook.yml
```

테스트를 실행한다.

```bash
ansible-navigator run -m stdout roles/firewall/tests/test.yml
```

## Directory Structure

권장 구조는 다음과 같다.

```text
$HOME/private_infrastructure/
├── ansible.cfg
├── ansible-navigator.yml
├── inventory
├── playbook.yml
└── roles/
    └── firewall/
        ├── defaults/
        │   └── main.yml
        ├── files/
        ├── handlers/
        │   └── main.yml
        ├── meta/
        │   └── main.yml
        ├── README.md
        ├── LICENSE
        ├── tasks/
        │   ├── 01_package.yml
        │   ├── 02_service.yml
        │   └── main.yml
        ├── templates/
        ├── tests/
        │   └── test.yml
        └── vars/
            └── main.yml
```

## Project Files

`ansible.cfg` 예시는 다음과 같다.

```ini
[defaults]
inventory = inventory
remote_user = ansible
roles_path = roles:/usr/share/ansible/roles:/etc/ansible/roles:/home/ansible/.ansible/roles

[privilege_escalation]
become = true
become_method = sudo
become_user = root
```

`ansible-navigator.yml` 예시는 다음과 같다. playbook artifact 파일이 남지 않도록 한다.

```yaml
---
ansible-navigator:
  playbook-artifact:
    enable: false
```

## Role Variables

사용자가 환경에 맞게 바꿀 수 있는 변수는 `defaults/main.yml`에 정의한다.

```yaml
---
firewall_enabled: true

firewall_started: true
```

변수 설명은 다음과 같다.

```text
firewall_enabled
  firewalld 서비스를 부팅 시 자동 시작하도록 설정할지 결정한다.
  기본값은 true 이다.

firewall_started
  firewalld 서비스를 현재 실행 상태로 만들지 결정한다.
  기본값은 true 이다.
```

role 내부에서 고정적으로 사용하는 값은 `vars/main.yml`에 정의한다. 일반적으로 사용자가 수정하지 않는다.

```yaml
---
firewall_packages:
  - firewalld

firewall_service_name: firewalld
```

## What To Edit

처음 사용하는 경우 보통 아래 파일만 수정하면 된다.

```text
inventory
playbook.yml
ansible-navigator.yml
```

firewalld를 설치하되 자동 시작하지 않으려면 다음처럼 설정한다.

```yaml
vars:
  firewall_enabled: false
```

firewalld를 설치하되 현재 실행하지 않으려면 다음처럼 설정한다.

```yaml
vars:
  firewall_started: false
```

일반적인 서버 구성에서는 두 값을 모두 `true`로 유지한다.

## Inventory Rule

이 role은 전체 managed node에 공통 적용하는 것을 기본 사용 방식으로 한다.

따라서 playbook에서는 보통 `hosts: all`을 사용한다.

```yaml
- name: FIREWALL 기본 구성
  hosts: all
  gather_facts: true

  tasks:
    - name: FIREWALL role 실행
      ansible.builtin.include_role:
        name: firewall
```

현재 프로젝트 기준 적용 대상은 다음과 같다.

```text
ansible1.example.com  DNS Server
ansible2.example.com  WEB Server
ansible3.example.com  FTP Server
ansible4.example.com  MAIL Server
```

## Generated Changes

기본 설정 기준으로 수행되는 변경은 다음과 같다.

```text
firewalld 패키지 설치
firewalld 서비스 started 상태 보장
firewalld 서비스 enabled 상태 보장
```

이 role은 개별 서비스 포트를 등록하지 않는다.

서비스별 방화벽 등록은 각 role에서 처리한다.

```text
roles/dns/tasks/04_firewall.yml
roles/web/tasks/04_firewall.yml
roles/ftp/tasks/04_firewall.yml
roles/mail/tasks/04_firewall.yml
```

## Recommended Playbook Order

전체 인프라 playbook 권장 순서는 다음과 같다.

```yaml
---
- name: FIREWALL 기본 구성
  hosts: all
  gather_facts: true

  tasks:
    - name: FIREWALL role 실행
      ansible.builtin.include_role:
        name: firewall

- name: DNS 서버 구성
  hosts: dnsservers
  gather_facts: true

  tasks:
    - name: DNS role 실행
      ansible.builtin.include_role:
        name: dns

- name: WEB 서버 구성
  hosts: webservers
  gather_facts: true

  tasks:
    - name: WEB role 실행
      ansible.builtin.include_role:
        name: web

- name: FTP 서버 구성
  hosts: ftpservers
  gather_facts: true

  tasks:
    - name: FTP role 실행
      ansible.builtin.include_role:
        name: ftp

- name: MAIL 서버 구성
  hosts: mailservers
  gather_facts: true

  tasks:
    - name: MAIL role 실행
      ansible.builtin.include_role:
        name: mail
```

이 순서에서는 firewalld가 모든 서버에서 먼저 설치 및 기동된다.

그 다음 각 서비스 role이 필요한 firewalld service를 등록한다.

## Run

프로젝트 루트에서 실행한다.

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout playbook.yml
```

문법만 확인하려면 다음 명령을 사용한다.

```bash
ansible-navigator run -m stdout playbook.yml --syntax-check
```

firewall role test 문법만 확인하려면 다음 명령을 사용한다.

```bash
ansible-navigator run -m stdout roles/firewall/tests/test.yml --syntax-check
```

inventory를 확인하려면 다음 명령을 사용한다.

```bash
ansible-navigator inventory -m stdout --list
```

## Expected Result

정상 실행 시 대략 다음 흐름으로 출력된다.

```text
PLAY [FIREWALL 기본 구성]

TASK [Gathering Facts]
ok: [ansible1.example.com]
ok: [ansible2.example.com]
ok: [ansible3.example.com]
ok: [ansible4.example.com]

TASK [FIREWALL role 실행]

TASK [firewall : 1) 패키지 설치 작업 포함]
included: .../roles/firewall/tasks/01_package.yml for ansible1.example.com, ansible2.example.com, ansible3.example.com, ansible4.example.com

TASK [firewall : 1-0) CentOS 환경 확인]
skipping: [ansible1.example.com]
skipping: [ansible2.example.com]
skipping: [ansible3.example.com]
skipping: [ansible4.example.com]

TASK [firewall : 1-1) firewall 패키지 설치 - firewalld]
ok: [ansible1.example.com]
ok: [ansible2.example.com]
ok: [ansible3.example.com]
ok: [ansible4.example.com]

TASK [firewall : 2) 서비스 기동 작업 포함]
included: .../roles/firewall/tasks/02_service.yml for ansible1.example.com, ansible2.example.com, ansible3.example.com, ansible4.example.com

TASK [firewall : 2-1) firewalld 서비스 기동 및 활성화]
ok: [ansible1.example.com]
ok: [ansible2.example.com]
ok: [ansible3.example.com]
ok: [ansible4.example.com]

PLAY RECAP
ansible1.example.com : ok=..., changed=..., unreachable=0, failed=0
ansible2.example.com : ok=..., changed=..., unreachable=0, failed=0
ansible3.example.com : ok=..., changed=..., unreachable=0, failed=0
ansible4.example.com : ok=..., changed=..., unreachable=0, failed=0
```

처음 실행하면 패키지 설치나 서비스 기동으로 인해 일부 task가 `changed`로 표시될 수 있다. 이미 적용된 상태에서 다시 실행하면 많은 task가 `ok`로 표시된다.

출력 상태의 의미는 다음과 같다.

```text
ok
  작업을 확인했지만 변경할 내용이 없었음

changed
  대상 시스템에 실제 변경이 발생했음

skipping
  when 조건에 따라 실행하지 않음

failed
  작업 실패

unreachable
  SSH 접속 실패 또는 대상 노드 접근 불가
```

## Test

test playbook 위치는 다음과 같다.

```text
roles/firewall/tests/test.yml
```

실행 명령은 다음과 같다.

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/firewall/tests/test.yml
```

테스트는 firewall role을 전체 managed node에 다시 적용한다.

firewalld 상태를 수동으로 확인하려면 다음 명령을 사용한다.

```bash
ansible all -m shell -a "systemctl is-active firewalld && systemctl is-enabled firewalld"
```

정상 예시는 다음과 같다.

```text
active
enabled
```

이 수동 확인 명령은 `shell` 모듈을 사용한다. role 코드에서는 사용하지 않는다.

서비스별 등록 상태는 각 서버에서 다음처럼 확인할 수 있다.

DNS 서버:

```bash
ssh ansible1.example.com "sudo firewall-cmd --query-service=dns"
```

WEB 서버:

```bash
ssh ansible2.example.com "sudo firewall-cmd --query-service=http"
```

FTP 서버:

```bash
ssh ansible3.example.com "sudo firewall-cmd --query-service=ftp"
```

MAIL 서버:

```bash
ssh ansible4.example.com "sudo firewall-cmd --query-service=smtp"
ssh ansible4.example.com "sudo firewall-cmd --query-service=imap"
```

정상인 경우 `yes`가 출력된다.

## SELinux

이 role은 SELinux 모드를 변경하지 않는다.

firewalld 설치와 기동은 SELinux 모드와 별개로 처리된다. 개별 서비스 접근 문제가 발생하는 경우에는 각 서비스 role의 SELinux 관련 주의사항을 확인해야 한다.

## Troubleshooting

firewalld 서비스 상태는 다음 명령으로 확인한다.

```bash
sudo systemctl status firewalld
```

firewalld가 실행 중인지 확인한다.

```bash
sudo systemctl is-active firewalld
```

firewalld가 부팅 시 자동 시작되도록 설정되었는지 확인한다.

```bash
sudo systemctl is-enabled firewalld
```

firewalld zone 상태는 다음 명령으로 확인한다.

```bash
sudo firewall-cmd --get-active-zones
```

등록된 서비스 목록은 다음 명령으로 확인한다.

```bash
sudo firewall-cmd --list-services
```

영구 설정 기준 서비스 목록은 다음 명령으로 확인한다.

```bash
sudo firewall-cmd --permanent --list-services
```

firewalld 패키지 설치 여부는 다음 명령으로 확인한다.

```bash
rpm -q firewalld
```

## Tested Environment

작성 및 테스트 기준 환경은 다음과 같다.

```text
Host OS: Windows 10
Virtualization: VMware
Network: VMnet8 NAT
Subnet: 192.168.10.0/24
Gateway: 192.168.10.2
External DNS: 8.8.8.8

Control Node:
  ansible.example.com
  192.168.10.10

Managed Nodes:
  ansible1.example.com 192.168.10.11 DNS Server
  ansible2.example.com 192.168.10.12 WEB Server
  ansible3.example.com 192.168.10.13 FTP Server
  ansible4.example.com 192.168.10.14 MAIL Server

OS:
  CentOS Stream 9

Ansible User:
  ansible

Privilege:
  ansible user is in wheel group
  passwordless SSH is configured
  sudo privilege is configured through ansible.cfg

SELinux:
  permissive
```

## License

MIT. See `LICENSE`.

## Author Information

tjung03
