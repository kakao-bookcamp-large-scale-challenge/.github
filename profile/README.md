# KakaoTech Bootcamp : 대규모 부하 테스트 대회

## 팀원 구성 및 역할

| 이름 | 직무 | 역할 |
| --- | --- | --- |
| kellyn.lee(이가은) | AI | - 팀장 <br> - 프라이머리-세컨더리 레플리카 셋 몽고디비 기반 서비스 개발 |
| 🙇‍♂ noah.kim(김지호)   | 풀스택 / 프론트엔드 | - artillery · playwright 기반 e2e 테스트 코드 작성 <br> - artillery.io 모니터링 서비스 구축 및 부하 테스트 진행 및 평가 <br> - 회원정보 변경 API 개발 <br> - 버그 픽스 |
| huni.kwon(권상훈) | 풀스택 / 프론트엔드 | - AWS S3 기반 이미지 업로드 방식 개발 <br> - 버그 픽스 |
| wren.kim(김시은) | 풀스택 / 백엔드 | - 마스터 슬레이브 기반 Redis 기능 개발 <br> - RabbmitMQ 기반 메세징 기능 개발 |
| izzy.lee(이주미) | 클라우드 (Devops) | - 아키텍쳐 기반 레플리카 셋 몽고디비 <br> - 레디스 클러스터 환경 구축 <br> - Prometeous · Grafana 모니터링 환경 구축 |
| andrea.jeon(전찬호) | 클라우드 (Devops) | - 30개 인스턴스의 아키텍쳐 설계 <br> - Prometeous · Grafana 모니터링 환경 구축 <br> - Docker + Github Actions 기반 CI/CD 개발 |
| mac.lee(이원호) | AI | - 채팅 삭제 기능 개발 |

<br>
 
## 목차 <a name = "index"></a>

### [1. 대회 설명](#idea)
### [2. 아키텍쳐 및 구현 내용](#structure)
### [3. 결과](#result)
### [4. 회고](#review)

<br>

## 대회 설명 <a name = "idea"></a>

주어진 시간 동안, 동일한 서비스를 대상으로 실시간 대규모 트래픽을 견딜 수 있는 서비스 및 아키텍쳐 설계 및 개발

### 1. 공통 서비스 소개

운영진에서 사전 제작한 채팅 서비스

> 바이브 코딩을 토대로 제작한 서비스로, 서비스를 실행할 수는 있지만 에러가 많은 상태
> 
- 스택 : Next.js, Express.js, MongoDB, socket.io, Redis

<img width="1443" height="601" alt="image" src="https://github.com/user-attachments/assets/2e689955-da44-4a0e-8dc4-cc128bd9f552" />

<br>

### 2. 대회 규칙

1. AWS 기반 서비스로 제한
    - **⭐️ EC2 : t3 small 인스턴스 30개 사용 가능 ⭐️**
    - Elastic Load Balancing
    - Auto Scaling
    - Route 53
    - ACM
    - S3
    - CloudFront
    - CloudWatch
2. 모니터링 서비스는 제한 없이 사용 가능
3. Redis Elastic Cache 사용 불가능
4. MongoDB → RDBMS 변경 불가능
5. 주어진 도메인 변경 불가능
6. AWS 사용 비용 $300 이내로 제한 (초과시 실격)

<img width="1443" height="431" alt="image" src="https://github.com/user-attachments/assets/35631917-541d-4fcb-8de2-ee15f4c890bd" />

<br>

### 3. 진행 기간

25.07.23. (수) ~ 25.07.25 (금) - 총 3일

<br>

### 4. 부하테스트 진행 방식

**[채점 규칙]**

- **1:1 토너먼트 형식**으로, 동일한 부하를 토대로 서비스 가용성이 우수한 팀이 승리
    - 먼저 접속 불가해진 팀이 패배
- 30초 이내 동시 접속 불가 시 무승부, **E2E 결과 레포트로 채점**
    - 성공 / 실패 개수
    - response_time
    - session_length

**[부하 전송 방법]**

- 초기 500명에서 시작하여 30초마다 500명씩 증가
- 최대 100,000명까지 순차적 증가 목표
- 각 사용자당 최소 1개 이상의 웹 소켓 연결 유지

<br>

## 아키텍쳐 및 구현 내용 <a name = "structure"></a>

<img width="679" height="777" alt="image" src="https://github.com/user-attachments/assets/fc82a615-8640-4a03-b1e9-ad66a6559abb" />

<br>

### 최종 인스턴스 구성

- FE (Next.js) : 9개
- BE (Node.js, Express.js) : 10개
- DB (MongoDB) : 5개
- Cache (Redis) : 6개

<br>

### 아키텍쳐 설계 상세 내역

