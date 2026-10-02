# 리소스 정리 체크리스트

## 정리 요약

실습 후 리소스를 정리했다고 기록했으며, 보존된 화면에서 EC2 종료·VPC 삭제·키페어 삭제를 확인할 수 있다. EBS 삭제와 Elastic IP 할당·반납 이력은 별도 증거가 없어 미확인이다. 현재 계정에 접속해 다시 검사한 결과는 아니다.

## 체크리스트

| 리소스 | 필요한 정리 작업 | 상태 | 증거 |
|---|---|---|---|
| EC2 인스턴스 `mission6-web-ec2` | 인스턴스 종료 | 완료 | `../screenshots/11-ec2-terminate-action.png`, `../screenshots/12-ec2-terminated.png` |
| EBS 루트 볼륨 | 인스턴스 종료와 함께 삭제되었는지 또는 미사용 볼륨이 남지 않았는지 확인 | 미확인 | EC2 종료 화면만 있으며 볼륨 목록·DeleteOnTermination 확인은 없음 |
| Elastic IP | 별도 할당했다면 Release | 할당·반납 이력 미확인 | 퍼블릭 IPv4 표시나 유지 증거 부재만으로 Elastic IP 미할당·반납을 확정할 수 없음 |
| Internet Gateway | VPC 삭제 흐름에서 Detach 및 삭제 | VPC 삭제 흐름으로 기록; 개별 ID 확인 없음 | `../screenshots/13-vpc-mission6-delete-success.png` |
| VPC `mission6-vpc` | VPC와 종속 리소스 삭제 | 완료 | `../screenshots/13-vpc-mission6-delete-success.png`, `../screenshots/15-vpc-list-empty-after-delete.png` |
| Public Subnet | VPC 종속 리소스 정리 흐름에서 삭제 | VPC 삭제 흐름으로 기록; 개별 ID 확인 없음 | `../screenshots/13-vpc-mission6-delete-success.png` |
| Route 테이블 | VPC 종속 리소스 정리 흐름에서 삭제 | VPC 삭제 흐름으로 기록; 개별 ID 확인 없음 | `../screenshots/13-vpc-mission6-delete-success.png` |
| Security Group | VPC 종속 리소스 정리 흐름에서 삭제 | VPC 삭제 흐름으로 기록; 개별 ID 확인 없음 | `../screenshots/13-vpc-mission6-delete-success.png` |
| 키페어 | 실습용 키페어 삭제 | 완료 | `../screenshots/17-keypair-delete-success.png` |
| NAT Gateway | 생성했다면 삭제 | 기록상 미생성; 별도 목록 증거 없음 | 단일 Public Subnet 구성이라 NAT Gateway가 필요하지 않았다 |
| ELB/ALB | 생성했다면 삭제 | 기록상 미생성; 별도 목록 증거 없음 | 서비스는 EC2 Security Group 규칙으로 직접 공개했다 |
| RDS | 생성했다면 삭제 | 기록상 미생성; 별도 목록 증거 없음 | 데이터베이스가 필요 없는 실습이었다 |

## 정리 순서와 과금 방지 이유

1. 컴퓨트 과금을 먼저 멈추기 위해 EC2 인스턴스를 종료한다.
2. EBS 볼륨이 삭제되었는지 또는 분리된 미사용 볼륨이 남지 않았는지 확인한다. 남은 볼륨은 계속 과금될 수 있다.
3. Elastic IP를 별도로 할당했다면 Release한다. 연결되지 않은 Elastic IP는 과금될 수 있다.
4. 네트워크 리소스는 의존성 역순으로 정리한다. Public Subnet, Route 테이블 연결, Internet Gateway Detach/Delete, Security Group, VPC 순서로 확인한다.
5. 인스턴스가 삭제된 뒤 실습용 키페어를 삭제한다.
6. 선택 유료 리소스인 NAT Gateway, Load Balancer, RDS가 생성되지 않았거나 삭제되었는지 확인한다.

## 참고

- VPC 삭제 성공 스크린샷에는 `mission6-vpc`와 관련 리소스가 삭제되었다는 성공 메시지가 표시된다.
- EC2 인스턴스 목록에는 실습 인스턴스가 종료됨 상태로 표시된다.
- VPC 목록 화면에는 삭제 후 조회되는 VPC가 없는 상태가 표시된다.

## 새 실습에서 남길 증거

종료 전 인스턴스의 볼륨 ID와 DeleteOnTermination 값을 기록하고, 종료 후 해당 볼륨 ID 조회 결과 또는 삭제 완료 화면을 남긴다. EC2 종료 화면을 볼륨 삭제 증거로 대신하지 않는다. Elastic IP는 Allocation ID 목록과 실제 반납 결과를 확인한다.

VPC 삭제 메시지의 세부 목록에서 Subnet·Route Table·Internet Gateway·Security Group ID를 함께 기록한다. 확인하지 못한 항목은 미확인으로 두고, 계정·리전·조회 시각을 명시한 목록 증거가 있을 때만 완료 상태를 갱신한다.
