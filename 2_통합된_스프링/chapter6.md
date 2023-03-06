# Rest 서비스 생성하기

## 학습 목표

- 스프링 애플리케이션을 다른 애플리케이션과 통합하는 데 도움을 주는 주제를 다룸.
- 스프링 MVC에서 REST 엔드포인트 정의
- 하이퍼링크 REST 리소스 활성화
- 스프링 데이터 REST를 사용해서 리퍼지터리 기반의 REST 엔드포인트를 자동으로 생성

#### 오늘날의 웹 브라우저

- 클라이언트 측에서 자바스크립트 애플리케이션으로 많이 실행되며, 많은 애플리케이션이 클라이언트에 더 다가갈 수 있는 사용자 인터페이스 설계를 적용함.
- 모든 종류의 클라이언트가 백엔드 기능과 상호작용할 수 있게 서버는 클라이언트가 필요로 하는 API를 노출함.


#### 실습 내용

- 스프링을 사용해서 타코 클라우드 애플리케이션에 REST API를 제공.
	- 스프링 MVC 패턴을 사용해서 REST 엔드포인트를 생성하기 위해 스프링 MVC 사용
    - 스프링 데이터 리퍼지터리의 REST 엔드포인트도 외부에서 사용할 수 있게 자동으로 노출
    
#### 스프링 엔드포인트

찾아서 쓰기

## 6.1 REST 컨트롤러 작성

- 타코 클라우드 UI는 앵귤러 프레임워크를 사용해서 SPA(단일-페이지 애플리케이션)로 프론트엔드를 구축함.
- 타코 데이터를 저장하거나 가져오기 위해 앵귤러 기반의 UI와 통신하는 REST API를 생성해야 함.

> SPA 장/단점
- SPA의 장점
	- 프리젠테이션 계층이 백엔드 처리와 거의 독립적
    - 벡엔드 기능은 같은 것을 사용하면서도 사용자 인터페이스만 다르게 개발 가능
    - 이런 API를 사용할 수 있는 다른 애플리케이션과 통합할 수 있는 기회 제공
- SPA의 단점
	- 모든 애플리케이션이 이와 같은 유연성을 필요로 하는 것은 아님
    - 웹 페이지에 정보를 보여주는 것이 전부라면 MPA가 훨씬 간단함
    
- 앵귤러 클라이언트 코드는 HTTP 요청을 통해 이 장 전체에 걸쳐 생성할 REST API로 통신함.
- 2장에서는 @GetMapping과 @PostMapping 애노테이션을 사용해서 서버에서 데이터를 가져오거나 전송. REST API를 정의할 때도 그런 애노테이션들은 여전히 사용됨.
- 스프링 MVC는 다양한 타입의 HTTP 요청에 사용되는 다른 애노테이션들도 제공.

### 6.1.1 서버에서 데이터 가져오기

- 가장 최근에 생성된 타코를 보여주는 RecentTacosComponent를 앵귤러 코드에 정의
- 앵귤러 컴포넌트가 수행하는 /desing/recent의 GET 요청을 처리

#### 6.1 최근 타코들의 내역을 보여주는 앵귤러 컴포넌트 - RecentTacosComponent

```
import { Component, OnInit, Injectable } from '@angular/core';
import { Http } from '@angular/http';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'recent-tacos',
  templateUrl: 'recents.component.html',
  styleUrls: ['./recents.component.css']
})

@Injectable()
export class RecentTacosComponent implements OnInit {
  recentTacos: any;

  constructor(private httpClient: HttpClient) { }

  ngOnInit() {
  // 최근 생성된 타코들을 서버에서 가져온다.
    this.httpClient.get('http://localhost:8080/design/recent') // <1>
        .subscribe(data => this.recentTacos = data);
  }
}
```

#### ngOnInit()

- RecentTacosComponent는 주입된 Http 모듈을 사용해 http://localhost:8080/design/recent에 대한 HTTP 요청 수행.
- recentTacosa 모델 변수로 참조되는 타코들의 내역이 응답에 포함됨.
- recnets.component.html의 뷰에서는 브라우저에 나타나는 HTML로 모델 데이터를 보여줌.

#### 6.2 타코 디자인 API 요청을 처리하는 REST 사용 컨트롤러 - DesignTacoController

