# MAIL Role

CentOS Stream 9 / RHEL 계열 9 환경에서 Postfix와 Dovecot 기반 MAIL 서버를 구성하는 Ansible role이다.

이 role은 `mailservers` 그룹의 대상 노드에 `postfix`, `dovecot`을 설치하고, SMTP와 IMAP 서비스를 기동하며, 기본적인 Maildir 기반 메일 송수신 환경을 구성한다.

DNS의 A 레코드와 MX 레코드는 이 role에서 직접 생성하지 않는다. DNS 레코드는 `dns` role에서 처리한다.

주요 작업 흐름은 다음과 같다.

```text
1) 패키지 설치
2) 서비스 기동
3) 서비스 설정
4) 방화벽 등록
```

## Scope

이 role이 처리하는 범위는 다음과 같다.

```text
postfix 패키지 설치
dovecot 패키지 설치
postfix 서비스 기동 및 활성화
dovecot 서비스 기동 및 활성화
/etc/postfix/main.cf 원본 1회 백업
/etc/dovecot/dovecot.conf 원본 1회 백업
Postfix 기본 도메인 설정
Postfix Maildir 설정
Dovecot IMAP 설정
Dovecot Maildir 설정
firewalld smtp 서비스 등록
firewalld imap 서비스 등록
```

이 role이 처리하지 않는 범위는 다음과 같다.

```text
firewalld 설치
firewalld 서비스 기동
DNS A 레코드 생성
DNS MX 레코드 생성
SELinux 모드 변경
SELinux context 설정
메일 사용자 계정 생성
SMTP 인증 세부 구성
TLS 인증서 구성
스팸 필터 구성
바이러스 필터 구성
외부 메일 릴레이 구성
실제 인터넷 메일 송수신 보장
```

firewalld 설치와 기동은 별도 `firewall` role에서 먼저 처리한다.

DNS A/MX 레코드 생성은 별도 `dns` role에서 먼저 처리한다.

## Requirements

대상 OS는 다음을 기준으로 한다.

```text
CentOS Stream 9
RHEL 계열 9
```

필요한 Ansible collection은 다음과 같다.

```text
ansible.posix
```

설치 명령은 다음과 같다.

```bash
ansible-galaxy collection install ansible.posix
```

방화벽 등록은 `ansible.posix.firewalld` 모듈을 사용한다.

단, firewalld 패키지 설치와 firewalld 서비스 기동은 이 role에서 처리하지 않는다. 해당 작업은 별도 `firewall` role에서 처리하는 것을 전제로 한다.

## DNS Relationship

MAIL 서버는 DNS와 관계가 있다.

메일 서버 자체의 Postfix/Dovecot 구성은 DNS 없이도 가능하다. 하지만 도메인 기준으로 메일 서버를 찾으려면 DNS에 MX 레코드가 있어야 한다.

이 프로젝트에서는 DNS role이 다음 레코드를 생성한다.

```text
ansible4    IN A    192.168.10.14
@           IN MX   10 ansible4.example.com.
```

따라서 전체 playbook에서는 `dns` role이 `mail` role보다 먼저 실행되어야 한다.

권장 실행 순서는 다음과 같다.

```text
1. firewall role
2. dns role
3. web role
4. ftp role
5. mail role
```

중요한 점은 `mail` role의 `meta/main.yml`에 `dns` role을 dependency로 등록하지 않는다는 것이다.

`mail` role은 `mailservers` 그룹에서 실행된다. 만약 `mail` role의 dependency로 `dns` role을 등록하면, dependency role도 같은 host 컨텍스트에서 실행될 수 있다. 그러면 DNS role이 `ansible4.example.com`에서 실행되는 잘못된 구조가 될 수 있다.

따라서 DNS와 MAIL의 선후관계는 `meta/main.yml`이 아니라 상위 `playbook.yml`의 play 순서로 관리한다.

## Dependencies

이 role은 다른 role dependency를 선언하지 않는다.

```yaml
dependencies: []
```

실제 인프라 구성 순서는 상위 playbook에서 다음처럼 관리한다.

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

