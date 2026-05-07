


# 1장 개요 — 오브젝트와 의존관계

핵심 메시지:

“좋은 객체지향 설계는 변화에 유연해야 하며, 스프링은 이를 돕는 프레임워크다.”

책에서는 가장 단순한 DAO(Data Access Object) 예제를 점진적으로 개선하면서 스프링 철학을 설명합니다.

## 1.초난감 DAO

처음에는 매우 단순한 UserDao 클래스가 등장합니다.

예시 구조:

~~~JAVA
public class UserDao {
    public void add(User user) throws Exception {
        Connection c = DriverManager.getConnection(...);
        PreparedStatement ps = c.prepareStatement(...);
        ...
    }
}
~~~

* 문제점:

    * DB 연결 코드가 DAO 안에 직접 들어감
    * MySQL → Oracle 같은 DB 변경 시 코드 수정 필요
    * 관심사가 뒤섞여 있음
    * 재사용성과 테스트성이 낮음

즉, 객체가 너무 많은 일을 하고 있습니다.

## 2. 관심사의 분리

책의 핵심 전환점입니다.

UserDao가 해야 할 일은:

* SQL 실행
 * User 저장/조회

그런데 실제로는:

* DB 연결 생성
* 드라이버 로딩
* 리소스 관리

까지 모두 담당합니다.

그래서 저자는:

  *  “변하는 이유가 다른 코드는 분리해야 한다”

라고 설명합니다.

## 3. 메서드 분리

DB 연결 코드를 별도 메서드로 추출합니다.

~~~JAVA
public Connection getConnection() {
    ...
}
~~~

장점:

* 중복 제거
* 변경 지점 축소

하지만 아직 한계가 있습니다.

* 상속 없이는 DB 변경이 어려움
* 여전히 강한 결합 존재

## 4. 상속을 통한 확장

UserDao를 추상 클래스로 만들고,
DB 연결 생성은 하위 클래스가 구현하도록 바꿉니다.

~~~JAVA
public abstract class UserDao {
    protected abstract Connection getConnection();
}
~~~

하위 클래스:

~~~JAVA
public class NUserDao extends UserDao {
    protected Connection getConnection() {
        ...
    }
}
~~~
의미:

* 기능은 유지
* 구현은 교체 가능

여기서 템플릿 메서드 패턴과 팩토리 메서드 패턴 개념이 등장합니다.

## 5. 디자인 패턴의 적용

1장에서 자연스럽게 소개되는 패턴들:

* 템플릿 메서드 패턴
* 팩토리 메서드 패턴

하지만 저자는 중요한 한계를 지적합니다.

상속 기반 구조는:

* 컴파일 시점 결합이 강함
* 유연성이 부족함
* 다중 확장이 어려움

그래서 더 나은 방법이 필요합니다.

## 6. 객체 협력과 DI의 시작

이후 핵심 개념인 “관계 설정 책임 분리”가 등장합니다.

UserDao가 직접 DB 연결 객체를 만들지 않게 하고,
외부에서 주입받도록 설계합니다.

~~~ JAVA
public UserDao(ConnectionMaker connectionMaker) {
    this.connectionMaker = connectionMaker;
}
~~~

즉:

* UserDao는 “무엇을 할지”만 관심
* 실제 연결 생성은 다른 객체 담당

이것이 스프링 DI의 출발점입니다.

## 7. 클라이언트의 책임

객체 생성과 관계 설정은 별도 클래스가 담당합니다.

~~~JAVA
UserDao dao = new UserDao(new DConnectionMaker());
~~~

여기서:

* 오브젝트 생성 책임
* 오브젝트 사용 책임

을 분리합니다.

이 개념이 뒤에서 스프링 컨테이너와 IoC로 이어집니다.

## 8. 제어의 역전(IoC)

기존 방식:

* 객체가 필요한 객체를 직접 생성

IoC 적용 후:

* 외부가 객체 관계를 결정

즉:

* 객체는 자신의 제어권 일부를 외부에 넘긴다.

스프링 컨테이너가 바로 이 역할을 합니다.

## 9. 핵심 정리

1장의 가장 중요한 키워드:

|개념	|의미|
|------|---|
|관심사의 분리	|역할이 다른 코드를 나눈다|
|확장 가능성 |	변화에 유연한 구조|
|OCP|	확장에는 열려 있고 수정에는 닫힘|
|DI	|필요한 객체를 외부에서 주입 |
|IoC	|객체 생성/관계를 외부가 관리 |
|객체 협력	|객체는 자신의 역할만 책임 |

## 한 줄 요약

1장은 단순한 DAO 리팩토링을 통해:

* “왜 스프링이 필요한가?”
* “왜 DI와 IoC가 중요한가?”

를 객체지향 설계 관점에서 설명하는 장입니다.