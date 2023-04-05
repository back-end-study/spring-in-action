
# 학습 목표
- 스프링 데이터의 리액티브 리퍼지터리
- 카산드라와 몽고DB의 리액티브 리퍼지터리 작성하기
- 리액티브가 아닌 리퍼지터리를 리액티브 사용에 맞추어 조정하기
- 카산드라를 사용한 데이터 모델링

- 이전에는 스프링 WebFlux를 사용해서 리액티브하고 블로킹이 없는 컨트롤러를 생성하는 방법을 배움. 이는 웹 계층의 확장성을 향상시키는데 도움이 됨.

- 그러나 **같이 작동되는 다른 컴포넌트도 블로킹이 없어야 진정한 블로킹 없는 컨트롤러**가 될 수 있음.

- 만일 블로킹 되는 리퍼지터리에 의존하는 스프링 WebFlux 리액티브 컨트롤러를 작성한다면, 이 컨트롤러는 해당 리퍼지터리의 데이터 생성을 기다리느라 블로킹될 것.

- 따라서 컨트롤러로부터 데이터베이스에 이르기까지 데이터의 전체 flow가 리액티브하고 블로킹되지 않는 것이 중요하다.

## 12.1 스프링 데이터의 리액티브 개념 이해하기
- 스프링 데이터는 Kay 릴리즈 트레인부터 Reactvie Repository의 지원을 제공 시작
- Reactvie Repository는 카산드라, 몽고DB, 카우치베이스, Redis 등을 지원

- 하지만 RDB나 JPA는 지원하지 않는데, 이들은 표준화된 비동기 API를 제공하지 않음
- 따라서 앞으로 **카산드라와 몽고DB를 이용**하여 스프링 데이터 리액티브를 사용해볼 것이다.

### 12.1.1 스프링 데이터 리액티브 개요
**Reactvie Repository**
- 도메인 타입이나 컬렉션 대신, Mono나 Flux를 인자로 받거나 반환하는 메서드를 가짐

데이터베이스로부터 식자재 타입으로 Ingredient 객체들을 가져오는 리퍼지터리 메서드

```
Flux<Ingredient> findByType(Ingredient.Type type);
```

Taco 객체를 저장하는 리액티브 리퍼지터리의 메서드 시그니처

```
<Taco> Flux<Taco> saveAll(Publisher<Taco> tacoPublisher);
```


### 12.1.2 리액티브와 리액티브가 아닌 타입 간의 변환

- 기존에 RDB를 사용중일 때도 Reactive Programming을 애플리케이션에 적용 가능
-  RDB가 블로킹 없는 Reactive Query를 지원하지 않더라도, 우선 블로킹 되는 방식으로 데이터를 가져와서 가능한 빨리 리액티브 타입으로 변환하여 상위 컴포넌트들이 Reactive의 장점을 활용 가능
- 예를 들어, RDB와 스프링 데이터 JPA를 사용하는 경우 OrderRepository는 다음과 같은 시그니처의 메서드를 가질 수 있다.

```
List<Order> findByUser(User user);
```

- findByUser()는 블로킹 방식으로 동작
- 이유 : List가 Reactive 타입이 아니므로 어떤 Reactive 오퍼레이션도 수행할 수 없기 때문
컨트롤러가 findByUser()를 호출했다면 결과를 리액티브하게 사용할 수 없어 확장성을 향상시킬 수 없음

이 경우엔 가능한 빨리 Reactive가 아닌 List를 Flux로 변환하여 결과를 처리

```
List<Order> orders = repo.findByUser(someUser);
Flux<Order> orderFlux = Flux.fromIterable(orders);
```

Mono를 사용할 경우엔 아래와 같이 작성하면 된다.

```
Order order = repo.findById(id);
Mono<Order> orderFlux = Mono.just(order);
```

- 이처럼 Mono의 just()나 Flux의 fromIterable(), fromArray(), fromStream()을 사용하면
Repository의 Reactive가 아닌 블로킹 코드를 격리시키고 애플리케이션의 어디서든 Reactive 타입으로 처리 가능

