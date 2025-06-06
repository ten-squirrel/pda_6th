# 인스턴스 전송/압축 성능 비교
## 📌 프로젝트 정의 및 목적

AWS EC2의 다양한 인스턴스 타입 중 T2 (범용형)와 C7g (컴퓨팅 최적화형) 인스턴스 간의 전송 속도와 파일 작업 속도를 실험적으로 분석한다. 이를 통해 실제 서비스 환경에서 가장 효과적이고 합리적인 인스턴스 선택 기준을 제시하는 것을 목표로 한다.


## 🧪 테스트 설명
### 테스트 항목
|항목|설명|
|--|------|
|압축 속도|1GB 파일을 gzip으로 압축하는 데 걸리는 시간 측정|
|전송 속도|10MB 파일을 scp로 전송하는 데 걸리는 시간 측정|

### 테스트 환경

범용 T2 인스턴스와, AWS에게 성능 우선으로 추천받은 C7g 인스턴스를 사용했다. storage 등 다른 요소는 최대한 통제하였다.

서울, 도쿄, 버지니아로 송신/수신 인스턴스를 바꿔가며 서로 다른 리전 간 전송 속도를 측정하였다.

|항목|CPU|리전 및 가용영역|
|--|------|-|
|T2.medium | Intel Xeon 2vCPU (x86)|도쿄(ap-northeast-1a), 버지니아(us-east-1b), 서울(ap-northeast-2c)
|C7g.medium | AWS Graviton3 1vCPU (ARM)|도쿄(ap-northeast-1d), 버지니아(us-east-1a), 서울(ap-northeast-2d)

## 💡 가설
### 작업 속도
인스턴스별로 특화된 작업이 존재하여, 압축 속도에 차이가 있을 것이다.

t형이 범용, c형이 컴퓨팅 최적화 인스턴스임을 고려할 때, c7g의 파일 작업 속도가 더 빠를 것이다.

### 전송 속도

동일 인스턴스 타입(t2.medium)이라도 리전에 따라 네트워크 품질(지연, 전송속도)이 다를 것이다.

서울 리전(ap-northeast-2)은 미국 리전보다 빠르지만 요금이 더 비싸다, 속도 대비 요금 효율은 낮을 수 있다.

|리전|전송 GB당 비용|
|-|-|
|서울|0.02$|
|일본|0.02$|
|미국|0.01$|

### 가설 결론

- 전송 관련해서는 t형이 속도가 빠를 것이고, c형이 압축 부분에서 빠를 것이며, 총 시간은 t형이 빠를 것이다.
- 이러한 속도 차이를 반영한 계산기를 통해 적합한 송수신 리전 추천을 해줄 수 있다.

## 📊 결과

### 압축 시간

|인스턴스|압축 시간 (s)|압축 속도(MB/s)|예상 요금($/시간)|
|-|-|-|-|
|t2.medium|38.7|27.8|0.0464|
|c7g.medium|32.3|33.2|0.0361|

![image](https://github.com/user-attachments/assets/ef027b34-b7a0-4a68-87c7-3ba6d2b3b017)




### 파일 전송 시간

| 인스턴스       | 구간        | 전송 시간 (s) |
| ---------- | --------- | --------- |
| t2.medium  | 서울 → 도쿄   | 20.176    |
| t2.medium  | 서울 → 버지니아 | 114.005   |
| t2.medium  | 버지니아 → 도쿄 | 99.100    |
| t2.medium  | 버지니아 → 서울 | 110.164   |
| t2.medium  | 도쿄 → 버지니아 | 94.319    |
| t2.medium  | 도쿄 → 서울   | 18.0      |
| c7g.medium | 서울 → 도쿄   | 3.511     |
| c7g.medium | 서울 → 버지니아 | 14.505    |
| c7g.medium | 버지니아 → 도쿄 | 13.299    |
| c7g.medium | 버지니아 → 서울 | 14.760    |
| c7g.medium | 도쿄 → 버지니아 | 13.148    |
| c7g.medium | 도쿄 → 서울   | 3.468     |

![image](https://github.com/user-attachments/assets/2c61a6f0-f781-4139-8244-082126ee11c3)


(3회 실행한 평균값이다.)

압축 속도와 전송 속도 모두 c7g가 비교적 빨랐다.


## ✅ 결론
CPU 연산 성능과 전송 속도가 중요한 작업에는 c7g 인스턴스가 적합하다.

같은 인스턴스 내에서는 가까운 리전 간 전송 시 크게 속도가 개선되었다.

인스턴스를 선택할 때는 단순 사양보다 작업 유형에 따라 알맞은 인스턴스와 리전을 선택해야 한다.

### 최적 인스턴스 계산기
https://kyun9-cloud.github.io/pracc/

## ⛔️ 잘못된 설계의 원인
> SCP(Secure Copy Protocol)의 오용

직접 파일을 리전 별 인스턴스 간 송신해보며 파일 송수신 시간을 테스트했는데, 이 프로토콜은 암호화 알고리즘을 포함하여 연산돼 cpu가 좋은 c7g가 빠를 수 밖에 없었다.

> 시간 측정: time 명령어의 오용

time 명령어는 cpu 시간을 포함한다. user 시간과 system 시간을 제해도 정확한 네트워크 속도를 알 순 없다.

`두 문제를 대신하여 iperf 를 사용해볼 수 있다.`

> 가용 영역 차이

인스턴스 유형별로 가용 영역에 차이가 있었다. 가용 영역별 이용량 혹은 속도 차이에 의한 결과일 수 있다.