- name: MAIL 서버 구성
  hosts: mailservers
  gather_facts: true

  tasks:
    - name: MAIL role 실행
      ansible.builtin.include_role:
        name: mail
```

## Quick Start

프로젝트 루트로 이동한다.

```bash
cd $HOME/private_infrastructure
```

inventory 예시는 다음과 같다.

```ini
[mailservers]
ansible4.example.com ansible_host=192.168.10.14
```

playbook 예시는 다음과 같다.

```yaml
---
- name: MAIL 서버 구성
  hosts: mailservers
  gather_facts: true

  tasks:
    - name: MAIL role 실행
      ansible.builtin.include_role:
        name: mail
```

단독 실행은 가능하지만, 전체 인프라 구성에서는 `firewall` role과 `dns` role이 먼저 실행되는 구조를 권장한다.

실행한다.

```bash
ansible-navigator run -m stdout playbook.yml
```

테스트를 실행한다.

```bash
ansible-navigator run -m stdout roles/mail/tests/test.yml
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
    └── mail/
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
        │   ├── 03_config.yml
        │   ├── 04_firewall.yml
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
mail_domain: "{{ inventory_hostname.split('.', 1)[1] if '.' in inventory_hostname else '' }}"

mail_hostname: "{{ inventory_hostname }}"

mail_home_mailbox: Maildir/

mail_location: maildir:~/Maildir

mail_firewall_services:
  - smtp
  - imap
```

변수 설명은 다음과 같다.

```text
mail_domain
  메일 도메인 이름이다.
  inventory_hostname이 ansible4.example.com이면 기본값은 example.com 이다.

mail_hostname
  메일 서버 hostname이다.
  기본값은 inventory_hostname이다.

mail_home_mailbox
  Postfix가 사용할 사용자 mailbox 형식이다.
  기본값은 Maildir/ 이다.

mail_location
  Dovecot이 사용할 mailbox 위치이다.
  기본값은 maildir:~/Maildir 이다.

mail_firewall_services
  firewalld에 등록할 서비스 목록이다.
  기본값은 smtp, imap 이다.
```

role 내부에서 고정적으로 사용하는 값은 `vars/main.yml`에 정의한다. 일반적으로 사용자가 수정하지 않는다.

```yaml
---
mail_packages:
  - postfix
  - dovecot

mail_services:
  - postfix
  - dovecot

mail_postfix_config_file: /etc/postfix/main.cf
mail_dovecot_config_file: /etc/dovecot/dovecot.conf
mail_dovecot_mail_config_file: /etc/dovecot/conf.d/10-mail.conf
mail_dovecot_auth_config_file: /etc/dovecot/conf.d/10-auth.conf
```

## What To Edit

처음 사용하는 경우 보통 아래 파일만 수정하면 된다.

```text
inventory
playbook.yml
ansible-navigator.yml
```

메일 도메인을 직접 지정하려면 다음처럼 설정한다.

```yaml
vars:
  mail_domain: example.com
```

메일 서버 hostname을 직접 지정하려면 다음처럼 설정한다.

```yaml
vars:
  mail_hostname: ansible4.example.com
```

방화벽에 추가 서비스를 등록하려면 다음처럼 설정한다.

```yaml
vars:
  mail_firewall_services:
    - smtp
    - imap
```

이 role은 기본적으로 `smtp`, `imap`만 등록한다. `smtps`, `submission`, `imaps`는 기본 구성에 포함하지 않는다.

## Inventory Rule

이 role은 FQDN 형식의 inventory hostname을 사용하는 것을 기본 전제로 한다.

좋은 예시는 다음과 같다.

```ini
[mailservers]
ansible4.example.com ansible_host=192.168.10.14
```

FQDN이 아니면 `mail_domain`을 자동 계산할 수 없다. 이 경우에는 직접 지정해야 한다.

```yaml
vars:
  mail_domain: example.com
```

## Generated Files

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text
/etc/postfix/main.cf.OLD
/etc/postfix/main.cf
/etc/dovecot/dovecot.conf.OLD
/etc/dovecot/dovecot.conf
/etc/dovecot/conf.d/10-mail.conf
/etc/dovecot/conf.d/10-auth.conf
```

