# 1. 스프링 스터디 1주차

## 난감한 DAO

### 자바 빈

- 원래 의미 : 비주얼 툴에서 조작 가능한 컴포넌트를 의미
- 현재의 의미 : 두가지 관례를 따라 만들어진 Object를 의미
    - 디폴트 생성자(파라미터가 없는 생성자)를 가질 것 → 툴, 프레임워크에서 리플렉션을 이용해서 오브젝트를 생성하기 때문에 필요하다.
    - 프로퍼티 : 자바빈이 노출하는 이름을 가진 속성을 프로퍼티 → getter,setter을 이용하여 수정, 조회 가능

```java
//. 유저 정보 저장용 user class -> 자바빈 규약을 따르는 object

public class User{

	String id;
	String name;
	String password;

	public String getId(){
		return id
	}

	public String setId(){
		this.name = name;
	}
	
	public String getPassword(){
		return password;
	}
	
	public void setPassword(String password){
		this.password = password;
	 
	}

}
```

### DAO

- DAO(data access object)란  DB를 사용해 데이터를 조회하거나 조작하는 기능을 전담하도록 만든 오브젝트를 말한다. →즉 DB에 접근해서 그 접근한 정보를 데이터를 관리 / 조작을 위한 클래스
- 이 조작 하는 DAO 클래스(UserDAO class)를 이용해서 데이터를 자바빈 object(User class)에 넣고 관리

```java
// UserDAO -> 사용자 정보를 DB에 넣고 관리할 수 있는 DAO Class
package springbook.user.dao;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

import springbook.user.domain.User;

public class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		
		// 1. DB연결을 위한 connection을 가져온후
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
		
		// 2. SQL을 담은 Statement를 만들고
		PreparedStatement ps = c.prepareStatement(
			"insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());

		//3. 만들어진 statement를 실행한다.
		ps.executeUpdate();
		
		ps.close();
		c.close();
	}

	public User get(String id) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
		PreparedStatement ps = c
				.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);

		// get함수는 조회하는 sql문을 이용하므로 -> resultset으로 쿼리 실행결과를 받아서
		ResultSet rs = ps.executeQuery();
		rs.next();
		
		// User object에 담아주는 작업을 진행한 후
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));

		rs.close();
		ps.close();
		c.close();

		//리턴을 진행한다.
		return user;
	}

}
```

```java
// 테스트용 main() 메서드
public static void main(String[] args){
	UserDao dao = new UserDao();
	
	User user = new User();
	user.setId("whiteship");
	user.setname("백기선");
	user.setPassword("married");
	
	//dao를 이용한 DB user table 데이터 삽입 테스트
	dao.add(user);

	System.out.println(user.getId() +"등록 성공");
	
	//dao를 이용한 user 정보 조회 테스트
	User user2 = dao.get(user.id());
	System.out.println(user2.getName());

}
```

해당 코드들에는 각 class마다 너무 많은 책임, 관심사의 분리가 전혀 되지 않고 있다.(기능은 동작하지만)

## 관심사의 분리

객체 지향 설계 → 프로그래밍의 절차적 프로그래밍 패러다임에 비해 초기에 조금 더 많은 번거로운 작업들을 이용하여, 변화에 효과적,효율적으로 대처할 수 있다는 기술적 특징을 가지고 있다.

즉 변화의 폭을 최소한으로 줄여서 코드를 활용할 수 있게 된다. → 이 것을 가능케 하는게 **분리(모든 변경과 발전은 한번에 한가지, 관심이 같은 것 끼리는 하나의 객체안으로, 그 외는 철저히 분리하여 영향 받지 않도록 구성)와 확장**을 고려한 설계를 통해서 가능하게 한다.

### UserDao의 관심사항

userdao : (1)DB연결과 관련된 관심, (2)SQL 실행에 대한 관심, (3)작업이 끝난 후 공유 리소스 정리하는 것에 관심이 한 번에 들어있다. → 분리하여 구성할 필요가 있다.

**중복 메서드 추출 (추출 전)**

```java
public void add(User user) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		
		// 1. DB연결을 위한 connection을 가져온후
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
				
		.....				
				
}

public User get(String id) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
								
}

```

→ 같은 클래스의 메서드에서 connection을 매번 여는 중복된 부분이 코드로 존재 → 분리 필요

**중복 메서드 추출 (추출 후)**

```java
public void add(User user) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		
		// 1. DB연결을 위한 connection을 가져온후
		Connection c =getConnection();
				
		.....				
				
}

public User get(String id) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = getConnection();								
}

// 중복되는 connection 여는 것을 따로 메서드로 빼서 구성
private Connection getConnection() {
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
								.....
}

```

