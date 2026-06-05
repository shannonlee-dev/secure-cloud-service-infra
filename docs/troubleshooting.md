# 트러블슈팅 보고서

## 문제 1. SSH 접속 실패: Security Group의 SSH 허용 IP 불일치

### 1. 증상

EC2 인스턴스를 생성한 뒤 WSL 환경에서 SSH 접속을 시도했지만 연결되지 않았다.

실행한 명령은 다음과 같다.

```bash
ssh -o ConnectTimeout=10 -i mission6-key.pem ubuntu@13.125.199.198
```

출력 결과는 다음과 같았다.

```text
ssh: connect to host 13.125.199.198 port 22: Connection timed out
```

이 에러는 인증 키 문제라기보다는, 클라이언트에서 EC2 인스턴스의 22번 포트까지 네트워크 연결이 도달하지 못하는 상황으로 판단했다.

---

### 2. 원인 가설

EC2에 연결된 Security Group의 SSH 인바운드 규칙에서 소스를 `My IP`로 설정했지만, 이 설정을 모바일 환경에서 진행했기 때문에 모바일 네트워크의 공인 IP가 등록되었을 가능성이 있다고 판단했다.

실제 SSH 접속은 Windows WSL 환경에서 시도했으므로, 모바일에서 등록된 IP와 Windows PC가 사용하는 공인 IP가 달라 22번 포트 접근이 차단되었을 가능성이 있었다.

---

### 3. 검증 방법

AWS 콘솔에서 EC2 인스턴스에 연결된 Security Group을 확인했다.

확인 경로는 다음과 같다.

```text
EC2
→ 인스턴스
→ mission6-web-ec2 선택
→ 보안 탭
→ mission6-web-sg 선택
→ 인바운드 규칙 확인
```

확인 결과, SSH 규칙은 존재했지만 소스가 현재 Windows WSL에서 접속하는 네트워크의 공인 IP와 일치하지 않는 것으로 판단했다.

또한 EC2 인스턴스 상태는 `running`이고 상태 검사는 `2/2 checks passed`였기 때문에, 인스턴스 자체 문제보다는 Security Group의 인바운드 접근 제어 문제일 가능성이 높았다.

---

### 4. 조치 내용

Security Group의 인바운드 규칙을 수정했다.

기존 SSH 규칙의 소스를 현재 Windows PC 환경 기준의 `My IP`로 다시 설정했다.

수정한 인바운드 규칙은 다음과 같다.

| 유형  | 프로토콜 | 포트 | 소스    | 목적               |
| ----- | -------- | ---: | --------- | --------------------- |
| HTTP  | TCP      |   80 | 0.0.0.0/0 | 외부 웹 접속 허용            |
| HTTPS | TCP      |  443 | 0.0.0.0/0 | HTTPS 접속 허용           |
| SSH   | TCP      |   22 | My IP/32  | 관리자 접속을 현재 작업자 IP로 제한 |

전체 포트 개방이나 SSH를 `0.0.0.0/0`으로 여는 방식은 사용하지 않았다.

---

### 5. 결과

Security Group의 SSH 소스를 현재 Windows PC의 공인 IP로 수정한 뒤, WSL에서 다시 SSH 접속을 시도했다.

```bash
ssh -i mission6-key.pem ubuntu@13.125.199.198
```

최초 접속 시 호스트 키 확인 메시지가 출력되었다.

```text
The authenticity of host '13.125.199.198' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

`yes`를 입력한 뒤 정상적으로 Ubuntu EC2 인스턴스에 접속되었다.

```text
Welcome to Ubuntu 24.04.4 LTS
ubuntu@ip-10-0-1-60:~$
```

이를 통해 SSH 접속 실패의 원인이 EC2 키페어 문제가 아니라 Security Group의 SSH 허용 Source IP 불일치였음을 확인했다.

---

### 6. 재발 방지

향후 동일한 문제가 발생하지 않도록 다음 사항을 체크리스트에 추가했다.

* Security Group에서 SSH 22번 포트는 반드시 `My IP/32`로 제한한다.
* 단, `My IP`는 설정을 수행하는 기기의 현재 공인 IP를 기준으로 잡히므로, 모바일과 PC 네트워크가 다를 수 있음을 확인한다.
* 실제 SSH 접속을 수행할 PC 또는 WSL 환경에서 Security Group의 `My IP`를 설정한다.
* SSH 접속 실패 시 먼저 다음 항목을 순서대로 확인한다.

  * EC2 인스턴스 상태가 `running`인지 확인
  * 상태 검사가 `2/2 checks passed`인지 확인
  * Public IPv4 주소가 존재하는지 확인
  * Security Group 인바운드에 SSH 22번이 있는지 확인
  * SSH 소스가 현재 접속 환경의 공인 IP와 일치하는지 확인
* 문제 해결을 위해 SSH를 `0.0.0.0/0`으로 열거나 전체 포트를 개방하지 않는다.

---

## 요약

이번 문제는 모바일 AWS 콘솔에서 Security Group의 SSH 소스를 `My IP`로 설정한 뒤, 실제 접속은 Windows WSL에서 수행하면서 발생한 IP 불일치 문제였다. Security Group의 SSH 허용 IP를 실제 접속 환경인 Windows PC의 공인 IP로 수정하자 SSH 접속이 정상적으로 성공했다.