- 6.1의 GET 요청을 처리해 최근에 디자인된 타코들의 내역을 응답하는 엔드포인트 필요
- 새로운 컨트롤러를 생성해 요청 처리

```
package tacos.web.api;
// import 생략

@RestController
@RequestMapping(path="/design",                      // /design 경로의 요청 처리
                produces="application/json")
@CrossOrigin(origins="*")       		     // 서로 다른 도메인 간의 요청 허용
public class DesignTacoController {
  private TacoRepository tacoRepo;
  
  @Autowired
  EntityLinks entityLinks;

  public DesignTacoController(TacoRepository tacoRepo) {
    this.tacoRepo = tacoRepo;
  }

  @GetMapping("/recent")
  public Iterable<Taco> recentTacos() {               //최근 생성된 타코 디자인들을 가져와서 반환
    PageRequest page = PageRequest.of(
            0, 12, Sort.by("createdAt").descending());
    return tacoRepo.findAll(page).getContent();
  }

```

- TacoRepository 객체에 호출되는 findAll() 메서드는 PageRequest 객체를 인자로 받아 페이징(응답 결과를 여러 페이지로 구성) 수행.
- TacoRepository 인터페이스에서 CrudRepository 인터페이스를 확장하는 대신 PagingAndSorting Repository 인터페이스를 확장하면 이 findAll() 메서드 사용.

#### @RestController

- @Controller나 @Service와 같이 스테레오타입 애노테이션이므로 이 애노테이션이 지정된 클래스를 스프링의 컴포넌트 검색으로 찾을 수 있음

- 컨트롤러의 모든 HTTP 요청 처리 메서드에서 HTTP 응답 몸체에 직접 쓰는 값을 반환한다는 것을 스프링에게 알려줌(뷰로 보여줄 값을 반환하는 스프링의 일반적인 @Controller와 다름)

- 반환값이 뷰를 통해 HTML로 변환되지 않고 직접 HTTP 응답으로 브라우저에 전달됨

위와 달리 스프링 MVC 컨트롤러처럼 @Controller을 사용한다면
- 이 클래스의 모든 요청 처리 메서드에 @ResponseBody 지정해야하만 @RestController와 같은 결과를 얻을 수 있음

이외에도 ResponseEntity 객체를 반환하는 또 다른 방법이 있음.

#### @RequestMapping
- /design 경로의 요청 처리
- produces 속성(값 "application/json") 설정
- 요청의 Accept 헤더에 "application/json"이 포함된 요청만을 DesignTacoController의 메서드에 처리한다는 것을 나타냄
	- 이 경우 응답 결과는 JSON 형식이 되지만 produces 속성의 값은 String 배열로 저장되므로, 다른 컨트롤러에서도 요청을 처리할 수 있도록 JSON 만이 아닌 다른 콘텐트 타입을 같이 지정 가능
	- 예) xml로 출력하려면 produces 속성에 "text/xml"를 추가
```
@RequestMapping(path="/design",
                produces={"application/json", "text/xml"})
@CrossOrigin(origins="*")
```

@GetMapping
- recentTacos() /recent 경로의 GET 요청 처리

-> recentTacos() 메서드는 /design/recent 경로의 GET 요청 처리

#### @CrossOrigin

- 현재 6.1의 앵귤러 코드는 6.2 API와 별도의 도메인에서 실행중이므로 앵귤러 클라이언트에서 API를 사용하지 못하게 웹 브라우저가 막음

- 이런 제약은 서버 응답에 CORS 헤더를 포함시켜 극복할 수 있고, 스프링에서는 위 애노테이션을 지정해 적용 가능

- 다른 도메인(프로토콜과 호스트 및 포트로 구성)의 클라이언트에서 해당 REST API를 사용할 수 있게 해줌.

#### recentTacos()

- 최근 생성일자 순으로 정렬된 처음 12개의 결과를 갖는 첫 번째 페이지만 원한다는 것을 PageRequest 객체에 지정(가장 최근에 생성된 12개의 타코 디자인을 원한다는 의미)

- TacoRepository의 findAll() 메서드 인자로 PageRequest 객체가 전달되어 호출된 후 결과 페이지의 콘텐츠가 클라이언트에게 반환

