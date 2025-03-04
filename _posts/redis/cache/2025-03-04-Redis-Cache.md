---
title: Redis 를 Cache 로 사용 기록
date: 2025-03-04 19:20 +0900
categories: [Redis,Cache]
tags: [redis,cache]
---
## Redis 를 Cache 로 사용 기록   
### 현재 상황   
메인 페이지에서 7일 이내 게시글 중 조회수와 좋아요가 많은 순으로 3개를 보여줍니다.   
메인 페이지여서 요청이 많을 api 지만 내용이 자주 바뀌지 않기 때문에 캐시를 이용해 응답하면 성능을 개선 할 수 있을 것 같습니다.   
스프링에서 제공하는 @EnableCaching 를 이용해 외부 저장소 없이도 가능하지만 서버가 여러 대라고 생각하고 Redis 를 적용해 봤습니다.   

### 캐시 적용 전 
#### Jmeter 테스트   
쓰레드 : 500개   
Ramp-up : 1초   
지속 시간 : 600초

#### 테스트 결과   
![before](/assets/images/redis/cache/before.png)_처리량 : 12092.9/sec_   

### Redis 적용    
#### 1. Dependencies
```
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```
위 의존성을 추가하고 application.properties 로 host, port 설정을 하면   
자동으로 redisTemplate 와 redisConnectionFactory 가 생성되어 Redis 와 연동이 됩니다.   
**PS :** 자동으로 생성된 template 의 직렬화 방식은 JDK 직렬화로 Java 가 아닌 환경에선 호환이 어려울 수 있습니다.   
저는 직접 확인 할 수 있는 JSON 형태로 저장하고 싶어 따로 생성해 Bean 으로 등록했습니다.
#### 2. Config
  - RedisConnectionFactory
```java
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }
 ```   
connectionFactory 로는 lettuce, jedis 가 있는데 성능이 lettuce 가 더 좋고 스프링에서도 기본값이 lettuce 입니다.   

  - RedisTemplate
```java
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();

        GenericJackson2JsonRedisSerializer serializer = new GenericJackson2JsonRedisSerializer(objectMapper());

        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(serializer);

        redisTemplate.setConnectionFactory(redisConnectionFactory());
        return redisTemplate;
    }
```   
JSON 직렬화를 위해 위 template 을 Bean 등록해 주었습니다.
  - ObjectMapper
```java
    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .deactivateDefaultTyping()
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
                .build();
    }
```   
게시글의 데이터 중 LocalDateTime 을 직렬화 하기 위해 objectMapper 를 따로 등록했습니다.   
addModule 이 LocalDateTime 을 직렬화하기 위한 부분이고 disable 은 타임스탬프를 문자열로 변환하기 위한 것입니다.   
deactivateDefaultTyping 은 @Class 정보를 담지 않기 위해서 설정하는 것인데 jackson 2.10 부터는 기본값으로 되어 생략해도 됩니다.   
#### 3. Service
```java
public List<Board> getBoardTop3() {
        Object o = redisTemplate.opsForValue().get(boardKey);
        if (o instanceof List<?> resultList) {
            return resultList.stream()
                    .map(board -> objectMapper.convertValue(board, Board.class))
                    .toList();
        }
        return new ArrayList<>();
    }

    public void setBoardTop3(List<Board> boardList) {
        redisTemplate.opsForValue().set(boardKey, boardList);
    }
```   
boardKey 에는 key 로 사용할 String 으로 "board:" 를 넣어주었습니다.   
getBoardTop3 에 if 문은 Redis 에서 받은 Object 를 List<Board> 인지 검증하고 반환하는 코드입니다.   

#### Scheduler   
캐시 데이터를 게시글의 조회수, 좋아요 수에 따라 변경해야 하는데 모든 게시글에 조회수, 좋아요가 증가할 때마다 비교 후 변경하는 건   
비효율적이라고 생각이 들어 스케쥴러를 이용해 10분마다 갱신해주기로 했습니다.   
```java
    @Scheduled(cron = "0 */10 * * * *", zone = "Asia/Seoul")
    public void updateBoardTop3() {
        Pageable pageable = PageRequest.of(0, 3);
        LocalDateTime sevenDaysAgo = LocalDateTime.now().minusDays(7);
        List<Board> boardList = boardRepository.getTop3Within7Days(sevenDaysAgo, pageable);
        redisService.setBoardTop3(boardList);
    }
```
### 캐시 적용 후   
#### Jmeter 테스트
쓰레드 : 500개   
Ramp-up : 1초   
지속 시간 : 600초

#### 테스트 결과
![after](/assets/images/redis/cache/after.png)_처리량 : 23327.6/sec_   

처리량 : 12092.9/sec -> 23327.6/sec 약 93% 증가   



 
