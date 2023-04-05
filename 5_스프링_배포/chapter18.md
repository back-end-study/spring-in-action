## 학습 목표
- 액추에이터 엔드포인트 MBeans 사용하기
- 스프링 빈을 MBeans로 노출하기
- 알림 발행(전송)하기

# JMX

**JMX**는 자바 애플리케이션을 모니터링하고 관리하는 표준 방법
	- **MBeans** : 컴포넌트를 노출해 외부의 JMX 클라이언트가 오퍼레이션 호출, 속성 검사, MBeans의 이벤트 모니터링을 통해 애플리케이션을 관리

- **JMX**는 스프링 부트 앱에 기본적으로 활성화됨. 이에 따라 모든 액추에이터 엔드포인트가 MBeans로 노출
- 스프링 애플리케이션 컨텍스트의 다른 빈도 노출 가능

이번 장에서는 액추에이터 엔드포인트가 어떻게 MBeans로 노출되는지 살펴보면서 스프링과 JMX를 알아본다.

## 18.1 액추에이터 MBeans 사용하기

- 16장의 표 16.1을 보면 /heapdump를 제외한 모든 액추에이터 엔드포인트가 MBeans로 노출되어 있음.
	- 어떤 JMX 클라이언트(예. JConsole)를 사용해도 현재 실행 중인 스프링 부트 애플리케이션의 앷에이터 엔드포인트 MBeans와 연결 가능.

- JConsole을 사용하면 그림 18.1과 같이 org.springframewor.boot 도메인 아래에 나타난 액추에이터 엔드포인트 MBeans들을 볼 수 있음.
	- **JConsole** : JDK에 포함되어 있으며, JDK가 설치된 후 홈 디렉토리의 /bin 서브 디렉터리에 있는 jconsole을 실행하면 된다.
    