- 타코ID로 특정 타코만 가져오는 엔트포인트를 제공하고 싶을 때는 메서드 경로에 플레이스홀더 변수를 지정하고 해당 변수를 통해 ID를 인자를 받는 메서드를 DesignTacoController에 추가한다. -> 해당 ID를 사용해서 리퍼지터리의 특정 객체를 찾을 수 있음.

```
@GetMapping("/{id}")
  public Taco tacoById(@PathVariable("id") Long id) {
    Optional<Taco> optTaco = tacoRepo.findById(id);
    if (optTaco.isPresent()) {
      return optTaco.get();
    }
    return null;
  }
```

- DesignTacoController 기본 경로가 /design이므로 tacoById()는 /design/{id} 경로의 GET 요청 처리.
- 경로의 {id} 부분이 플레이스 홀더
- @PathVariable에 의해 {id} 플레이스 홀더와 대응되는 id 매개변수에 해당 요청의 실제 값 지정
- tacoById() 내부에서는 Taco 객체를 가져오기 위해 id 매개변수 값이 타코 리퍼지터리의 findById() 메서드 인자로 전달됨
- 지정된 타코 ID가 없을 수 있으므로 findById()는 Optional<Taco> 반환
- 값 반환 이전에 해당 ID와 일치하는 타코가 있는지 확인한 후, Optional<Taco> 객체의 get()을 호출해 Taco 객체를 반환해야 함.
  
만약 ID가 일치하지 않으면 null을 반환하는데 이는 좋은 방법이 아니다.
null을 반환하면 콘텐츠가 없는데도 정상 처리를 나타내는 HTTP 200(OK) 상태 코드를 클라이언트가 받기 때문이다. 따라서 HTTP 404(NOT FOUND) 상태 코드를 응답으로 반환하는 게 더 좋다.
  
```
@GetMapping("/{id}")
  public ResponseEntity<Taco> tacoById(@PathVariable("id") Long id) {
    Optional<Taco> optTaco = tacoRepo.findById(id);
    if (optTaco.isPresent()) {
      return new ResponseEntity<>(optTaco.get(), HttpStatus.OK);
    }
    return new ResponseEntity<>(null, HttpStatus.NOT_FOUND);
  }

```

- Taco 객체 대신 ResponseEntity<Taco>가 반환됨.
  - 찾은 타코가 있는 경우 HTTP 200<OK> 상태 코드를 갖는 ResponseEntity에 Taco 객체가 포함됨.
  - 이와 반대의 경우 HTTP 404(NOT FOUND) 상태 코드를 갖는 ResponseEntity에 null이 포함되어 클라이언트에서 가져오려는 타코가 없다는 것을 나타냄
  
