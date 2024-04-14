---
layout: post
title: java Optional을 제대로 쓰기 위한 12가지 레시피
date: 2024-04-14 11:30 +0900
categories: ["요약 및 정리"]
tags: [java]
---


## [java Optional을 제대로 쓰기 위한 12가지 레시피](https://blogs.oracle.com/javamagazine/post/12-recipes-for-using-the-optional-class-as-its-meant-to-be-used)


- Optional 클래스는 값이 없을 수도 있는 값을 위한 컨테이너 유형입니다.
- java SE11 문서에서 아래와 같이 말합니다

> "Optional은 '결과 없음'을 나타내는 명확한 필요성이 있고 null 사용이 오류를 유발할 가능성이 높은 경우에 메서드 반환 유형으로 주로 사용됩니다. Optional 형식인 변수 자체는 null이 되어서는 안 되며, 항상 Optional 인스턴스를 가리켜야 합니다."

- 아래 레시피들을 통해 3개의 질문에 대한 해법을 얻을 수 있습니다

### 질문

1. Optional을 사용할 때도 null pointer exception이 발생하는 경우는 무엇인가요?
2. 값이 존재하지 않을 때 무엇을 반환하거나 설정해야 하나요?
3. Optional 값을 효과적으로 소비하는 방법은 무엇인가요?

## 내가 Optional을 쓰는데도 null 을 얻게 되는 이유?
#### 1. optional에 null을 할당하지마라
```java
1 public Optional<Employee> getEmployee(int id) {
2    // perform a search for employee 
3    Optional<Employee> employee = null; // in case no employee
4    return employee; 
5 }
```
- 아래와 같이 변경하자
```java
Optional<Employee> employee = Optional.empty(); 
```


#### 2. get() 을 바로 호출하지마라

```java
Optional<Employee> employee = HRService.getEmployee();
Employee myEmployee = employee.get();
log.info("직원 ID : {}", myEmployee.getEmployeeId());
// myEmployee 가 null인 경우 NPE 유발
```

```java
if (employee.isPresent()) {
    Employee myEmployee = employee.get();
    log.info("직원 ID : {}", myEmployee.getEmployeeId()); 
} else {
    log.warn("직원이 없습니다."); 
}
```

- 사실 get(), isPresent() 를 활용하는건 추천되는 방법이 아니다. 대체안이 많다. 위 예제처럼 하기보단 아래 레시피들을 참고하자

#### 3. null reference 를 얻기 위해 null을 직접 쓰지마라
- 어떤 경우들에는 null을 가져와야하는 경우들이 존재한다
- Optional을 쓴다면 null을 바로 쓰지말고 `orElse(null)`을 사용하라

```java
if (employee.isPresent()) {
    Employee myEmployee = employee.get();
    HRService.startWork(myEmployee); 
} else {
    HRService.startWork(null);
}
```

```java

    HRService.startWork(employee.orElse(null));
```

- 위 케이스같은 경우엔 orElse(null) 을 쓰지만 일반적으로는 지양해야한다


## 값이 없을 때 반환하거나 set하는 방법은?
#### 4. 반환이나 세터 호출시 4. isPresent(), get() 쌍을 사용하는걸 지양해라

- 3번 과 비슷한 내용을 적용 가능하다

```java
public static final String DEFAULT_STATUS = "Unknown";
...
public String getEmployeeStatus(long id) {
    Optional<String> empStatus = ... ;
    if (empStatus.isPresent()) {
        return empStatus.get();
    } else {
        return DEFAULT_STATUS;
    }
}
```

- 위 코드는 아래처럼 바뀔 수 있다

```java
public String getEmployeeStatus(long id) {
    Optional<String> empStatus = ... ;
    return empStatus.orElse(DEFAULT_STATUS); 
}
```

> 중요한 사실은 바로 성능 제약이다. orElse() 는 옵셔널 값이 존재하던 하지않던 계산된다. 그래서 여기서의 규칙은, 이미 생성된 값을 할당하고, 매번 계산을 요구하는 값 (computed)을 사용하지 말아야한다.


#### 5. computed 반환 값으로 orElse()를 쓰지말라
- 4의 내용에 이어서, 계산이 요구되는 경우엔 성능하락의 원인이 될 수 있다

```java
Optional<Employee> getFromCache(int id) {
    System.out.println("search in cache with Id: " + id);
    // get value from cache
}

Optional<Employee> getFromDB(int id) {
    System.out.println("search in Database with Id: " + id);    
    // get value from database
}
```

```java
public Employee findEmployee(int id) {        
    return getFromCache(id)
            .orElse(getFromDB(id)
                    .orElseThrow(() -> new NotFoundException("Employee not found with id" + id)));}
```

- 위 코드는, 캐시에 있으면 캐시로부터 가져오고, 만약 없다면 DB를 조회하는 예제다
- 만약 직원이 캐시,데이터베이스에 둘다 없다면 NotFoundException을 뱉어낸다
- 만약 캐시에 이 직원이 있다하더라도, 결과는 예상과 다르게 나온다


```
Search in cache with Id: 1
Search in Database with Id: 1
```

