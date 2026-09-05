# FIREWALL Role

CentOS Stream 9의 Managed Node에 firewalld를 설치하고 실행·부팅 시 자동 시작 상태를 관리한다. 전체 Playbook에서는 모든 Inventory 호스트에 먼저 적용된다.

## 역할과 구현

[Task 진입점](tasks/main.yml)은 CentOS 검사와 패키지 설치 후 [서비스 상태](tasks/02_service.yml)를 적용한다. built-in 모듈만 사용하며, Handler와 Role dependency는 사용하지 않는다.

개별 서비스의 허용 규칙은 이 Role이 아닌 각 서비스 Role에서 등록한다.

| Role | 등록하는 firewalld 서비스 |
|---|---|
| dns | `dns` |
| web | `http` |
| ftp | `ftp` |
| mail | `smtp`, `imap` |

서비스 Role의 등록 Task는 `permanent: true`, `immediate: true`를 사용한다. Zone을 명시하지 않으므로 인터페이스의 활성 Zone과 등록 대상도 적용 환경에서 확인해야 한다. 기존 규칙을 초기화하거나 불필요한 서비스를 제거하는 기능은 없다.

## 변수

| [기본 변수](defaults/main.yml) | 기본값 | 동작 |
|---|---|---|
| `firewall_enabled` | `true` | 부팅 시 자동 시작 |
| `firewall_started` | `true` | 현재 실행; false이면 중지 |

[내부 변수](vars/main.yml)는 설치 패키지와 서비스명을 `firewalld`로 정의한다. 서비스 Role이 뒤에서 실행되는 전체 구성에서는 위 두 기본값을 유지한다. 특히 `firewall_started: false`이면 후속 firewalld 등록 Task의 실행 전제와 충돌한다.

## 실행과 확인

[공통 실행 조건](../../README.md#실행-조건)을 준비하고 저장소 루트에서 실행한다.

```bash
ansible-navigator run -m stdout roles/firewall/tests/test.yml
```

이 [테스트 Playbook](tests/test.yml)은 전체 Managed Node에 Role을 재적용한다. 별도 검증 Task는 없다.

대상 서버에서 현재 상태와 등록 상태를 확인하는 절차:

```bash
sudo systemctl is-active firewalld
sudo systemctl is-enabled firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-services
sudo firewall-cmd --permanent --list-services
```

기본값을 적용했다면 서비스 상태는 `active`, 자동 시작은 `enabled`가 판단 기준이다. 서비스 목록은 각 서비스 Role 적용 이후 확인한다. 이 절차 자체가 실행 성공 기록을 의미하지는 않는다.

## 범위

CentOS만 허용하는 OS 검사를 수행한다. VM·SSH 계정 준비, SELinux 변경, Zone 설계, Rich Rule, 개별 포트 등록은 포함하지 않는다. 전체 순서와 환경 정보는 [루트 README](../../README.md)에 있다.

작성자: tjung03 · [MIT License](LICENSE)