개발 시에 API 테스트를 위해서 curl이나 HTTPie(https://httpie.org/)를 사용해도 된다.
  
```
$ curl localhost:8080/design/recent
```
```
http :8080/design/recent
```

정보 반환만 하는 엔드포인트 API 정의.
다음 6.2에서는 요청의 입력 데이터를 처리하는 컨트롤러 메서드를 작성하는 방법을 배움.
  
### 6.1.2 서버에 데이터 전송하기 - POST
  
- 타코 클라우드를 SPA로 변환하기 위해 앵귤러 컴포넌트와 엔드포인트를 생성
- DesignComponent(**design.component.ts**)라는 이름의 새로운 앵귤러 컴포넌트를 정의해 타코 디자인 폼의 클라이언트 코드 처리
- onSubmit() 메서드는 타코 디자인 폼의 제출을 처리


```
onSubmit() {
  this.httpClient.post(
      'http://localhost:8080/design',
      this.model, {
          headers: new HttpHeaders().set('Content-type', 'application/json'),
      }).subscribe(taco => this.cart.addToCart(taco));

  this.router.navigate(['/cart']);
}
```

- HttpClient post() 메서드는 API로부터 데이터를 가져오는 대신 API로 데이터를 전송하는 것을 의미한다.
- 즉, model 변수에 저장된 데이터를 API 엔드포인트(/design 경로의 HTTP POST 요청애 대한)로 전송

따라서 DesignTacoController에 디자인 데이터를 요청하고 저장하는 메서드를 추가해야 한다.
  
```
@PostMapping(consumes="application/json")
@ResponseStatus(HttpStatus.CREATED)
public Taco postTaco(@RequestBody Taco taco) {
  return tacoRepo.save(taco);
}
```
- postTaco() 메서드는 DesignTacoController 클래스에 지정된 @RequestMapping /design 경로에 대한 요청 처리
- **consumes="application/json"**
Content-type이 application/json인 요청만 처리한다.

- **@RequestBody**
요청 몸체의 JSON 데이터가 Taco 객체로 변환되어 taco 매개변수와 바인딩 
이게 지정되지 않으면 매개변수가 곧바로 Taco 객체와 바인딩되는 것으로 스프링 MVC가 간주함
  
- **@ResponseStatus(HttpStatus.CREATED)**
해당 요청이 성공적이면서 요청의 결과로 리소스가 생성되면 HTTP 201(CREATED)가 전달
	- HTTP 200(OK)보다 더 정확한 상태 설명을 전달
  
### 6.1.3 서버의 데이터 변경하기 (PUT, PATCH)
- (Taco 객체를) 변경
- 데이터 변경을 위한 HTTP 메서드로 put, patch가 존재

**PUT**데이터 전체를 변경할 때 사용
**PATCH** 데이터의 일부만 변경할 때 사용

예를 들어, 특정 주문 데이터의 주소를 변경하고 싶다면 PUT을 사용한다면 해당 주문 데이터 전체를 PUT 요청으로 제출. 해당 주문의 속성이 생략되면 값이 null로 변경됨.
  
```
@PutMapping(path="/{orderId}")
  public Order putOrder(@RequestBody Order order) {
    return repo.save(order);
  }
```
**PATCH**

- 클라이언트에서 변경할 속성만 전송하면 됨
- 서버에서는 클라이언트에서 지정하지 않은 속성의 기존 데이터를 그대로 보존 가능
  
```
@PatchMapping(path="/{orderId}", consumes="application/json")
  public Order patchOrder(@PathVariable("orderId") Long orderId,
                          @RequestBody Order patch) {
    
    Order order = repo.findById(orderId).get();
    if (patch.getDeliveryName() != null) {
      order.setDeliveryName(patch.getDeliveryName());
    }
    if (patch.getDeliveryStreet() != null) {
      order.setDeliveryStreet(patch.getDeliveryStreet());
    }
    if (patch.getDeliveryCity() != null) {
      order.setDeliveryCity(patch.getDeliveryCity());
    }
    if (patch.getDeliveryState() != null) {
      order.setDeliveryState(patch.getDeliveryState());
    }
    if (patch.getDeliveryZip() != null) {
      order.setDeliveryZip(patch.getDeliveryState());
    }
    if (patch.getCcNumber() != null) {
      order.setCcNumber(patch.getCcNumber());
    }
    if (patch.getCcExpiration() != null) {
      order.setCcExpiration(patch.getCcExpiration());
    }
    if (patch.getCcCVV() != null) {
      order.setCcCVV(patch.getCcCVV());
    }
    return repo.save(order);
  }
```
  
**patchOrder() 메서드 제약 사항**

- 특정 필드의 데이터를 변경하지 않음을 나타내는 것을 null로 정한다면 해당 필드를 null로 변경하고 싶을 때 클라이언트에서 이를 표현할 방법이 필요

- 컬렉션에 저장된 항목을 삭제 혹은 추가할 방법 x.
따라서 클라이언트가 컬렉션의 항목을 삭제 혹은 추가하려면 변경될 컬렉션 테이터 전체를 전송
  
### 6.1.4 서버에서 데이터 삭제하기 - DELETE
  
-ㅜ데이터를 삭제할 때에는 DELETE 요청을 처리하는 메서드에 @DeleteMapping을 지정
  
**@ResponseStatus(HttpStatus.NO_CONTENT)**

```
@DeleteMapping("/{orderId}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void deleteOrder(@PathVariable("orderId") Long orderId) {
  try {
    repo.deleteById(orderId);
  } catch (EmptyResultDataAccessException e) {}
}
```
- deleteOrder()는 특정 주문 데이터를 삭제, 해당 주문이 존재하지 않으면 EmptyResultDataAccessException이 발생
- @ResponseStatus가 지정되어 있는데 응답의 상태코드가 204(NO CONTENT) 되게 함, 주문 데이터를 삭제하는 것이므로 클라이언트에게 데이터를 반환할 필요가 없기 때문임
  
## 6.2 하이퍼미디어 사용하기
  
- 과거 클라이언트가 API 요청을 위해 URL스킴 사용, API 클라이언트 코드에서 하드코딩된 URL 패턴을 사용해 문자열로 처리
- API URL을 하드코딩하고 문자열로 처리하면 클라이언트 코드가 불안정해짐
  
**HATEOAS**
- REST API를 구현하는 또 다른 방법인 HATEOAS(Hypermedia As The Engine Of Application)
- API로부터 반환되는 리소스에 해당 리소스와 관련된 하이퍼링크들이 포함
- 클라이언트가 최소한의 API URL만 알면 다른 API URL들을 알아내 사용 가능
  
예) 클라이언트가 최근 생성된 타코 리스트를 요청했을 때, 하이퍼링크가 없는 형태의 API 요청은 (JSON 형식으로) 타코 관련 정보만 클라이언트에서 수신될 것
  