이번엔 저장하는 경우, Mono나 Flux 모두 자신들이 발행하는 데이터를 도메인 타입이나 Iterable 타입으로 추출하는 오퍼레이션을 가짐

```
Taco taco = tacoMono.block();
tacoRepo.save(taco);
```

```
Iterable<Taco> tacos = tacoFlux.toIterable();
tacoRepo.saveAll(tacos);

```
- Mono의 block()이나 Flux의 toIterable()은 추출 작업을 할 때 블로킹
따라서 이런식의 Mono와 Flux를 사용을 최소화 해야 한다.

- 블로킹되는 타입을 더 Reactive하게 추출 가능
Mono나 Flux를 구독하면서 발행되는 요소 각각에 대해 원하는 오퍼레이션을 수행

```
tacoFlux.subscribe(
        taco -> {
            tacoRepo.save(taco);
        }
);
```

tacoRepo의 save는 여전히 블로킹 오퍼레이션이다.

그러나 Flux나 Mono가 발행하는 데이터를 소비하고 처리하는 Reactive 방식의 subscribe()를 사용하므로
블로킹 방식의 일괄처리보다는 더 바람직하다.

### 12.1.3 리액티브 리퍼지터리 개발

- 데이터 퍼시스턴스를 제공하는 백엔드로 이 데이터베이스들을 사용하면, 스프링 애플리케이션이 웹 계층부터 데이터베이스까지에 걸쳐 진정한 엔드-to-엔드 리액티브 플로우 제공

## 12.2 리액티브 카산드라 리퍼지터리 사용하기

### 카산드라
- 분산처리, 고성능, 상시 가용, 궁극적인 일관성을 갖는 NoSQL DB
- 데이터를 테이블에 저장된 행으로 처리, 각 행은 일 대 다 관계의 많은 분산 노드에 걸쳐 분할됨
- 한 노드가 모든 데이터를 갖지 않으나 특정 행은 다수의 노드에 걸쳐 복제될 수 있어 단일 장애점을 없앰

- 스프링 데이터 카산드라는 카산드라 데이터베이스의 자동화된 리퍼지터리 자원 제공
- 애플리케이션의 도메인 타입을 데이터베이스 구조에 매핑하는 애노테이션 제공

### 12.2.1 스프링 데이터 카산드라 활성화하기

1. 카산드라의 리액티브가 아닌 리퍼지터리를 작성 시
2. 리액티브 리퍼지터리 활성화를 위한 스타터 의존성

- 최소한 리퍼지터리가 운용되는 키 공간의 이름을 구성 -> **키 공강 생성**
- 카산드라 CQL 쉘에서 다음과 같은 create keyspace 명령을 사용하면 타코 클라우드 애플리케이션의 키 공간 생성

### 12.2.2 카산드라 데이터 모델링 이해하기

- 카산드라 테이블은 얼마든지 많을 열을 가질 수 있음.
	- 모든 행이 같은 열을 갖지 않고, 행마다 서로 다른 열을 가질 수 있음.
- 카산드라 데이터베이스는 다수의 파티션에 결쳐 분할됨.
- 카산드라 테이블은 두 종류의 키(파티션 키와 클러스터링 키)를 가짐.
	- 파티션 키 : 각 행이 유지 관리되는 파티션을 결정하기 위한 해시 오퍼레이션이 각 행의 파티션 키에 수행됨
    - 클러스터링 키 : 각 행이 파티션 내부에서 유지 관리되는 순서 결정.
- 카산드라는 읽기 오퍼레이션에 최적화됨.


### 12.2.3 카산드라 퍼시스턴스의 도메인 타입 매핑

스프링 데이터 카산드라는 유사한 목적의 매핑 애노테이션 제공


#### 카산드라에서 사용 가능한 Ingredient 클래스
```
package tacos;

import lombok.Data;

@Data
public class Ingredient {

  private final String name;
  private final Type type;
  
  public static enum Type {
    WRAP, PROTEIN, VEGGIES, CHEESE, SAUCE
  }
  
}
```
- JPA 퍼시스턴스에서 클래스에 지정했던 @Entity 대신 **@Table** 지정
- @Table : 식재료 데이터가 ingredients 테이블에 저장 및 유지되어야 한다는 것을 나타냄
- id 속성에 지정했던 @Id 대신 @PrimaryKey 애노테이션 지정

