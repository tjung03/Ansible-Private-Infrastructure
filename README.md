# Private Infrastructure Ansible Project

CentOS Stream 9 기반의 사설 인프라 서버 구성을 자동화하기 위한 Ansible 프로젝트이다.

이 프로젝트는 `firewall`, `dns`, `web`, `ftp`, `mail` role을 사용해 기본 방화벽, DNS, WEB, FTP, MAIL 서버를 구성한다. 각 role은 `roles/<role_name>/README.md`에 세부 설명이 있으며, 이 문서는 프로젝트 루트에서 전체 구조와 실행 방법을 한 번에 확인하기 위한 README이다.

## Table of Contents

1. [개요](#1-개요)
2. [실습 환경](#2-실습-환경)
3. [전체 tree 구조](#3-전체-tree-구조)
4. [각 role 설명 및 반드시 변경해서 사용해야 하는 값](#4-각-role-설명-및-반드시-변경해서-사용해야-하는-값)
5. [Usage](#5-usage)
6. [저자](#6-저자)

## 1. 개요

이 프로젝트는 Ansible control node에서 여러 managed node에 접속하여 사설 인프라 서버를 자동 구성하는 것을 목표로 한다.

기본 구성 대상은 다음과 같다.

| 구분      | Inventory group | 기본 host 예시             | 기본 IP 예시        | 구성 role    |
| ------- | --------------- | ---------------------- | --------------- | ---------- |
| DNS 서버  | `dnsservers`    | `ansible1.example.com` | `192.168.10.11` | `dns`      |
| WEB 서버  | `webservers`    | `ansible2.example.com` | `192.168.10.12` | `web`      |
| FTP 서버  | `ftpservers`    | `ansible3.example.com` | `192.168.10.13` | `ftp`      |
| MAIL 서버 | `mailservers`   | `ansible4.example.com` | `192.168.10.14` | `mail`     |
| 공통 방화벽  | `all`           | 전체 managed node        | 전체 managed node | `firewall` |

전체 실행 순서는 다음과 같다.

```text
1. firewall role
2. dns role
3. web role
4. ftp role
5. mail role
```

`firewall` role은 모든 managed node에 먼저 적용된다. 이후 각 서비스 role이 필요한 firewalld service를 등록한다.

각 role의 기본 처리 범위는 다음과 같다.

```text
firewall role
  firewalld 설치 및 기동

DNS role
  bind, bind-utils 설치
  named 서비스 기동
  /etc/named.conf 구성
  zone 파일 생성
  DNS firewalld service 등록

WEB role
  httpd 설치
  httpd 서비스 기동
  기본 index.html 생성
  HTTP firewalld service 등록

FTP role
  vsftpd 설치
  익명 다운로드용 /var/ftp/pub 구성
  기본 README.txt 생성
  FTP firewalld service 등록

MAIL role
  postfix, dovecot 설치
  Postfix Maildir 구성
  Dovecot IMAP 구성
  SMTP, IMAP firewalld service 등록
```

이 프로젝트는 기본 서비스 구성에 초점을 둔다. SELinux 상세 설정, TLS 인증서 구성, DNS slave 구성, 실제 인터넷 메일 송수신 보장, 외부 웹 콘텐츠 배포 등은 포함하지 않는다.

## 2. 실습 환경

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

주의할 점은 다음과 같다.

```text
1. managed node는 SSH 접속이 가능해야 한다.
2. inventory의 ansible_host 값은 실제 managed node IP와 일치해야 한다.
3. ansible.cfg의 remote_user 값은 실제 Ansible 접속 계정과 일치해야 한다.
4. 현재 role의 OS 확인 task는 CentOS 기준으로 작성되어 있다.
5. DNS role과 MAIL role은 FQDN 형식의 inventory hostname 사용을 기본 전제로 한다.
```

필요한 Ansible collection은 다음과 같다.

```bash
ansible-galaxy collection install ansible.posix
```

`ansible.posix.firewalld` 모듈은 `dns`, `web`, `ftp`, `mail` role에서 firewalld service를 등록할 때 사용한다.

## 3. 전체 tree 구조

프로젝트 루트는 다음 위치를 기준으로 한다.

```bash
$HOME/private_infrastructure
```

전체 구조는 다음과 같다.

```text
private_infrastructure/
├── README.md
├── ansible-navigator.yml
├── ansible.cfg
├── inventory
├── playbook.yml
└── roles/
    ├── dns/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── files/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── meta/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   ├── 01_package.yml
    │   │   ├── 02_service.yml
    │   │   ├── 03_config.yml
    │   │   ├── 04_firewall.yml
    │   │   └── main.yml
    │   ├── templates/
    │   │   ├── named.conf.j2
    │   │   ├── named.zone.include.j2
    │   │   └── zone.j2
    │   ├── tests/
    │   │   └── test.yml
    │   ├── vars/
    │   │   └── main.yml
    │   ├── LICENSE
    │   └── README.md
    ├── firewall/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── files/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── meta/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   ├── 01_package.yml
    │   │   ├── 02_service.yml
    │   │   └── main.yml
    │   ├── templates/
    │   ├── tests/
    │   │   └── test.yml
    │   ├── vars/
    │   │   └── main.yml
    │   ├── LICENSE
    │   └── README.md
    ├── ftp/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── files/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── meta/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   ├── 01_package.yml
    │   │   ├── 02_service.yml
    │   │   ├── 03_config.yml
    │   │   ├── 04_firewall.yml
    │   │   └── main.yml
    │   ├── templates/
    │   ├── tests/
    │   │   └── test.yml
    │   ├── vars/
    │   │   └── main.yml
    │   ├── LICENSE
    │   └── README.md
    ├── mail/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── files/
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── meta/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   ├── 01_package.yml
    │   │   ├── 02_service.yml
    │   │   ├── 03_config.yml
    │   │   ├── 04_firewall.yml
    │   │   └── main.yml
    │   ├── templates/
    │   ├── tests/
    │   │   └── test.yml
    │   ├── vars/
    │   │   └── main.yml
    │   ├── LICENSE
    │   └── README.md
    └── web/
        ├── defaults/
        │   └── main.yml
        ├── files/
        ├── handlers/
        │   └── main.yml
        ├── meta/
        │   └── main.yml
        ├── tasks/
        │   ├── 01_package.yml
        │   ├── 02_service.yml
        │   ├── 03_config.yml
        │   ├── 04_firewall.yml
        │   └── main.yml
        ├── templates/
        ├── tests/
        │   └── test.yml
        ├── vars/
        │   └── main.yml
        ├── LICENSE
        └── README.md
```

루트 주요 파일의 역할은 다음과 같다.

| 파일                          | 설명                                                     |
| --------------------------- | ------------------------------------------------------ |
| `ansible.cfg`               | inventory 위치, 접속 사용자, role 경로, privilege escalation 설정 |
| `ansible-navigator.yml`     | ansible-navigator 실행 옵션 설정                             |
| `inventory`                 | managed node 그룹과 IP 정의                                 |
| `playbook.yml`              | 전체 role 실행 순서 정의                                       |
| `roles/*/README.md`         | 각 role의 상세 설명                                          |
| `roles/*/defaults/main.yml` | 사용자가 변경할 수 있는 기본 변수                                    |
| `roles/*/vars/main.yml`     | role 내부 고정 변수                                          |
| `roles/*/tasks/main.yml`    | role task 진입점                                          |
| `roles/*/tests/test.yml`    | role별 테스트 playbook                                     |

## 4. 각 role 설명 및 반드시 변경해서 사용해야 하는 값

### 4-1. 전체 프로젝트 공통 변경값

처음 실행하기 전에 반드시 확인해야 하는 파일은 다음과 같다.

```text
inventory
ansible.cfg
playbook.yml
ansible-navigator.yml
```

#### inventory

현재 inventory 예시는 다음과 같다.

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

반드시 변경해야 하는 값은 다음과 같다.

| 항목                     | 설명                        |
| ---------------------- | ------------------------- |
| `ansible1.example.com` | 실제 DNS 서버 hostname으로 변경   |
| `ansible2.example.com` | 실제 WEB 서버 hostname으로 변경   |
| `ansible3.example.com` | 실제 FTP 서버 hostname으로 변경   |
| `ansible4.example.com` | 실제 MAIL 서버 hostname으로 변경  |
| `ansible_host`         | 각 managed node의 실제 IP로 변경 |

DNS와 MAIL role은 inventory hostname에서 domain 부분을 자동 추출한다. 따라서 기본적으로 FQDN 형식으로 작성해야 한다.

좋은 예시는 다음과 같다.

```ini
ansible1.example.com ansible_host=192.168.10.11
```

FQDN을 사용하지 않는 경우에는 `dns_zone_name`, `mail_domain`을 직접 지정해야 한다.

#### ansible.cfg

현재 설정은 다음과 같다.

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

반드시 확인해야 하는 값은 다음과 같다.

| 항목            | 기본값         | 변경 기준                      |
| ------------- | ----------- | -------------------------- |
| `remote_user` | `ansible`   | SSH 접속 계정이 다르면 변경          |
| `inventory`   | `inventory` | inventory 파일명이 다르면 변경      |
| `roles_path`  | `roles:...` | 프로젝트 경로 또는 role 경로가 다르면 변경 |
| `become`      | `true`      | root 권한 상승을 사용하지 않는 구조면 변경 |

일반적인 실습 구성에서는 `remote_user`가 가장 중요하다. managed node에 `ansible` 계정이 없거나 다른 계정을 사용할 경우 반드시 바꿔야 한다.

#### playbook.yml

현재 전체 playbook은 다음 순서로 실행된다.

```text
FIREWALL 기본 구성 -> all
DNS 서버 구성      -> dnsservers
WEB 서버 구성      -> webservers
FTP 서버 구성      -> ftpservers
MAIL 서버 구성     -> mailservers
```

role 실행 대상 group을 바꾸려면 `hosts` 값을 수정한다. 특정 role을 전체 실행에서 제외하려면 해당 play를 주석 처리하거나 별도 playbook으로 분리한다.

#### ansible-navigator.yml

현재 설정은 다음과 같다.

```yaml
---
ansible-navigator:
  playbook-artifact:
    enable: false
```

기본 실행에는 변경이 필요하지 않다. 실행 artifact 파일을 남기고 싶으면 `enable: true`로 바꿀 수 있다.

### 4-2. firewall role

`firewall` role은 전체 managed node에 `firewalld`를 설치하고 서비스를 기동한다.

처리 범위는 다음과 같다.

```text
firewalld 패키지 설치
firewalld 서비스 started 상태 보장
firewalld 서비스 enabled 상태 보장
```

사용자가 변경할 수 있는 기본 변수는 `roles/firewall/defaults/main.yml`에 있다.

| 변수                 | 기본값    | 설명                      | 변경 필요성   |
| ------------------ | ------ | ----------------------- | -------- |
| `firewall_enabled` | `true` | 부팅 시 firewalld 자동 시작 여부 | 일반적으로 유지 |
| `firewall_started` | `true` | 현재 firewalld 실행 여부      | 일반적으로 유지 |

내부 고정 변수는 `roles/firewall/vars/main.yml`에 있다.

| 변수                      | 기본값         | 설명         |
| ----------------------- | ----------- | ---------- |
| `firewall_packages`     | `firewalld` | 설치할 패키지    |
| `firewall_service_name` | `firewalld` | 관리할 서비스 이름 |

일반적인 서버 구성에서는 `firewall_enabled`, `firewall_started`를 모두 `true`로 유지한다.

### 4-3. dns role

`dns` role은 `dnsservers` 그룹의 대상 노드에 BIND 기반 DNS 서버를 구성한다.

처리 범위는 다음과 같다.

```text
bind, bind-utils 설치
named 서비스 기동 및 활성화
/etc/named.conf 원본 1회 백업
/etc/named.conf 배포
role 전용 zone include 파일 생성
zone 파일 생성
inventory 기반 A 레코드 생성
inventory 기반 MX 레코드 생성
firewalld dns 서비스 등록
```

사용자가 변경할 수 있는 기본 변수는 `roles/dns/defaults/main.yml`에 있다.

| 변수                      | 기본값                                    | 설명                           | 변경 필요성                      |
| ----------------------- | -------------------------------------- | ---------------------------- | --------------------------- |
| `dns_forwarders`        | `8.8.8.8`                              | 외부 질의를 전달할 DNS 서버            | 내부 DNS나 다른 forwarder를 쓰면 변경 |
| `dns_zone_name`         | inventory hostname의 domain 부분          | DNS zone 이름                  | FQDN이 아니거나 도메인이 다르면 변경      |
| `dns_zone_file`         | `{{ dns_zone_name }}.zone`             | `/var/named` 아래 생성할 zone 파일명 | 보통 유지                       |
| `dns_zone_include_file` | `/etc/named.{{ dns_zone_name }}.zones` | role 전용 zone include 파일      | 보통 유지                       |
| `dns_record_groups`     | `all`                                  | A 레코드로 등록할 inventory group   | 일부 서버만 DNS에 등록하려면 변경        |
| `dns_mx_record_groups`  | `mailservers`                          | MX 레코드로 등록할 inventory group  | 메일 서버 group이 다르면 변경         |
| `dns_mx_priority`       | `10`                                   | MX priority                  | 우선순위를 다르게 줄 때 변경            |
| `dns_test_record`       | `{{ inventory_hostname }}`             | DNS 테스트 대상 레코드               | 테스트 대상을 바꿀 때 변경             |

내부 고정 변수는 `roles/dns/vars/main.yml`에 있다.

| 변수                 | 기본값                  | 설명         |
| ------------------ | -------------------- | ---------- |
| `dns_packages`     | `bind`, `bind-utils` | 설치할 패키지    |
| `dns_service_name` | `named`              | 관리할 서비스 이름 |

반드시 확인해야 하는 값은 다음과 같다.

```text
inventory의 dnsservers host
inventory의 ansible_host
inventory hostname의 FQDN 형식
dns_forwarders
dns_record_groups
dns_mx_record_groups
```

FQDN을 사용하지 않는 경우에는 다음처럼 `dns_zone_name`을 직접 지정해야 한다.

```yaml
vars:
  dns_zone_name: example.com
```

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text
/etc/named.conf.OLD
/etc/named.conf
/etc/named.example.com.zones
/var/named/example.com.zone
```

이 role은 `/etc/named.rfc1912.zones`를 직접 수정하지 않는다. 별도 include 파일을 만들어 `/etc/named.conf`에서 포함한다.

### 4-4. web role

`web` role은 `webservers` 그룹의 대상 노드에 Apache HTTP Server를 구성한다.

처리 범위는 다음과 같다.

```text
httpd 패키지 설치
httpd 서비스 기동 및 활성화
웹 문서 루트 디렉토리 생성 또는 확인
기본 index 파일 생성
firewalld http 서비스 등록
```

사용자가 변경할 수 있는 기본 변수는 `roles/web/defaults/main.yml`에 있다.

| 변수                     | 기본값                              | 설명                       | 변경 필요성                  |
| ---------------------- | -------------------------------- | ------------------------ | ----------------------- |
| `web_document_root`    | `/var/www/html`                  | 웹 문서 루트                  | 문서 루트를 바꾸면 변경           |
| `web_index_file`       | `index.html`                     | 기본 index 파일명             | 파일명을 바꾸면 변경             |
| `web_manage_index`     | `true`                           | role에서 index 파일을 생성할지 여부 | 기존 웹 콘텐츠를 보존하려면 `false` |
| `web_firewall_service` | `http`                           | firewalld에 등록할 서비스       | HTTP가 아닌 다른 서비스명을 쓰면 변경 |
| `web_test_url`         | `http://{{ ansible_host ... }}/` | 테스트 URL                  | 테스트 URL을 바꾸면 변경         |
| `web_index_content`    | HTML 기본 내용                       | 생성할 index 파일 내용          | 화면 내용을 바꾸면 변경           |

내부 고정 변수는 `roles/web/vars/main.yml`에 있다.

| 변수                 | 기본값     | 설명         |
| ------------------ | ------- | ---------- |
| `web_packages`     | `httpd` | 설치할 패키지    |
| `web_service_name` | `httpd` | 관리할 서비스 이름 |

반드시 확인해야 하는 값은 다음과 같다.

```text
inventory의 webservers host
inventory의 ansible_host
web_manage_index
```

기존 웹 콘텐츠가 있는 서버에 적용할 경우 다음 설정을 고려한다.

```yaml
vars:
  web_manage_index: false
```

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text
/var/www/html
/var/www/html/index.html
```

이 role은 기본적으로 `/etc/httpd/conf/httpd.conf`를 직접 수정하지 않는다.

### 4-5. ftp role

`ftp` role은 `ftpservers` 그룹의 대상 노드에 vsftpd 기반 FTP 서버를 구성한다.

처리 범위는 다음과 같다.

```text
vsftpd 패키지 설치
vsftpd 서비스 기동 및 활성화
/etc/vsftpd/vsftpd.conf 원본 1회 백업
익명 다운로드 설정 적용
/var/ftp/pub 디렉토리 생성
기본 README.txt 생성
firewalld ftp 서비스 등록
```

사용자가 변경할 수 있는 기본 변수는 `roles/ftp/defaults/main.yml`에 있다.

| 변수                     | 기본값                                           | 설명                  | 변경 필요성           |
| ---------------------- | --------------------------------------------- | ------------------- | ---------------- |
| `ftp_root_dir`         | `/var/ftp`                                    | FTP 기본 디렉토리         | FTP root를 바꾸면 변경 |
| `ftp_public_dir`       | `{{ ftp_root_dir }}/pub`                      | 익명 다운로드용 공개 디렉토리    | 공개 디렉토리를 바꾸면 변경  |
| `ftp_sample_file`      | `README.txt`                                  | 공개 디렉토리에 생성할 기본 파일명 | 파일명을 바꾸면 변경      |
| `ftp_sample_content`   | FTP 서버 정보                                     | 기본 파일 내용            | 표시 내용을 바꾸면 변경    |
| `ftp_firewall_service` | `ftp`                                         | firewalld에 등록할 서비스  | 보통 유지            |
| `ftp_test_url`         | `ftp://127.0.0.1/pub/{{ ftp_sample_file }}`   | 테스트 URL             | 테스트 대상을 바꾸면 변경   |
| `ftp_test_dest`        | `/tmp/ansible_ftp_test_{{ ftp_sample_file }}` | 테스트 다운로드 경로         | 보통 유지            |

내부 고정 변수는 `roles/ftp/vars/main.yml`에 있다.

| 변수                 | 기본값                       | 설명              |
| ------------------ | ------------------------- | --------------- |
| `ftp_packages`     | `vsftpd`                  | 설치할 패키지         |
| `ftp_service_name` | `vsftpd`                  | 관리할 서비스 이름      |
| `ftp_config_file`  | `/etc/vsftpd/vsftpd.conf` | vsftpd 설정 파일 경로 |

반드시 확인해야 하는 값은 다음과 같다.

```text
inventory의 ftpservers host
inventory의 ansible_host
ftp_public_dir
ftp_sample_file
ftp_sample_content
```

기본적으로 적용하는 주요 FTP 설정은 다음과 같다.

```text
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

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text
/etc/vsftpd/vsftpd.conf.OLD
/etc/vsftpd/vsftpd.conf
/var/ftp/pub
/var/ftp/pub/README.txt
```

이 role은 익명 다운로드용 FTP 서버 구성을 기준으로 한다. 로컬 사용자 업로드, 인증 기반 FTP, TLS 구성은 포함하지 않는다.

### 4-6. mail role

`mail` role은 `mailservers` 그룹의 대상 노드에 Postfix와 Dovecot 기반 MAIL 서버를 구성한다.

처리 범위는 다음과 같다.

```text
postfix, dovecot 패키지 설치
postfix, dovecot 서비스 기동 및 활성화
/etc/postfix/main.cf 원본 1회 백업
/etc/dovecot/dovecot.conf 원본 1회 백업
Postfix 기본 도메인 설정
Postfix Maildir 설정
Dovecot IMAP 설정
Dovecot Maildir 설정
firewalld smtp 서비스 등록
firewalld imap 서비스 등록
```

사용자가 변경할 수 있는 기본 변수는 `roles/mail/defaults/main.yml`에 있다.

| 변수                       | 기본값                           | 설명                    | 변경 필요성                     |
| ------------------------ | ----------------------------- | --------------------- | -------------------------- |
| `mail_domain`            | inventory hostname의 domain 부분 | 메일 도메인                | FQDN이 아니거나 도메인이 다르면 변경     |
| `mail_hostname`          | `{{ inventory_hostname }}`    | 메일 서버 hostname        | 실제 메일 hostname이 다르면 변경     |
| `mail_home_mailbox`      | `Maildir/`                    | Postfix mailbox 형식    | 보통 유지                      |
| `mail_location`          | `maildir:~/Maildir`           | Dovecot mailbox 위치    | mailbox 방식을 바꾸면 변경         |
| `mail_firewall_services` | `smtp`, `imap`                | firewalld에 등록할 서비스 목록 | submission, imaps 등을 쓰면 변경 |

내부 고정 변수는 `roles/mail/vars/main.yml`에 있다.

| 변수                              | 기본값                                | 설명                 |
| ------------------------------- | ---------------------------------- | ------------------ |
| `mail_packages`                 | `postfix`, `dovecot`               | 설치할 패키지            |
| `mail_services`                 | `postfix`, `dovecot`               | 관리할 서비스 목록         |
| `mail_postfix_config_file`      | `/etc/postfix/main.cf`             | Postfix 설정 파일      |
| `mail_dovecot_config_file`      | `/etc/dovecot/dovecot.conf`        | Dovecot 기본 설정 파일   |
| `mail_dovecot_mail_config_file` | `/etc/dovecot/conf.d/10-mail.conf` | Dovecot mail 설정 파일 |
| `mail_dovecot_auth_config_file` | `/etc/dovecot/conf.d/10-auth.conf` | Dovecot 인증 설정 파일   |

반드시 확인해야 하는 값은 다음과 같다.

```text
inventory의 mailservers host
inventory의 ansible_host
inventory hostname의 FQDN 형식
mail_domain
mail_hostname
mail_firewall_services
```

FQDN을 사용하지 않는 경우에는 다음처럼 `mail_domain`을 직접 지정해야 한다.

```yaml
vars:
  mail_domain: example.com
```

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

기본 설정 기준으로 생성 또는 수정되는 파일은 다음과 같다.

```text
/etc/postfix/main.cf.OLD
/etc/postfix/main.cf
/etc/dovecot/dovecot.conf.OLD
/etc/dovecot/dovecot.conf
/etc/dovecot/conf.d/10-mail.conf
/etc/dovecot/conf.d/10-auth.conf
```

중요한 제한 사항은 다음과 같다.

```text
1. 이 role은 메일 사용자 계정을 생성하지 않는다.
2. 이 role은 DNS A/MX 레코드를 직접 생성하지 않는다.
3. DNS A/MX 레코드는 dns role에서 생성한다.
4. 실제 인터넷 메일 송수신, SMTP 인증 세부 구성, TLS 인증서 구성은 포함하지 않는다.
```

MAIL 서버를 도메인 기준으로 사용하려면 DNS role이 먼저 실행되어야 한다. 이 프로젝트에서는 `dns` role이 `mail` role보다 먼저 실행되도록 `playbook.yml`에서 순서를 관리한다.

## 5. Usage

### 5-1. 실행 전 확인

프로젝트 루트로 이동한다.

```bash
cd $HOME/private_infrastructure
```

필요한 collection을 설치한다.

```bash
ansible-galaxy collection install ansible.posix
```

inventory를 확인한다.

```bash
ansible-navigator inventory -m stdout --list
```

playbook 문법을 확인한다.

```bash
ansible-navigator run -m stdout playbook.yml --syntax-check
```

### 5-2. 전체 실행 방법

전체 인프라 구성을 실행한다.

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout playbook.yml
```

정상 실행 시 play는 다음 순서로 진행된다.

```text
PLAY [FIREWALL 기본 구성]
PLAY [DNS 서버 구성]
PLAY [WEB 서버 구성]
PLAY [FTP 서버 구성]
PLAY [MAIL 서버 구성]
```

처음 실행하면 패키지 설치, 설정 파일 생성, 서비스 기동 때문에 여러 task가 `changed`로 표시될 수 있다. 이미 적용된 상태에서 다시 실행하면 많은 task가 `ok`로 표시된다.

출력 상태의 의미는 다음과 같다.

| 상태            | 의미                       |
| ------------- | ------------------------ |
| `ok`          | 작업을 확인했지만 변경할 내용이 없음     |
| `changed`     | 대상 시스템에 실제 변경 발생         |
| `skipping`    | `when` 조건에 따라 실행하지 않음    |
| `failed`      | 작업 실패                    |
| `unreachable` | SSH 접속 실패 또는 대상 노드 접근 불가 |

### 5-3. role별 실행 및 테스트 방법

각 role은 `roles/<role_name>/tests/test.yml`로 단독 적용 및 최소 테스트를 수행할 수 있다.

#### firewall role

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/firewall/tests/test.yml
```

적용 후 managed node에서 확인할 수 있는 명령은 다음과 같다.

```bash
systemctl status firewalld
firewall-cmd --state
```

#### dns role

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/dns/tests/test.yml
```

기본 테스트는 DNS 서버 자신에게 `dns_test_record`의 A 레코드를 질의한다.

수동 확인 예시는 다음과 같다.

```bash
dig +short @192.168.10.11 ansible1.example.com
dig +short @192.168.10.11 example.com MX
```

#### web role

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/web/tests/test.yml
```

기본 테스트는 HTTP 200 응답을 확인한다.

수동 확인 예시는 다음과 같다.

```bash
curl http://192.168.10.12/
```

#### ftp role

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/ftp/tests/test.yml
```

기본 테스트는 FTP 서버 자신에서 기본 파일을 다운로드한다.

수동 확인 예시는 다음과 같다.

```bash
curl ftp://192.168.10.13/pub/README.txt
```

#### mail role

```bash
cd $HOME/private_infrastructure
ansible-navigator run -m stdout roles/mail/tests/test.yml
```

기본 테스트는 Postfix와 Dovecot 설정 검증을 수행한다.

수동 확인 예시는 다음과 같다.

```bash
postfix check
doveconf -n
ss -lntup | grep ':25'
ss -lntup | grep ':143'
dig +short @192.168.10.11 example.com MX
```

### 5-4. 일부 role만 실행하는 방법

가장 단순한 방법은 role별 test playbook을 실행하는 것이다.

```bash
ansible-navigator run -m stdout roles/web/tests/test.yml
```

전체 `playbook.yml`을 기준으로 특정 group만 제한해서 실행할 수도 있다.

```bash
ansible-navigator run -m stdout playbook.yml --limit webservers
```

단, `mail` role은 DNS 레코드와 관계가 있으므로 전체 구성에서는 `dns` role이 먼저 적용되어 있어야 한다.

### 5-5. 변수 변경 방법

이 프로젝트의 role 변수는 기본적으로 각 role의 `defaults/main.yml`에 정의되어 있다.

```text
roles/firewall/defaults/main.yml
roles/dns/defaults/main.yml
roles/web/defaults/main.yml
roles/ftp/defaults/main.yml
roles/mail/defaults/main.yml
```

간단한 실습에서는 해당 파일을 직접 수정할 수 있다. 다만 재사용성을 고려하면 `playbook.yml`의 각 play에 `vars`를 지정하거나, 필요 시 `group_vars`, `host_vars` 구조를 추가해서 관리할 수 있다.

예를 들어 DNS forwarder를 바꾸려면 DNS play에 다음처럼 지정할 수 있다.

```yaml
- name: DNS 서버 구성
  hosts: dnsservers
  gather_facts: true
  vars:
    dns_forwarders:
      - 192.168.10.2
      - 8.8.8.8
  tasks:
    - name: DNS role 실행
      ansible.builtin.include_role:
        name: dns
```

기존 WEB index 파일을 보존하려면 WEB play에 다음처럼 지정할 수 있다.

```yaml
- name: WEB 서버 구성
  hosts: webservers
  gather_facts: true
  vars:
    web_manage_index: false
  tasks:
    - name: WEB role 실행
      ansible.builtin.include_role:
        name: web
```

### 5-6. 전체 실행 후 기본 확인 명령

전체 실행 후 control node 또는 각 managed node에서 다음 항목을 확인한다.

```bash
# inventory 확인
ansible-navigator inventory -m stdout --list

# DNS 확인
dig +short @192.168.10.11 ansible1.example.com
dig +short @192.168.10.11 ansible2.example.com
dig +short @192.168.10.11 example.com MX

# WEB 확인
curl http://192.168.10.12/

# FTP 확인
curl ftp://192.168.10.13/pub/README.txt

# MAIL 확인
ssh ansible4.example.com 'sudo postfix check'
ssh ansible4.example.com 'sudo doveconf -n'
ssh ansible4.example.com "sudo ss -lntup | grep ':25'"
ssh ansible4.example.com "sudo ss -lntup | grep ':143'"
```

## 6. 저자

```text
Author: tjung03
License: MIT
```

각 role의 상세 설명은 다음 파일에서 확인한다.

```text
roles/firewall/README.md
roles/dns/README.md
roles/web/README.md
roles/ftp/README.md
roles/mail/README.md
```
