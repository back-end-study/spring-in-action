# Chapter7. REST 서비스 사용하기

스프링을 사용해서 다른 REST API와 상호작용하는 방법을 알아보자

- **RestTemplate** : 스프링 프레임워크에서 제공하는 간단하고 동기화된 REST 클라이언트
- **Traverson** : 스프링 HATEOAS에서 제공하는 하이퍼링크를 인식하는 동기화 REST 클라이어늩로 같은 이름의 자바스크립트 라이브러리로부터 비롯된 것이다
- **WebClient** : 스프링 5에서 소개된 반응형 비동기 REST 클라이언트 (11장에서 더 알아보자)

## 7.1 RestTemplate으로 REST 엔드 포인트 사용하기

클라이언트 입장에서 REST 리소스와 상호작용하려면 코드가 장황해진다

저수준의 HTTP 라이브러리로 작업하면서 클라이언트는 

- 클라이언트 인스턴스와 요청 객체를 생성하고
- 해당 요청을 실행하고
- 응답을 분석하여 관련 도메인 객체와 연관시켜 처리하고
- 그 와중에 발생될 수 있는 예외도 처리해야한다

그리고 매 HTTP 요청마다 위 작업들이 반복된다

이처럼 장황한 코드를 피하기 위해 스프링은 RestTemplate을 제공한다

아래는 RestTemplate이 정의하는 고유한 작업을 수행하는 12개의 메서드이다

(REST 리소스와 상호작용하기 위한 41개의 메서드가 있지만 나머지는 12개의 오버로딩된 버전이다)

| 메서드 | 기능 설명 |
| --- | --- |
| delete(…) | 지정된 URL의 리소스에 HTTP DELETE 요청을 수행한다 |
| exchange(…) | 지정된 HTTP 메서드를 URL에 대해 실행하며, 응답 몸체와 연결되는 객체를 포함하는 ResponseEntity를 반환 |
| execute(…) | 지정된 HTTP 메서드를 URL에 대해 실행하며, 응답 몸체와 연결되는 객체를 반환 |
| getForEntity(…) | HTTP GET 요청을 전송하며, 응답 몸체와 연결되는 객체를 포함하는 ResponseEntity를 반환 |
| getForObject(…) | HTTP GET 요청을 전송하며, 응답 몸체와 연결되는 객체를 반환 |
| headForHeaders(…) | HTTP HEAD 요청을 전송하며, 지정된 리소스의 URL의 HTTP헤더를 반환 |
| optionsForAllow(…) | HTTP OPTIONS 요청을 전송하며, 지정된 URL의 Allow 헤더를 반환 |
| patchForObject(…) | HTTP PATCH 요청을 전송하며, 응답 몸체와 연결되는 결과 객체를 반환 |
| postForEntity(…) | URL에 데이터를 POST하며, 응답 몸체와 연결되는 객체를 포함하는 ResponseEntity를 반환 |
| postForLocation(…) | URL에 데이터를 POST하며, 새로 생성된 리소스의 URL을 반환 |
| postForObject(…) | URL에 데이터를 POST하며, 응답 몸체와 연결되는 객체를 반환 |
| put(…) | 리소스 데이터를 지정된 URL에 PUT |

위 12개의 메소드는 세가지 형태로 오버로딩 되어있다