`/etc/postfix/main.cf.OLD`는 Postfix 원본 설정 파일을 1회 백업한 파일이다. 이미 존재하면 덮어쓰지 않는다.

`/etc/dovecot/dovecot.conf.OLD`는 Dovecot 원본 설정 파일을 1회 백업한 파일이다. 이미 존재하면 덮어쓰지 않는다.

이 role은 설정 파일 전체를 새로 덮어쓰지 않는다. 필요한 설정만 `lineinfile`로 관리한다.

기본적으로 적용하는 주요 Postfix 설정은 다음과 같다.

```text
myhostname = ansible4.example.com
mydomain = example.com
myorigin = $mydomain
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
home_mailbox = Maildir/
```

기본적으로 적용하는 주요 Dovecot 설정은 다음과 같다.

```text
protocols = imap
mail_location = maildir:~/Maildir
disable_plaintext_auth = no
auth_mechanisms = plain login
```

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

MAIL role test 문법만 확인하려면 다음 명령을 사용한다.

```bash
ansible-navigator run -m stdout roles/mail/tests/test.yml --syntax-check
```

inventory를 확인하려면 다음 명령을 사용한다.

```bash
ansible-navigator inventory -m stdout --list
```

## Expected Result

정상 실행 시 대략 다음 흐름으로 출력된다.

```text
PLAY [MAIL 서버 구성]

TASK [Gathering Facts]
ok: [ansible4.example.com]

TASK [MAIL role 실행]

TASK [mail : 1) 패키지 설치 작업 포함]
included: .../roles/mail/tasks/01_package.yml for ansible4.example.com

TASK [mail : 1-0) CentOS 환경 확인]
skipping: [ansible4.example.com]

TASK [mail : 1-1) MAIL 패키지 설치 - postfix, dovecot]
ok: [ansible4.example.com]

TASK [mail : 2) 서비스 기동 작업 포함]
included: .../roles/mail/tasks/02_service.yml for ansible4.example.com

TASK [mail : 2-1) MAIL 서비스 기동 및 활성화 - postfix]
ok: [ansible4.example.com]

TASK [mail : 2-1) MAIL 서비스 기동 및 활성화 - dovecot]
ok: [ansible4.example.com]

TASK [mail : 3) 서비스 설정 작업 포함]
included: .../roles/mail/tasks/03_config.yml for ansible4.example.com

TASK [mail : 3-0) mail domain 이름 확인]
skipping: [ansible4.example.com]

TASK [mail : 3-1) 원본 파일 백업 생성 - /etc/postfix/main.cf.OLD]
ok: [ansible4.example.com]

TASK [mail : 3-2) 원본 파일 백업 생성 - /etc/dovecot/dovecot.conf.OLD]
ok: [ansible4.example.com]

TASK [mail : 3-3) Postfix 기본 설정 수정 - /etc/postfix/main.cf]
ok: [ansible4.example.com] => (item=myhostname = ansible4.example.com)
ok: [ansible4.example.com] => (item=mydomain = example.com)
ok: [ansible4.example.com] => (item=myorigin = $mydomain)
ok: [ansible4.example.com] => (item=inet_interfaces = all)
ok: [ansible4.example.com] => (item=inet_protocols = ipv4)
ok: [ansible4.example.com] => (item=mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain)
ok: [ansible4.example.com] => (item=home_mailbox = Maildir/)

TASK [mail : 3-4) Dovecot protocol 설정 수정 - /etc/dovecot/dovecot.conf]
ok: [ansible4.example.com]

TASK [mail : 3-5) Dovecot mail location 설정 수정 - /etc/dovecot/conf.d/10-mail.conf]
ok: [ansible4.example.com]

TASK [mail : 3-6) Dovecot plaintext auth 설정 수정 - /etc/dovecot/conf.d/10-auth.conf]
ok: [ansible4.example.com]

TASK [mail : 3-7) Dovecot auth mechanisms 설정 수정 - /etc/dovecot/conf.d/10-auth.conf]
ok: [ansible4.example.com]

TASK [mail : 3-8) postfix 서비스 재시작]
skipping: [ansible4.example.com]

TASK [mail : 3-9) dovecot 서비스 재시작]
skipping: [ansible4.example.com]

TASK [mail : 4) 방화벽 등록 작업 포함]
included: .../roles/mail/tasks/04_firewall.yml for ansible4.example.com

TASK [mail : 4-1) firewalld MAIL 서비스 등록 - smtp]
ok: [ansible4.example.com]

TASK [mail : 4-1) firewalld MAIL 서비스 등록 - imap]
ok: [ansible4.example.com]

PLAY RECAP
ansible4.example.com : ok=..., changed=..., unreachable=0, failed=0
```

