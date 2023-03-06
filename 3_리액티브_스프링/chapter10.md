## 리액터 개요

**리액티브 프로그래밍**은 데이터 흐름(data flows)과 변화 전파에 중점을 둔 프로그래밍 패러다임이다.

(명령형 프로그래밍의 대안)

애플리케이션 코드를 개발할 때는 명령형과 리액티브의 두 가지 형태로 코드를 작성할 수 있다.

- 명령형 코드는 이것은 순차적으로 연속되는 작업이며, 각 작업은 한 번에 하나씩 그리고 이전 작업 다음에 실행된다. 데이터는 모아서 처리되고 이전 작업이 데이터 처리를 끝낸 후에 다음 작업으로 넘어갈 수 있다.
- 리액티브 코드는 데이터 처리를 위해 일련의 작업들이 정의되지만, 이 작업들은 병렬로 실행될 수 있다. 그리고 각 작업은 부분 집합의 데이터를 처리할 수 있으며, 처리가 끝난 데이터를 다음 작업에 넘겨주고 다른 부분 집합의 데이터로 계속 작업할 수 있다.

> 리액티브 프로그래밍은 만능이 아니다. 상황에 맞게 사용하는것이 중요하다..!

### 1-1. 리액티브 스트림 정의하기

리액티브 프로그맹의 비동기 특성은 동시에 여러 작업을 수행하여 더 큰 확장성을 얻게 해준다.

백 프레셔는 데이터를 소비하는 컨슈머가 처리할 수 있는 만큼으로 전달 데이터를 제한함으로써 지나치게 빠른 데이터 소스로부터의 데이터 전달 폭주를 방지한다.

리액티브 스트림의 4가지 인터페이스

1. publisher (발행자)
2. subscriber (구독자)
3. subsription (구독)
4. Processor (프로세서)

**1. Publisher**

```java
public interface Publisher<T> {
		void subscribe(Subscriber<? super T> subscriber);
}
```

하나의 publiser는 Subscribtion당 하나의 Subscriber에 발행(전송)하는 데이터를 생성한다.

**2. Subscriber**

```java
public interface Subscriber<T> {
		void onSubscribe(Subscription sub); // 구독할 첫 이벤트를 호출
		void onNext(T item); // publisher가 전송 하는 데이터가 subscriber 에게 전달
		void onError(Throwable ex); // 에러 발생시 호출 됨
		void onComplete(); // publisher가 subscriber 에게 작업종료를 공지
}
```

subscriber가 구독 신청 되면 publisher 로 부터 이벤트를 수신할 수 있다.

**3. Subscription**

```java
public interface Subscription {
		void request(long n); // 전송하는 데이터 요청, n은 백프레셔 즉 데이터의 항목수 이다.
		void cancel(); // 구독을 취소 요청한다.
}
```

subscriber는 subscription 객체를 통해서 구독을 관리할 수 있다.

**4. Processor**

```
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

processor 인터페이스느 subsriber와 publisher를 결합한 인터페이스이다.

### 1-2. 리액터 시작하기

리액티브 프로그래밍은 일련의 작업 단계를 기술하는 것이 아니라 데이터가 전달될 파이프라인을 구성하는 것이다.

그리고 이 파이프라인을 통해 데이터가 전달되는 동안 어떤 형태로든 변경 또는 사용될 수 있다.

> 명령형 프로그래밍
>

```java
String name = "sawook";
String capitalName = name.toUpperCase();
String greeting = "Hello, " + capitalName + "!";
System.out.println(greeting);
```

> 리액티브 프로그래밍
>

```java
Mono.just("sawook2")
		.map(n -> n.toUpperCase())
		.map(cn -> "Hello, " + cn + "!")
		.subscribe(System.out::println);
