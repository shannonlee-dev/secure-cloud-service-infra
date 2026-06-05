# 미션 6.1 AWS 웹 서비스 인프라 구축

## 외부 접속 정보

- 검증 방식: A, 브라우저로 공개 웹 서비스 접속
- 퍼블릭 IPv4: `13.125.199.198`
- HTTP URL: `http://13.125.199.198`
- 도메인: `cody-aws-web.duckdns.org`
- HTTPS URL: `https://cody-aws-web.duckdns.org`
- 검증 결과: 브라우저에서 nginx 시작 페이지가 정상 표시되었고, 터미널 확인에서도 내부 `localhost`와 외부 HTTPS 요청이 HTTP 200으로 응답했다.

## 환경 요약

| 항목 | 값 |
|---|---|
| AWS 리전 | `ap-northeast-2` 서울 |
| 가용 영역 | `ap-northeast-2a` |
| EC2 인스턴스 | `mission6-web-ec2`, 인스턴스 ID `i-055ca0e81bf05d493` |
| 인스턴스 타입 | `t3.micro` |
| 운영체제 | Ubuntu 24.04.4 LTS |
| 프라이빗 IPv4 | `10.0.1.60` |
| 퍼블릭 IPv4 | `13.125.199.198` |
| 웹 서버 | nginx |
| Docker 이미지 | `nginx:alpine` |
| Docker 포트 매핑 | `0.0.0.0:80->80/tcp`, `[::]:80->80/tcp` |
| 도메인 | `cody-aws-web.duckdns.org` |

## 네트워크 아키텍처

인프라는 서울 리전의 `mission6-vpc` 안에 구성했다.

- VPC 안에 `ap-northeast-2a`의 Public Subnet 1개를 배치했다.
- Internet Gateway를 VPC에 연결했다.
- Public Subnet의 Route Table은 `0.0.0.0/0` 트래픽을 Internet Gateway로 전달한다.
- EC2 인스턴스는 Public Subnet에 배치했고, 프라이빗 IP와 퍼블릭 IPv4를 모두 가진다.
- 외부 사용자는 도메인 또는 퍼블릭 IP로 접속하고, 요청은 Internet Gateway, Public Subnet Route Table, Security Group을 거쳐 EC2의 nginx 서비스까지 도달한다.

구성도는 `docs/architecture.png`에 정리했다.

## Security Group과 IAM

Security Group은 EC2 인스턴스로 들어오는 네트워크 트래픽을 제어한다. IAM은 실습 사용자 또는 Role이 AWS API로 어떤 리소스를 생성, 조회, 태그 지정, 연결, 삭제할 수 있는지 제어한다. 즉 Security Group은 서버 접속 경로를 제한하고, IAM은 AWS 리소스 조작 권한을 제한한다.

인바운드 규칙은 필요한 서비스 포트만 허용했다.

| 포트 | 소스 | 목적 |
|---:|---|---|
| 80/tcp | `0.0.0.0/0` | 공개 HTTP 웹 접속 |
| 443/tcp | `0.0.0.0/0` | 보너스 HTTPS 웹 접속 |
| 22/tcp | `My IP/32` | 현재 작업자 IP에서만 SSH 관리 접속 허용 |

전체 트래픽 허용 규칙이나 `0.0.0.0/0` 전체 포트 개방 규칙은 사용하지 않았다. `docs/troubleshooting.md`에 기록한 SSH 접속 실패는 SSH를 전체 공개로 여는 대신, 실제 접속을 수행한 PC/WSL 환경의 공인 IP로 SSH Source CIDR을 바로잡아 해결했다.

작업은 AWS 루트 계정이 아니라 별도로 준비한 실습용 IAM 사용자 또는 Role로 수행했다. 권한 범위는 EC2, VPC, Subnet, Route Table, Internet Gateway, Security Group, 태그, 연결 작업처럼 실습에 필요한 범위로 제한했고, S3/RDS 같은 무관 서비스 권한이나 `AdministratorAccess`는 사용하지 않는 구성을 목표로 했다.

## 배포 및 검증 증거

