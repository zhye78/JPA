# Chapter 8 - 프록시와 연관관계 관리

이번 장에서는 프록시, 즉시 로딩, 지연 로딩, 영속성 전이, 고아 객체에 대해 알아본다.


## 프록시

엔티티를 조회할 때 연관된 엔티티들이 항상 사용되는 것은 아니다.

JPA는 연관 엔티티를 즉시 로딩하는 것이 아니라 실제 사용될때 DB에서 조회하도록 `지연로딩`이라는 기능을 제공한다.

이 지연로딩을 사용하려면 실제 엔티티 객체 대신 데이터베이스 조회를 지연하도록 하는 가짜 객체가 필요한데, 이것을 `프록시 객체`라고 한다.

### 프록시 기초

엔티티를 실제 사용하는 시점까지 DB 조회를 미루고 싶은 경우 `EntityManager.find()` 가 아닌 `EntityManager.getReference()` 를 사용하면 된다.

이 메소드를 통해 반환되는 엔티티 객체는 `프록시 객체` 이다.

이 프록시 객체는 실제 객체에 대한 `참조를 보관`하고 있어 프록시 객체의 메소드를 호출하면 실제 객체의 메소드를 대신 호출하여 반환한다.

실제 엔티티가 사용될 떄, DB를 조회하여 실제 엔티티 객체를 생성하는 작업을 `프록시 객체의 초기화` 라고 한다.

</br>

- 프록시 클래스 예상 코드

```java
// 프록시 객체 조회 후 실제 데이터 조회
Member member = em.getReference(Member.class, "id1");
member.getName(); // 1. getName()
```

```java
public class MemberProxy extends Member {

    Member target = null;

    public String getName() {
        if (target == null) {
            // 2. 초기화 요청
            // 3. DB 조회
            // 4. 실제 엔티티 생성 및 참조 보관
            this.target = ...;
        }
        //5. target.getName()
        return this.target.getName();
    }
}
```

<img src = "./프록시.png"></br>


- 프록시의 특징

1. 프록시 객체는 처음 사용할때 `한 번만 초기화`
2. 프록시 객체를 초기화한다고 프록시 객체가 실제 엔티티로 바뀌는것은 아님
    - 실제 엔티티의 참조(`target`)를 가지는 프록시 객체가 되는 것
3. 프록시 객체는 원본 엔티티를 상속받은 구조이기에 타입 체크 시 주의
4. 영속성 컨텍스트에 찾는 엔티티가 있다면, `getReference` 메서드 호출하더라도 프록시가 아닌 원본 엔티티를 반환
5. 초기화는 영속성 컨텍스트의 도움을 받아야 가능
    - 때문에 준영속 상태의 프록시를 초기화하려면 예외 발생
        - `org.hibernate.LazyInitializationException` 예외 발생


### 프록시와 식별자

프록시 객체는 식별자(PK) 값을 보관하므로, 식별자 값을 조회하는 코드에서는 프록시 초기화가 되지 않는다.

```java
Team team = em.getReference(Team.class, "team1");
team.getId(); // 초기화 X
```

But, 엔티티 접근 방식을 AccessType.FIELD 로 설정하면 JPA는 위의 getId()가 무슨 역할을 하는지 모르므로 초기화가 진행된다.


### 프록시 확인

JPA가 제공하는 `PersistenceUnitUtil.isLoaded(Object entity)` 메소드를 사용하면 프록시 인스턴스의 초기화 여부를 확인할 수 있다.


## 즉시 로딩과 지연 로딩

프록시 객체는 주로 연관된 엔티티를 지연 로딩할 때 사용한다. 연관 엔티티의 DB 조회 시점에 따라 두 가지 로딩 방법이 있다.

- `즉시 로딩` : 엔티티를 조회할 때 연관된 엔티티도 함께 조회
    - 사용법 : @ManyToOne(fetch = FetchType.EAGER)
- `지연 로딩` : 연관된 엔티티는 실제 사용할 때 조회
    - 사용법 : @ManyToOne(fetch = FetchType.LAZY)


### 즉시 로딩

JPA 구현체는 대부분 많은 sql이 날라가는것을 방지하여 즉시 로딩을 최적화하기 위해 가능하면 조인쿼리를 사용한다.

```java
Member member = em.find(Member.class, "member1"); //1
Team team = member.getTeam(); //2
```

위 코드의 1번 라인을 실행하는 순간 Team도 함께 조회된다.


### 지연 로딩

위 코드의 1번 라인을 실행할 때 Member만 조회하고 Team은 조회하지 않으며, team 멤버 변수에 프록시 객체를 넣는다. 이후 2번 라인이 실행될 때 Team을 조회한다.


### 즉시 로딩, 지연 로딩 정리

항상 같이 조회하는 것은 즉시로딩, 필요시 조회하는 관계이면 지연로딩을 사용하자.


## 지연로딩 활용

사내 주문관리시스템을 예로 들어본다.

<img src = "./주문관리시스템.png"></br>

- Member는 Team 하나에만 속할 수 있음 (N:1)
- Member는 여러 주문 내역을 가짐 (1:N)
- Order는 하나의 Product를 가짐 (N:1)