```

### 1-3. 리액티브 플로우의 마블 다이어그램

Flux : 0, 1 또는 다수의 데이터를 갖는 파이프라인

Mono: 하나의 데이터 항목만을 갖는 데이터 셋

> 리엑터는 두가지 타입으로 스트림을 정한다

+ Flux 마블 다이어그램 플로우
+ Mono 마블 다이어그램 플로우


마블 다이어그램의 제일 위에는 Flux나 Mono를 통해 전달되는 데이터의 타임라인을 나타내고,

중앙에는 오퍼레이션을, 제일 밑에는 결과로 생성되는 Flux나 Mono의 타임라인을 나타낸다.

### 1-4. 리액터 의존성 추가하기

리액터를 시작하려면 아래와 같이 의존성을 추가해야한다.

```xml
<dependency>
	<groupId>io.projectreactor</groupId>
    	<artifactId>reactor-core</artifactId>
</dependency>
```

리액터 테스트 코드 관련 의존성

```xml
<dependency>
	<groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>

```

### 1-5. 리액티브 오퍼레이션 적용하기

Flux와 Mono에는 500개 이상의 오퍼레이션이 있으며, 각 오퍼레이션은 다음과 같이 분류될 수 있다.

1. 생성(creation) 오퍼레이션
2. 조합(combination) 오퍼레이션
3. 변환(transformation) 오퍼레이션
4. 로직(logic) 오퍼레이션

### 1-6. 리액티브 타입 생성하기

### | `just()`

Flux 나 Mono의 `just()` 메서드를 사용하여 객체로부터 리액티브 타입을 생성할 수 있다.

```java
@Test
public void createAFlux_just() {
    Flux<String> fruitFlux = Flux
            .just("Apple", "Orange", "Grape", "Banana", "Strawberry");

    StepVerifier.create(fruitFlux)
        .expectNext("Apple")
        .expectNext("Orange")
        .expectNext("Grape")
        .expectNext("Banana")
        .expectNext("Strawberry")
        .verifyComplete();
}
```

### | `fromArray()`

배열로부터 Flux를 생성하려면 static 메서드인 `fromArray()`를 호출하며, 이때 소스 배열을 인자로 전달한다.

```java
@Test
public void createAFlux_fromArray() {
    String[] fruits = new String[] {"Apple", "Orange", "Grape", "Banana", "Strawberry" };
    Flux<String> fruitFlux = Flux.fromArray(fruits);

    StepVerifier.create(fruitFlux)
        .expectNext("Apple")
        .expectNext("Orange")
        .expectNext("Grape")
        .expectNext("Banana")
        .expectNext("Strawberry")
        .verifyComplete();
}
```

### | `fromIterable()`

java.util.List, java.util.Set 또는 java.lang.Iterable 로부터 Flux를 생성해야 한다면 해당 컬렉션을 인자로 전달하여 `fromIterable()`을 호출하면 된다.

```java
@Test
public void createAFlux_fromIterable() {
    List<String> fruitList = new ArrayList<>();
    fruitList.add("Apple");
    fruitList.add("Orange");
    fruitList.add("Grape");
    fruitList.add("Banana");
    fruitList.add("Strawberry");

    Flux<String> fruitFlux = Flux.fromIterable(fruitList);
}
```

### | `fromStream()`

Flux를 생성하는 소스로 자바 Stream 객체를 사용해야 한다면 `fromStream()`을 호출하면 된다.

```java
@Test
public void createAFlux_fromStream() {
    Stream<String> fruitStream =
        Stream.of("Apple", "Orange", "Grape", "Banana", "Strawberry");
    Flux<String> fruitFlux = Flux.fromStream(fruitStream);
}
```

### | `range()`

데이터 없이 매번 새 값으로 숫자를 방출하는 카운터 역할의 flux를 생성할 수 있다.

```java
@Test
public void createAFlux_range() {
    Flux<Integer> intervalFlux = Flux.range(1, 5);

    StepVerifier.create(intervalFlux)
        .expectNext(1)
        .expectNext(2)
        .expectNext(3)
        .expectNext(4)
        .expectNext(5)
        .verifyComplete();
}
```

### | `interval()`

`interval()`는 시작 값과 종료 값 대신 값이 방출되는 시간 간격이나 주기를 설정한다.

```java
@Test
public void createAFlux_interval() {

    Flux<Long> intervalFlux =
        Flux.interval(Duration.ofSeconds(1))
            .take(3);

    StepVerifier.create(intervalFlux)
        .expectNext(0L)
        .expectNext(1L)
        .expectNext(2L)
        .expectNext(3L)
        .expectNext(4L)
        .verifyComplete();
}
```

### 1-7 리액티브 타입 조합하기

두 개의 리액티브 타입을 결합해야 하거나 하나의 Flux를 두 개 이상의 리액티브 타입으로 분할해야 하는 경우가 있을 수 있다.

여기서는 리액터의 Flux나 Mono를 결합하거나 분할하는 오퍼레이션을 알아보자

### | `**mergeWith()**`

두 개의 Flux 스트림을 하나의 결과 Flux로 생성해보자.

```java
@Test
public void mergeFluxes() {

    Flux<String> characterFlux = Flux.just("Garfield", "Kojak", "Barbossa").delayElements(Duration.ofMillis(500));
    Flux<String> foodFlux = Flux.just("Lasagna", "Lollipops", "Apples")
                                .delaySubscription(Duration.ofMillis(250))
                                .delayElements(Duration.ofMillis(500));

    Flux<String> mergedFlux = characterFlux.mergeWith(foodFlux);

    StepVerifier.create(mergedFlux)
                .expectNext("Garfield")
                .expectNext("Lasagna")
                .expectNext("Kojak")
                .expectNext("Lollipops")
                .expectNext("Barbossa")
                .expectNext("Apples")
                .verifyComplete();
}
```

두 Flux 객체가 결합되면 하나의 Flux가 새로 생성된다. 그리고 mergedFlux를 StepVerifier가 구독할 때 데이터의 흐름이 시작되면서 두 소스 Flux 스트림을 번갈아 구독하게 된다.

mergedFlux로부터 방출되는 항목 순서는 두 Flux 로 방출되는 시간에 맞추어 결정된다. 여기서는 두 Flux 객체 모두 일정한 속도로 방출되게 설정되었으므로 번갈아 mergedFlux에 끼워진다.

### 1-8 리액티브 스트림의 변환과 필터링

데이터가 스트림을 통해 흐르는 동안 일부 값을 필터링 하거나 다른 값으로 변경해야 할 경우가 있는데, 이럴 때 주로 사용된다.

### | `skip()`

`skip()` 은 지정된 수의 메시지를 건너뛴 후에 나머지 메시지를 결과 Flux로 전달한다.`skip()`은 지정된 시간이 경과할 때 까지 기다렸다가 결과 flux 로 메시지를 전달한다.

```java
@Test
public void skipAFew() {
    Flux<String> countFlux = Flux.just(
        "one", "two", "skip a few", "ninety nine", "one hundred")
        .skip(3); 
    StepVerifier.create(countFlux)
        .expectNext("ninety nine", "one hundred")
        .verifyComplete();
    }