| 증거 | 파일 |
|---|---|
| Ubuntu EC2 SSH 접속 성공 | `screenshots/01-ssh-success.png` |
| EC2 내부 `curl -I http://localhost`가 `HTTP/1.1 200 OK` 반환 | `screenshots/02-localhost-http-200.png` |
| 브라우저에서 `http://13.125.199.198` 접속 시 nginx 페이지 표시 | `screenshots/03-http-ip-browser.png` |
| DuckDNS 도메인이 `13.125.199.198`로 해석됨 | `screenshots/04-domain-dns-lookup.png` |
| 브라우저에서 `http://cody-aws-web.duckdns.org` 접속 시 nginx 페이지 표시 | `screenshots/05-http-domain-browser.png` |
| HTTP 도메인 접속이 HTTPS로 `301 Moved Permanently` 리다이렉트됨 | `screenshots/06-http-to-https-redirect.png` |
| Certbot으로 HTTPS 인증서 발급 및 nginx 배포 성공 | `screenshots/07-certbot-success.png` |
| `curl -I https://cody-aws-web.duckdns.org`가 `HTTP/1.1 200 OK` 반환 | `screenshots/08-https-terminal-200.png` |
| 브라우저에서 HTTPS URL 접속 시 자물쇠 아이콘과 nginx 페이지 표시 | `screenshots/09-https-browser.png` |
| `docker ps`에서 `nginx:alpine` 컨테이너가 실행 중이고 80 포트가 매핑됨 | `screenshots/10-docker-ps.png` |

## Docker and HTTPS 적용

- `cody-aws-web.duckdns.org` 도메인에 Let's Encrypt HTTPS 인증서를 적용했다.
- EC2 인스턴스에 Docker를 설치했다.
- `nginx:alpine` 컨테이너 이미지로 웹 서비스를 실행했다.
- 포트 매핑은 `0.0.0.0:80->80/tcp`, `[::]:80->80/tcp`로 확인했다.
- Docker 실행 증거와 외부 접속 증거는 `screenshots/10-docker-ps.png`, `screenshots/09-https-browser.png`에 포함했다.

## 리소스 정리

검증 후 실습 리소스를 삭제했다. 상세 내용은 `docs/cleanup-checklist.md`에 정리했다.

정리 증거는 다음과 같다.

- EC2 종료 작업 화면: `screenshots/11-ec2-terminate-action.png`
- EC2 종료 완료 상태: `screenshots/12-ec2-terminated.png`
- VPC 및 관련 리소스 삭제 성공: `screenshots/13-vpc-mission6-delete-success.png`
- 삭제 후 VPC 목록 비어 있음: `screenshots/15-vpc-list-empty-after-delete.png`
- 키페어 삭제 성공: `screenshots/17-keypair-delete-success.png`

## 리소스 추적 기준

이번 실습에서는 AWS 리소스를 정리할 때 **이름 규칙(Name tag)** 을 기준으로 추적했다.

공통적으로 `mission6-` 접두어를 사용하여 실습 리소스를 구분했다.

| 리소스              | 이름                       |
| ---------------- | ------------------------ |
| VPC              | `mission6-vpc`           |
| Public Subnet    | `mission6-public-subnet` |
| Internet Gateway | `mission6-igw`           |
| Route Table      | `mission6-public-rt`     |
| Security Group   | `mission6-web-sg`        |
| EC2 Instance     | `mission6-web-ec2`       |
| Key Pair         | `mission6-key`           |
| Docker Container | `mission6-nginx`         |

별도의 공통 태그를 일괄 부여하지는 않았지만, 모든 주요 리소스 이름에 `mission6-` 접두어를 붙여 실습 리소스와 기본 리소스를 구분했다. 정리 단계에서는 EC2, EBS Volume, Elastic IP, Internet Gateway, Subnet, Route Table, Security Group, VPC 순서로 `mission6-` 관련 리소스가 남아 있는지 확인했다.

---

## Public Subnet Route Table

Public Subnet에 배치된 EC2가 외부 인터넷과 통신하려면 Route Table에 기본 인터넷 경로가 필요하다.

이번 구성에서는 Route Table `mission6-public-rt`에 다음 경로를 설정했다.

| Destination   | Target         | 의미        |
| ------------- | -------------- | --------- |
| `10.0.0.0/16` | `local`        | VPC 내부 통신 |
| `0.0.0.0/0`   | `mission6-igw` | 외부 인터넷 통신 |

`10.0.0.0/16 → local`은 VPC 내부 주소 간 통신을 위한 기본 경로이다. 반면 `0.0.0.0/0 → mission6-igw`는 VPC 내부 범위가 아닌 모든 외부 목적지로 나가는 트래픽을 Internet Gateway로 보내기 위한 경로이다.

이 경로가 없으면 EC2에 Public IP가 있더라도 외부 인터넷으로 나가거나, 외부 사용자가 HTTP/HTTPS로 접근하는 흐름이 정상적으로 완성되지 않는다. 따라서 Public Subnet이 실제로 인터넷과 연결되려면 Internet Gateway 연결뿐 아니라 Route Table의 `0.0.0.0/0 → IGW` 설정이 함께 필요하다.

---

## 외부 접속 실패 시 점검 순서

