# 클라우드 실습 재현 안내

## 준비와 네트워크

아래 절차는 자신의 **새 Ubuntu 24.04 실습 EC2**에서 수행할 안내입니다. 저장소 검사와 CI는 이 명령을 실행하지 않습니다. 기존 배포 기록의 IP·도메인·리소스 ID는 과거 값이며 재사용하지 않습니다.

AWS 콘솔에서 실습용 IAM 사용자/Role, 키페어, VPC `10.0.0.0/16`, Public Subnet `10.0.1.0/24`, Internet Gateway를 준비합니다. Subnet에 연결된 Route Table은 VPC local 경로와 `0.0.0.0/0 → IGW`를 가져야 합니다. EC2에 공개 IP를 할당하고 Security Group은 HTTP 80·HTTPS 443을 서비스 범위에, SSH 22는 실제 관리 PC의 공인 IPv4 `/32`에 허용합니다. 아래 Docker 기본 단계에는 공개 8080 규칙이 필요 없습니다.

작업에 사용한 계정/Role과 정책 Action·Resource 범위를 기록합니다. IAM 허용 여부는 실패한 작업·연결 정책·CloudTrail 등 실제 증거로 확인합니다. 자격 증명이나 키 내용을 증빙에 넣지 않습니다.

관리 PC에서 자신의 값으로 SSH 접속합니다.

```bash
read -r -p '새 EC2 공개 IP: ' lab_public_ip
read -r -p '자신의 키 파일 경로: ' lab_key_file
chmod 400 "$lab_key_file"
ssh -i "$lab_key_file" "ubuntu@${lab_public_ip}"
```

이후 설치·설정 명령은 EC2 안에서 실행합니다. 패키지 설치와 인증서 발급은 자신의 서버와 외부 패키지/인증 서비스에 영향을 주는 수동 실습입니다.

## 단계 A: 호스트 nginx HTTP·HTTPS

```bash
sudo apt-get update
sudo apt-get install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
sudo nginx -t
curl -I http://127.0.0.1
lsb_release -ds
nginx -v
certbot --version
```

자신이 관리하는 도메인의 A 레코드를 새 EC2 공개 IP로 설정합니다. IPv6를 구성하지 않았다면 해당 도메인에 잘못된 AAAA 레코드가 없어야 합니다. 실제 공개 IP와 DNS 결과를 비교한 뒤 도메인을 입력합니다.

```bash
read -r -p '자신이 관리하는 DNS 이름: ' lab_domain
getent ahostsv4 "$lab_domain"
sudo certbot --nginx -d "$lab_domain"
sudo nginx -t
curl -I "http://${lab_domain}"
curl -I "https://${lab_domain}"
sudo systemctl list-timers --all # 목록에서 certbot 관련 갱신 작업을 확인
```

Certbot의 이메일·약관·인증 안내를 완료합니다. 설치한 Certbot 버전이 HTTP 리다이렉트 선택을 제공하면 HTTPS 리다이렉트를 선택하고, 그렇지 않으면 생성된 nginx 설정을 확인합니다. HTTP가 HTTPS로 리다이렉트되는지와 HTTPS 200 응답을 각각 기록합니다. 인증서 경로는 호스트 `/etc/letsencrypt/live/<자신의 도메인>/`이며 적용된 호스트 nginx 설정도 확인합니다.

관리 PC에서도 자신의 도메인으로 HTTP·HTTPS 응답과 브라우저 인증서 정보를 확인합니다. 인증서 갱신 검증이 필요하면 자신의 실습 서버에서 `sudo certbot renew --dry-run`을 별도 실행하고 결과를 기록합니다. 인증서 개인키 내용은 출력하지 않습니다.

## 단계 B: 포트 충돌 없는 Docker HTTP 확인

호스트 nginx HTTPS를 유지하며 컨테이너는 EC2 내부의 `127.0.0.1:8080`에만 노출합니다. 이 단계는 Docker HTTP 실행 검증이며 컨테이너 HTTPS나 공개 컨테이너 서비스를 의미하지 않습니다.

```bash
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo docker pull nginx:alpine
sudo docker image inspect nginx:alpine --format '{{json .RepoDigests}}'
sudo docker run -d --name lab-nginx -p 127.0.0.1:8080:80 nginx:alpine
sudo docker ps --filter name=lab-nginx
curl -I http://127.0.0.1:8080
sudo docker logs --tail 30 lab-nginx
docker --version
```

동일한 이미지를 다시 사용하려면 출력된 이미지 digest를 기록하고 다음 실행에서 `nginx@sha256:<기록한 digest>`를 사용합니다. 태그와 apt 저장소 버전은 시간에 따라 달라지므로 OS·nginx·Docker·Certbot 버전과 이미지 digest를 실행 기록에 함께 남깁니다. 같은 이름의 컨테이너가 이미 있으면 새로 실행하기 전에 상태를 확인합니다.

### 과거 기록의 Docker 80 포트 매핑을 재현할 때

과거 화면의 `0.0.0.0:80 → 80`을 재현하려면 단계 A를 마친 뒤 호스트 nginx를 중지해야 합니다. 이 시간에는 호스트 HTTPS도 중단됩니다. 아래는 자신의 실습 컨테이너만 정리하고 HTTP 80 실습 후 호스트 nginx로 돌아오는 순서입니다.

```bash
sudo docker stop lab-nginx
sudo docker rm lab-nginx
sudo systemctl stop nginx
sudo ss -ltnp
sudo docker run -d --name lab-nginx -p 80:80 nginx:alpine
curl -I http://127.0.0.1
sudo docker ps --filter name=lab-nginx
# 관리 PC에서 자신의 새 공개 IP로 HTTP를 확인한 뒤 EC2에서 복원합니다.
sudo docker stop lab-nginx
sudo docker rm lab-nginx
sudo systemctl start nginx
sudo nginx -t
curl -I "https://${lab_domain}"
```

컨테이너에 443 매핑·TLS 설정·인증서가 없으므로 이 전환을 Docker HTTPS 구성으로 기록하지 않습니다. [현재 단계 구분 구성도](architecture.svg)와 [과거 배포 기록](deployment-record.md)을 함께 참고합니다.

## 정리와 증빙

[정리 체크리스트](cleanup-checklist.md)에 따라 자신의 실습 리소스 ID·리전·조회 시각과 삭제 결과를 기록합니다. EC2 종료 화면과 EBS 삭제 화면, Elastic IP 미할당/반납 결과는 따로 확인합니다. IAM 실제 정책 증빙이 없으면 최소권한 적용을 완료로 표기하지 않습니다.

현재 리소스 삭제는 문서 검사나 CI에서 수행하지 않습니다. 과거 실습의 EBS·Elastic IP·IAM 상태가 미확인이라는 사실과 새 실습의 확인 결과를 섞지 않습니다. 접속 실패는 [오류 분석](troubleshooting.md)의 라우팅·보안 그룹·DNS·서버 점검 순서를 따릅니다.