```

```java
@Test
public void skipAFewSeconds() {
    Flux<String> countFlux = Flux.just(
        "one", "two", "skip a few", "ninety nine", "one hundred")
        .delayElements(Duration.ofSeconds(1))
        .skip(Duration.ofSeconds(4));
    StepVerifier.create(countFlux)
        .expectNext("ninety nine", "one hundred")
        .verifyComplete();
}
```

### | `take()`

`take()` 은 입력 flux로 부터 처음 지정된 수의 메시지만 전달하고 구독을 취소 시킨다.

```java
@Test
public void take() {
    Flux<String> nationalParkFlux = Flux.just(
        "Yellowstone", "Yosemite", "Grand Canyon", "Zion", "Acadia")
        .take(3); // 3개만 방출

    StepVerifier.create(nationalParkFlux)
        .expectNext("Yellowstone", "Yosemite", "Grand Canyon")
        .verifyComplete();
}
```

```java
@Test
public void takeForAwhile() {
    Flux<String> nationalParkFlux = Flux.just(
        "Yellowstone", "Yosemite", "Grand Canyon", "Zion", "Grand Teton")
        .delayElements(Duration.ofSeconds(1))
        .take(Duration.ofMillis(3500)); //3.5 초 동안만  방출

    StepVerifier.create(nationalParkFlux)
        .expectNext("Yellowstone", "Yosemite", "Grand Canyon")
        .verifyComplete();
}
```

### |`filter()`

`filter()` 로 지정된 조건 식에 일치되는 메시지만 결과 Flux가 수신하도록 입력 flux를 필터링 할 수 있다.

```java
@Test
public void filter() {
    // 공백 있는 메시지 필터링
    Flux<String> nationalParkFlux = Flux.just(
        "Yellowstone", "Yosemite", "Grand Canyon", "Zion", "Grand Teton")
        .filter(np -> !np.contains(" "));

    StepVerifier.create(nationalParkFlux)
        .expectNext("Yellowstone", "Yosemite", "Zion")
        .verifyComplete();
}
```

### | `distinct()`

`distinct()` 는 중복 메시지를 제거해준다.

```java
@Test
public void distinct() {
    Flux<String> animalFlux = Flux.just(
        "dog", "cat", "bird", "dog", "bird", "anteater")
        .distinct(); // 중복 필터링

    StepVerifier.create(animalFlux)
        .expectNext("dog", "cat", "bird", "anteater")
        .verifyComplete();
}
```

### `map()`

`map()` 은 입력 메시지의 변환을 수행하여 결과 스트림의 새로운 메시지로 발행한다.

```java
@Test
public void map() {
    Flux<Player> playerFlux = Flux
    .just("Michael Jordan", "Scottie Pippen", "Steve Kerr")
    .map(n -> {
        String[] split = n.split("\\s");
        return new Player(split[0], split[1]);
    });

    StepVerifier.create(playerFlux)
        .expectNext(new Player("Michael", "Jordan"))
        .expectNext(new Player("Scottie", "Pippen"))
        .expectNext(new Player("Steve", "Kerr"))
        .verifyComplete();
}
```

### |`flatMap()`

`flatMap()` 오퍼레이션은 수행 도중 생성되는 임시 Flux 를 사용하여 변환을 수행하므로 비동기 변환이 가능하다.

`subscribe()` : 리액티브 플로우를 구독 요청하고 실제로 구독`subscribeOn()` : 구독이 동시적으로 처리

```java
@Test
public void flatMap() {
    Flux<Player> playerFlux = Flux
        .just("Michael Jordan", "Scottie Pippen", "Steve Kerr")
        .flatMap(n -> Mono.just(n)
            .map(p -> {
                String[] split = p.split("\\s");
                return new Player(split[0], split[1]);
            })
            .subscribeOn(Schedulers.parallel()) // 구독을 병렬 스레드로 수행
        );

    List<Player> playerList = Arrays.asList(
        new Player("Michael", "Jordan"),
        new Player("Scottie", "Pippen"),
        new Player("Steve", "Kerr"));

    StepVerifier.create(playerFlux)
        .expectNextMatches(p -> playerList.contains(p))
        .expectNextMatches(p -> playerList.contains(p))
        .expectNextMatches(p -> playerList.contains(p))
        .verifyComplete();
    }