처음 실행하면 파일 생성, 패키지 설치, 설정 변경으로 인해 일부 task가 `changed`로 표시될 수 있다. 이미 적용된 상태에서 다시 실행하면 많은 task가 `ok`로 표시된다.

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
roles/mail/tests/test.yml
```

실행 명령은 다음과 같다.

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/mail/tests/test.yml
```

테스트는 role을 적용한 뒤 다음 명령으로 설정 검증을 수행한다.

```text
postfix check
doveconf -n
```

`postfix check`, `doveconf -n` 실행은 테스트 목적상 `ansible.builtin.command` 모듈을 예외적으로 사용한다.

수동 검증 예시는 다음과 같다.

```bash
ssh ansible4.example.com "sudo systemctl status postfix --no-pager"
ssh ansible4.example.com "sudo systemctl status dovecot --no-pager"
```

Postfix 주요 설정 확인:

```bash
ssh ansible4.example.com "sudo grep -E '^(myhostname|mydomain|myorigin|inet_interfaces|inet_protocols|mydestination|home_mailbox)' /etc/postfix/main.cf"
```

Dovecot 주요 설정 확인:

```bash
ssh ansible4.example.com "sudo grep -E '^(protocols)' /etc/dovecot/dovecot.conf"
ssh ansible4.example.com "sudo grep -E '^(mail_location)' /etc/dovecot/conf.d/10-mail.conf"
ssh ansible4.example.com "sudo grep -E '^(disable_plaintext_auth|auth_mechanisms)' /etc/dovecot/conf.d/10-auth.conf"
```

DNS MX 레코드 확인:

```bash
dig +short @192.168.10.11 example.com MX
```

정상 예시는 다음과 같다.

```text
10 ansible4.example.com.
```

## SELinux

테스트 환경에서는 SELinux가 permissive 상태였다.

이 role은 SELinux 모드를 변경하지 않는다.

Maildir 기본 경로는 사용자 홈 디렉토리 아래의 `~/Maildir`이다. SELinux enforcing 환경에서 실제 사용자 메일 송수신 테스트를 확장하는 경우 mailbox context와 관련 정책을 추가로 확인해야 할 수 있다.

## Troubleshooting

`ansible.posix.firewalld` 관련 오류가 발생하면 collection 설치 여부를 확인한다.

```bash
ansible-galaxy collection list ansible.posix
```

없으면 설치한다.

```bash
ansible-galaxy collection install ansible.posix
```

firewalld 서비스 관련 오류가 발생하면 `firewall` role이 먼저 실행되었는지 확인한다.

```bash
sudo systemctl status firewalld
```

Postfix 서비스 상태는 다음 명령으로 확인한다.

```bash
sudo systemctl status postfix
```

Dovecot 서비스 상태는 다음 명령으로 확인한다.

```bash
sudo systemctl status dovecot
```

SMTP 포트 대기 상태는 다음 명령으로 확인한다.

```bash
sudo ss -lntup | grep ':25'
```

IMAP 포트 대기 상태는 다음 명령으로 확인한다.

```bash
sudo ss -lntup | grep ':143'
```

Postfix 설정 검증은 다음 명령으로 확인한다.

```bash
sudo postfix check
```

Dovecot 설정 검증은 다음 명령으로 확인한다.

```bash
sudo doveconf -n
```

DNS MX 레코드는 다음 명령으로 확인한다.

```bash
dig +short @192.168.10.11 example.com MX
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

Managed Node:
  ansible1.example.com 192.168.10.11 DNS Server
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