**하이퍼링크가 없는 타코 리스트**
  
```
{
	{
		"id": 4,
		"name":"Veg-Out",
		"createdAt":"2018-01-31T20:15:53.219+0000",
		"ingredients": [
			{"id": "FLTO", "name": "Flour Totila", "type": "WRAP"},
			...
		]
	},
	...
}
```
  
**하이퍼링크를 포함한 타코 리스트**
  
```
{
 "_embedded": {
   "tacoResourceList": [{
     "name": "Veg-Out",
     "createdAt": "2018-01-31T20:15:53.219+0000",
     "ingredients": [{
       "name": "Flour Tortilla", "type": "WRAP",
       "_links": {
         "self": { "href": "http://localhost:8080/ingredients/FLTO" }
       }
     },
     {
       "name": "Corn Tortilla", "type": "WRAP",
       "_links": {
         "self": { "href": "http://localhost:8080/ingredients/COTO" }
       }
     }],
     "_links": {
       "self": { "href": "http://localhost:8080/design/4" }
     }
   },
   { 
 ...
 ]},
 "_links": {
   "recents": {
     "href": "http://localhost:8080/design/recent"
   }
 }
}
```
  
- 이런 형태의 HATEOAS를 HAL이라고 하며, JSON응답에 하이퍼링크를 포함시킬 떄 주로 사용
  
**"_link"**

- 클라이언트가 관련 API를 수행할 수 있는 하이퍼링크 포함(self, recent)
	- self는 타코와 해당 타코의 식자재 모두 그들 리소르를 참조
  	- recent는 리스트 전체는 자신을 참조
  
- 클라이언트 애플리케이션이 타코 리스트의 특정 타코에 대해 HTTP 요청을 수행해야 할 때 타코 리소스의 URL을 지정하지 않아도 됨, 대신에 원하는 self링크를 요청하면 됨.
  
- Spring HATEOAS는 스프링 MVC 컨트롤러에서 리소스를 반환하기 전에 해당 리소스에 링크를 추가하는 데 사용할 수 있는 클래스와 리소스 어셈블러 제공

#### 6.2.1 하이퍼링크 추가하기
  
- 스프링 HATEOAS는 하이퍼링크 리소스를 나타내는 두 개의 기본 타입인 Resource(단일 리소스)와 Resources(리소스 컬렉션) 제공.
- 두 타입이 전달하는 링크는 스프링 MVC 컨트롤러 메서드에 반환될 때 클라이언트가 받는 JSON(혹은 XML)에 포함
 
**ControllerLinkBuilder**
- 스프링 HATEOAS 링크 빌더 중 가장 유용한 빌더로 URL을 하드코딩하지 않고 호스트 이름을 알 수 있음
- 컨트롤러의 기본 URL에 관련된 링크의 빌드를 도와주는 편리한 API

#### 리소스 어셈블러 생성
  
- 리소스에 포함된 각 타코 리소스에 대한 링크 추가
 	- 반복 루프에서 Resources 객체가 가지는 각 Resource<Taco> 요소에 Link를 추가하기 -> 좀 번거로운 방법