이러한 작업 → 리팩터링(refactoring) : 기능이 추가되거나 바뀐것은 전혀 없지만 UserDao는 이전보다 훨씬 깔끔 해졌으며, 변화에 손쉽게 대응할 수 있는 코드로 만드는 작업 → 지금의 사용한 기법은 메서드 추출(extract method)기법이라고 한다.

 

### DB 커넥션 만들기의 독립

1. UserDAO 내부 소스 코드를 공개하지 않고, 유저DB관리 기능만을 제공해주고, DB와 관련된 부분은 다른 회사에서 구현해서 쓸 수 있게끔 하고 싶다. →(userDAO를 구현체가 아닌 추상체로, 인터페이스로 정의, 이 추상체를 상속받아 구현해서 사용할 수 있게 끔 구성)

```java
public abstract class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();

		PreparedStatement ps = c.prepareStatement(
			"insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());

		ps.executeUpdate();

		ps.close();
		c.close();
	}

	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = getConnection();
		PreparedStatement ps = c
				.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);

		ResultSet rs = ps.executeQuery();
		rs.next();
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));

		rs.close();
		ps.close();
		c.close();

		return user;
	}

	//getConnection 부분을 추상메서드로 전환하여, 추후 각 회사에서 자신의 입맛에 맞는 connection을
	//사용할 수 있도록 구성
	abstract protected Connection getConnection() throws ClassNotFoundException, SQLException ;
```

**템플릿 메서드 패턴** : 슈퍼 클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메서드나, 오버라이딩이 가능한 protected 메서드를 맞게 구현

**팩토리 메서드 패턴 :** 서브클래스에서 어떻게 오브젝트를 생성할 것인지를 결정하는 방법 (서브 클래스에서 다양한 방법으로 오브젝트를 생성하는 메서드를 재정의 할 수 있다.)

→ 단점 : 상속을 사용, 자바는 다중 상속을 허용하지 않기 때문에 다른 목적으로 UserDao에 상속을 적용하기 힘들고, 상속을 사용함으로써 상하위 클래스의 관계는 밀접해져버린다(긴밀한 결합).

### 클래스 분리

 class 내부에서는 중복을 제거하는 리팩터링을 거쳤지만 여전이 UserDAO에는 DB 커넥션 만드는 책임이 존재 → 별도의 클래스로 분해하자

1. 먼저 구현체인  (simpleConnectionMaker) 도입
    
    문제 (1) : simpleConnectionMaker의 메서드가 문제 → 이 구현체를 사용하는 코드에서 변경이 필요하면 구현체 메서드 사용부분일 일일이 하나하나씩 다 변경해야한다. → 작업의 양이 커짐
    
    문제 (2) : userDao → 바뀔 수 있는 정보인, DB커넥션을 가지고 오는 클래스에 대해 너무 알게된다. → 구체적인 방법에 대해 종속적이게 되어 기능 확장에 문제가 생긴다.
    
2. 구현체 대신 인터페이스를 도입하자
    
    기능만 정의 → 구현은 각 세부 클래스에서 구현→ 문제1,2를 해결 가능 (구현체만 수정해서 바꿔끼워 사용하면 되고, 확장을 할 때에도 구현체만 확장하여 구현해서 사용하면 되니까)
    
    문제점 : 여전히 userDao가 connectionmaker에 대한 구현체에 대해 의존(정보를 알고 있다.)한다.
    
    해결책 :  생성자를 변경해서 외부에서 구현체를 생성하여 주입하는 방식으로 해결하여 userDao가 connectionmaker 구현체에 접근, 의존하지 못하도록 구성을 한다.
    
    ## 제어의 역전(IoC)
    
    **프로그램의 제어 흐름 구조가 뒤바뀌는 것**
    
    - 오브젝트가 **자신**이 **직접** 사용할 **오브젝트를 생성하거나 선택하지 않음** 심지어는 **자기자신이 어디서 어떻게 만들어지고 사용되는지 모름**
    - 제어의 역전의 예
        - **템플릿 메소드 패턴**의 **서브클래스가 작성한 확장한 메소드**는 자기자신이 **언제 어떻게 사용될지 모름,** 슈퍼클래스가 **필요에 의해 호출**되서 사용이 되는 형태
        - 라이브러리와 달리 **프레임워크**는 자신이 흐름을 주도하며 **필요에 의해 개발자가 만든 코드를 호출해**서 사용하는 형태이다**프레임워크**에 의해 **코드가 수동적으로 사용**되는 형태반면 **라이브러리**는 **코드가 라이브러리를 호출**해서 쓰는 **능동적** 형태
    
    ## 오브젝트 팩토리를 이용한 스프링Io
    
    ### 팩토리
    
    **객체의 생성 방법을 결정**하고 만들어진 **오브젝트를 돌려주는 오브젝트**
    
    ```java
    public class DaoFactory{
    	public UserDao userDao{
        	// UserDao가 사용할 ConnectionMaker설정
         ConnectionMaker connectionMaker = new DConnectionMaker(); 
         UserDao dao = new UserDao(connectionMaker); // 오브젝트 제공하여 의존관계 설정
    
         return userDao;
        }
    }
    
    public class UserDaoTest {
    	public static void main(String[] args) throws ClassNotFoundException,
         		SQLException{
    
        	UserDao dao = DaoFactory().userDao();//팩토리가 대신 생성
        }
    }
    ```
    

