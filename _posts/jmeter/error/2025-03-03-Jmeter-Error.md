---
title: Jmeter Error
date: 2025-03-03 22:53 +0900
categories: [Jmeter,error]
tags: [jmeter,error]
---
## java.net.BindException: Address already in use: connect 오류
### 현재 상황
Jmeter 를 통해 아래 설정으로 부하테스트를 진행하던 중    
쓰레드 수 : 500개   
Ramp-up 시간 : 1초   
지속 시간 : 600초   
java.net.BindException: Address already in use: connect 라는 오류가 발생했습니다.
### 원인   
구글링을 해보니 짧은 시간동안 네트워크 연결이 너무 많을 때 발생하는 문제로   
소켓 통신은 종료된 이후에 바로 포트를 해제하지 않고 time_wait 상태로 일정 시간 기다리는데    
그 사이 최대 접속 가능한 포트 수를 요청 수가 뛰어 넘어 발생한다고 합니다.
### 의문점   
동일한 조건으로 다른 api 를 테스트 했을 땐 오류가 발생하지 않았는데  
Redis 로 캐시를 적용시킨 데이터를 조회하는 api 는 에러가 발생했고   
DB 쿼리로 데이터를 조회하는 api 에서는 오류가 발생하지 않았습니다.   

1. 캐시 api 테스트   
![cache](/assets/images/jmeter/error/cache.png)_오류 : 0.59%_
2. DB api 테스트
![db](/assets/images/jmeter/error/db.png)_오류 : 0%_   

#### 의문 해결   
두 테스트 결과의 표본 수가 다른 것을 보고 알 수 있었습니다.   
쓰레드 500 개에 지속 시간 600초면 500 * 600 으로 30만개가 나와야 하지 않나 생각을 하고 있었는데   
표본 수 계산은 500 * 1초당 요청 횟수 * 600 으로 계산되는 거였고 역으로 계산해보면   

캐시 api 는 1초당 약 27번   
db api 는 1초당 약 13번   
요청을 보내고 있었습니다.   

따라서 캐시 api 는 db api 보다 2개 가량 요청을 보냈고 이 때문에     
db api 에선 발생하지 않은 에러가 캐시 api 에선 발생한 것이였습니다.   
### 해결 방법   
os 설정을 통해 최대 포트 수를 늘릴 수 있습니다.     
윈도우 기준   
레지스트리 편집기 -> HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters   
편집 -> 새로 만들기 -> DWORD(32-bit Value)  
이름 : MaxUserPort   
값 : 65534   
10진수   
이렇게 추가하고 재부팅하면 설정이 끝납니다.

#### 설정 후 캐시 api 테스트   
![해결](/assets/images/jmeter/error/jmeter-error-해결.png)_오류 : 0%_   






