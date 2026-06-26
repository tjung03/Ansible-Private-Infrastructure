# WEB Role

CentOS Stream 9 / RHEL 계열 9 환경에서 Apache HTTP Server를 구성하는 Ansible role이다.

이 role은 inventory 정보를 기준으로 WEB 서버에 `httpd`를 설치하고, 서비스를 기동하며, 기본 index 파일을 생성하고, firewalld에 `http` 서비스를 등록한다.

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
httpd 패키지 설치
httpd 서비스 기동 및 활성화
웹 문서 루트 디렉토리 생성 또는 확인
기본 index 파일 생성
firewalld http 서비스 등록
```

이 role이 처리하지 않는 범위는 다음과 같다.

```text
firewalld 설치
firewalld 서비스 기동
SELinux 모드 변경
SELinux context 설정
가상 호스트 구성
HTTPS/TLS 구성
Apache 세부 튜닝
외부 웹 컨텐츠 배포
```

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

단, firewalld 패키지 설치와 firewalld 서비스 기동은 이 role에서 처리하지 않는다. 해당 작업은 별도 firewall role에서 처리하는 것을 전제로 한다.

## Dependencies

이 role은 다른 role dependency를 선언하지 않는다.

```yaml
dependencies: []
```

firewalld 설치와 기동은 별도 firewall role에서 처리하는 것을 권장하지만, 이 role의 `meta/main.yml`에는 dependency로 등록하지 않는다.

## Quick Start

프로젝트 루트로 이동한다.

```bash
cd $HOME/private_infrastructure
```

inventory 예시는 다음과 같다.

```ini
[webservers]
ansible2.example.com ansible_host=192.168.10.12
```

playbook 예시는 다음과 같다.

```yaml
---
- name: WEB 서버 구성
  hosts: webservers
  gather_facts: true

  tasks:
    - name: WEB role 실행
      ansible.builtin.include_role:
        name: web
```

실행한다.

```bash
ansible-navigator run -m stdout playbook.yml
```

테스트를 실행한다.

```bash
ansible-navigator run -m stdout roles/web/tests/test.yml
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
    └── web/
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
web_document_root: /var/www/html

web_index_file: index.html

web_manage_index: true

web_firewall_service: http

web_test_url: "http://{{ ansible_host | default(ansible_facts['default_ipv4']['address']) }}/"

web_index_content: |
  <!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8">
    <title>{{ inventory_hostname }}</title>
  </head>
  <body>
    <h1>WEB Server</h1>
    <p>Hostname: {{ inventory_hostname }}</p>
    <p>IP Address: {{ ansible_host | default(ansible_facts['default_ipv4']['address']) }}</p>
    <p>OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}</p>
  </body>
  </html>
```

변수 설명은 다음과 같다.

```text
web_document_root
  웹 문서 루트 디렉토리이다.
  기본값은 /var/www/html 이다.

web_index_file
  기본 index 파일 이름이다.
  기본값은 index.html 이다.

web_manage_index
  role에서 기본 index 파일을 생성할지 결정한다.
  기존 웹 컨텐츠를 건드리고 싶지 않으면 false로 설정한다.

web_firewall_service
  firewalld에 등록할 서비스 이름이다.
  기본값은 http 이다.

web_test_url
  roles/web/tests/test.yml에서 확인할 HTTP URL이다.

web_index_content
  role에서 생성할 기본 index.html 내용이다.
  inventory_hostname, ansible_host, ansible_facts 값을 사용한다.
```

role 내부에서 고정적으로 사용하는 값은 `vars/main.yml`에 정의한다. 일반적으로 사용자가 수정하지 않는다.

```yaml
---
web_packages:
  - httpd

web_service_name: httpd
```

## What To Edit

처음 사용하는 경우 보통 아래 파일만 수정하면 된다.

```text
inventory
playbook.yml
ansible-navigator.yml
```

기존 웹 컨텐츠를 보존하고 싶으면 다음처럼 설정한다.

```yaml
vars:
  web_manage_index: false
```

문서 루트를 바꾸려면 다음처럼 설정한다.

```yaml
vars:
  web_document_root: /srv/www/html
```

단, 문서 루트를 기본값이 아닌 경로로 바꾸는 경우 Apache 설정과 SELinux context는 별도로 고려해야 한다. 이 role은 `/etc/httpd/conf/httpd.conf`를 직접 수정하지 않는다.

index 파일 내용을 바꾸려면 다음처럼 설정한다.

```yaml
vars:
  web_index_content: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>Custom WEB Page</h1>
    </body>
    </html>