- Taco 객체를 새로운 TacoResource 객체로 변환하는 유틸리티 클래스를 정의
  
```
package tacos.web.api;

import java.util.List;

import org.springframework.hateoas.Resources;

public class TacoResources extends Resources<TacoResource> {
  public TacoResources(List<TacoResource> tacoResources) {
    super(tacoResources);
  }
}
```
  
- TacoResource는 Taco의 id 속성을 갖지 않음. 왜냐하면 데이터베이스에서 필요한 ID를 API에 노출시킬 필요가 없기 때문. 그리고 API 클라이언트 관점에선 해당 리소스의 self 링크가 리소스 식별자 역할을 할 것이다.
  
- TacoResource는 Taco 객체를 인자로 받는 하나의 생성자를 가지며, Taco 객체의 속성 값을 자신의 속성에 복사함 -> Taco 객체를 TacoResource 객체로 쉽게 변환시킴
  - 이를 위한 반복 루프 필요 TacoResourceAssembler
  
```
package tacos.web.api;

import org.springframework.hateoas.mvc.ResourceAssemblerSupport;

import tacos.Taco;

public class TacoResourceAssembler
       extends ResourceAssemblerSupport<Taco, TacoResource> {

  public TacoResourceAssembler() {
    super(DesignTacoController.class, TacoResource.class);
  }
  
  @Override
  protected TacoResource instantiateResource(Taco taco) {
    return new TacoResource(taco);
  }

  @Override
  public TacoResource toResource(Taco taco) {
    return createResourceWithId(taco.getId(), taco);
  }

}
```
  
- TacoResource를 생성하면서 만들어지는 링크에 포함되는 URL이 기본 경로를 결정하기 위해 DesignTacoController를 사용

- toResource() : ResourceAssemblerSupport로부터 상속을 받을 때 반드시 오버라이드해야 하는 메서드로 Taco 객체로 TacoResource 인스턴스를 생성하면서 객체의 id 속성 값으로 생성되는 self 링크가 URL에 자동 지정

#### 6.2.3 embeded 관계 이름 짓기
  
- 만약 변경 전의 이름을 사용하는 클라이언트 코드가 있다면 @Relation 애노테이션을 사용해볼 것
@Relation : 자바로 정의된 리소스 타입 클래스 이름과 JSON 필드 이름 간의 결합도를 낮춰줌
  

## 6.3 데이터 기반 서비스 활성화하기
  
**스프링 데이터 REST**는 스프링 데이터의 또 다른 모듈, 스프링 데이터가 생성하는 리퍼지터리의 REST API 자동 생성. 

- 의존성 지정 후, 이미 스프링 데이터를 사용 중인 프로젝트에서 REST API를 노출 가능
- 스프링 데이터 REST 엔드포인트를 사용하고 싶으면 지금까지 생성했던 @RestController이 지정된 모든 클래스들을 이 시점에서 제거해야 함
- 스프링 데이터 REST가 생성한 엔드포인트들은 GET, POST, PUT, DELETE 메서드 지원
- API 기본 경로 지정 필요, 왜냐하면 해당 API의 엔드포인트가 우리가 작성한 모든 다른 컨트롤러와 충돌하지 않게 하기 위해서임.

#### 6.3.1 리소스 경로와 관계 이름 조정
- 스프링 데이터 리퍼지터리의 엔드포인트 생성 시 스프링 데이터 REST는 해당 엔드포인트의 관련된 엔터티 클래스 이름의 복수형 사용
- 스프링 데이터 REST는 노출된 모든 엔드포인트의 링크를 갖는 홈 리소스도 노출 시킴
- @RestResource 지정 시 관계 이름과 경로를 우리가 원하는 것으로 변경 가능.  

#### 6.3.2 페이징과 정렬
  
- 홈 리소스의 모든 링크는 선택적 매개변수인 page, size, sort 제공

  - 페이지 크기가 5인 첫 번째 페이지를 요청하는 경우
```
$curl "localhost:8080/api/tacos?size=5"
```
  - 5개 이상의 타코가 있고 page 매개변수를 추가하면 두 번째 페이지의 타코 요청 가능