![](https://velog.velcdn.com/images/jiyeong/post/24e0098c-8fe3-4c1d-85d1-5d0d5130235b/image.png)


- 액추에이터 엔드포인트 MBeans : HTTP의 경우처럼 명시적으로 포함시킬 필요 없이 기본으로 노출

#### 엔드포인트만 엑추에이터 엔드포인트 MBeans로 노출할 때
```
management:
	endpoints:
    	jmx:
        	exposure:
            	include: health, info, bean, conditions
```

- exclude는 include와 반대로 적으면 됨.

- JConsole에서 액추에이터 MBeans 중 하나의 관리용 오퍼레이션을 호출할 때
	- 왼쪽 패널 트리의 해당 엔드포인트 MBeans를 확장한 후 Operations 아래의 원하는 오페레이션을 선택하면 됨.

#### tacos.ingredients 패키지의 로깅 레벨 지정
![](https://velog.velcdn.com/images/jiyeong/post/52e0dc37-c2b1-45bc-b289-a070b584664d/image.png)

- Loggers MBean을 확장하고 loggerLevels 오퍼레이션 클릭
- 우측 상단의 Name 필드에 패키지 이름을 입력 후 loggerLevels 클릭

![](https://velog.velcdn.com/images/jiyeong/post/3fc55e38-5d9b-4eba-b78f-56b5ff44466c/image.png)

- 버튼을 누르면 /loggers 엔드포인트 MBean 응답을 보여주는 대화상자가 나옴

## 18.2 우리의 MBeans 생성하기

- 빈 클래스에 @ManagedResource 애노테이션 지정
- 메서드에는 @ManagedOperation
- 속성에는 @ManagedAttribute 지정

#### 18.1 생성된 타코의 수량을 세는 MBean

```
package tacos.tacos;

import java.util.concurrent.atomic.AtomicLong;

import javax.management.Notification;

import org.springframework.data.rest.core.event.AbstractRepositoryEventListener;
import org.springframework.jmx.export.annotation.ManagedAttribute;
import org.springframework.jmx.export.annotation.ManagedOperation;
import org.springframework.jmx.export.annotation.ManagedResource;
import org.springframework.jmx.export.notification.NotificationPublisher;
import org.springframework.jmx.export.notification.NotificationPublisherAware;
import org.springframework.stereotype.Service;

@Service
@ManagedResource
public class TacoCounter 
       extends AbstractRepositoryEventListener<Taco>
       implements NotificationPublisherAware {

  private AtomicLong counter;
  private NotificationPublisher np;

  public TacoCounter(TacoRepository tacoRepo) {
    long initialCount = tacoRepo.count();
    this.counter = new AtomicLong(initialCount);
  }
  
  @Override
  public void setNotificationPublisher(NotificationPublisher np) {
    this.np = np;
  }

  @Override
  protected void onAfterCreate(Taco entity) {
    counter.incrementAndGet();
  }

  @ManagedAttribute
  public long getTacoCount() {
    return counter.get();
  }

  @ManagedOperation
  public long increment(long delta) {
   return counter.addAndGet(delta);
  }

}
```

- TacoCounter 클래스에는 @Service 애노테이션이 지정되어 있어 컴포넌트를 찾아줌
- 이 클래스 인스턴스는 스프링 애플리케이션 컨텍스트의 빈으로 등록됨
- @ManagedResource가 이 빈이 MBeans가 된다고 알려줌
- getTacoCount() 메서드는 @ManagedAttribute가 지정되었으므로 MBeans 속성으로 노출
- increment() 메서드는 @ManagedOperation이 지정되어 MBeans 오퍼레이션으로 노출됨

#### TacoCounter MBeans가 어떻게 나타나는 지 보여주는 부분

![](https://velog.velcdn.com/images/jiyeong/post/27d76731-ce16-447b-8451-58a4d358a1ed/image.png)

- Abstract Repository EventListener의 서브클래스이므로 Taco 객체가 TacoRepositroy를 통해 저장될 때 퍼시스턴스 관련 이벤트를 받을 수 있음 -> 새로운 Taco 객체가 생성되어 리퍼지터리에 저장될 때마다 onAftewrCreate() 메서드가 호출되어 카운터를 1씩 증가시킴. 

- MBeans 오퍼레이션과 속성은 풀 방식을 사용함 -> MBeans 속성의 값이 변경되더라도 자동으로 알려주지 않아 JMX 클라이언트를 통해 봐야만 알 수 있음
- MBeans는 JMX 크랄이언트에 알림을 푸시할 수 있는 방법이 있음

## 18.3 알림 전송하기

- 스프링의 NotificiationPublisher를 사용하면 MBeans가 JMX 클라이언트에 알림을 푸시 가능
	- NotificiationPublisher는 하나의 sendNotification() 메서드를 가짐
    - 이 메서드는 Notification 객체를 인자로 받아 MBeans을 구독하는 JMX 클라이언트에게 발행(전송)
    
- MBeans가 알림을 발행하려면 NotificiationPublisherAware 인터페이스의 setNotificationPublisher() 메서드 구현 필요

#### 18.2 100개의 타코가 생성될 때마다 알림 전송하기

```
package tacos.tacos;

import java.util.concurrent.atomic.AtomicLong;

import javax.management.Notification;

import org.springframework.data.rest.core.event.AbstractRepositoryEventListener;
import org.springframework.jmx.export.annotation.ManagedAttribute;
import org.springframework.jmx.export.annotation.ManagedOperation;
import org.springframework.jmx.export.annotation.ManagedResource;
import org.springframework.jmx.export.notification.NotificationPublisher;
import org.springframework.jmx.export.notification.NotificationPublisherAware;
import org.springframework.stereotype.Service;

@Service
@ManagedResource
public class TacoCounter 
       extends AbstractRepositoryEventListener<Taco>
       implements NotificationPublisherAware {

  private AtomicLong counter;
  private NotificationPublisher np;

  public TacoCounter(TacoRepository tacoRepo) {
    long initialCount = tacoRepo.count();
    this.counter = new AtomicLong(initialCount);
  }
  
  @Override
  public void setNotificationPublisher(NotificationPublisher np) {
    this.np = np;
  }

  @Override
  protected void onAfterCreate(Taco entity) {
    counter.incrementAndGet();
  }

  @ManagedAttribute
  public long getTacoCount() {
    return counter.get();
  }

  @ManagedOperation
  public long increment(long delta) {
    long before = counter.get();
    long after = counter.addAndGet(delta);
    if ((after / 100) > (before / 100)) {
      Notification notification = new Notification(
          "taco.count", this,
          before, after + "th taco created!");
      np.sendNotification(notification);
    }
    return after;
  }

}
```

- JMX 클라이언트에서 알림을 받으려면 TacoCounter MBeans를 구독해야 함. 

## 18.4 TacoCounter MBeans 빌드 및 사용하기

1. STS가 실행 중이라면 종료하기
- 각자 STS 작업 영역 디렉터리에 생성한 .metadata 서브 디렉터리 삭제

# 요약

- 대부분의 액추에이터 앤드포인트는 JMX 클라이언트로 모니터링할 수 있는 Mbean로 사용 가능
- 스프링은 스프링 애플리케이션 컨텍스트의 빈을 모니터링하기 위해 자동으로 JMX를 활성화
- 스프링 빈에 @ManagedResource 애노테이션을 지정하면 Mbean로 노출 가능
	- 그리고 해당 빈의 메서드에 @ManagedOperation을 지정하면 관리 오퍼레이션으로 노출 가능
	- 속성에 @ManagedAttribute를 지정하면 관리 속성으로 노출 가능
- 스프링 빈은 NotificationPublisher를 사용하여 JMX 클라이언트에게 알림을 전송 가능

    
[그림 출처](https://livebook.manning.com/book/spring-in-action-fifth-edition/chapter-18/1)