```

## Inventory Rule

이 role은 `webservers` 그룹에 속한 host를 WEB 서버로 구성하는 것을 기본 사용 방식으로 한다.

inventory 예시는 다음과 같다.

```ini
[webservers]
ansible2.example.com ansible_host=192.168.10.12
```

## Generated Files

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text
/var/www/html
/var/www/html/index.html
```

이 role은 기본적으로 `/etc/httpd/conf/httpd.conf`를 직접 수정하지 않는다.

기본 index 파일에는 다음 정보가 포함된다.

```text
inventory_hostname
ansible_host 또는 default_ipv4 address
distribution
distribution_version
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
PLAY [WEB 서버 구성]

TASK [Gathering Facts]
ok: [ansible2.example.com]

TASK [WEB role 실행]

TASK [web : 1) 패키지 설치 작업 포함]
included: .../roles/web/tasks/01_package.yml for ansible2.example.com

TASK [web : 1-0) CentOS 환경 확인]
skipping: [ansible2.example.com]

TASK [web : 1-1) WEB 패키지 설치 - httpd]
ok: [ansible2.example.com]

TASK [web : 2) 서비스 기동 작업 포함]
included: .../roles/web/tasks/02_service.yml for ansible2.example.com

TASK [web : 2-1) httpd 서비스 기동 및 활성화]
ok: [ansible2.example.com]

TASK [web : 3) 서비스 설정 작업 포함]
included: .../roles/web/tasks/03_config.yml for ansible2.example.com

TASK [web : 3-1) 웹 문서 루트 디렉토리 생성 - /var/www/html]
ok: [ansible2.example.com]

TASK [web : 3-2) 기본 index 파일 생성 - /var/www/html/index.html]
ok: [ansible2.example.com]

TASK [web : 4) 방화벽 등록 작업 포함]
included: .../roles/web/tasks/04_firewall.yml for ansible2.example.com

TASK [web : 4-1) firewalld http 서비스 등록 - http]
ok: [ansible2.example.com]

PLAY RECAP
ansible2.example.com : ok=..., changed=..., unreachable=0, failed=0
```

처음 실행하면 파일 생성이나 패키지 설치로 인해 일부 task가 `changed`로 표시될 수 있다. 이미 적용된 상태에서 다시 실행하면 많은 task가 `ok`로 표시된다.

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
roles/web/tests/test.yml
```

실행 명령은 다음과 같다.

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/web/tests/test.yml
```

테스트는 role을 적용한 뒤 HTTP 200 응답을 확인한다.

기본 테스트 URL은 다음과 같다.

```yaml
web_test_url: "http://{{ ansible_host | default(ansible_facts['default_ipv4']['address']) }}/"
```

네 환경에서는 다음 URL을 확인하는 것과 같다.

```text
http://192.168.10.12/
```

수동 검증 예시는 다음과 같다.

```bash
curl -I http://192.168.10.12/
```

예상 결과는 다음과 같다.

```text
HTTP/1.1 200 OK
```

## SELinux

테스트 환경에서는 SELinux가 permissive 상태였다.

이 role은 SELinux 모드를 변경하지 않는다.

기본 문서 루트 `/var/www/html`을 사용할 경우 일반적인 CentOS/RHEL Apache 기본 정책과 맞는다. 단, 문서 루트를 다른 경로로 변경하는 경우 SELinux enforcing 환경에서는 context 문제로 httpd가 파일을 읽지 못할 수 있다.

## Troubleshooting

`ansible.posix.firewalld` 관련 오류가 발생하면 collection 설치 여부를 확인한다.

```bash
ansible-galaxy collection list ansible.posix
```

없으면 설치한다.

```bash
ansible-galaxy collection install ansible.posix
```

firewalld 서비스 관련 오류가 발생하면 firewalld가 설치 및 기동되어 있는지 확인한다.

```bash
sudo systemctl status firewalld
```

httpd 서비스 상태는 다음 명령으로 확인한다.

```bash
sudo systemctl status httpd
```

80번 포트 대기 상태는 다음 명령으로 확인한다.

```bash
sudo ss -lntup | grep ':80'
```

HTTP 응답은 다음 명령으로 확인한다.

```bash
curl -I http://192.168.10.12/
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
  ansible2.example.com 192.168.10.12 WEB Server

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
