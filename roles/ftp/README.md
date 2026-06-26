# FTP Role

CentOS Stream 9 / RHEL 계열 9 환경에서 vsftpd 기반 FTP 서버를 구성하는 Ansible role이다.

이 role은 `ftpservers` 그룹의 대상 노드에 `vsftpd`를 설치하고, 서비스를 기동하며, 익명 다운로드용 공개 디렉토리와 기본 파일을 생성하고, firewalld에 `ftp` 서비스를 등록한다.

주요 작업 흐름은 다음과 같다.

```text id="gu8g49"
1) 패키지 설치
2) 서비스 기동
3) 서비스 설정
4) 방화벽 등록
```

## Scope

이 role이 처리하는 범위는 다음과 같다.

```text id="nk0yhw"
vsftpd 패키지 설치
vsftpd 서비스 기동 및 활성화
/etc/vsftpd/vsftpd.conf 원본 1회 백업
익명 다운로드용 FTP 설정 적용
FTP 공개 디렉토리 생성
기본 README.txt 파일 생성
firewalld ftp 서비스 등록
```

이 role이 처리하지 않는 범위는 다음과 같다.

```text id="oo19x7"
firewalld 설치
firewalld 서비스 기동
SELinux 모드 변경
SELinux context 설정
FTP 사용자 계정 생성
로컬 사용자 FTP 로그인 구성
FTP 업로드 허용 구성
FTPS/TLS 구성
passive port range 세부 구성
외부 FTP 컨텐츠 배포
```

## Requirements

대상 OS는 다음을 기준으로 한다.

```text id="sjkrzx"
CentOS Stream 9
RHEL 계열 9
```

필요한 Ansible collection은 다음과 같다.

```text id="qvsvnl"
ansible.posix
```

설치 명령은 다음과 같다.

```bash id="yujv6m"
ansible-galaxy collection install ansible.posix
```

방화벽 등록은 `ansible.posix.firewalld` 모듈을 사용한다.

단, firewalld 패키지 설치와 firewalld 서비스 기동은 이 role에서 처리하지 않는다. 해당 작업은 별도 firewall role에서 처리하는 것을 전제로 한다.

## Dependencies

이 role은 다른 role dependency를 선언하지 않는다.

```yaml id="fi7pdu"
dependencies: []
```

firewalld 설치와 기동은 별도 firewall role에서 처리하는 것을 권장하지만, 이 role의 `meta/main.yml`에는 dependency로 등록하지 않는다.

## Quick Start

프로젝트 루트로 이동한다.

```bash id="svi4i8"
cd $HOME/private_infrastructure
```

inventory 예시는 다음과 같다.

```ini id="r9x1hz"
[ftpservers]
ansible3.example.com ansible_host=192.168.10.13
```

playbook 예시는 다음과 같다.

```yaml id="3nmw4b"
---
- name: FTP 서버 구성
  hosts: ftpservers
  gather_facts: true

  tasks:
    - name: FTP role 실행
      ansible.builtin.include_role:
        name: ftp
```

실행한다.

```bash id="42lw5s"
ansible-navigator run -m stdout playbook.yml
```

테스트를 실행한다.

```bash id="uo6boi"
ansible-navigator run -m stdout roles/ftp/tests/test.yml
```

## Directory Structure

권장 구조는 다음과 같다.

```text id="rr48kz"
$HOME/private_infrastructure/
├── ansible.cfg
├── ansible-navigator.yml
├── inventory
├── playbook.yml
└── roles/
    └── ftp/
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

```ini id="jj3bsp"
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

```yaml id="es7r2k"
---
ansible-navigator:
  playbook-artifact:
    enable: false
```

## Role Variables

사용자가 환경에 맞게 바꿀 수 있는 변수는 `defaults/main.yml`에 정의한다.

```yaml id="tfwnk8"
---
ftp_root_dir: /var/ftp

ftp_public_dir: "{{ ftp_root_dir }}/pub"

ftp_sample_file: README.txt

ftp_sample_content: |
  FTP Server
  Hostname: {{ inventory_hostname }}
  IP Address: {{ ansible_host | default(ansible_facts['default_ipv4']['address']) }}
  OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}

ftp_firewall_service: ftp

ftp_test_url: "ftp://127.0.0.1/pub/{{ ftp_sample_file }}"

ftp_test_dest: "/tmp/ansible_ftp_test_{{ ftp_sample_file }}"
```

변수 설명은 다음과 같다.

```text id="c3a6mw"
ftp_root_dir
  FTP 기본 디렉토리이다.
  기본값은 /var/ftp 이다.

ftp_public_dir
  익명 다운로드용 공개 디렉토리이다.
  기본값은 {{ ftp_root_dir }}/pub 이다.

ftp_sample_file
  공개 디렉토리에 생성할 기본 파일 이름이다.
  기본값은 README.txt 이다.

ftp_sample_content
  role에서 생성할 기본 파일 내용이다.
  inventory_hostname, ansible_host, ansible_facts 값을 사용한다.

ftp_firewall_service
  firewalld에 등록할 서비스 이름이다.
  기본값은 ftp 이다.

ftp_test_url
  roles/ftp/tests/test.yml에서 확인할 FTP URL이다.
  테스트는 FTP 서버 자신에서 실행하므로 기본값은 127.0.0.1을 사용한다.

ftp_test_dest
  테스트 시 다운로드 파일을 저장할 임시 경로이다.
```