- 캐시로부터 값이 있더라도 디비 쿼리는 실행될 것이다
- 그래서 대신 `orElseGet(Supplier<? extends T> supplier)`을 써야한다
- 위 함수는 Optional이 비어있을 때에만 supplier가 실행된다
```java
public Employee findEmployee(int id) {        
    return getFromCache(id)
        .orElseGet(() -> getFromDB(id)
            .orElseThrow(() -> {
                return new NotFoundException("Employee not found with id" + id);
            }));
}
// Search in cache with Id: 1
```


#### 6. 값이 없다면 예외를 던져라
- 값이 없음을 표시하기위해 예외를 던지는 경우가 필요할 것
- 디비나 다른 자원과 연결하는 서비스를 개발할때 필요하다
```java
public Employee findEmployee(int id) {        
    var employee = p.getFromDB(id);
    if(employee.isPresent())
        return employee.get();
    else
        throw new NoSuchElementException();
}
```
- 위 코드는 아래처럼 아름답게 변경 가능
```java
public Employee findEmployee(int id) {        
    return getFromDB(id).orElseThrow();
}
```


#### 7. 값이 없을 때 명시적 예외를 던지는 방법

```java
@GetMapping("/employees/{id}")
public Employee getEmployee(@PathVariable("id") String id) {
    return HrRepository
    .findByEmployeeId(id)
    .orElseThrow(
        () -> new NotFoundException("Employee not found with id " + id));
}
```

## Optional의 값을 효율적으로 소비하는 방법
#### 8. 값이 있을때만 실행하고싶은 행위가 있다면 isPresent(),get() 을 쓰지마라

- `ifPresent(Consumer<? super T> action)` 을 사용하라. 옵셔널의 값을 사용해 액션을 취하게한다

```java

1 Optional<String> confName = Optional.of("CodeOne");
2 if(confName.isPresent())
3    System.out.println(confName.get().length());
```

```java
confName.ifPresent( s -> System.out.println(s.length()));
```

#### 9. 값이 없을 때 실행하고 싶은 행위에도 쓰지마라

```java
1 Optional<Employee> employee = ... ;
2 if(employee.isPresent()) {
3    log.debug("Found Employee: {}" , employee.get().getName());
4 } else {
5    log.error("Employee not found");
6 }
```

- 위 코드대신 아래와 같이 작성 가능하다. 6줄을 2줄로 까지 줄일 수 있다
```java

employee.ifPresentOrElse(emp -> 
						 log.debug("Found Employee: {}",emp.getName()), 
						 () -> log.error("Employee not found"));
```


#### 10. 값이 없을 때에는 다른 Optional을 반환하라

```java
Optional<String> defaultJobStatus = Optional.of("Not started yet.");
public Optional<String> fetchJobStatus(int jobId) {
    Optional<String> foundStatus = ... ; // fetch declared job status by id
    if (foundStatus.isPresent())
        return foundStatus;
    else
        return defaultJobStatus; 
}
```
- 위를 보통 아래로 바꿀 것이다
```java
public Optional<String> fetchJobStatus(int jobId) {
    Optional<String> foundStatus = ... ; // fetch declared job status by id
    return foundStatus.orElseGet(() -> Optional.<String>of("Not started yet."));
}
```
- orElseGet() 의 남용이며 문제점은 unwrap된 겨로가를 리턴한다는 것이다
- 아래 처럼 하면 완벽하다
```java
1 public Optional<String> fetchJobStatus(int jobId) {
2    Optional<String> foundStatus = ... ; // fetch declared job status by id
3    return foundStatus.or(() -> defaultJobStatus);
	// return foundStatus.or(() -> Optional.of("Not started yet."));
4 }
```
- `defaultJobStatus` 을 정의할필요도 없게 할 수 있다

#### 11. Optional의 상태를 가져와라

- 자바 11부터 생긴 `isEmpty()` 메서드를 활용하면 바로 Optional이 비었는지 확인 가능하다
```java
1 public boolean isMovieListEmpty(int id){
2    Optional<MovieList> movieList = ... ;
3    return !movieList.isPresent();
4 }
```
```java
return movieList.isEmpty();
```

#### 12. Optional을 남용하지마라
- 때때로 개발자들은 Optional을 좋아하는 나머지 과용하는 경향이 있다
- 값을 얻기 위해 모든 곳에 chaning을 해서 쓰는걸 볼 수 있는데, 이러면 명확함도 떨어지고 메모리 풋프린트와 직관성을 잊어버림

```java
1 public String fetchJobStatus(int jobId) {
2    String status = ... ; // fetch declared job status by id
3    return Optional.ofNullable(status).orElse("Not started yet.");
4 }
```
```java
return status == null ? "Not started yet." : status;
```

- 값처리가 필요한 모든 영역에 Optional을 쓰기보다, 자바 기본 문법을 활용해도 충분한 경우가 있는지 잘 고민하자

## 결론

- 결국 Optional을 다른 자바 기능들과 동일하게, 적절히 쓰일 수도 있고 막무가내로 쓰일 수도 있다
- 잘 쓰려면 위 레시피들을 이해해두어야 한다
- 더 자세한 내용은 아래를 참고하라
- [documentation for Optional](https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/util/class-use/Optional.html).
- [java8 in action](https://www.manning.com/books/java-8-in-action)


- 위 내용은 [Mohamed Taman의 글](https://blogs.oracle.com/javamagazine/post/12-recipes-for-using-the-optional-class-as-its-meant-to-be-used)을 요약한 것입니다