- **인 메모리 이미지 업로드 → AWS S3 이미지 업로드**
    - 파일이 앱 서버와 무관하여 배포 시 무리 없음
    - 글로버 CDN 연동을 통한 접근 속도 향상
    - fileData 전송으로 인한 서버 I/O 감소
- **Redis → Redis 마스터 슬레이브 구조**
    - 슬레이브에서 읽기 처리를 분산하여 성능 향상
    - 마스터 장애 시 슬레이브로 빠르게 페일오버 가능하도록 설계
- **MongoDB → 프라이머리-세컨더비 레플리카 셋 구조**
    - 프라이머리 장애 시 자동으로 세컨더리가 승격되어 서비스 지속
    - 여러 노드에 데이터가 복제되어 데이터 안정성 확보
- **Redis + RabbitMQ를 통한 읽음 상태 처리**
    - Redis에서 읽음 상태 실시간 캐싱으로 사용자에게 빠른 응답 제공
    - 읽음 상태 기능을 비동기로 처리하여 DB 부담 감소

### 모니터링 방법

- **artillery.io를 활용하여 부하 테스트 성공/실패 분석**
    
    <img width="867" height="771" alt="image" src="https://github.com/user-attachments/assets/7852005b-a657-44db-b9c9-df27e1626eaf" />

    
- **Prometeous + Grafana를 이용한 실시간 부하 분석**
    
    <img width="1254" height="775" alt="image" src="https://github.com/user-attachments/assets/4bdc71c8-1364-474a-9fa8-0fef3e3e2cb3" />

    

### 기타 기능 추가 및 버그 픽스 내역
- (feat) 채팅 삭제하기
- (feat) 비밀방 비밀번호 입력 기능 추가
- (feat) 비밀번호 변경 API 추가
- (fix) 자신을 제외한 읽음 표시 처리
- (fix) 조합형 언어 2번 전달 오류
- (fix) 채팅 링크 클릭시 2개 입력 오류
- (fix) 블릿 포인트, 숫자 리스트 렌더링 오류
- (fix) 프로필 비밀번호 변경 엔드포인트 변경
- (fix) 파일 렌더링 시 아이콘 표시 오류
- (fix) 디폴트 이미지를 사용하는 사용자의 프로필 이미지 통일

<br>

## 🥇 결과 <a name = "result"></a>

### 대상 수상 🎉🎉

[결과]

- 5,000명 부하테스 진행시
    - 상대팀 : 6520 테스트 케이스 실패
    - 우리팀 (노아컹컹팀) : 1821 테스트 케이스 실패
- 🔥 동시 접속 100,000명 부하에도 서비스가 가용하도록 유지됨 (카카오테크 부트캠프 사상 최초)

<img width="580" height="775" alt="image" src="https://github.com/user-attachments/assets/9fdf369a-d7be-4e06-a4be-c348c4ae60be" />


## 뜨거웠던 결승전 현장 영상 보러가기 [(링크)](https://drive.google.com/drive/folders/1ODPKPgdPVW8G5ROmSO-a8sviuzgTvUQG?usp=sharing)


<br>

## ✏️ 회고 <a name = "review"></a>

지금까지는 **서비스를 만드는 데에만 집중**하며 개발을 진행해왔습니다. 

특히, **사용자에게 더 나은 사용 경험을 제공하겠다는 명분 아래**, 프론트엔드 최적화 작업에 몰두해왔습니다.

하지만 이번 부하 테스트를 계기로, **프론트엔드에 국한된 개발이 아닌 전체 서비스 아키텍처와 구조에 대한 이해**가 얼마나 중요한지를 깨닫게 되었습니다. 서비스를 성장시키기 위해서는 다양한 기술에 대한 이해와 관심이 필수적임을 체감했습니다. 카카오테크 부트캠프에서의 멘토링과 현업자의 조언을 통해, **실무에서는 “내 분야”라는 경계가 없으며**, 회사의 상황과 필요에 따라 **다양한 역할을 유연하게 수행하는 개발자**가 되어야 함을 알게 되었습니다. 이 경험을 통해서도 앞으로는 기술과 분야에 제한을 두지 않고 **더 열린 자세로 임하는 개발자**가 되어야겠다는 다짐을 하게 되었습니다.

또한 **서비스의 가용성**에 대해 깊이 고민하면서, 지금까지 접하지 못했던 개념들을 새롭게 배우는 계기가 되었고, 이는 개발자로서 한 단계 더 성장할 수 있는 좋은 기회였습니다.

지난 해커톤과 마찬가지로, 처음엔 단순한 단기 프로젝트라고 생각했던 경험이 실제로는 **저를 더욱 단단한 개발자로 성장시키고 있음을 다시금 깨닫게 되었습니다.**
