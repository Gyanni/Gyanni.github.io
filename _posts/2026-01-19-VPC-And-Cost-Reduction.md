---
layout: post
title:  "[AWS] VPC 실습 비용 아끼기 & Cloud Nuke로 삭제되지 않는 리소스 해결기"
date:   2025-01-19 23:37:00 +0900
categories: [Retrospect]
tags: [aws, terraform]
---

# [AWS] VPC 실습 비용 아끼기 & Cloud Nuke로 삭제되지 않는 리소스 해결기  
  <br><br>
  
## 1. 꼬여버린 코드와 불어나는 요금

Terraform을 이용해 AWS VPC 환경을 구축하는 실습을 진행하던 중이었습니다. 며칠에 걸쳐 다양한 시도를 하다 보니 Terraform 상태 파일(state)과 실제 리소스 간에 정합성이 맞지 않는 상황이 발생했고, 코드는 점점 복잡해졌습니다.

무엇보다 가장 큰 문제는 **'비용'**이었습니다. 실습을 위해 띄워둔 리소스들이 야금야금 요금을 발생시키고 있었기 때문입니다.

> **[📸 스크린샷 1: 전체 서비스 중 EC2-Other가 차지하는 비용 그래프]** > *(설명: AWS Cost Explorer에서 '서비스별'로 필터링했을 때, EC2-Other 항목이 비용의 대부분을 차지하고 있는 막대그래프)*

청구서를 확인해 보니, 순수 서버(EC2-Instances) 비용보다 부가적인 네트워크 비용(EC2-Other)이 훨씬 높게 나오고 있었습니다. 결국 비용 절감과 환경 초기화를 위해 **"일단 싹 밀고(Reset), 다시 깔끔하게 시작하자"**는 결심을 하게 되었습니다.
<br>

---
<br>

## 2. 본론 1: 강력한 청소부 'Cloud Nuke' 설치 및 실행

일일이 AWS 콘솔에 들어가서 리소스를 삭제하는 건 비효율적이라 판단하여, AWS 리소스를 강력하게 초기화해주는 오픈소스 도구 **Cloud Nuke**를 사용하기로 했습니다.
<br>

### 2-1. Cloud Nuke란?
Gruntwork에서 만든 오픈소스로, 특정 리전의 모든 AWS 리소스를 한 번에 삭제(Nuke)해주는 무시무시하면서도 편리한 도구입니다.
<br>

### 2-2. 설치 방법
OS 별로 패키지 매니저를 통해 간단하게 설치할 수 있습니다.

**macOS (Homebrew)**
```bash
brew install cloud-nuke
Windows (Chocolatey)
```
<br>

**Window**
```Bash
choco install cloud-nuke
Linux (Script)
```
<br>

**Linux**
```Bash
# 최신 바이너리 다운로드 (버전은 GitHub Release 확인 필요)
wget [https://github.com/gruntwork-io/cloud-nuke/releases/download/v0.1.0/cloud-nuke_linux_amd64](https://github.com/gruntwork-io/cloud-nuke/releases/download/v0.1.0/cloud-nuke_linux_amd64)
mv cloud-nuke_linux_amd64 cloud-nuke
chmod +x cloud-nuke
sudo mv cloud-nuke /usr/local/bin
```
<br>

### 2-3. 실행 그리고 실패

설치를 마치고, 서울 리전(ap-northeast-2)의 모든 리소스를 삭제하는 명령어를 실행했습니다.

Bash
cloud-nuke aws --region ap-northeast-2
명령어를 실행하자 리소스들이 삭제되는 듯했으나, 마지막 순간 VPC가 삭제되지 않고 에러를 뱉어냈습니다.

[📸 스크린샷 2: Cloud Nuke 실행 시 에러 로그] > (설명: 터미널 창에서 "DependencyViolation" 또는 "cannot delete" 메시지와 함께 붉은색 에러가 발생한 화면)

분명 다 지운 것 같은데, 무언가 남아서 VPC 삭제를 막고 있었습니다.

<br>

## 3. 본론 2: 범인은 'Load Balancer 삭제 방지'
로그를 분석하고 콘솔을 확인해 보니, VPC 내부에 Application Load Balancer(ALB) 하나가 덩그러니 남아 있었습니다.