role 내부에서 고정적으로 사용하는 값은 `vars/main.yml`에 정의한다. 일반적으로 사용자가 수정하지 않는다.

```yaml id="bazjpg"
---
ftp_packages:
  - vsftpd

ftp_service_name: vsftpd

ftp_config_file: /etc/vsftpd/vsftpd.conf
```

## What To Edit

처음 사용하는 경우 보통 아래 파일만 수정하면 된다.

```text id="c7qcxb"
inventory
playbook.yml
ansible-navigator.yml
```

기본 공개 디렉토리를 바꾸려면 다음처럼 설정한다.

```yaml id="wwzv5a"
vars:
  ftp_public_dir: /srv/ftp/pub
```

단, 기본값이 아닌 경로를 사용하는 경우 vsftpd 설정과 SELinux context는 별도로 고려해야 한다.

기본 파일 이름을 바꾸려면 다음처럼 설정한다.

```yaml id="c3g9dz"
vars:
  ftp_sample_file: welcome.txt
```

기본 파일 내용을 바꾸려면 다음처럼 설정한다.

```yaml id="hssioz"
vars:
  ftp_sample_content: |
    Welcome to FTP Server
    Managed by Ansible
```

테스트 URL을 바꾸려면 다음처럼 설정한다.

```yaml id="zu4b7q"
vars:
  ftp_test_url: ftp://127.0.0.1/pub/welcome.txt
```

## Inventory Rule

이 role은 `ftpservers` 그룹에 속한 host를 FTP 서버로 구성하는 것을 기본 사용 방식으로 한다.

inventory 예시는 다음과 같다.

```ini id="sidg6m"
[ftpservers]
ansible3.example.com ansible_host=192.168.10.13
```

## Generated Files

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text id="yubhra"
/etc/vsftpd/vsftpd.conf.OLD
/etc/vsftpd/vsftpd.conf
/var/ftp/pub
/var/ftp/pub/README.txt
```

`/etc/vsftpd/vsftpd.conf.OLD`는 원본 설정 파일을 1회 백업한 파일이다. 이미 존재하면 덮어쓰지 않는다.

이 role은 `/etc/vsftpd/vsftpd.conf` 전체를 새로 덮어쓰지 않는다. 필요한 설정만 `lineinfile`로 관리한다.

기본적으로 적용하는 주요 FTP 설정은 다음과 같다.

```text id="h0a4ty"
anonymous_enable=YES
local_enable=NO
write_enable=NO
anon_upload_enable=NO
anon_mkdir_write_enable=NO
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
listen=NO
listen_ipv6=YES
```

기본 파일 예시는 다음과 같다.

```text id="vj4zjz"
FTP Server
Hostname: ansible3.example.com
IP Address: 192.168.10.13
OS: CentOS 9
```

## Run

프로젝트 루트에서 실행한다.

```bash id="qbkqp1"
cd $HOME/private_infrastructure
ansible-navigator run -m stdout playbook.yml
```

문법만 확인하려면 다음 명령을 사용한다.

```bash id="55ub3c"
ansible-navigator run -m stdout playbook.yml --syntax-check
```

inventory를 확인하려면 다음 명령을 사용한다.

```bash id="j6w1lv"
ansible-navigator inventory -m stdout --list
```

## Expected Result

정상 실행 시 대략 다음 흐름으로 출력된다.

```text id="ptfwrs"
PLAY [FTP 서버 구성]

TASK [Gathering Facts]
ok: [ansible3.example.com]

TASK [FTP role 실행]

TASK [ftp : 1) 패키지 설치 작업 포함]
included: .../roles/ftp/tasks/01_package.yml for ansible3.example.com

TASK [ftp : 1-0) CentOS 환경 확인]
skipping: [ansible3.example.com]

TASK [ftp : 1-1) FTP 패키지 설치 - vsftpd]
ok: [ansible3.example.com]

TASK [ftp : 2) 서비스 기동 작업 포함]
included: .../roles/ftp/tasks/02_service.yml for ansible3.example.com

TASK [ftp : 2-1) vsftpd 서비스 기동 및 활성화]
ok: [ansible3.example.com]

TASK [ftp : 3) 서비스 설정 작업 포함]
included: .../roles/ftp/tasks/03_config.yml for ansible3.example.com

TASK [ftp : 3-1) 원본 파일 백업 생성 - /etc/vsftpd/vsftpd.conf.OLD]
ok: [ansible3.example.com]