```

### |`buffer()`

`buffer()` 는 지정된 최대 크기의 리스트(입력 Flux로부터 수집된)로 된 flux를 생성한다.

```arduino
@Test
public void buffer() {
    Flux<String> fruitFlux = Flux.just(
        "apple", "orange", "banana", "kiwi", "strawberry");

    Flux<List<String>> bufferedFlux = fruitFlux.buffer(3); // 각각 3개 미만을 포함하도록한다.

    StepVerifier
        .create(bufferedFlux)
        .expectNext(Arrays.asList("apple", "orange", "banana"))
        .expectNext(Arrays.asList("kiwi", "strawberry"))
        .verifyComplete();
}

@Test
public void bufferAndFlatMap() throws Exception {
    Flux.just(
        "apple", "orange", "banana", "kiwi", "strawberry")
        .buffer(3)  // 각각 3개 미만을 포함하도록한다
        .flatMap(x ->
            Flux.fromIterable(x)
            .map(y -> y.toUpperCase())
            .subscribeOn(Schedulers.parallel())
            .log()
        ).subscribe();
}
```

### `collectionList()`

`collectionList()` 는 입력 Flux가 방출한 모든 메시지를 갖는 List의 Mono를 생성한다.

```arduino
@Test
public void collectList() {
    Flux<String> fruitFlux = Flux.just(
        "apple", "orange", "banana", "kiwi", "strawberry");

    Mono<List<String>> fruitListMono = fruitFlux.collectList();

    StepVerifier
        .create(fruitListMono)
        .expectNext(Arrays.asList(
            "apple", "orange", "banana", "kiwi", "strawberry"))
        .verifyComplete();
}
```

### `collectMap()`

`collectMap()` 은 Map을 포함하는 Mono를 생성한다. 이때 입력 Flux가 방출한 메시지가 해당 Map의 항목으로 저장되며, 각 항목의 키는 입력 메시지의 특성에 따라 추출된다.

```arduino
@Test
public void collectMap() {
    Flux<String> animalFlux = Flux.just(
        "aardvark", "elephant", "koala", "eagle", "kangaroo");

    Mono<Map<Character, String>> animalMapMono =
        animalFlux.collectMap(a -> a.charAt(0));

    StepVerifier
        .create(animalMapMono)
        .expectNextMatches(map -> {
            return
                map.size() == 3 &&
                map.get('a').equals("aardvark") &&
                map.get('e').equals("eagle") &&
                map.get('k').equals("kangaroo");
        })
        .verifyComplete();
}
```

### 10.3.4 리액티브 타입에 로직 오퍼레이션 수행하기

### |`all, any()`

`all()` 은 모든 메시지가 조건을 충족하는지 확인한다.`any()` 는 최소한 하나의 메시지가 조건을 충족하는지 검사한다.

```java
@Test
public void all() {
    Flux<String> animalFlux = Flux.just(
        "aardvark", "elephant", "koala", "eagle", "kangaroo");

    Mono<Boolean> hasAMono = animalFlux.all(a -> a.contains("a"));
    StepVerifier.create(hasAMono)
        .expectNext(true)
        .verifyComplete();

    Mono<Boolean> hasKMono = animalFlux.all(a -> a.contains("k"));
    StepVerifier.create(hasKMono)
        .expectNext(false)
        .verifyComplete();
}

@Test
public void any() {
    Flux<String> animalFlux = Flux.just(
        "aardvark", "elephant", "koala", "eagle", "kangaroo");

    Mono<Boolean> hasAMono = animalFlux.any(a -> a.contains("a"));
    StepVerifier.create(hasAMono)
        .expectNext(true)
        .verifyComplete();

    Mono<Boolean> hasZMono = animalFlux.any(a -> a.contains("z"));
    StepVerifier.create(hasZMono)
        .expectNext(false)
        .verifyComplete();
}
```

## 요약

- 리액티브 프로그래밍에서는 데이터가 흘러가는 파이프라인을 생성한다.
- 리액티브 스트림은 Publisher, Subscriber, Subscription, Transformer의 4가지 타입을 정의한다.
- 프로젝트 리액터는 리액티브 스트림을 구현하며 수많은 오퍼레이션을 제공하는 Flux와 Mono 두 가지 타입으로 스트림을 정의한다.
- 스프링 5는 리액터를 사용해서 리액티브 컨트롤러, 리퍼지토리, 레스트 클라이언트 를 생성하고 다른 리액티브 프레임워크를 지원한다.
