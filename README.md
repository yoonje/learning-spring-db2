# Spring Framework 정리 자료
스프링 DB 2편 - 데이터 접근 활용 기술

Table of contents
=================
<!--ts-->
   * [스프링 트랜잭션 이해](#스프링-트랜잭션-이해)
   * [스프링 트랜잭션 전파](#스프링-트랜잭션-전파)
<!--te-->

스프링 트랜잭션 이해
=======
- `스프링 트랜잭션 추상화`
  - 각각의 데이터 접근 기술들은 트랜잭션을 처리하는 방식에 차이가 있어서 스프링은 추상화를 통해서 트랜잭션을 동일한 방식으로 사용할 수 있음
  - 스프링은 트랜잭션을 추상화해서 제공할 뿐만 아니라, 실무에서 주로 사용하는 데이터 접근 기술에 대한 트랜잭션 매니저의 구현체도 제공하여 빈으로 등록한 이후 주입 받아서 사용
  - JdbcTemplate, MyBatis 를 사용하면 구현체인 DataSourceTransactionManager(JdbcTransactionManager) 를 스프링 빈으로 등록하고, JPA를 사용하면 구현체인  JpaTransactionManager 를 스프링 빈으로 등록해줌
  - @Transactional 애노테이션 하나만 선언해서 매우 편리하게 트랜잭션을 적용하는 선언적 트랜잭션 관리
- `스프링 트랜잭션 AOP`
  - AOP 도입 후 전체 과정
  
  <img width="819" alt="스크린샷 2022-08-03 오후 3 50 12" src="https://user-images.githubusercontent.com/38535571/182542996-1cdcf30e-77f5-4ec5-a14b-f617bb59c97c.png">
  
  - 개발자는 트랜잭션 처리가 필요한 곳에 @Transactional 애노테이션만 붙여주면 되며 스프링의 트랜잭션 AOP는 이 애노테이션을 인식해서 트랜잭션을 처리하는 프록시를 적용
- `스프링 트랜잭션 적용 확인`
  - `application.properties 확인`: logging.level.org.springframework.transaction.interceptor=TRACE 설정하면 트랜잭션 시작과 종료 로그 명확히 확인 가능
  - `TransactionSynchronizationManager.isActualTransactionActive()`: 현재 쓰레드에 트랜잭션이 적용되어 있는지 확인할 수 있는 기능으로 결과가 true 면 트랜잭션이 적용되어 있는 것
  - 트랜잭션 적용 위치는 스프링에서 우선순위는 항상 더 구체적이고 자세한 것이 높은 우선순위를 가짐 -> `메서드 > 클래스 > 인터페이스`
- `트랜잭션 AOP 주의 사항 - 내부 호출`
  - 트랜잭션을 적용하려면 항상 프록시를 통해서 대상 객체(Target)을 호출해야하는데 프록시를 거치지 않고 대상 객체를 직접 호출하게 되면 AOP가 적용되지 않아서 트랜잭션도 적용되지 않는데, 대상 객체의 내부에서 트랜잭션 어노테이션이 있있는  메서드 호출이 발생하면 프록시를 거치지 않기 때문에 대상 객체를 `직접 호출해서` 트랜잭션 처리가 되지 않는 문제가 발생
  - 자바 언어에서 메서드 앞에 별도의 참조가 없으면 this 라는 뜻으로 자기 자신의 인스턴스(실제 대상 객체)를 가리키므로 this에 의한 내부 호출은 프록시를 거치지 않음
  - @Transactional 를 사용하는 트랜잭션 AOP는 프록시를 사용하여 프록시를 사용하면 메서드 내부 호출에 프록시를 적용할 수 없음
  - 내부 호출을 피하기 위해 트랜잭션을 적용해야하는 메서드를 별도의 클래스로 분리해서 해결
- `트랜잭션 AOP 주의 사항 - 초기화 시점`
  - 초기화 코드가 먼저 호출되고 그 다음에 트랜잭션 AOP가 적용되기 때문에, @PostConstruct와 @Transactional을 함께 사용하면 트랜잭션이 적용되지 않음
- `트랜잭션 옵션 설정`
  - `적용 범위`
    - 스프링의 트랜잭션 AOP 기능은 클래스에 어노테이션을 걸어도 `public 메서드`에만 트랜잭션을 적용하도록 기본 설정이 되어 있음
  - `value, transactionManager`
    - 사용할 트랜잭션 매니저를 지정할 때는 value, transactionManager 둘 중 하나에 트랜잭션 매니저의 스프링 빈의 이름을 적어주면됨
    - 이 값을 생략하면 기본으로 등록된 트랜잭션 매니저를 사용하기 때문에 대부분 생략
  - `rollbackFor`
    - 기본적으로 언체크 예외인 RuntimeException, Error 와 그 하위 예외가 발생하면 롤백하고 체크 예외인 Exception 과 그 하위 예외들은 커밋하는데 rollbackFor를 통해서 어떤 예외가 발생할 때 추가 롤백을 지정할 수 있음
    - `@Transactional(rollbackFor = Exception.class)` 처럼 주로 체크 예외인 Exception 이 발생해도 커밋 대신 롤백 처리할 때 사용
    - 스프링 기본 설정에서 `체크 예외는 커밋`하고, `언체크(런타임) 예외는 롤백`하는 이유는 스프링 기본적으로 체크 예외는 비즈니스 의미가 있을 때 사용하고, 런타임(언체크) 예외는 복구 불가능한 예외로 가정하기 때문인데 굳이 이 정책을 따를 필요는 없음
  - `noRollbackFor`
    - 기본 정책에 추가로 어떤 예외가 발생했을 때 롤백하면 안되는지 지정
    - @Transactional(noRollbackFor={IgnoreRollbackException.class}) 처럼 주로 언체크 예외가 발생했을 때 롤백 처리를 하지 않으려할 때 사용
  - `propagation`
    - 트랜잭션이 이미 진행중인데, 여기에 추가로 트랜잭션을 수행될 때 어떻게 동작할지를 결정하는 것
    - 서비스 계층과 레포지토리 계층 둘 다 트랜잭션 어노테이션을 사용할 때 주로 적용되는 개념
    - @Transactional(propagation = Propagation.REQUIRES_NEW) 형식으로 옵션 설정 가능
    - `REQUIRED`: 많이 사용하는 기본 설정으로 기존 트랜잭션이 없으면 생성하고 있으면 참여
    - `REQUIRES_NEW`: 항상 새로운 트랜잭션을 생성
    - SUPPORT: 기존 트랜잭션이 없으면 없는대로 진행하고 있으면 참여
    - NOT_SUPPORT: 트랜잭션을 지원하지 않음
    - MANDATORY: 트랜잭션이 반드시 있어야하는 옵션으로 기존 트랜잭션이 있으면 참여하고 없으면 예외가 발생
    - NEVER: 트랜잭션을 사용하지 않는다는 의미로 기존 트랜잭션이 있으면 예외가 발생하고 없으면 없는대로 진행
    - NESTED: 트른잭션이 없으면 새로운 트랜잭션을 생성하고 있으면 중첩 트랜잭션을 만듦
  - `isolation`
    - 트랜잭션 `격리 수준`을 지정할 수 있는데, 기본 값은 데이터베이스에서 설정한 트랜잭션 격리 수준을 사용하는 `DEFAULT`로 대부분 데이터베이스에서 설정한 기준을 따름
    - DEFAULT: 데이터베이스에서 설정한 격리 수준을 따름
    - READ_UNCOMMITTED: 커밋되지 않은 읽기 
    - READ_COMMITTED: 커밋된 읽기 (대부분 이 격리 수준 사용)
    - REPEATABLE_READ: 반복 가능한 읽기
    - SERIALIZABLE: 직렬화 가능
  - `timeout`
    - 트랜잭션 수행 시간에 대한 `타임아웃`을 초 단위로 지정
    - 기본 값은 트랜잭션 시스템의 타임아웃을 사용하는데, 운영 환경에 따라 동작하는 경우도 있고 그렇지 않은 경우도 있기 때문에 꼭 확인하고 사용
  - `label`
    - 트랜잭션 애노테이션에 있는 값을 직접 읽어서 어떤 동작을 하고 싶을 때 사용
  - `readOnly`
    - 트랜잭션은 기본적으로 읽기 쓰기가 모두 가능한 트랜잭션이 생성하는데, `readOnly=true` 옵션을 사용하면 읽기 전용 트랜잭션이 생성되고 이 경우 등록, 수정, 삭제가 안되고 읽기 기능에만 트랜잭션이 작동
    - readOnly 옵션을 사용하면 읽기에서 다양한 성능 최적화가 발생
    - readOnly 옵션은 프레임워크, JDBC 드라이버, 데이터베이스 3곳에 적용되는데 각각의 종류에 따라 조금씩 다르게 동작하여 설정 시에 사용하는 것에 대해 확인해야함
  - `트랜잭션 전파에서 옵션 사용 시 주의 사항`
    - 트랜잭션 전파에서 옵션 isolation , timeout , readOnly 는 트랜잭션이 처음 시작될 때만 적용되고 트랜잭션에 참여하는 경우에는 적용되지 않음
    - 예를 들어서 REQUIRED 를 통한 트랜잭션 시작, REQUIRES_NEW 를 통한 트랜잭션 시작 시점에만 적용
    - 중첩 트랜잭션은 JPA에서는 사용할 수 없음
    - 내부 트랜잭션에서 예외가 발생하면 `rollbackOnly`를 설정

스프링 트랜잭션 전파
=======
- `트랜잭션 전파 기본(REQUIRED) 동작`
  - 트랜잭션이 수행 중이고 아직 끝나지 않았는데 그 도중에 내부 트랜잭션이 수행되면 외부 트랜잭션과 내부 트랜잭션을 묶어서 하나의 `물리 트랜잭션`을 만들어주고 내부 트랜잭션과 외부 트랜잭션이 각각 논리 트랜잭션이 되고 내부 트랜잭션이 외부 트랜잭션에 참여 (REQUIRED)
  - 트랜잭션이 참여했는지 안했는지의 여부는 `isNewTransaction`를 통해서 파악가능한데 `false`로 나와야 참여한 것
  - 물리 트랜잭션은 우리가 이해하는 실제 데이터베이스에 적용되는 트랜잭션을 뜻하는데 실제 커넥션을 통해서 트랜잭션을 시작하고 실제 커넥션을 통해서 커밋, 롤백하는 단위
  - `모든 논리 트랜잭션이 커밋되어야 물리 트랜잭션이 커밋되며, 하나의 논리 트랜잭션이라도 롤백되면 물리 트랜잭션은 롤백 처리`
  - 스프링은 이렇게 여러 트랜잭션이 함께 사용되는 경우, 처음 트랜잭션을 시작한 외부 트랜잭션이 실제 물리 트랜잭션을 관리하도록 해서 트랜잭션 중복 커밋 문제를 해결
  - 내부 트랜잭션에서 예외가 발생하면 rollbackOnly로 설정되고 rollbackOnly이 설정되어 있으면 외부 트랜잭션에서 커밋했을 때 `UnexpcetedRollbackException`이 발생 
- `트랙재션 전파 REQUIRES_NEW 동작`
  - 항상 새로운 트랜잭션을 생성하여 외부 트랜잭션과 내부 트랜잭션을 완전히 분리해서 각각 별도의 물리 트랜잭션을 사용하는 방법으로 커밋과 롤백이 각각 처리됨
  - REQUIRED는 하나의 트랜잭션이라도 롤백이 되면 전체가 롤백이 되어야하는데 그렇게 처리하고 싶지 않을 때 새로운 트랜잭션을 만드는 REQUIRES_NEW를 사용
  - `isNewTransaction`이 true로 완전히 새로운 트랜잭션 생성
  - REQUIRES_NEW 를 사용하면 `데이터베이스 커넥션을 동시에 2개 사용`