- Member와 연관된 Team은 자주 함께 사용됨(즉시 로딩)
- Member와 연관된 Order는 가끔 사용됨(지연 로딩)
- Order와 Product는 자주 함께 사용됨(즉시 로딩)

```java
@Getter @Setter
@Entity
public class Member {

    @Id
    private String id;
    private String username;
    private int age;

    @ManyToOne(fetch = FetchType.EAGER)
    private Team team;

    @OneToMany(mappedBy = "member", fetch = FetchType.LAZY)
    private List<Order> orders;
}
```

- Team은 바로 조회
- Orders는 실제 데이터 조회 시 조회
    - Ex: `member.getOrders().get(0)`


### 프록시와 컬렉션 래퍼

JPA의 구현체인 하이버네이트는 엔티티를 영속 상태로 만들 때 엔티티에 컬렉션이 있으면, 이를 추적 및 관리하기 위해 하이버네이트 내장 컬렉션으로 변경하는데, 이를 `컬렉션 래퍼`라고 한다.


### JPA 기본 패치 전략

JPA의 기본 패치 속성값은 아래와 같습니다.

- @ManyToOne, @OneToOne: 즉시 로딩(FetchType.EAGER)
- @OneToMany, @ManyToMany: 지연 로딩(FetchType.LAZY)

책에서 추천하는 방법은 `모든 연관관계를 지연로딩`, `필요한 곳만 즉시로딩으로 최적화` 이다.


### 컬렉션에 FetchType.EAGER 사용 시 주의점

- 컬렉션을 하나 이상 즉시 로딩하는것은 권장하지 않음
    - 일대다 조인의 경우 결과 데이터가 무수히 많을 수 있기 때문
- 컬렉션 즉시 로딩은 항상 외부 조인을 사용

*FetchType.LAZY 일 경우: 조인 없이 쿼리를 따로 날리기 때문에 상관 없음


## 영속성 전이: CASCADE

특정 엔티티를 영속 상태로 만들 때 `연관된 엔티티도 함께 영속 상태로 만드는 것` 이다.

JPA에서는 이 영속성 전이를 `CASCADE` 옵션으로 제공한다.


### 영속성 전이: 저장

```java
@Entity
public class Parent {

    @Id @GeneratedValue
    @Column(name = "PARENT_ID")
    private long id;

    private String name;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.PERSIST)
    private List<Child> childList = new ArrayList<>();
}
```

```java
private static void saveWithCascade(EntityManager em) {
    Child child1 = new Child();
    Child child2 = new Child();

    Parent parent = new Parent();
    child1.setParent(parent);
    child2.setParent(parent);

    parent.getChildList().add(child1);
    parent.getChildList().add(child2);

    em.persist(parent); // 부모만 저장해도 자식까지 영속성 전이
}
```

영속성 전이는 연관관계 매핑과 아무 관련 없고, 단지 `엔티티 영속화시 연관된 엔티티도 같이 영속화하는 편리함을 제공`할 뿐이다.


### 영속성 전이: 삭제

영속성 전이는 엔티티 삭제시에도 `CascadeType.REMOVE`로 설정하여 사용할 수 있다. 


### CASCADE의 종류

- ALL : 모두 적용
- PERSIST : 영속
- MERGE : 병합
- REMOVE : 삭제
- REFRESH : REFRESH
- DETACH : DETACH

이 Cascade 속성은 여러 개 같이 사용 가능


## 고아 객체

JPA는 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제하는 기능을 제공하며, 이를 `고아 객체 제거`라고 한다.

```java
@Entity
public class Parent {

    @Id @GeneratedValue
    @Column(name = "PARENT_ID")
    private long id;

    private String name;

    @OneToMany(mappedBy = "parent", orphanRemoval = true)
    private List<Child> childList = new ArrayList<>();
}
```

```java
Parent parent1 = em.find(Parent.class, id);
parent1.getChildList().remove(0);
```

`orphanRemoval = true` 로 설정으로 인해 컬렉션에서 부모에서 자식 엔티티를 제거하면, 데이터베이스의 데이터도 삭제된다.

고아 객체 제거 기능은 영속성 컨텍스트를 `플러시할 때` 적용된다.

고아 객체 제거는 참조하는곳이 하나일 때만 사용해야 하며, 다른 곳에서도 참조하는 부분이 있다면 문제가 될 수 있다.

orphanRemoval은 @OneToOne, @OneToMany 에서만 사용 가능.

또한, 부모를 제거하면 자식도 제거 되는데 이는 `CascadeType.REMOVE` 를 적용한 것과 동일하다.


## 영속성 전이 + 고아 객체, 생명 주기

`CascadeType.ALL + orphanRemoval = true`를 동시에 사용하면

부모 엔티티에 자식엔티티만 추가 및 삭제하면 자동으로 영속화 및 삭제 기능이 동작한다.

```java
// 저장하려면, 부모에 등록만 하면 됨
Parent parent = em.find(Parent.class, parentId);
parent.addChild(child1);

// 삭제하려면, 부모에서만 제거하면 됨
Parent parent = em.find(Parent.class, parentId);
parent.getChildren().remove(removeObject);
```

