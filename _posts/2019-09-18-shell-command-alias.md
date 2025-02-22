---
layout: post
title: "쉘 명령어 단축 지정하기 - alias"
tags: [shell, alias, zsh, productivity, snippet]
comments: true
---


쉘 명령어에 단축 명령어를 지정하는 **`alias`**에 관해 알아본다. 
그리고 추가로 `zsh` 환경에서 함수 선언과 함께 사용하여 **alias에 파라미터를 넣는 방법**도 알아본다. 

## Why?
로컬 개발 환경에서 사용하는 카프카는 도커 컨테이너로 띄워놓고 사용하고 있다. 
그렇기 때문에 아침에 출근하면 카프카 컨테이너부터 실행시킨다. 
실행시키는 과정은 다음과 같다. 

1. `docker start zookeeper` : 주키퍼를 먼저 실행  
1. `docker ps` : 주키퍼 정상 실행 확인
1. `docker start kafka-01 kafka-02 kafka-03` : 카프카 클러스터 실행 확인
1. `docker ps` : 카프카 클러스터 정상 실행 확인 

**참 번거로운 과정이다.**
특히 3번의 경우 카프카 클러스터에 해당하는 컨테이너 이름들을 다 적어줘야 한다. 
이런 번거로운 상황을 해결할 수 있는 명령어가 `alias`이다. 
alias는 자주 쓰는 명령어들을 **단축 형태로 지정할 수 있는 명령어다.** 
그럼 어떻게 등록, 조회, 삭제할 수 있는 지 알아보자.

## alias
### 등록 
alias는 다음과 같이 지정할 수 있다. 

```
alias xxxx='so long long order'
```

여기서 `xxxx`는 우리가 사용할 단축 명령어고, `=''`으로 실제 명령어를 입력한다. 
쉘은 띄어쓰기에 민감한 녀석이니 `xxx=''`는 **꼭 붙혀쓰도록 하자** 

### 목록 확인 
적용된 단축 명령어를 확인하고자 할 때는 `alias` 를 입력하면 된다.

```
$ alias              
-='cd -'
...=../..
....=../../..
.....=../../../..
......=../../../../..
...
```

### 삭제 
존재하는 단축 명령어를 제거하고자 할 때는 `unalias xxxx`를 입력하면 된다. 

![](https://media.makeameme.org/created/easy-peasy-lemon-5bcb39.jpg)

## alias에 파라미터 적용하기 (zsh에서) 
우리는 실행한 컨테이너 쉘에 접속해야 하는 경우가 종종 있다. 
도커의 특정 컨테이너 쉘에 접속하기 위한 명령어는 다음과 같다.

```
docker exec -it <container_name> bash
```

이 명령어는 기존 명령어와 다르게 **명령어 사이에 파라미터가 존재한다.** 
하지만 **alias는 아쉽게도 파라미터를 지원하지 않는다.** 
그렇지만 `zsh`의 함수를 같이 사용하여 설정하면 파라미터와 함께 단축 명령어를 사용할 수 있다. 

![](https://i.redd.it/2zzjcb780br21.jpg)

먼저, 도커 쉘 접속을 위한 함수를 선언한다. 
위 도커 쉘 접속 명령어를 예로 들면, 다음 코드와 같이 함수를 설정할 수 있다.  

```
function dsh() { docker exec -it $1 bash } 
```

여기서 **`$1`은 첫번째 파라미터를 지칭한다.** 
위 코드를 zsh 에 입력하면 `dsh` 라는 이름을 가진 함수가 지정된다. 
이렇게 지정된 `dsh` 함수를 다음과 같이 alias 지정하면 파라미터를 가진 alias가 설정된다. 

```
alias dsh='dsh’
```

적절하게 적용되었는 지 파라미터와 함께 테스트해보자. 

```
$ dsh kafka-01
root@d809c2ddbec2:/# 
```

**아주 적절하다.** :) 

## 결론
티끌모아 태산이라 했다. 
얼른 alias를 등록하여 생산성을 향상시켜보잣!

---

## 추가 내용 
### alias 유지 
터미널 세션에서 alias를 지정하고 **세션을 종료하게 되면 해당 alias가 날라간다.** 
세션이 종료되더라도 alias 정보가 유지하기 위해 `~/.bashrc`(나는 zsh을 사용하고 있으므로 `~/.zshrc`)에 alias 명령어들을 명시해준다. 
물론 함수도 동일하게 명시하면 된다. 

```
...
## Donald Alias
alias dps='docker ps'
alias dpsA='docker ps -a'
...
alias dsh='dsh'

function dsh() {
  docker exec -it $1 bash
}
```

관련해서 피드백해주신 글또의 이동규님 감사합니다 :) 