TASK [ftp : 3-2) FTP 기본 설정 수정 - /etc/vsftpd/vsftpd.conf]
ok: [ansible3.example.com] => (item=anonymous_enable=YES)
ok: [ansible3.example.com] => (item=local_enable=NO)
ok: [ansible3.example.com] => (item=write_enable=NO)
ok: [ansible3.example.com] => (item=anon_upload_enable=NO)
ok: [ansible3.example.com] => (item=anon_mkdir_write_enable=NO)
ok: [ansible3.example.com] => (item=dirmessage_enable=YES)
ok: [ansible3.example.com] => (item=xferlog_enable=YES)
ok: [ansible3.example.com] => (item=connect_from_port_20=YES)
ok: [ansible3.example.com] => (item=listen=NO)
ok: [ansible3.example.com] => (item=listen_ipv6=YES)

TASK [ftp : 3-3) FTP 공개 디렉토리 생성 - /var/ftp/pub]
ok: [ansible3.example.com]

TASK [ftp : 3-4) FTP 기본 파일 생성 - /var/ftp/pub/README.txt]
ok: [ansible3.example.com]

TASK [ftp : 3-5) vsftpd 서비스 재시작]
skipping: [ansible3.example.com]

TASK [ftp : 4) 방화벽 등록 작업 포함]
included: .../roles/ftp/tasks/04_firewall.yml for ansible3.example.com

TASK [ftp : 4-1) firewalld ftp 서비스 등록 - ftp]
ok: [ansible3.example.com]

PLAY RECAP
ansible3.example.com : ok=..., changed=..., unreachable=0, failed=0
```

처음 실행하면 파일 생성, 패키지 설치, 설정 변경으로 인해 일부 task가 `changed`로 표시될 수 있다. 이미 적용된 상태에서 다시 실행하면 많은 task가 `ok`로 표시된다.

출력 상태의 의미는 다음과 같다.

```text id="b1tg8a"
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

```text id="cob1k0"
roles/ftp/tests/test.yml
```

실행 명령은 다음과 같다.

```bash id="h5z99p"
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/ftp/tests/test.yml
```

테스트는 role을 적용한 뒤 FTP 서버 자신에서 `get_url` 모듈로 기본 파일 다운로드를 확인한다.

기본 테스트 URL은 다음과 같다.

```yaml id="2v1opx"
ftp_test_url: "ftp://127.0.0.1/pub/{{ ftp_sample_file }}"
```

네 환경에서는 다음 URL을 확인하는 것과 같다.

```text id="c9oj3w"
ftp://127.0.0.1/pub/README.txt
```

다운로드 대상 파일은 다음과 같다.

```text id="2fnuyo"
/tmp/ansible_ftp_test_README.txt
```

수동 검증 예시는 다음과 같다.

FTP 서버 자신에서 확인:

```bash id="ygtbdq"
curl ftp://127.0.0.1/pub/README.txt
```

제어 노드에서 확인:

```bash id="0337g8"
curl ftp://192.168.10.13/pub/README.txt
```

예상 결과는 다음과 같다.

```text id="8pjcuf"
FTP Server
Hostname: ansible3.example.com
IP Address: 192.168.10.13
OS: CentOS 9
```

## SELinux

테스트 환경에서는 SELinux가 permissive 상태였다.

이 role은 SELinux 모드를 변경하지 않는다.

기본 FTP 디렉토리 `/var/ftp/pub`을 사용할 경우 일반적인 CentOS/RHEL vsftpd 기본 정책과 맞는다. 단, FTP 공개 디렉토리를 다른 경로로 변경하는 경우 SELinux enforcing 환경에서는 context 문제로 vsftpd가 파일을 읽지 못할 수 있다.

## Troubleshooting

`ansible.posix.firewalld` 관련 오류가 발생하면 collection 설치 여부를 확인한다.

```bash id="1lghyi"
ansible-galaxy collection list ansible.posix
```

없으면 설치한다.

```bash id="1tjzv7"
ansible-galaxy collection install ansible.posix
```

firewalld 서비스 관련 오류가 발생하면 firewalld가 설치 및 기동되어 있는지 확인한다.

```bash id="puidif"
sudo systemctl status firewalld
```

vsftpd 서비스 상태는 다음 명령으로 확인한다.

```bash id="jktnvd"
sudo systemctl status vsftpd
```

21번 포트 대기 상태는 다음 명령으로 확인한다.

```bash id="m1sk7x"
sudo ss -lntup | grep ':21'
```

FTP 응답은 다음 명령으로 확인한다.

```bash id="ffzgg8"
curl ftp://192.168.10.13/pub/README.txt
```

설정 파일을 확인하려면 다음 명령을 사용한다.

```bash id="0024rd"
sudo grep -E '^(anonymous_enable|local_enable|write_enable|listen|listen_ipv6)=' /etc/vsftpd/vsftpd.conf
```

## Tested Environment

작성 및 테스트 기준 환경은 다음과 같다.

```text id="0lyh8d"
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
  ansible3.example.com 192.168.10.13 FTP Server

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