![https://user-images.githubusercontent.com/68587990/220318714-a1656eb6-cce8-468a-b5e4-e19a9223ada8.png](https://user-images.githubusercontent.com/68587990/220318714-a1656eb6-cce8-468a-b5e4-e19a9223ada8.png)

- 가변 인자 리스트에 지정된 URL 매개변수에 URL 문자열(String 타입)을 인자로 받는다
- Map<String, String>에 지정된 URL 매개변수에 URL 문자열을 인자로 받는다
- java.net.URI를 URL에 대한 인자로 받으며, 매개변수화된 URL은 지원하지 않는다

RestTemplate는 객체를 생성하거나 Bean으로 주입받아서 사용할 수 있다

```java
RestTemplate rest = new RestTemplate();

// or

@Bean
public RestTemplate restTemplate() {
    retrun new RestTemplate();
}
```

### 7.1.1 리소스 가져오기(GET)

타코 클라우드 API로부터 식자재(ingredient)를 가져온다고 해보자****

```java
public Ingredient getIngredientById(String ingredientId) {
    return rest.getForObject("http://localhost:8080/ingredients/{id}",
                             Ingredient.class, 
                             ingredientId);
}
```

다른 방법으로는 Map을 사용해서 URL 변수들을 지정할 수 있다

```java
public Ingredient getIngredientById(String ingredientId) {
    Map<String, String> urlVariables = new HashMap<>();
    urlVariables.put("id", ingredientId);
    return rest.getForObject("http://localhost:8080/ingredients/{id}",
                             Ingredient.class, 
                             urlVariables);
}
```

이와는 달리 URI 매개변수를 사용할 때는 URI 객체를 구성하여 getForObject()를 호출해야 한다

```java
public Ingredient getIngredientById(String ingredientId) {
    Map<String, String> urlVariables = new HashMap<>();
    urlVariables.put("id", ingredientId);
    URI url = UriComponentsBuilder
              .fromHttpUrl("http://localhost:8080/ingredients/{id}")
              .build(urlVariables);
    return rest.getForObject(url, Ingredient.class);
}
```

getForEntity()는 getForObject()와 같은 방법으로 작동하지만, 응답 결과를 나타내는 도메인 객체를 반환하는 대신 도메인 객체를 포함하는 ResponseEntity 객체를 반환한다

ResponseEntity에는 응답 헤더와 같은 더 상세한 응답 콘텐츠가 포함될 수 있다.

```java
public Ingredient getIngredientById(String ingredientId) {
    ResponseEntity<Ingredient> responseEntity =
        rest.getForEntity("http://localhost:8080/ingredients/{id}",
                          Ingredient.class, 
                          ingredientId);
    log.info("Fetched time: " + responseEntity.getHeaders().getDate());
    return responseEntity.getBody();
}
```

### 7.1.2 리소스 쓰기(PUT)

HTTP PUT 요청을 전송하기 위해 RestTemplate은 put() 메서드를 제공한다

```java
public void updateIngredientById(Ingredient ingredientId) {
    rest.put("http://localhost:8080/ingredients/{id}",
             ingredient,
             ingredient.getId());
}
```

### 7.1.3 리소스 삭제하기(DELETE)

타코 클라우드에서 특정 식자재를 더 이상 제공하지 않으므로 해당 식자재를 완전히 삭제하고 싶다고 해보자

```java
public void deleteIngredient(Ingredient ingredientId) {
    rest.delete("http://localhost:8080/ingredients/{id}",
                ingredient.getId());
}
```

### 7.1.4 리소스 데이터 추가하기(POST)

새로운 식자재를 타코 클라우드 메뉴에 추가한다고 해보자

RestTemplate은 POST 요청을 전송하는 오버로딩된 3개의 메서드를 갖고 있으며, URL을 지정하는 형식은 모두 같다

```java
public Ingredient createIngredient(Ingredient ingredientId) {
    return rest.postForObject("http://localhost:8080/ingredients/{id}",
                              ingredient, 
                              Ingredient.class);
}
```

만일 클라이언트에서 새로 생성된 리소스의 위치가 추가로 필요하다면 postForObject() 대신 postForLocation()을 호출할 수 있다

```java
public URI createIngredient(Ingredient ingredient) {
    return rest.postForLocation("http://localhost:8080/ingredients",
                                ingredient, 
                                Ingredient.class);
```

postForLocation()은 postForObject()와 동일하게 작동하지만, 리소스 객체 대신 새로 생성된 리소스의 URI를 반환한다는 것이 다르다

```java
public Ingredient createIngredient(Ingredient ingredient) {
    ResponseEntity<Ingredient> responseEntity =
        rest.postForEntity("http://localhost:8080/ingredients",
                           ingredient, 
                           Ingredient.class);
    log.info("New resource created at " + responseEntity.getHeaders().getLocation());
    return responseEntity.getBody();
}
```

## 7.2 Traverson으로 REST API 사용하기

Traverson은 스프링 데이터 HATEOAS에 같이 제공되며, 스프링 애플리케이션에서 하이퍼미디어 API를 사용할 수 있는 솔루션이다. 

Traverson은 ‘돌아다닌다는 traverse on’의 의미로, 여기서는 관계 이름으로 원하는 API를 이동하며 사용할 것이다

Traverson을 사용할 때는 우선 해당 API의 기본 URI를 갖는 객체를 생성해야한다

```java
Traverson traverson = new Traverson(URI.create("http://localhost:8080/api"), MediaTypes.HAL_JSON);
```

Traverson 객체가 생성되었으므로 이제는 링크를 따라가면서 API를 사용할 수 있다

예를 들어, 모든 식자재 리스트를 가져온다고 해보자. 각 ingredients 링크들은 해당 식자재 리소스를 링크하는 href 속성을 가지므로 그 링크를 따라가면 된다.

```java
public Iterable<Ingredient> getAllIngredientsWithTraverson() {
    ParameterizedTypeReference<Resources<Ingredient>> ingredientType = new ParameterizedTypeReference<Resources<Ingredient>>() {};

    Resources<Ingredient> ingredientRes = traverson
                                          .follow("ingredients")
                                          .toObject(ingredientType);
	
    Collection<Ingredient> ingredients = ingredientRes.getContent();
	
    return ingredients;
}
```

다음은 가장 최근에 생성된 타코들을 갖져온다고 하자. 이때는 다음과 같이 홈 리소스에서 시작해서 가장 최근에 생성된 타코 리소스로 이동할 수 있다.

```java
public Iterable<Taco> getRecentTacosWithTraverson() {
    ParameterizedTypeReference<Resources<Taco>> tacoType = new ParameterizedTypeReference<Resources<Taco>>() {};

    Resources<Taco> tacoRes = traverson
                              .follow("tacos")
                              .follow("recents")      // follow("tacos", "recents")로 사용 가능
                              .toObject(tacoType);

    Collection<Taco> tacos = tacoRes.getContent();
    return tacos;
}
```

여기서는 tacos 링크 다음에 recents 링크를따라간다. 

그러면 최근 생성된 타코 리소스에 도달하므로 toObject()를 호출하여 해당 리소스를 가져올 수 있다. 

여기서 tacoType은 ParameterizedTypeReference 객체로 생성되었으며, 우리가 원하는 Resources<Taco> 타입이다.

Traverson은 API에 리소스를 쓰거나 삭제하는 메서드를 제공하지 않는다.

RestTemplate은 리소스를 쓰거나 삭제할 수 있지만, API를 이동하는 것은 쉽지 않다

따라서 API의 이동과 리소스의 변경이나 삭제를 모두 해야 한다면 RestTemplate과 Traverson을 함께 사용해야한다

예를 들어, 새로운 식자재를 타코 클라우드 메뉴에 추가하고 싶다면, 다음의 addIngredient() 메서드와 같다

```java
public Ingredient addIngredient(Ingredient ingredient) {
    String ingredientsUrl = traverson
                             .follow("ingredients")
                             .asLink()
                             .getHref();
    return rest.postForObject(ingredientsUrl,
                              ingredient,
                              Ingredient.class);
  }
```

```java
public Ingredient addIngredient(Ingredient ingredient) {
    String ingredientsUrl = traverson
                            .follow("ingredients")
                            .asLink()
                            .getHref();

    return rest.postForObject(ingredientsUrl,
                              ingredient,
                              Ingredient.class);
}
```

## 요약

- 클라이언트는 RestTemplate을 사용해서 REST API에 대한 HTTP 요청을 할 수 있다.
- Traverson을 사용하면 클라이언트가 응답에 포함된 하이퍼링크를 사용해서 원하는  API로 이동할 수 있다.