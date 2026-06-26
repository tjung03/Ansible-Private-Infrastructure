# DNS Role

CentOS Stream 9 / RHEL 계열 9 환경에서 BIND 기반 DNS 서버를 구성하는 Ansible role이다.

이 role은 `dnsservers` 그룹의 대상 노드에 `bind`, `bind-utils`를 설치하고, `named` 서비스를 기동하며, inventory 정보를 기준으로 DNS zone 파일을 생성한다.

이 role은 A 레코드와 MX 레코드를 생성할 수 있다.

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
bind 패키지 설치
bind-utils 패키지 설치
named 서비스 기동 및 활성화
/etc/named.conf 원본 1회 백업
/etc/named.conf 배포
role 전용 zone include 파일 생성
zone 파일 생성
inventory 기반 A 레코드 생성
inventory 기반 MX 레코드 생성
firewalld dns 서비스 등록
```

이 role이 처리하지 않는 범위는 다음과 같다.

```text
firewalld 설치
firewalld 서비스 기동
SELinux 모드 변경
SELinux context 설정
역방향 zone 구성
외부 public DNS 구성
DNSSEC 세부 구성
slave DNS 구성
dynamic DNS 구성
```

firewalld 설치와 기동은 별도 `firewall` role에서 먼저 처리한다.

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

## Dependencies

이 role은 다른 role dependency를 선언하지 않는다.

```yaml
dependencies: []
```

firewalld 설치와 기동은 별도 `firewall` role에서 먼저 처리한다.

MAIL 서버의 MX 레코드는 이 DNS role에서 생성할 수 있다. 하지만 `mail` role을 이 role의 dependency로 등록하지 않는다. DNS와 MAIL의 실행 순서는 상위 `playbook.yml`에서 관리한다.

권장 실행 순서는 다음과 같다.

```text
1. firewall role
2. dns role
3. web role
4. ftp role
5. mail role
```

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

playbook 예시는 다음과 같다.

```yaml
---
- name: DNS 서버 구성
  hosts: dnsservers
  gather_facts: true

  tasks:
    - name: DNS role 실행
      ansible.builtin.include_role:
        name: dns
```

실행한다.

```bash
ansible-navigator run -m stdout playbook.yml
```

테스트를 실행한다.

```bash
ansible-navigator run -m stdout roles/dns/tests/test.yml
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
    └── dns/
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
        │   ├── named.conf.j2
        │   ├── named.zone.include.j2
        │   └── zone.j2
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
dns_forwarders:
  - 8.8.8.8

dns_zone_name: "{{ inventory_hostname.split('.', 1)[1] if '.' in inventory_hostname else '' }}"

dns_zone_file: "{{ dns_zone_name }}.zone"

dns_zone_include_file: "/etc/named.{{ dns_zone_name }}.zones"

dns_record_groups:
  - all

dns_mx_record_groups:
  - mailservers

dns_mx_priority: 10

dns_test_record: "{{ inventory_hostname }}"
```

변수 설명은 다음과 같다.

```text
dns_forwarders
  외부 질의를 전달할 DNS 서버 목록이다.
  기본값은 8.8.8.8 이다.

dns_zone_name
  DNS zone 이름이다.
  inventory_hostname이 ansible1.example.com이면 기본값은 example.com 이다.

dns_zone_file
  /var/named 아래에 생성할 zone 파일 이름이다.
  기본값은 example.com.zone 형태이다.

dns_zone_include_file
  role 전용 zone include 파일 경로이다.
  기본값은 /etc/named.example.com.zones 형태이다.

dns_record_groups
  A 레코드로 등록할 inventory group 목록이다.
  기본값은 all 이다.

dns_mx_record_groups
  MX 레코드로 등록할 inventory group 목록이다.
  기본값은 mailservers 이다.

dns_mx_priority
  MX 레코드 priority 값이다.
  기본값은 10 이다.

dns_test_record
  roles/dns/tests/test.yml에서 확인할 A 레코드 대상이다.
```

role 내부에서 고정적으로 사용하는 값은 `vars/main.yml`에 정의한다. 일반적으로 사용자가 수정하지 않는다.

```yaml
---
dns_packages:
  - bind
  - bind-utils

dns_service_name: named
```

## What To Edit

처음 사용하는 경우 보통 아래 파일만 수정하면 된다.

```text
inventory
playbook.yml
ansible-navigator.yml
```

외부 DNS forwarder를 바꾸려면 다음처럼 설정한다.

```yaml
vars:
  dns_forwarders:
    - 8.8.8.8
    - 1.1.1.1
```

A 레코드 생성 대상을 바꾸려면 다음처럼 설정한다.

```yaml
vars:
  dns_record_groups:
    - webservers
    - ftpservers
    - mailservers
```

MX 레코드 생성 대상을 바꾸려면 다음처럼 설정한다.

```yaml
vars:
  dns_mx_record_groups:
    - mailservers
```

MX priority를 바꾸려면 다음처럼 설정한다.

```yaml
vars:
  dns_mx_priority: 20
```

## Inventory Rule

이 role은 FQDN 형식의 inventory hostname을 사용하는 것을 기본 전제로 한다.

좋은 예시는 다음과 같다.

```ini
[dnsservers]
ansible1.example.com ansible_host=192.168.10.11
```

FQDN이 아니면 `dns_zone_name`을 자동 계산할 수 없다. 이 경우에는 직접 지정해야 한다.

```yaml
vars:
  dns_zone_name: example.com
