---
layout: post
title: java-to-kotlin-13 스트림에서 이터러블이나 시퀀스로
date: 2024-04-02 23:52 +0900
---

## 13장 java-to-kotlin-13 스트림에서 이터러블이나 시퀀스로

> 자바와 코틀린은 모두 컬렉션을 변환하고 축약하도록 해 준다. 하지만 각각의 설계 목표와 구현은 서로 다르다. 코틀린이 자바 스트림 대신 무엇을 사용할까? 어떻게 하면 자바 스트림을 이런 코틀린 구조로 변환할 수 있고, 언제 이런 변환을 수행해야 할까

## 자바 스트림

- 자바8에 스트림이 도입 (2014년, 생각보다 얼마 안되었다?!)
- 스트림에는 거의 대부분 람다를 활용한다

```java
# 일반적인 스트림 사용 예제
    public static double averageNonBlankLength(List<String> strings) {
  return strings.stream().filter(s -> !s.isBlank()).mapToInt(String::length).sum() / (double) strings.size();
  }
```

- 자바스트림은 지연계산을 수행한다. 뒤쪽 파이프라인 단계가 앞쪽에서 수행해야할 작업 양을 제한한다는 뜻

### 성능에 관하여
- 자바 스트림은 컬렉션이 작을경우 놀랍도록 느리다
- 자바 스트림은 대규모 데이터 처리에 특화되어 있다


## 코틀린에서는 두가지 추상화를 제공

### 이터러블

- 코틀린 이터러블은 Iterable에 대한 확장함수를 제공한다
- 자바에서는 Stream을 반환하는 filter method가 있는데, 코틀린에서는 List를 반환한다
```kotlin
class Test {
    fun averageNonBlankLength(strings: List<String>): Double =
        (strings.filter { it.isNotBlank() }.map(String::length).sum() / strings.size.toDouble())
}
```

> map, filter 모두 리스트를 반환한다. 즉 메모리에 2개의 리스트를 추가로 만든다..

- 장점은 뭘까? **바로 성능을 신경쓰지 않아도 된다는 것. 컬렉션 크키가 크지않다면 빠르게 동작한다**


### 시퀀스

- `Sequence` 추상화는 지연계산을 제공한다. 마치 자바 스트림처럼.
- `asSequence()` 라는 확장함수를 Iterable, Iterator, Collection에서 제공한다
- 코틀린 확장함수의 활용도를 보여주는 부분이라고 느꼈다

```kotlin
fun averageNonBlankLength2(strings: List<String>): Double = (strings.asSequence().filter
    { it.isNotBlank() }.map(String::length).sum() / strings.size.toDouble())
```


### 이터러블과 시퀀스 바꿔사용하기

- Iterable<T> vs Sequence<T> 는 서로 성격이 다르다. 같은 확장함수들 (map,filter,reduce)을 가지지만.
- 이터러블은 즉시계산, 시퀀스는 지연계산이기때문에 아무 문제없이 바꿔 쓸수는 없다
- 그저 장점은, 서로 바꿔쓸떄 코드를 많이 안바꿔도 된다는 사실
