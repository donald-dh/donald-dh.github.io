---
layout: post
title: "JPA 네이티브 쿼리와 하이버네이트 1차 캐시"
tags: [jpa, native-query, hibernate, cache, transaction]
comments: true
---

Spring에서 제공하는 JPA(Java Persistent Api)는 특정 키워드로 간편하게 쿼리를 생성해주는 쿼리 메소드(`Query Method`)를 제공해준다. 
또한 사용자가 직접 쿼리를 작성하는 네이티브 쿼리(`Native Query`) 방식도 지원한다. 
이번 글은 JPA의 네이티브 쿼리 방식으로 쿼리를 사용할 때, 하이버네이트 1차 캐시와 관련하여 발생할 수 있는 이슈에 대해 다뤄본다. 
    
## 하이버네이트 1차 캐시 (Hibernate first-level cache)
데이터베이스와 커넥션을 맺고 조회하여 결과를 가져오는 과정은 매우 무거운 과정이다. 
이런 무거운 과정을 줄이고 성능을 향상하기 위해 하이버네이트는 1차, 2차 캐시를 제공한다.   
     
![hibernate-cache-architecture](https://thoughts-on-java.org/wp-content/uploads/2015/11/caches1.png)      
   

하이버네이트 캐시는 DB에서 조회한 결과를 일시적으로 가지고 있는다. 
이 후 동일한 엔티티에 관한 조회가 있을 경우 DB에 접근하지 않고 메모리(캐시)에서 바로 가져와 사용한다. 
다만 1차, 2차 캐시는 저장, 유지, 사용하는 범위가 다른데, 
2차 캐시는 Spring Application 범위에서 공용으로 사용되는 캐시이고 1차 캐시는 하이버네이트 세션 영역에서 사용되는 캐시이다.           
   
하이버네이스 세션이 종료될 때는 `flush` 과정을 거치게된다. 
flush는 세션 내에서 일어났던 변경 사항들을 DB와 동기화하는 과정이다. 
즉, 세션 내에서 특정 엔티티를 생성하거나(`CREATE`), 변경하거나(`UPDATE`), 삭제했을 경우(`DELETE`), 
*메모리 내부에서 해당 변경사항들을 가지고 있다가 flush 요청에 따라 DB에 변경사항들을 반영한다.* 
**다만, flush 이 후에 1차 캐시가 제거되는 것은 아니다.**

## JPA 네이티브 쿼리(Native Query)
JPA에서 제공하는 쿼리 메소드 방식은 로직의 요구사항을 모두 만족할 수 없다. 
그래서 JPA는 쿼리문을 직접 작성하여 메소드와 맵핑하는 네이티브 쿼리 방식도 제공한다. 
밑은 네이티브 쿼리 방식의 예시 코드이다.   
   
```java
public interface PersonRepository extends JpaRepository<Person, Long> {

	@Modifying
	@Query("delete from Person where no = :no")
	void deleteHumanByNo(Long no);
}
```   
   
네이티브 쿼리의 특징은 *하이버네이트 캐시를 거치지 않고 바로 DB에 접근하여 처리된다.* 
이런 특징 때문에 네이티브 쿼리로 DML 쿼리를 생성하여 사용할 경우, 캐시와 실제 DB 사이에 데이터 불일치가 발생할 수 있다.    

## 1차 캐시와 DB 간 데이터 불일치 발생
> 이슈 설명에 앞서, 문제 상황을 [간단한 예제](https://github.com/donald-dh/study-hibernate)로 재현해봤다. 
로컬에서 작성된 테스트 케이스를 실행하면 문제 상황을 재현할 수 있다. 
본 글에서는 코드에 관한 자세한 묘사가 없으므로 예제와 같이 보는 것을 추천한다. 

### 상황 예시
다음과 같은 삭제 API에 관한 테스트 코드가 있다고 가정해보자. 테스트 시나리오는 

1. 특정 엔티티를 생성
2. `MockMvc` 객체를 통해 삭제 요청
3. 삭제되었는 지 검증
   
이다. 
 
```java
@Test
@Tranactional
public void deleteApi() throws Exception {
        // given
	Person donald = repository.save(new Person("Donald", 29, 180));     // 저장 후

	// when
	mockMvc.perform(delete("/people/{no}", donald.getNo()))     // 삭제 api를 요청하여 삭제한다.
			.andExpect(status().isOk());

	// then
	assertThat(repository.findById(donald.getNo())).isNotPresent();     // 정말 잘 삭제되었는 지 검증한다.
}
```

또한, 삭제 API에서 사용된 삭제 쿼리는 네이티브 쿼리 방식으로 작성되었다(본 문 상단 코드블록 참조).    

### 테스트 실패
위의 테스트 코드를 실행하면 다음과 같이 실패 메세지가 나타난다.

```
java.lang.AssertionError: 
Expecting an empty Optional but was containing value: <Person(no=1, name=Donald, age=29, height=180)>.
```    

메세지 내용은 비어있어야 할 `Optional` 객체에 값이 있고, 그렇기 때문에 테스트에 실패했다는 것이다. 
그리고 직접 실행해보면 삽입과 삭제 쿼리 또한 정상적으로 실행되었음을 확인할 수 있다. 
결국 *삽입 삭제 모두 잘 되었지만, 삭제는 되지 않았다는 것이다.*   
   
우리는 이 기이한 현상의 원인을 앞서 언급한 하이버네이트 1차 캐시와 네이티브 쿼리의 특징에서 찾을 수 있다.
(한번 곰곰히 생각해보는 것을 추천한다.) 

### 테스트 실패 이유
테스트 상단에는 `@Tranactional`이 명시되어 있다. 
즉, 해당 테스트 진행은 하나의 트랜젝션으로 처리하겠다는 것이다. 
이는 곧 하이버네이트 세션이 테스트 진행동안 하나의 세션으로 유지되는 것을 나타낸다.   

하나의 세션으로 유지되기 때문에 테스트 진행 중 발생하는 변경, 조회에 관한 요청이 세션에서 유지된다. 
그래서 테스트 과정에서 삽입된 `donald` 엔티티 또한 *세션에(만) 저장된다.* 
하지만 삭제 API에서 사용된 JPA 쿼리는 네이티브 방식이기 때문에 세션을 거치지 않고 바로 *데이터베이스에서 삭제한다.* 
그렇기 때문에 세션에는 여전히 `donald` 엔티티가 삽입된 상태로 존재하게 되고, 
조회 부분에서 캐시를 먼저 조회하기 때문에 `donald` 엔티티를 조회할 수 있다. 
그래서 결국 테스트를 실패하게 된 것이다.   
   
그렇다면 테스트가 올바르게 작동하게 하려면 어떻게 해야할까? 
바로 트랜젝션을 분리하고 서로 다른 세션에서 삽입, 삭제, 조회를 처리하는 것이다. 
그렇게 되면 삽입 내용이 데이터베이스에 저장되고, 
조회할 때에도 세션 내부에 엔티티 정보가 존재하지 않기 때문에 정상적인 테스트를 진행할 수 있게 된다. 

## 마무리 
이번 이슈는 테스트 코드를 작성하면서 발생했던 이슈이다. 
내부 동작으로 발생되는 이슈이기 때문에 쉽게 파악할 수 없는 이슈이기도 하다. 
앞으로 네이티브 쿼리를 사용할 경우에는 트랜젝션와 1차 캐시(하이버네이트 세션)을 잘 고려해야겠다.   

## 참고 
* [jpa nativequery returns cached resultlist](https://stackoverflow.com/questions/34484203/jpa-nativequery-returns-cached-resultlist)
* [JPA 영속성](http://wonwoo.ml/index.php/post/997)