### 애플리케이션 컨택스트와 설정정보

### 빈(bean)

**스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트**

- 스프링 빈 : 스프링 컨테이너가 생성, 관계설정, 사용을 제어하는 제어의 역전이 적용된 오브젝트

### 빈 팩토리(bean factory)

**빈의 생성과 관계설정**과 같은 제어를 **담당하는 IoC 오브젝트**

### 애플리케이션 컨텍스트(application context)

**IoC방식을 따라 만들어진 빈 팩토리(bean factory)**

- 빈 팩토리와 동일하지만 **IoC엔진의 의미가 좀 더 부각**됨
- 오브젝트 팩토리에서 사용했던 원리와 같은 원리를 사용

```java
@Configuration //애플리케이션 컨텍스트 또는 빈 팩토리가 사용할 설정정보라는 표시
public class DaoFactory{

    @Bean //오브젝트 생성을 담당하는 IoC용 메소드라는 표시
    public UserDao userDao{
     	return new UserDao(connectionMaker());
    }

    @Bean
    public ConncetionMaker connectionMaker() {
    	return new DConnectionMaker();
        //return new NConnectionMaker();
    }
}

public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException,
     		SQLException{
        //애플리케이션 컨텍스트 적용
    	ApplicationContext context =
        	new AnnotationConfigApplicationContext(DaoFactory.class);
        UserDao dao =
        		context.getBean("userDao", UserDao.class); //빈 이름만 알면 접근가능
    }
}
```

**팩토리와 동일**하게 클라이언트가 **필요에 의해 애플리케이션 컨택스트를 호출**

- **애플리케이션 컨텍스트**는 **빈의 이름**으로 **사용할 오브젝트를 알아서 찾음**
- 만들어진 오브젝트를 클라이언트가 받아서 사용

### 애플리케이션 컨텍스트의 장점

• 클라이언트는 **구체적인 팩토리 클래스를 알 필요가 없다**

• 애플리케이션 컨텍스트는 **종합 IoC서비스를 제공**해준다

• 애플리케이션 컨텍스트는 **빈을 검색하는 다양한 방법을 제공**한다

## 싱글톤 레지스트리와 오브젝트 스코프

### 오브젝트의 동일성과 동등성

### **동일성**

**완전히 같은** 오브젝트 **'=='연산자로 비교**

- 주소값까지 같은 오브젝트

### **동등성**

**동일한 정보**를 가진 오브젝트 **'equals()'메소드로 비교**

- 주소값은 다르지만 동일한 정보를 지님

### 싱글톤

여러번에 걸쳐 오브젝트를 요청해도 **매번 동일한 오브젝트가 나오는 것**

- **하나의 오브젝트만 만들어서 그것을 공유**하는 형태

### 싱글톤 패턴

**오브젝트를 하나만 만들도록 강제**하는 패턴 하나만 만들어진 오브젝트는 **전역적으로 접근가능**

한계

- **private 생성자**를 가지고 있어 **상속불가**
- **테스트 하기가 힘듦**
- 서버환경에서는 **싱글톤이 하나만 만들어지는 것을 보장하지 못함**
- 싱글톤의 사용은 **전역상태**를 만들어 **바람직하지 못함**

### 싱글톤 레지스트리

스프링이 제공하는 **직접 싱글톤 형태의 오브젝트를 만들고 관리**하는 기능

싱글톤 패턴의 문제점을 해결

### 스프링 빈의 스코프

**빈이 생성되고, 존재하고, 적용되는 범위**