#### 12.1 Taco 클래스를 카산드라 tacos 테이블로 매핑
```
package tacos;

import java.util.Date;
import java.util.List;
import java.util.UUID;

import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;

import org.springframework.data.cassandra.core.cql.Ordering;
import org.springframework.data.cassandra.core.cql.PrimaryKeyType;
import org.springframework.data.cassandra.core.mapping.Column;
import org.springframework.data.cassandra.core.mapping.PrimaryKeyColumn;
import org.springframework.data.cassandra.core.mapping.Table;
import org.springframework.data.rest.core.annotation.RestResource;

import com.datastax.driver.core.utils.UUIDs;

import lombok.Data;

@Data
@RestResource(rel = "tacos", path = "tacos")
@Table("tacos") //tacos 테이블에 저장&유지
public class Taco {

  @PrimaryKeyColumn(type=PrimaryKeyType.PARTITIONED)//파티션 키를 정의함
  private UUID id = UUIDs.timeBased();
  
  @NotNull
  @Size(min = 5, message = "Name must be at least 5 characters long")
  private String name;
  
  @PrimaryKeyColumn(type=PrimaryKeyType.CLUSTERED,
                    ordering=Ordering.DESCENDING)//클러스터링 키 정의
  private Date createdAt = new Date();
  
  @Size(min=1, message="You must choose at least 1 ingredient")
  @Column("ingredients")//List를 ingredients열에 매핑
  private List<IngredientUDT> ingredients;

}
```

- 테이블 이름을 tacos로 지정하기 위해 @Table 애노테이션 사용
- id 속성 : 
	- PrimaryKeyType.PARTITIONED 타입으로 @PrimaryKeyColumn에 지정. 타코 데이터의 각 행이 저장된 카산드라 파티션을 결정하기 위해 사용되는 파티션 키가 id 속성인 것을 나타냄.
    - id 속성의 타입은 Long 대신 UUID인데 자동으로 생성되는 ID 값을 저장하는 속성에 흔히 사용하는 타입. (UUID는 새로운 Taco 객체가 생성될 때 시간 기반의 UUID 값으로 초기화됨)
    
- createdAt 속성 : PrimaryKeyType.CLUSTERED 속성으로 클러스터링 키임. 
	- 클러스터링 키 : **파티션 내부**에서 행의 순서를 결정하기 위해 사용

- ingredients 속성 : List 대신 IngredientUDT 객체를 저장하는 List로 정의됨. -> 카산드라 테이블은 비정규화 되어 다른 테이블과 중복된 데이터를 포함할 수 있어 tacos 테이블의 ingredients 열에 중복 저장 가능.
	- IngredientUDT 클래스 사용 이유 : ingredients 열처럼 데이터의 컬렉션을 포함하는 열은 네이티브 타입(정수, 문자열 등)의 컬렉션이나 사용자 정의 타입 UDT의 컬렉션이어야 하기 때문.
    - UDT : 
    	- 다채로운 테이블 열 선언 가능. 
        - 비정규화된 관계형 데이터베이스 외부 키처럼 사용됨. 단, 다른 테이블의 한 행에 대한 참조만 갖는 외부 키와 대조적으로 UDT의 열은 다른 테이블의 한 행으로부터 복사될 수 있는 데이터를 실제로 가짐. -> tacos 테이블의 ingredients 열은 식재료 자체를 정의하는 클래스 인스턴스의 컬렉션 포함

#### IngredientUDT

```
package tacos;

import org.springframework.data.cassandra.core.mapping.UserDefinedType;

import lombok.AccessLevel;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.RequiredArgsConstructor;

@Data
@RequiredArgsConstructor
@NoArgsConstructor(access = AccessLevel.PRIVATE, force = true)
@UserDefinedType("ingredient")
public class IngredientUDT {
  private final String name;
  private final Ingredient.Type type;
}
```

