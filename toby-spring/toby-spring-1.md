# 1장. 오브젝트와 의존관계

스프링이 자바에서 가장 중요하게 가치를 두는 것은 객체지향 프로그래밍이 가능한 언어라는 점이다. 
스프링은 오브젝트를 어떻게 효과적으로 설계하고, 구현하고, 사용하고, 개선해나갈 것인가에 대한 명쾌한 기준을 마련해준다. 
동시에 객체지향 기술과 설계, 구현에 관한 실용적인 전략과 검증된 베스트 프랙티스를 개발자가 쉽게 적용할 수 있도록 프레임워크 형태로 제공한다.

# 1) 객체지향

## DAO 코드 변천사

기존 코드
```java
public void add(User user) throws ClassNotFoundException, SQLException {
        Class.forName("org.h2.Driver");
        Connection connection = DriverManager.getConnection(
            "jdbc:h2:tcp://localhost/springbook", "spring", "book"
        );

        PreparedStatement ps = connection.prepareStatement(
            "insert into users(id, name, password) values(?,?,?)"
        );
        ps.setString(1, user.getId());
        ps.setString(2, user.getName());
        ps.setString(3, user.getPassword());

        ps.executeUpdate();

        ps.close();
        connection.close();
    }
```

관심사의 분리
- 관심사의 분리: 관심이 같은 것끼리는 하나의 객체 안으로, 관심이 다른 것끼리는 서로 영향을 주지 않도록 분리
- 개발자가 객체를 설계할 때 가장 염두에 둬야 할 상황은 미래의 변화를 어떻게 대비할 것인가 이다. 미래를 준비하는 데 있어 가장 중요한 과제는 변화에 대비할 것인가이고, 가장 좋은 대책은 변화의 폭을 최소한으로 줄여주는 것이다.

### 상속을 통한 확장
```java
public abstract class UserDao {
    public void add(User user) throws ClassNotFoundException, SQLException {
            Connection connection = getConnection();
            //...
        }

        public abstract Connection getConnection() throws ClassNotFoundException, SQLException;
    }
}

public class NUserDao extends UserDao{
    public Connection getConnection() throws ClassNotFoundException, SQLException {
        // N사 DB 커넥션 생성 코드
    }
}
```

### 클래스의 분리
관심사와 변화의 성격이 다른 코드를 별도의 클래스로 분리
```java
public class UserDao {
    private final SimpleConnectionMaker connectionMaker = new SimpleConnectionMaker();

		public void add(User user) throws ClassNotFoundException, SQLException {
        Connection connection = connectionMaker.openConnection();

        PreparedStatement ps = connection.prepareStatement(
            "insert into users(id, name, password) values(?,?,?)"
        );
        ps.setString(1, user.getId());
        ps.setString(2, user.getName());
        ps.setString(3, user.getPassword());

        ps.executeUpdate();

        ps.close();
        connection.close();   
		}
}

public class SimpleConnectionMaker {
    public Connection openConnection() throws ClassNotFoundException, SQLException {
        Class.forName("org.h2.Driver");
        Connection connection = DriverManager.getConnection(
             "jdbc:h2:tcp://localhost/springbook", "spring", "book"
        );

        return connection;
    }
}
```
하지만, UserDao 수정 없이 DB 커넥션 생성 기능을 변경할 방법이 없다

### 인터페이스의 도입
두 클래스 중간에 추상적인 느슨한 연결고리. 
```java
public class UserDao {
    private final ConnectionMaker connectionMaker = new NConnectionMaker();
}

public interface ConnectionMaker {

    Connection openConnection() throws ClassNotFoundException, SQLException;
}

public class NConnectionMaker implements ConnectionMaker{
    public Connection openConnection() throws ClassNotFoundException, SQLException {
        Class.forName("org.h2.Driver");

        return DriverManager.getConnection(
            "jdbc:h2:tcp://localhost/springbook", "spring", "book"
        );
    }
}
```
하지만, 초기에 한 번 어떤 클래스의 오브젝트를 사용할지 결정하는 생성자의 코드가 제거되지 않고 남아있다.

### 관계설정 책임의 분리
UserDao와 ConnectionMaker 구현 클래스의 관계를 결정해주는 기능을 분리해서 UserDaoTest로 런타임 사용관계, 의존관계를 맺어주는 역할

```java
public class UserDaoTest { // 클라이언트로 생성 이동
    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        UserDao dao = new UserDao(new NConnectionMaker());
		}
}

public class UserDao { // UserDao는 직접적으로 자신이 사용할 클래스를 모른다!
    private final ConnectionMaker connectionMaker;

    public UserDao(ConnectionMaker connectionMaker) {
        this.connectionMaker = connectionMaker;
    }
}
```

### 제어의 역전 
팩토리: 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 역할
(오브젝트 생성과, 사용의 역할과 책임을 분리)
애플리케이션의 컴포넌트 역할을 하는 오브젝트와 애플리케이션 구조를 결정하는 오브젝트를 분리할 수 있다.