```

## Generated Files

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text
/etc/named.conf.OLD
/etc/named.conf
/etc/named.example.com.zones
/var/named/example.com.zone
```

`/etc/named.conf.OLD`는 원본 설정 파일을 1회 백업한 파일이다. 이미 존재하면 덮어쓰지 않는다.

이 role은 `/etc/named.rfc1912.zones`를 직접 수정하지 않는다.

대신 role 전용 zone include 파일을 생성한다.

```text
/etc/named.example.com.zones
```

`/etc/named.conf`에는 다음 include가 들어간다.

```text
include "/etc/named.rfc1912.zones";
include "/etc/named.example.com.zones";
include "/etc/named.root.key";
```

기본 zone 파일 예시는 다음과 같다.

```text
$TTL 86400
@       IN SOA  ansible1.example.com. root.example.com. (
                2026062601 ; serial
                1D         ; refresh
                1H         ; retry
                1W         ; expire
                3H )       ; minimum

        IN NS   ansible1.example.com.

@       IN MX   10 ansible4.example.com.

ansible1    IN A    192.168.10.11
ansible2    IN A    192.168.10.12
ansible3    IN A    192.168.10.13
ansible4    IN A    192.168.10.14
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

inventory를 확인하려면 다음 명령을 사용한다.

```bash
ansible-navigator inventory -m stdout --list
```

## Expected Result

정상 실행 시 대략 다음 흐름으로 출력된다.

```text
PLAY [DNS 서버 구성]

TASK [Gathering Facts]
ok: [ansible1.example.com]

TASK [DNS role 실행]

TASK [dns : 1) 패키지 설치 작업 포함]
included: .../roles/dns/tasks/01_package.yml for ansible1.example.com

TASK [dns : 1) CentOS 환경 확인]
skipping: [ansible1.example.com]

TASK [dns : 1) DNS 패키지 설치]
ok: [ansible1.example.com]

TASK [dns : 2) 서비스 기동 작업 포함]
included: .../roles/dns/tasks/02_service.yml for ansible1.example.com

TASK [dns : 2) named 서비스 기동 및 활성화]
ok: [ansible1.example.com]

TASK [dns : 3) 서비스 설정 작업 포함]
included: .../roles/dns/tasks/03_config.yml for ansible1.example.com

TASK [dns : 3-0) DNS zone 이름 확인]
skipping: [ansible1.example.com]

TASK [dns : 3-1) 원본 파일 백업 생성 - /etc/named.conf.OLD]
ok: [ansible1.example.com]

TASK [dns : 3-2) DNS zone 파일 생성 - /var/named/example.com.zone]
ok: [ansible1.example.com]

TASK [dns : 3-3) DNS zone include 파일 생성 - /etc/named.example.com.zones]
ok: [ansible1.example.com]

TASK [dns : 3-4) DNS 기본 설정 파일 배포 - /etc/named.conf]
ok: [ansible1.example.com]

TASK [dns : 3-5) named 서비스 재시작]
skipping: [ansible1.example.com]

TASK [dns : 4) 방화벽 등록 작업 포함]
included: .../roles/dns/tasks/04_firewall.yml for ansible1.example.com

TASK [dns : 4) firewalld dns 서비스 등록]
ok: [ansible1.example.com]

PLAY RECAP
ansible1.example.com : ok=..., changed=..., unreachable=0, failed=0
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
roles/dns/tests/test.yml
```

실행 명령은 다음과 같다.

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/dns/tests/test.yml
```

기본 테스트는 role을 적용한 뒤 A 레코드 응답을 확인한다.

기본 테스트 대상은 다음과 같다.

```yaml
dns_test_record: "{{ inventory_hostname }}"
```

네 환경에서는 다음 질의를 확인하는 것과 같다.

```bash
dig +short @192.168.10.11 ansible1.example.com
```

다른 A 레코드를 확인하려면 다음처럼 실행한다.

```bash
ansible-navigator run -m stdout roles/dns/tests/test.yml \
  -e dns_test_record=ansible2.example.com
```

MX 레코드는 다음 명령으로 수동 확인할 수 있다.

```bash
dig +short @192.168.10.11 example.com MX
```

정상 예시는 다음과 같다.

```text
10 ansible4.example.com.
```

제어 노드에 `dig`가 없으면 다음 패키지가 필요하다.

```bash
sudo dnf install -y bind-utils
```

## SELinux

테스트 환경에서는 SELinux가 permissive 상태였다.

이 role은 SELinux 모드를 변경하지 않는다.

기본 zone 파일 경로 `/var/named/{{ dns_zone_file }}`을 사용할 경우 일반적인 CentOS/RHEL BIND 기본 정책과 맞는다. 단, zone 파일 경로나 include 파일 경로를 변경하는 경우 SELinux enforcing 환경에서는 context 문제로 `named`가 파일을 읽지 못할 수 있다.

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

named 서비스 상태는 다음 명령으로 확인한다.

```bash
sudo systemctl status named
```

DNS 설정 문법은 다음 명령으로 확인한다.

```bash
sudo named-checkconf /etc/named.conf
```

zone 파일은 다음 명령으로 확인한다.

```bash
sudo named-checkzone example.com /var/named/example.com.zone
```

A 레코드는 다음 명령으로 확인한다.

```bash
dig +short @192.168.10.11 ansible1.example.com
```

MX 레코드는 다음 명령으로 확인한다.

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