기본 스코프는 싱글톤이다 (경우에 따라 다른 스코프를 가질 수 있음

## 의존관계 주입(Dependency Injection)

IoC라는 용어가 폭넓게 사용되는 용어여서 스프링의 IoC 기능을 명확하게 설명하지 못함

**의존관계 주입(DI)**라는 의도가 명확히 드러나는 이름 사용

### 의존관계란?

**다른 클래스나 모듈이 변화**하면 그에 영향을 받아 **자신도 변화하는 것**

**A가 B에게 의존**하고 있다 - **B가 변화하면 A에 영향**을 미친다

의존관계엔 **방향성이 존재**한다 (**A가 변화해도 B에는 영향이 없다**)

### 의존관계 주입(Dependency Injection, DI)이란?

**오브젝트 레퍼런스**를 **외부로부터 제공(주입)**받고 이를 통해 **여타 오브젝트와 다이나믹하게 의존관계가 만들어지는 것**

조건

- **클래스 모델이나 코드**에는 **런타임 시점의 의존관계가 드러나지 않는다**이를 위해서 **인터페이스에만 의존해야함!**
- **런타임 시점의 의존관계**는 **컨테이너나 팩토리 같은 제3의 존재가 결정**
- 의존관계는 **사용할 오브젝트에 대한 래퍼런스**를 **외부에서 제공(주입) 해줌**으로써 만들어짐

### UserDao의 의존관계

```java
public class DaoFactory{
	public UserDao userDao{
     	return new UserDao(connectionMaker());
    }
}

public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException,
     		SQLException{

    	UserDao dao = DaoFactory().userDao();//팩토리가 대신 생성
    }
}
```

**UserDao는 ConnectionMaker에만 의존**하고있음DConnectionMaker가 바뀌어도 아무런 영향이 없다인터페이스에만 의존을 하고있으면 결합도가 낮다

UserDao와 DConnectionMaker는 **런타임시점에 의존관계가 형성**됨**⇒ 의존관계 주입의 조건 만족!**

사전에 DB를 연결하겠다고 설계는 미리 해놨지만 DB코드의 구체적인 내용은 UserDao가 알 수 없음DConnectionMaker는 **런타임 시에 의존관계를 맺는 대상 - 의존 오브젝트의존관계 주입 : 클라이언트와 의존 오브젝트를 연결해주는 작업** (클라이언트에 의존 오브젝트를 주입해주는 작업)

### 의존관계 검색(Dependency Lookup, DL)

의존관계를 맺는 방법이 **외부에서의 주입이 아닌 스스로 검색**을 이용

자신이 스스로 어떤 클래스의 오브젝트를 선택하는건 아님

외부에서 만든 **오브젝트를 가져올때, 스스로 컨테이너에게 요청하는 방식**

```java
public class UserDao {
	...

    //이전코드
    public UserDao(ConnectionMaker connectionMaker){
		this.connectionMaker = connectionMaker;
	}

    //의존관계 검색코드
    public UserDao(){ 										//UserDao가
		DaoFactory daofactory = new DaoFactory();
    	this.connectionMaker = daoFactory.connectionMaker();//직접 DaoFactory에게 요청
	}

    //의존관계 검색코드 - 애플리케이션 컨택스트 사용
    public UserDao() {
    	AnnotationConfigApplicationContext context =
        	new AnnotationConfigApplicationContext(DaoFactory.class);
        this.connectionMaker =
        		context.getBean("connectionMaker", ConnectionMaker.class);
    }
    ...
}
```

### 의존관계 검색과 의존관계 주입

기능자체는 거의 동일함

- **의존관계 주입**이 **코드가 더욱 간결**함
- **의존관계 주입**은 **성격이 다른 오브젝트에 의존하지 않음 ⇒ 대개는 의존관계 주입을 쓰는게 바람직** 하다
- **의존관계 검색**에서는 **검색하는 오브젝트 자신**이 **스프링의 빈일 필요가 없다**
- **의존관계 주입**에서는 **주입받는 오브젝트**도 **컨테이너가 만드는 빈 오브젝트여야한다**

### 메소드를 통한 의존관계 주입

- 수정자(setter) 메소드를 이용한 주입
- 일반 메소드를 이용한 주입

### 궁금했던 점

---

애플리케이션 컨텍스트 기본 설정을 사용할 때 기본 설정이 싱글턴으로 관리를 한다고 했을 때 오브젝트(빈으로 등록된 것)이 최초의 호출될 때 생성되서 제공되는줄 알았는데

![image.png](1%20%EC%8A%A4%ED%94%84%EB%A7%81%20%EC%8A%A4%ED%84%B0%EB%94%94%201%EC%A3%BC%EC%B0%A8/image.png)

이렇게 되면 구조가 잘 쓰이지 않는, 호출되지 않는 빈들은 리소스를 잡아 먹는게 아닌가 라는 생각을 했었는데

![image.png](1%20%EC%8A%A4%ED%94%84%EB%A7%81%20%EC%8A%A4%ED%84%B0%EB%94%94%201%EC%A3%BC%EC%B0%A8/image%201.png)

## @Lazy singleton

- 시작 빠름
- 첫 요청 때 느림
- 실제 안 쓰는 빈은 생성 안 됨

```
부팅 비용 ↓
첫 사용 비용 ↑
```