```java
public class DaoFactory {
    public UserDao userDao() {
        return new UserDao(connectionMaker());
    }
    public ConnectionMaker connectionMaker() {
        return new NConnnectionMaker();
    }
}

public class UserDaoTest {
    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        UserDao dao = new DaoFactory().userDao();
		}
}
```

## 원칙과 패턴
- 개방 폐쇄 원칙: 클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다
- 높은 응집도와 낮은 결합도
응집도가 높다: 하나의 클래스가 하나의 책임 또는 관심사에만 집중되고, 변화가 일어날 때 해당 모듈에서 변하는 부분이 크다
결합도가 낮다: 하나의 오브젝트에 변경이 일어날 때 관계를 맺고 있는 다른 오브젝트에게 변화를 요구하는 상태가 적도록. 하나의 변경이 다른 객체의 변경으로 전파되지 않도록
- 전략 패턴: 자신의 기능 맥락에서 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 외부로 분리시키고, 이를 구현한 구체적인 클래스를 필요에 따라 바꿔서 사용할 수 있도록 하는 패턴


# 2) 스프링

## IoC (제어의 역전)

일반적인 프로그램에서는 main() 에서 시작해서 개발자가 정한 순서를 따라 오브젝트가 생성, 실행된다.
하지만, 서블릿같은 경우 서블릿에 대한 코드를 직접 실행할 수는 없지만, 제어 권한을 가진 컨테이너가 서블릿 객체를 만들고 사용하게 된다.

이렇게 스스로 제어권을 가지고 있지 않고, 프로그램의 제어 흐름 구조가 뒤바뀐 것이 제어의 역전. 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지 않고, 모든 제어 권한을 특별한 오브젝트에 위임한다. 

프레임워크 또는 컨테이너와 같이 애플리케이션 컴포넌트의 생성, 관계설정, 사용, 생명주기 관리 등을 관장하는 존재가 필요 => 스프링의 IoC 컨테이너 (BeanFactory, ApplicationContext)

### 용어 정리
- 빈: 스프링이 IoC 방식으로 관리하는 오브젝트. 스프링이 제어권을 가지고 직접 생성, 제어를 담당하는 오브젝트
- 빈 팩토리: 스프링의 IoC를 담당하는 핵심 컨테이너. 빈을 등록, 조회, 관리하는 기능
- 애플리케이션 컨텍스트: 빈 팩토리를 확장한 IoC 컨테이너. 스프링이 제공하는 부가 서비스들이 추가
- 설정 정보(Configuration Metadata): 애플리케이션 컨텍스트가 IoC를 적용하기 위해 사용하는 메타정보
IoC 컨테이너에 의해 관리되는 애플리케이션 오브젝트를 구성할 때나 컨테이너에 기능을 설정하고 싶을 때 사용
- 컨테이너: IoC 방식으로 빈을 관리.

### ApplicationContext 동작

1. DaoFactory 클래스를 설정정보로 등록
2. @Bean이 붙은 메소드의 이름을 가져와 빈 목록을 만듦
3. .getBean()이 호출되면 빈 목록에서 요청한 이름이 있는지 찾음
4. 있다면 빈 생성 메소드를 호출해서 오브젝트를 생성 후 클라이언트에게 돌려준다.

### 왜 애플리케이션 컨텍스트 사용? 
범용적이고 유연한 방법으로 IoC 기능을 확장
- 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다
- 애플리케이션 컨텍스트는 종합 IoC 서비스(자동 생성, 후처리, 인터셉팅 등)를 제공
- 빈을 검색하는 다양한 방법을 제공

## 싱글톤 

### 왜 스프링은 싱글톤으로 빈을 만들까?
애플리케이션 컨텍스트는 디폴트로 내부에서 생성하는 빈 오브젝트는 싱글톤으로 만든다.
스프링은 고성능을 요구하는 서버 환경에서 쓰는 기술이고, 보통 서버는 하나의 요청을 처리하기 위한 계층형 구조로 이루어져 있다. 매번 요청이 올 때마다 각 로직을 담당하는 오브젝트를 새로 만들어 사용하면 성능 부하가 많다.
그래서, 서비스 오브젝트라는 개념이 있다. 싱글톤으로 구성된 각 계층의 로직이 담긴 오브젝트를 말한다.
서블릿은 대부분 멀티스레드에서 싱글톤으로 동작하는데 서블릿 클래스당 하나의 오브젝트만 만들어두고, 사용자의 요청을 담당하는 여러 스레드들이 이 오브젝트를 공유해서 사용한다.

### 자바 싱글톤 구현 한계
```
public class SingleTonInstance {
    private static SingleTonInstance INSTANCE;

    public synchronized static SingleTonInstance getINSTANCE() {
        
        if(INSTANCE == null){
           INSTANCE = new SingleTonInstance(); 
        }
        return INSTANCE;
    }
}
```
1) private 생성자를 가지고 있어 상속할 수 없다.
2) 싱글톤은 테스트하기가 힘들다
3) 서버 환경에서는 싱글톤이 하나만 만들어지는 것을 보장하지 못한다.
4) 싱글톤은 static 메소드를 사용해 쉽게 접근이 가능해 전역 상태로 사용되기 쉬워서 바람직하지 못하다.