[📸 스크린샷 3: 덤프 ALB의 '삭제 방지'가 활성화된 AWS 콘솔 화면] > (설명: 로드 밸런서 속성 탭에서 Deletion Protection이 체크되어 있는 모습)

원인은 바로 실습 때 설정해 둔 '삭제 방지(Deletion Protection)' 기능 때문이었습니다. 이 기능이 켜져 있으면 Cloud Nuke 같은 자동화 도구라도 API를 통해 강제로 삭제할 수 없습니다.

결국 콘솔에서 체크를 해제하고 나서야 깔끔하게 삭제할 수 있었습니다. **"도구만 믿지 말고 리소스의 속성을 이해해야 한다"**는 것을 배운 순간이었습니다.
<br>

## 4. 본론 3: 비용의 주범, NAT Gateway 잡아내기
VPC 초기화 후, 도대체 어디서 비용이 그렇게 많이 나왔는지 세부 항목을 뜯어보았습니다. 범인은 바로 NAT Gateway였습니다.

[📸 스크린샷 4: EC2-Other 상세 내역 중 NAT Gateway 비용 화면] > (설명: Cost Explorer에서 Usage Type으로 그룹화했을 때, 'NatGateway-Hours' 항목이 비용의 대부분을 차지하고 있는 화면)

NAT Gateway는 시간당 요금이 꽤 비싼 편입니다. 약 30시간 정도 켜뒀을 뿐인데 벌써 커피 한 잔 값($1.42)이 나가 있었습니다.

여기서 중요한 점은 Elastic IP(EIP) 였습니다. 청구서에는 EIP 비용이 없었는데, 이는 NAT에 붙어 사용하여 무료 처리되었기 때문입니다.

🚨 주의: 만약 비용을 아끼겠다고 NAT만 지우고 EIP를 남겨두면, 그때부터는 '유휴 IP(Idle Address)' 요금 폭탄을 맞게 됩니다. 반드시 세트로 관리해야 합니다.
<br>

## 5. 본론 4: 헝그리 개발자의 Terraform 비용 절약 꿀팁
VPC를 다시 구축했지만, 매일 destroy 하고 다시 만드는 건 시간이 너무 걸렸습니다. 그렇다고 켜두자니 NAT 요금이 무서웠죠.

그래서 **"실습 안 할 땐 비싼 부품(NAT, EIP)만 빼놓자"**는 전략을 세웠습니다. Terraform 코드에서 해당 리소스만 주석 처리를 하는 것입니다.

[📸 스크린샷 5: Terraform 코드에서 NAT와 EIP를 주석 처리한 화면] > (설명: VS Code 등에서 resource "aws_nat_gateway"와 resource "aws_eip" 부분이 주석(#) 처리된 모습)

```terraform
# 실습 종료 시 주석 처리하여 비용 절감

# resource "aws_eip" "nat_eip" {
#   vpc = true
# }

# resource "aws_nat_gateway" "nat" {
#   allocation_id = aws_eip.nat_eip.id
#   subnet_id     = aws_subnet.public_a.id
# }
```
이렇게 주석 처리를 하고 terraform apply를 하면, Terraform은 전체 인프라(VPC 뼈대)는 유지한 채 비용이 발생하는 NAT와 EIP만 쏙 골라 삭제합니다. 다음 날 주석만 풀면 1분 만에 실습 환경이 복구됩니다.
<br>

## 6. 결론: 생성보다 중요한 건 '관리'
이번 경험을 통해 클라우드 엔지니어로서 중요한 교훈을 얻었습니다.

리소스 속성 이해: Cloud Nuke도 AWS의 안전장치(삭제 방지)를 뚫을 순 없습니다. 에러 로그를 보고 원인을 파악하는 능력이 중요합니다.

비용 관리 능력: 리소스를 잘 만드는 것만큼, NAT/EIP 같은 고비용 리소스를 효율적으로 관리(ON/OFF)하는 것이 핵심 역량입니다.

앞으로는 코드 한 줄을 짤 때도 "이게 얼마짜리 리소스인가?", "어떻게 지울 것인가?"를 함께 고민하는 습관을 들여야겠습니다.