외부에서 `http://<Public IP>` 또는 `https://cody-aws-web.duckdns.org` 접속이 실패할 경우, 다음 순서로 점검한다.

#### 1단계. 라우팅 확인

먼저 네트워크 경로가 열려 있는지 확인한다.

* EC2가 `mission6-public-subnet`에 배치되어 있는지 확인한다.
* Public Subnet에 연결된 Route Table이 `mission6-public-rt`인지 확인한다.
* Route Table에 `0.0.0.0/0 → mission6-igw` 경로가 있는지 확인한다.
* Internet Gateway `mission6-igw`가 `mission6-vpc`에 attach 되어 있는지 확인한다.

라우팅이 잘못되어 있으면 서버 프로세스가 정상이어도 외부 트래픽이 EC2까지 도달하지 못한다.

#### 2단계. Security Group 확인

다음으로 EC2 앞단 방화벽 역할을 하는 Security Group을 확인한다.

`mission6-web-sg`의 인바운드 규칙은 다음과 같이 구성했다.

| Type  | Port | Source      | 목적             |
| ----- | ---: | ----------- | -------------- |
| HTTP  |   80 | `0.0.0.0/0` | 외부 HTTP 접속 허용  |
| HTTPS |  443 | `0.0.0.0/0` | 외부 HTTPS 접속 허용 |
| SSH   |   22 | `My IP/32`  | 관리자 접속 제한      |

외부 웹 접속이 안 될 경우 HTTP 80 또는 HTTPS 443 규칙이 누락되었는지 먼저 확인한다. SSH 접속 실패 시에는 SSH 22번 Source가 실제 접속 중인 PC의 공인 IP와 일치하는지 확인한다.

#### 3단계. Public IP와 DNS 확인

EC2에 Public IPv4 주소가 할당되어 있는지 확인한다.

또한 DuckDNS 도메인을 사용하는 경우 다음 명령으로 도메인이 EC2 Public IP를 가리키는지 확인한다.

```bash
nslookup cody-aws-web.duckdns.org
```

결과 IP가 EC2 Public IPv4와 다르면 DuckDNS의 current IP 값을 EC2 Public IP로 수정한 뒤 다시 확인한다.

#### 4단계. 서버 프로세스와 로그 확인

라우팅, Security Group, DNS가 정상이라면 EC2 내부의 웹서버 상태를 확인한다.

nginx 직접 설치 버전에서는 다음을 확인한다.

```bash
sudo systemctl status nginx --no-pager
curl -I http://localhost
sudo nginx -t
```

Docker 버전에서는 다음을 확인한다.

```bash
sudo docker ps
curl -I http://localhost
```

필요하면 nginx 로그도 확인한다.

```bash
sudo tail -n 50 /var/log/nginx/access.log
sudo tail -n 50 /var/log/nginx/error.log
```

이 순서를 따르면 네트워크 경로 문제, 방화벽 문제, DNS 문제, 서버 프로세스 문제를 단계적으로 분리해서 확인할 수 있다.

---

## IAM 권한 부족 시 최소권한 원칙에 따른 대응 방식

IAM 권한 부족 오류가 발생하면 권한을 무작정 `AdministratorAccess`로 올리지 않고, 필요한 권한 범위를 좁혀서 확인한다.

예상되는 오류 메시지는 다음과 같다.

```text
AccessDenied
UnauthorizedOperation
You are not authorized to perform this operation
```

대응 순서는 다음과 같다.

1. 오류 메시지에서 실패한 AWS Action을 확인한다.
   예: `ec2:CreateVpc`, `ec2:RunInstances`, `ec2:AuthorizeSecurityGroupIngress`

2. CloudTrail Event history에서 실패한 API 호출을 확인한다.
   어떤 사용자 또는 Role이 어떤 API를 호출했고, 어떤 권한 부족으로 실패했는지 확인한다.

3. IAM Policy Simulator를 사용해 현재 IAM 사용자 또는 그룹 정책으로 해당 Action이 허용되는지 테스트한다.

4. 필요한 경우 정책을 추가하되, 전체 관리자 권한을 부여하지 않고 실습에 필요한 EC2/VPC/Security Group 관련 권한만 추가한다.

5. 실습과 무관한 S3, RDS, Lambda, Billing 관리 권한은 부여하지 않는다.

이번 실습에서는 루트 계정 대신 `mission6-user` IAM 사용자를 사용했고, 실습 목적에 맞게 EC2/VPC 중심의 권한을 부여했다. IAM 권한 문제 발생 시에는 전체 관리자 권한으로 우회하지 않고, CloudTrail과 정책 시뮬레이터를 통해 부족한 최소 Action을 확인한 뒤 제한적으로 보완하는 방식을 원칙으로 한다.