### Singleton Registry
위의 문제 때문에, 스프링은 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공.
평범한 자바 클래스도 싱글톤으로 활용 가능하고, public 생성자를 가질 수 있어 테스트하기 쉽다. 또한 스프링이 지지하는 객체지향적인 설계 방식과 원칙, 디자인 패턴을 적용하는데 제약이 없다. 

### 싱글톤과 무상태
싱글톤이 멀티스레드 환경에서 서비스 오브젝트로 사용되는 경우 상태정보를 내부에 갖고 있지 않은 무상태 방식으로 만들어야 한다. 상태값을 다뤄야할 때는 파라미터, 메소드 안 로컬변수, 리턴값을 사용한다.

다만, 자신이 사용하는 다른 싱글톤 빈을 저장하려는 용도라면 인스턴스 변수를 사용해도 된다. (읽기 전용 속성)

### 빈의 스코프
스프링이 관리하는 빈이 생성, 적용되는 범위
기본 스코프: 싱글톤
프프로토타입: 빈을 요청할 때마다 새로운 오브젝트 생성
요청: HTTP 요청이 생길 때마다 생성
세션: 웹 세션과 동일한 라이프 사이클

## DI (의존 관계 주입)
IoC는 많은 의미를 담고 있는 용어라서 스프링을 IoC 컨테이너라고만 하면 스픞링의 기능을 명확하게 설명하지 못한다. 스프링 IoC의 동작 원리는 '의존관계 주입'이라는 용어를 쓸 때 분명하게 드러난다.

### 용어
의존 관계: 의존관계에는 방향성이 있다. A가 B에 의존한다는 것은 B가 변하면 A에 영향을 끼친다는 뜻이다.
의존 오브젝트: 런타임 시에 의존 관계를 맺는 대상, 실제 사용대상인 오브젝트를 말한다.
의존 관게 주입: 구체적인 의존 오브젝트와 그것을 사용할 클라이언트 오브젝트를 런타임 시에 연결해주는 것.

### 의존 관계 주입의 세가지 조건
1) 클래스 모델이나 코드에는 런타임 시점의 의존관계가 드러나지 않는다. 그러기 위해 인터페이스에만 의존해야한다.
2) 런타임 시점의 의존관계는 컨테이너 같은 제 3의 존재가 결정한다.
3) 의존관계는 사용할 오브젝트에 대한 레퍼런스를 외부에서 제공해줌으로 만들어진다.

DI는 자신이 사용할 오브젝트에 대한 선택과 생성 제어권을 외부로 넘기고, 자신은 수동적으로 주입받은 오브젝트를 사용한다는 점에서 IoC 개념에 잘 맞는다. 

### DI vs DL (의존관계 주입과 검색)
스프링이 IoC 방법을 DI만 제공하는 것이 아니다.
의존관계를 맺는 방법이 외부로부터의 주입이 아니라 스스로 검색을 할 때인 의존관계 검색이라는 방법도 있다.
```java 
public UserDao() {
    AnnotationConfigAppcationContext context = new AnnotationConfigAppcationContext(DaoFactory.class);
    this.connectionMaker = context.getBean("connectionMaker", ConnectionMaker.class);
}
```

의존 관계 검색은 오브젝트 팩토리 클래스나 스프링 API가 기존 코드 안에 들어가므로 역할 분리가 명확하지 않다.
그래서 등록된 빈을 직접 가져와야하는 경우나 가져오는 대상이 빈에 등록되지 않은 경우에만 사용한다.

### 응용 사례
1) 기능 구현의 교환
로컬, 운영, QA DB 연결 클래스를 사용할 때 
설정 정보에 해당하는 DaoFactory의 connectionMaker()만 다르게 설정하면  나머지 코드에 손대지 않고 의존관계 설정이 가능하다.

2) 부가 기능 추가
DAO가 DB를 연결한 횟수를 카운드하고 싶은 경우,
DAO 와 DB 커넥션을 만드는 오브젝트 사이에 연결횟수를 카운팅하는 오브젝트 추가하면 된다.
UserDao는 새 클래스를 의존하게 된다. 

### 의존관계 주입 방법
1) 생성자
2) 수정자 메소드 (setter)
3) 일반 메소드
- 여러개의 파라미터로 의존관계 주입을 원할 때 사용

스프링은 어떻게 오브젝트가 설계되고 만들어지고 어떻게 관계를 맺고 사용되는지에 관심을 갖는 프레임워크다. 하지만 오브젝트를 어떻게 설계하고, 분리하고, 개선하고, 어떤 의존관계를 가질지 결정하는 것은 개발자의 역할이다.