- @Table 애노테이션이 이미 Ingredient 클래스르 카산드라에 저장하는 엔터티(도메인 타입)로 매핑했기에 UDT 사용 불가
- 카산드라의 UDT인 것을 알 수 있도록 @UserDefinedType 지정
- id 속성 미포함. 왜냐하면 소스 클래스인 Ingredient의 id 속성을 가질 필요가 없음.

**그림 12.1 참고**

- 외부 키와 조인을 사용하는 대신 카산드라 테이블 비정규화
- 관련된 테이블로부터 복사된 데이터를 포함하는 사용자 정의 타입을 가짐
- Order, User, Taco, Ingredient는 엔터티(도메인 타입)

**tacos 테이블 쿼리**
- ingredient 열은 IngredientUDT 컬렉션을 포함하고 있어 JSON 객체로 채워진 JSON 배열이 됨

#### 12.2 Order 클래스를 카산드라 tacoorders 테이블로 매핑

```
package tacos;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.UUID;

import org.springframework.data.cassandra.core.mapping.Column;
import org.springframework.data.cassandra.core.mapping.PrimaryKey;
import org.springframework.data.cassandra.core.mapping.Table;

import com.datastax.driver.core.utils.UUIDs;

import lombok.Data;

@Data
@Table("tacoorders")
public class Order implements Serializable {
  private static final long serialVersionUID = 1L;
  
  @PrimaryKey
  private UUID id = UUIDs.timeBased();
  private Date placedAt = new Date();
  
  @Column("user")
  private UserUDT user;

  private String deliveryName;

  private String deliveryStreet;

  private String deliveryCity;

  private String deliveryState;

  private String deliveryZip;

  private String ccNumber;

  private String ccExpiration;

  private String ccCVV;


  @Column("tacos")
  private List<TacoUDT> tacos = new ArrayList<>();
  
  public void addDesign(TacoUDT design) {
    this.tacos.add(design);
}

}
```
- 사용자 정의 타입을 저장하는 컬렉션을 포함함.

### 12.2.4 리액티브 카산드라 리퍼지터리 작성하기

- ReactiveCassandraRepository : ReactiveCrudRepository를 확장해 새 객체가 저장될 때 사용되는 insert() 메서드의 몇 가지 변형 버전 제공, 이외에는 ReactiveCrudRepository아 동일한 메서드 제공. 많은 데이터를 추가할 때 선택.


## 12.3 리액티브 몽고DB 리퍼지터리 작성하기

### 몽고DB
- NoSql 중 하나로 문서형 DB
- BSON(Binary JSON) 형식의 문서로 데이터를 저장
- 다른 DB에서 데이터를 쿼리하는 것과 거의 유사한 방법으로 문서를 쿼리하거나 검색 가능
- 스프링 데이터로 사용하는 방법은 JPA를 스프링 데이터로 사용하는 방법과 크게 다르지 않음
즉, 도메인 타입을 문서 구조로 매핑하는 애노테이션을 도메인 클래스에 지정
- JPA와 동일한 프로그래밍 모델을 따르는 Repository Interface를 작성하면 됨

#### 12.3.1 스프링 데이터 몽고DB 활성화하기

- 리액티브 스프링 데이터 몽고DB 스타터 의존성을 추가하기

```
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>

```
- 스프링 데이터 리액티브 몽고DB 지원을 활성화하는 자동-구성이 수행 (Repository Interface 자동 구현)
- 기본적으로 27017 포트를 리스닝

그러나 테스트와 개발에 편리하도록 **in-memory 내장 몽고DB**를 사용 가능
아래와 같이 Flapdoodle 의존성을 빌드에 추가

```
<dependency>
   <groupId>de.flapdoodle.embed</groupId>
   <artifactId>de.flapdoodle.embed.mongo</artifactId>
</dependency>
```

#### 12.3.2 도메인 타입을 문서로 매핑하기
- 스프링 데이터 몽고DB는 몽고DB에 저장되는 문서 구조로 도메인 타입을 매핑하는 데 유용한 애노테이션들을 제공

그 중 아래 3개가 가장 많이 사용된다.

@Id : 지정된 속성을 문서 ID로 지정한다. Serializable 타입인 어떤 속성에도 지정 가능

