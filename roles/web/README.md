# WEB Role

CentOS Stream 9의 `webservers`에 Apache HTTP Server를 설치하고 기본 웹 페이지를 배포한다.

## 적용 동작

[Task 진입점](tasks/main.yml)은 CentOS 검사·httpd 설치 → 서비스 기동·자동 시작 → 문서 디렉터리·페이지 생성 → firewalld `http` 등록 순서다.

[페이지 배포](tasks/03_config.yml)는 `copy`로 처리하며, 기존 파일 변경 시 백업을 생성한다. 기본 페이지는 호스트명·IP·OS 정보를 표시한다. Apache 설정 파일을 수정하지 않으므로 재시작 Handler도 사용하지 않는다.

## 변수

| [기본 변수](defaults/main.yml) | 기본값 | 의미 |
|---|---|---|
| `web_document_root` | `/var/www/html` | 생성할 문서 디렉터리 |
| `web_index_file` | `index.html` | 배포할 파일명 |
| `web_manage_index` | `true` | false이면 페이지 복사를 생략 |
| `web_index_content` | 호스트·IP·OS를 표시하는 HTML | 페이지 내용 |
| `web_firewall_service` | `http` | firewalld 서비스 |
| `web_test_url` | 대상 IP의 `http://.../` | HTTP 점검 URL |

[내부 변수](vars/main.yml)의 패키지와 서비스명은 `httpd`다. `web_test_url`의 주소는 `ansible_host`를 우선하고, 없으면 기본 IPv4 Fact를 사용한다.

`web_manage_index: false`는 페이지 복사만 생략한다. 디렉터리 생성·권한 관리, 패키지·서비스·방화벽 적용은 계속 수행한다.

`web_document_root`는 파일 배포 경로이며 Apache의 `DocumentRoot` 설정을 변경하지 않는다. 기본 경로를 바꾸면 Apache 설정과 SELinux Context를 별도로 맞춰야 한다. 파일명 변경 역시 Apache의 기본 인덱스 설정을 변경하지 않는다.

## 실행과 검증

[공통 실행 조건](../../README.md#실행-조건)을 준비하고 저장소 루트에서 실행한다.

```bash
ansible-navigator run -m stdout playbook.yml --limit webservers
ansible-navigator run -m stdout roles/web/tests/test.yml
```

첫 명령은 WEB 노드에 firewall과 web을 적용한다. [테스트](tests/test.yml)는 web을 재적용한 뒤 **WEB 노드에서** `uri` 모듈로 HTTP 200을 검사한다. 응답 본문은 반환하지만 특정 내용과 일치하는지 검사하지 않는다.

별도 클라이언트에서의 접근 확인:

```bash
curl -I http://192.168.10.12/
```

HTTP 200이 응답 상태 판단 기준이다. 실패하면 WEB 서버에서 서비스·포트·방화벽을 확인한다.

```bash
sudo systemctl status httpd
sudo ss -lntup
sudo firewall-cmd --query-service=http
```

원격 클라이언트 접근 성공과 현재 환경의 실행 결과는 테스트 코드 존재만으로 보장되지 않는다.

## 범위

CentOS만 허용하며 firewalld는 사전에 설치·기동되어 있어야 한다. 가상 호스트, HTTPS/TLS, Apache 튜닝, 외부 콘텐츠 배포, SELinux 모드·Context 설정은 포함하지 않는다. 기존 실습 환경의 SELinux는 permissive다.

작성자: tjung03 · [MIT License](LICENSE)