```
$curl "localhost:8080/api/tacos?size=5&page=1"
```
 
- HATEOAS는 처음, 마지막, 다음, 이전 페이지의 링크를 요청 응답에 제공
- sort 매개변수를 지정 시 엔터티의 속성을 기준으로 결과 리스트 정렬 가능
  
	- 최근 생성된 12개의 타코를 가져오는 경우
```
$curl "localhost:8080/api/tacos?sort=createdAt,desc&size=12"
```
#### 6.3.3 커스텀 엔드포인트 추가
- 해당 엔드포인트 컨트롤러는 스프링 데이터 REST의 기본 경로로 매핑되지 x
- 해당 컨트롤러에 정의한 엔드포인트는 스프링 데이터 REST 엔드포인트에서 반환되는 리소스의 하이퍼링크에 자동으로 포함 x

- 스프링 데이터 REST는 @RepositoryRestController 포함하여 기본 경로롤 매핑되는 컨트롤러 클래스에 지정하는 새로운 애노테이션

```
package tacos.web.api;

import static org.springframework.hateoas.mvc.ControllerLinkBuilder.*;

import java.util.List;

import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.data.rest.webmvc.RepositoryRestController;
import org.springframework.hateoas.Resources;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;

import tacos.Taco;
import tacos.data.TacoRepository;

@RepositoryRestController
public class RecentTacosController {

  private TacoRepository tacoRepo;

  public RecentTacosController(TacoRepository tacoRepo) {
    this.tacoRepo = tacoRepo;
  }

  @GetMapping(path="/tacos/recent", produces="application/hal+json")
  public ResponseEntity<Resources<TacoResource>> recentTacos() {
    PageRequest page = PageRequest.of(
                          0, 12, Sort.by("createdAt").descending());
    List<Taco> tacos = tacoRepo.findAll(page).getContent();

    List<TacoResource> tacoResources = 
        new TacoResourceAssembler().toResources(tacos);
    Resources<TacoResource> recentResources = 
            new Resources<TacoResource>(tacoResources);
    
    recentResources.add(
        linkTo(methodOn(RecentTacosController.class).recentTacos())
            .withRel("recents"));
    return new ResponseEntity<>(recentResources, HttpStatus.OK);
  }

}
```
  
- @RepositoryRestController 애노테이션이 지정되어 있어 맨 앞에 스프링 데이터 REST의 기본 경로가 추가되어야 한다.
  - recentTacos() 메서드는 /api/tacos/recent의 GET 요청 처리

- @RepositoryRestController 애노테이션은 핸들러 메서드의 반환값을 요청 응답의 몸체에 자동으로 수록하지 않음. 
- 해당 메서드에 @ResponseBody 애노테이션을 지정하거나 해당 메서드에서 응답 데이터를 포함하는 ResponseEntity를 반환해야 함.

#### 6.3.4 커스텀 하이퍼링크를 스프링 데이터 엔드포인트에 추가
- 스프링 데이터 HATEOAS는 ResourceProcessor 제공. 이는 API를 리소스가 반환되기 전에 리소스를 조작하는 인터페이스.
  

  
  
### 6.4 앵귤러 IDE 이클립스 플러그인 설치와 프로젝트 빌드 및 실행
  
## 요약
  
- Rest 엔드포인트는 Spring MVC, 그리고 브라우저 지향의 컨트롤러와 동일한 프로그래밍 모델을 따르는 컨트롤러로 생성할 수 있다.
- 모델과 뷰를 거치지 않고 요청 응답 몸체에 직접 데이터를 쓰기 위해 컨트롤러의 핸들러 메서드에는 @ResponseBody 애노테이션을 지정할 수 있으며, ResponseEntity 객체를 반환할 수 있다.
- @RestController 애노테이션을 컨트롤러에 지정하면 해당 컨트롤러의 각 핸들러 메서드에 @ResponseBody를 지정하지 않아도 되므로 컨트롤러를 단순화해 준다.
- 스프링 HATEOAS는 스프링 MVC에서 반환되는 리소스의 하이퍼링크를 추가할 수 있게 한다.
- 스프링 데이터 리퍼지터리는 스프링 데이터 REST를 사용하는 REST API로 자동 노출 될 수 있다.
