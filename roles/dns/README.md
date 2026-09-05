# DNS Role

CentOS Stream 9에 BIND를 설치하고 Inventory에서 A·MX 레코드를 생성한다. 적용 대상은 `dnsservers`다.

## 구성 방식

[Task 진입점](tasks/main.yml)은 패키지 설치 → named 기동 → 설정 → firewalld 등록 순서로 실행된다.

[설정 Task](tasks/03_config.yml)는 다음 순서로 처리한다.

1. Zone 이름이 비어 있으면 중단한다.
2. 기존 `/etc/named.conf`를 `.OLD`로 복사한다. 이미 있는 백업은 덮어쓰지 않는다.
3. [Zone 템플릿](templates/zone.j2)을 생성하고 `named-checkzone`으로 검사한다.
4. [전용 Include](templates/named.zone.include.j2)를 생성한다.
5. [named 설정](templates/named.conf.j2)을 `named-checkconf`로 검사한 뒤 배포한다.
6. 세 템플릿 중 변경된 파일이 있으면 named를 한 번 재시작한다.

`/etc/named.rfc1912.zones`는 수정하지 않고 별도 Include를 추가한다. named 설정 배포에는 `backup: true`도 사용한다. 검사는 개별 파일 배포 시 수행되며, 뒤 단계 실패 시 앞서 변경한 파일까지 자동 Rollback하는 구성은 아니다.

## 변수와 Inventory

[기본 변수](defaults/main.yml)는 다음과 같다.

| 변수 | 기본값 | 의미 |
|---|---|---|
| `dns_forwarders` | `[8.8.8.8]` | 외부 질의 전달 대상 |
| `dns_zone_name` | Inventory 호스트명의 첫 점 뒤 부분 | 기본 `example.com`; 빈 값이면 실패 |
| `dns_zone_file` | `{{ dns_zone_name }}.zone` | `/var/named` 아래 Zone 파일 |
| `dns_zone_include_file` | `/etc/named.{{ dns_zone_name }}.zones` | 전용 Include 경로 |
| `dns_record_groups` | `[all]` | A 레코드 생성 대상 |
| `dns_mx_record_groups` | `[mailservers]` | MX 대상 호스트 그룹 |
| `dns_mx_priority` | `10` | MX 우선순위 |
| `dns_test_record` | `{{ inventory_hostname }}` | 테스트할 Inventory 호스트 |

[내부 변수](vars/main.yml)는 패키지 `bind`·`bind-utils`와 서비스 `named`를 정의한다.

기본 Inventory에서 `ansible1.example.com`은 `example.com` Zone의 DNS 서버이고, `ansible4.example.com`은 MX 대상이다. A 레코드는 `ansible_host`가 있는 호스트만 생성하며, 레코드 이름은 호스트명의 첫 부분이다.

따라서 단일 도메인의 FQDN과 IPv4 `ansible_host`를 사용하는 기본 구성을 전제로 한다. 다른 도메인에 같은 짧은 호스트명이 있으면 레코드가 충돌할 수 있다. `dns_zone_name` 지정만으로 SOA·NS·MX에 사용되는 호스트명까지 보정되지는 않는다. 그룹을 겹쳐 지정하면 레코드가 중복될 수 있다.

Zone Serial은 수집한 날짜의 `YYYYMMDD01`로 생성한다. 날짜가 바뀌면 Zone 변경과 재시작이 발생할 수 있고, 같은 날짜의 재실행에서는 Serial이 증가하지 않는다. Secondary DNS 동기화용 Serial 관리 기능은 제공하지 않는다.

## 실행과 확인

[공통 실행 조건](../../README.md#실행-조건)을 준비하고, firewalld를 먼저 설치·기동한다. 모든 명령은 저장소 루트 기준이다.

```bash
ansible-navigator run -m stdout playbook.yml --limit dnsservers
ansible-navigator run -m stdout roles/dns/tests/test.yml
```

첫 명령은 DNS 노드에 firewall과 dns를 적용한다. 두 번째 [테스트](tests/test.yml)는 dns를 재적용한 뒤 **DNS 노드에서** A 질의를 실행하고, 응답에 해당 Inventory 호스트의 `ansible_host`가 포함되는지 검사한다.

`dns_test_record`는 `ansible_host`가 정의된 Inventory 호스트여야 한다. 임의의 외부 도메인을 지정하는 테스트가 아니다.

```bash
ansible-navigator run -m stdout roles/dns/tests/test.yml -e dns_test_record=ansible2.example.com
```

별도 클라이언트에서 A·MX를 확인하는 절차는 다음과 같다. 주소는 기본 Inventory 기준이며, `dig`가 필요하다.

```bash
dig +short @192.168.10.11 ansible1.example.com
dig +short @192.168.10.11 example.com MX
```

기대값은 각각 `192.168.10.11`, `10 ansible4.example.com.`이다. 실제 실행 결과가 아닌 응답 판단 기준이다. 서버에서 설정·서비스 상태를 좁혀 확인할 때는 다음 명령을 사용한다.

```bash
sudo named-checkconf /etc/named.conf
sudo named-checkzone example.com /var/named/example.com.zone
sudo systemctl status named
sudo firewall-cmd --query-service=dns
```

## 범위와 제한

- CentOS만 허용하는 OS 검사이며, RHEL 계열 전체에 대한 호환성 검증은 없다.
- firewalld 설치·기동은 이 Role에 포함되지 않는다. `meta/main.yml`의 dependency도 비어 있다.
- `listen-on`·`allow-query`는 `any`, 재귀 질의는 활성화한다. 접근 범위를 제한하는 운영용 ACL 구성은 포함하지 않는다.
- 역방향 Zone, Secondary DNS, Dynamic DNS, 클라이언트 Resolver 설정은 구성하지 않는다.
- SELinux 모드·Context를 변경하지 않는다. 기존 실습 환경은 permissive이며 경로 변경·enforcing 환경은 별도 확인 대상이다.

작성자: tjung03 · [MIT License](LICENSE)