@Document : 지정된 도메인 타입을 몽고DB에 저장되는 문서로 선언

@Field : 몽고DB의 문서에 속성을 저장하기 위해 필드 이름(과 선택적으로 순서)을 지정

@Field가 지정되지 않은 도메인 타입의 속성들은 필드 이름과 속성 이름을 같은 것으로 간주

이 애노테이션들을 이용하여 Ingredient 클래스를 작성

```
package tacos;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import lombok.AccessLevel;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.RequiredArgsConstructor;

@Data
@RequiredArgsConstructor
@NoArgsConstructor(access=AccessLevel.PRIVATE, force=true)
@Document
public class Ingredient {
 
  @Id
  private final String id;
  private final String name;
  private final Type type;
 
  public static enum Type {
    WRAP, PROTEIN, VEGGIES, CHEESE, SAUCE
  }

}

```
#### Taco의 몽고DB 매핑

```
package tacos;

import java.util.Date;
import java.util.List;

import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.rest.core.annotation.RestResource;

import lombok.Data;

@Data
@RestResource(rel = "tacos", path = "tacos")
@Document
public class Taco {

  @Id
  private String id;
 
  @NotNull
  @Size(min = 5, message = "Name must be at least 5 characters long")
  private String name;
 
  private Date createdAt = new Date();
 
  @Size(min=1, message="You must choose at least 1 ingredient")
  private List<Ingredient> ingredients;

}
```

- ID로 String 타입의 속성을 사용하면 이 속성값이 DB에 저장될 때 몽고DB가 자동으로 ID 값을 지정해 줌. (null일 경우에 한함)


몽고DB 애노테이션이 지정된 다음의 Order 클래스
```

package tacos;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import lombok.Data;
import org.springframework.data.mongodb.core.mapping.Field;

@Data
@Document
public class Order implements Serializable {
  private static final long serialVersionUID = 1L;

  @Id
  private String id;
  private Date placedAt = new Date();

  @Field("customer")
  private User user;

  private String deliveryName;

  private String deliveryStreet;

  private String deliveryCity;

  private String deliveryState;

  private String deliveryZip;

  private String ccNumber;

  private String ccExpiration;

  private String ccCVV;


  private List<Taco> tacos = new ArrayList<>();

  public void addDesign(Taco design) {
    this.tacos.add(design);
  }

}
```

- customer 열을 문서에 저장한다는 것을 나타내기 위해서 user 속성에 @Field를 지정

User 도메인 클래스
```

package tacos;
import java.util.Arrays;
import java.util.Collection;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.
                                          SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import lombok.AccessLevel;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.RequiredArgsConstructor;

@Data
@NoArgsConstructor(access=AccessLevel.PRIVATE, force=true)
@RequiredArgsConstructor
@Document
public class User implements UserDetails {

  private static final long serialVersionUID = 1L;

  @Id
  private String id;
 
  private final String username;
 
  private final String password;
  private final String fullname;
  private final String street;
  private final String city;
  private final String state;
  private final String zip;
  private final String phoneNumber;
  private final String email;
 
  @Override
  public Collection<? extends GrantedAuthority> getAuthorities() {
    return Arrays.asList(new SimpleGrantedAuthority("ROLE_USER"));
  }

  @Override
  public boolean isAccountNonExpired() {
    return true;
  }

  @Override
  public boolean isAccountNonLocked() {
    return true;
  }

  @Override
  public boolean isCredentialsNonExpired() {
    return true;
  }

  @Override
  public boolean isEnabled() {
    return true;
  }

}

```
#### 12.3.3 리액티브 몽고DB 리퍼지터리 인터페이스 작성하기

- 스프링 데이터 몽고DB는 스프링 데이터 JPA가 제공하는 것과 유사한 자동 Repository 지원을 제공
- 몽고DB의 Reactvie Repository를 작성할 때는 ReactiveCrudRepository나 ReactiveMongoRepository를 선택 가능
- ReactiveCrudRepository는 새로운 문서나 기존 문서의 save() 메서드에 의존
- ReactiveMongoRepository는 새로운 문서의 저장에 최적화된 소수의 특별한 insert() 메서드를 제공

