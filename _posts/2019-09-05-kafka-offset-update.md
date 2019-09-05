---
layout: post
title: "Kafka 오프셋 변경하기"
tags: [kafka, offset]
comments: true
---

## 오프셋 정보 변경하기
프로듀서가 메세지를 발행하는 과정에서 에러가 발생하여, 컨슈머가 해당 메세지를 소비하지 못하고 Hang이 걸리는 경우가 있다(고승범님 글 참조). 
이러한 경우 오프셋 정보를 강제로 변경할 필요가 있다. 
이 글에서는 브로커 CLI 환경에서 오프셋 정보를 변경하는 방법에 대해 살펴본다. 

### 메세지 발행/구독 
* 프로듀서에 메세지 발행

```
# 발행할 메세지 입력
/usr/bin/kafka-console-producer --broker-list kafka-server-host:9092 --topic donald-offset-update
> ...
```

* 컨슈머에서 메세지 구독 (-> 오프셋 최신화)

```
# 브로커에 쌓인 메세지 구독
/usr/bin/kafka-console-consumer --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --topic donald-offset-update
> ...
```

### 각 파티션의 offset 10 감소
* Before
    * 50개의 메세지를 3개의 파티션이 17개 정도씩 나눠가진 상태
    * 컨슈머가 파티션의 모든 메세지를 가져감 (LAG = 0, CURRENT-OFFSET = LOG-END-OFFSET)
    
```
# 토픽(파티션) 상태
/usr/bin/kafka-consumer-groups --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --offsets --describe
Consumer group 'donald-consumer-group' has no active members.

GROUP                 TOPIC                PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
donald-consumer-group donald-offset-update 0          17              17              0               -               -               -
donald-consumer-group donald-offset-update 1          18              18              0               -               -               -
donald-consumer-group donald-offset-update 2          15              15              0               -               -               -
```

* After
    * CURRENT-OFFSET 이 10 감소
    * 각 파티션의 LAG = 10

``` 
# 오프셋 -10 이동
/usr/bin/kafka-consumer-groups --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --reset-offsets --shift-by -10 --topic donald-offset-update --execute

GROUP                          TOPIC                          PARTITION  NEW-OFFSET     
donald-consumer-group          donald-offset-update           0          7              
donald-consumer-group          donald-offset-update           2          5              
donald-consumer-group          donald-offset-update           1          8    

# 토픽(파티션) 상태
/usr/bin/kafka-consumer-groups --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --offsets --describe
Consumer group 'donald-consumer-group' has no active members.

GROUP                 TOPIC                PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
donald-consumer-group donald-offset-update 0          7               17              10              -               -               -
donald-consumer-group donald-offset-update 1          8               18              10              -               -               -
donald-consumer-group donald-offset-update 2          5               15              10              -               -               -

# 메세지 구독
/usr/bin/kafka-console-consumer --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --topic donald-offset-update
> ... (10 * 3 개의 메세지)
```

### 각 파티션의 offset 처음으로 update
* 파티션의 가장 처음(오프셋 최소값)으로 오프셋 이동

```
# 오프셋 처음으로 이동
/usr/bin/kafka-consumer-groups --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --reset-offsets --to-earliest --topic donald-offset-update --execute

GROUP                          TOPIC                          PARTITION  NEW-OFFSET     
donald-consumer-group          donald-offset-update           0          0              
donald-consumer-group          donald-offset-update           2          0              
donald-consumer-group          donald-offset-update           1          0 

# 토픽(파티션) 상태
/usr/bin/kafka-consumer-groups --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --offsets --describe

Consumer group 'donald-consumer-group' has no active members.

GROUP                 TOPIC                PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
donald-consumer-group donald-offset-update 0          0               17              17              -               -               -
donald-consumer-group donald-offset-update 1          0               18              18              -               -               -
donald-consumer-group donald-offset-update 2          0               15              15              -               -               -

# 메세지 구독
/usr/bin/kafka-console-consumer --bootstrap-server kafka-server-host:9092 --group donald-consumer-group --topic donald-offset-update
> ... (전체 메세지)
```

### 기타 오프셋 이동 옵션
```
.----------------------.-----------------------------------------------.----------------------------------------------------------------------------------------------------------------------------------------------.
|      Scenario        |                   Arguments                   |                                                                    Example                                                                   |
:----------------------+-----------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------:
| Reset to Datetime    |  --to-datetime YYYY-MM-DDTHH:mm:SS.sss±hh:mm  |  Reset to first offset since 01 January 2017, 00:00:00 hrs: --reset-offsets –group test.group --topic foo --to-datetime 2017-01-01T00:00:00Z |
:----------------------+-----------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------:
| Reset by Duration    |  --by-duration  PnDTnHnMnS                    |  Reset to first offset since one week ago (from current timestamp): --reset-offsets --group test.group --topic foo --by-duration P7D         |
:----------------------+-----------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------:
| Reset to Earliest    |  --to-earliest                                |  Reset to earliest offset available: --reset-offsets --group test.group --topic foo --to-earliest                                            |
:----------------------+-----------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------:
| Reset to Latest      |  --to-latest                                  |  Reset to latest offset available: --reset-offsets --group test.group --topic foo --to-latest                                                |
:----------------------+-----------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------:
| Reset to Offset      |  --to-offset                                  |  Reset to offset 1 in all partitions: --reset-offsets --group test.group --topic foo --to-offset 1                                           |
:----------------------+-----------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------:
| Shift Offset by 'n'  |  --shift-by n                                 |  Reset to current offset plus 5 positions: --reset-offsets --group test.group –topic foo --shift-by 5                                        |
:----------------------+-----------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------:
| Reset from File      |  --from-file PATH_TO_FILE                     |  Reset using a file with reset plan: --reset-offsets --group test.group --from-file reset-plan.csv                                           |
'----------------------'-----------------------------------------------'----------------------------------------------------------------------------------------------------------------------------------------------'
```

### 참고
* [Kafka 운영자가 말하는 Kafka 서버 실전 로그 분석 - 고승범](https://www.popit.kr/%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%9A%B4%EC%98%81%EC%9E%90%EA%B0%80-%EB%A7%90%ED%95%98%EB%8A%94-%EC%B9%B4%ED%94%84%EC%B9%B4-%EC%84%9C%EB%B2%84-%EC%8B%A4%EC%A0%84-%EB%A1%9C%EA%B7%B8-%EB%B6%84%EC%84%9D/)
* [how to change start offset for topic](https://stackoverflow.com/questions/29791268/how-to-change-start-offset-for-topic)