1. Ingredient 객체를 문서로 저장하는 Repository를 정의
- 식재료를 저장한 문서는 초기에 식재료 데이터를 DB에 추가할 때 생성, 이외에는 거의 추가 X
새로운 문서의 저장에 최적화된 ReactiveMongoRepository 보다는 ReactiveCrudRepository를 확장

```
package tacos.data;

import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import org.springframework.web.bind.annotation.CrossOrigin;

import tacos.Ingredient;

@CrossOrigin(origins="*")
public interface IngredientRepository extends ReactiveCrudRepository<Ingredient, String> {

}
```

- IngredientRepository는 ReactiveRepository이므로 이것의 메서드는 그냥 도메인 타입이나 컬렉션이 아닌 Flux나 Mono 타입으로 도메인 객체를 처리
- 예를 들어, findAll() 메서드는 Iterable<Ingredient> 대신 Flux<Ingredient>를 반환. 그리고 findById() 메서드는 Optional<Ingredient> 대신 Mono<Ingredient>를 반환.
따라서 이 Reactive Repository는 엔드-to-엔드 Reactive flow의 일부가 될 수 있음

2. 몽고DB의 문서로 Taco 객체를 저장하는 Repository를 정의

- Taco 객체를 저장하는 Repository를 정의하자. 색재료 문서와는 다르게 타코 문서는 자주 생성되어 ReactiveMongoRepository의 최적화된 insert() 메서드를 사용

```
package tacos.data;

import org.springframework.data.mongodb.repository.ReactiveMongoRepository;
import reactor.core.publisher.Flux;
import tacos.Taco;


public interface TacoRepository extends ReactiveMongoRepository<Taco, String> {
    Flux<Taco> findByOrderByCreatedAtDesc();
}

```
- ReactiveCrudRepository에 비해 ReactiveMongoRepository를 사용할 때의 유일한 단점은 **바로 몽고DB에 특화되어 있다는 점**
그래서 다른 DB에는 사용할 수 없으므로 이 단점을 감안하고 사용 필요

- TacoRepository의 새로운 메서드
- 이 메서드는 최근 생성된 타코들의 리스트를 조회하여 반환
- findByOrderByCreatedAtDesc()는 Flux<Taco>를 반환
 take() 오퍼레이션을 적용하여 Flux에서 발행되는 처음 12개의 Taco 객체만 반환 가능

- 예를 들어, 최근 생성된 타코들을 보여주는 컨트롤러에서는 다음과 같이 코드를 작성 가능

```
Flux<Taco> recents = repo.findByOrderByCreatedAtDesc().take(12);

- OrderRepository 인터페이스

package tacos.data;

import org.springframework.data.mongodb.repository.ReactiveMongoRepository;
import tacos.Order;

public interface OrderRepository extends ReactiveMongoRepository<Order, String> {

}

```
- Order 문서는 자주 생성될 것이다. 따라서 ReactiveMongoRepository를 확장

```
package tacos.data;

import org.springframework.data.mongodb.repository.ReactiveMongoRepository;
import reactor.core.publisher.Mono;
import tacos.User;

public interface UserRepository extends ReactiveMongoRepository<User, String> {

  Mono<User> findByUsername(String username);

}
 
```

## 요약

- 스프링 데이터는 카산드라, 몽고DB, 카우치베이스, 레디스 데이터베이스의 리액티브 리퍼지터리를 지원

- 스프링 데이터의 리액티브 리퍼지터리는 리액티브가 아닌 리퍼지터리와 동일한 프로그래밍 모델을 따름
단, Flux나 Mono와 같은 리액티브 타입을 사용

- JPA 리퍼지터리와 같은 리액티브가 아닌 리퍼지터리는 Mono나 Flux를 사용하도록 조정 가능
그러나 데이터를 가져오거나 저장할 때 여전히 블로킹이 생김!

- 관계형이 아닌 데이터베이스를 사용하려면 (NoSQL) 해당 데이터베이스에서 데이터를 저장하는 방법에 맞게 데이터를 모델링하는 방법을 알아야 함
