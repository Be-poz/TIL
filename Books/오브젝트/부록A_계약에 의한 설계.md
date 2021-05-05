# 계약에 의한 설계

인터페이스만으로는 객체의 행동에 관한 다양한 관점을 전달하기 어렵다. 명령의 부수효과를 쉽고 명확하게 표현할 수 있는 커뮤니케이션 수단이 필요한데 이 , **계약에 의한 설계(Design By Contract, DBC)** 가 주는 혜택을 눈여겨봐야한다.  

계약에 의한 설계를 사용하면 협력에 필요한 다양한 제약과 부수효과를 명시적으로 정의하고 문서화할 수 있다. 클라이언트 개발자는 오퍼레이션의 구현을 살펴보지 않더라도 객체의 사용법을 쉽게 이해할 수 있다. 계약은 실행 가능하기 때문에 구현에 동기화돼 있는지 여부를 런타임에 검증할 수 있다.  

<br/>

## 협력과 계약

### 부수효과를 명시적으로

객체지향의 핵심은 협력 안에서 객체들이 수행하는 행동이다. 작성된 계약은 문서화로 끝나는 것이 아니라 제약 조건의 만족 여부를 실행 중에 체크할 수 있다. 계약에 의한 설계를 사용하면 제약 조건을 명시적으로 표현하고 자동으로 문서화할 수 있을뿐만 아니라 실행을 통해 검증할 수 있다.  

<br/>

### 계약

현재 살고 있는 집을 리모델링하고 싶다고 가정해보자 계약은 대략 다음과 같은 특성을 가진다.  

* 각 계약 당사자는 계약으로부터 **이익**을 기대하고 이익을 얻기 위해 **의무**를 이행한다.
* 각 계약 당사자의 이익과 의무는 계약서에 **문서화**된다. 

한쪽의 의무는 반대쪽의 권리가 된다. 계약은 협력을 명확하게 정의하고 커뮤니케이션할 수 있는 범용적인 아이디어다.  

<br/>

## 계약에 의한 설계

* 협력에 참여하는 각 객체는 계약으로부터 **이익**을 기대하고 이익을 얻기 위해 **의무**를 이행한다.
* 협력에 참여하는 각 객체의 이익과 의무는 객체의 인터페이스 상에 **문서화**된다.

``public Reservation reserve (Customer customer, int audienceCount)`` 반환 타입과 메서드 이름 파라미터 타입 등 오퍼레이션이 클라이언트에게 어떤 것을 제공하려고 하는지 충분히 설명할 수 있다. 6장에서 설명한 **의도를 드러내는 인터페이스**를 만들면 오퍼레이션의 시그니처만으로도 어느 정도까지는 클라이언트와 서버가 협력을 위해 수행해야 하는 제약조건을 명시할 수 있다.  

계약은 여기서 한걸음 더 나아간다. reserve 메서드를 호출할 때 클라이언트 개발자는 customer의 값으로 null을 전달할 수 있고 audienceCount로 음수를 전달할 수도 있다. 이런 정상적인 상태가 아닌 객체일 떄에 reserve 메서드를 호출할 수 없어야 한다.  

서버는 자신이 처리할 수 있는 범위의 값들을 클라이언트가 전달할 것이라고 기대한다. 클라이언트는 자신이 원하는 값을 서버가 반환할 것이라고 예상한다. 클라이언트는 메세지 전송 전과 후의 서버의 상태가 정상일 것이라고 기대한다. 이  세 가지 기대가 바로 계약에 의한 설계를 구성하는 세 가지 요소에 대응된다.  

* **사전조건(precondition)** : 메서드가 호출되기 위해 만족돼야 하는 조건. 메서드의 요구사항을 명시하고 이것이 만족되지 않을 경우 메서드가 실행되서는 안 된다. 사전조건을 만족시키는 것은 메서드를 실행하는 클라이언트의 의무다.
* **사후조건(postcondition)** : 메서드가 실행된 후에 클라이언트에게 보장해야 하는 조건. 클라이언트가 사전조건을 만족시켰다면 메서드는 사후조건에 명시된 조건을 만족시켜야 한다. 만약 클라이언트가 사전조건을 만족시켰는데도 사후조건을 만족시키지 못한 경우에는 클라이언트에게 예외를 던져야 한다. 사후조건을 만족시키는 것은 서버의 의무다.
* **불변식(invariant)** : 항상 참이라고 보장되는 서버의 조건. 메서드가 실행되는 도중에는 불변식을 만족시키지 못할 수도 있지만 메서드를 실행하기 전이나 종료된 후에 불변식은 항상 참이어야 한다.

위 셋을 기술할 때는 실행 절차를 기술할 필요 없이 상태 변경만을 명시하기 때문에 코드를 이해하고 분석하기 쉬워진다. 클라이언트 개발자는 사전조건에 명시된 조건을 만족시키지 않으면 메서드가 실행되지 않을 것이라는 사실을 알고 있다. 불변식을 사용하면 클래스의 의미를 쉽게 설명할 수 있고 클라이언트 개발자가 객체를 더욱 쉽게 예측할 수 있다. 사후조건을 믿는다면 객체가 내부적으로 어떤 방식으로 동작하는지 걱정할 필요가 없다. 위 세 가지 요소에는 클라이언트 개발자가 알아야 하는 모든 것이 포함돼 있을 것이다.  

C#의 Code Contract를 이용해서 알아보자.  

<br/>

### 사전조건

사전조건이란 메서드가 정상저긍로 실행되기 위해 만족해야 하는 조건이다. 일반적으로 사전조건은 메서드에 전달된 인자의 정합성을 체크하기 위해 사용된다. 메서드의 사전조건을 정의하기 위해 사용되는 메서드는 Contract의 Requires 메서드다.  

```java
public Reservation Reserve(Customer customer, int audienceCount) {
    Contract.Requires(customer != null);
    Contract.Requires(audienceCount >= 1);
    return new Reservation(customer, this, calculateFee(audienceCount), audienceCount);
}
```

만약 클라이언트가 사전조건을 만족시키지 못할 경우 Reserve 메서드는 최대한 빨리 실패해서 클라이언트에게 버그가 있다는 사실을 알린다. Contract.Requires 메서드는 클라이언트가 계약에 명시된 조건을 만족시키지 못할 경우 ContractException 예외를 발생시킨다.  

<br/>

### 사후조건

사후조건은 메서드의 실행 결과가 올바른지를 검사하고 실행 후에 객체가 유효한 상태로 남아 있는지를 검증한다. 간단히 말해서 사후조건을 통해 메서드를 호출한 후에 어떤 일이 일어났는지를 설명할 수 있는 것이다. 사후조건은 다음과 같은 용도로 사용된다.  

* 인스턴스 변수의 상태가 올바른지를 서술하기 위해
* 메서드에 전달된 파라미터의 값이 올바르게 변경됐는지를 서술하기 위해
* 반환값이 올바른지를 서술하기 위해

그리고 다음과 같은 이유로 사전조건보다 사후조건을 정의하는 것이 더 어려울 수 있다.  

* **한 메서드 안에서 return 문이 여러 번 나올 경우** : 모든 return 문마다 결괏값이 올바른지 검증하는 코드를 추가해야 한다. 다행히도 계약에 의한 설계를 지원하는 대부분의 라이브러리는 결괏값에 대한 사후조건을 한 번만 기술할 수 있게 해준다.
* **실행 전과 실행 후의 값을 비교해야 하는 경우** : 실행 전의 값이 메서드 실행으로 인해 다른 값으로 변경됐을 수 있기 때문에 두 값을 비교하기 어려울 수 있다. 다행히 계약에 의한 설계를 지원하는 대부분의 라이브러리는 실행 전의 값에 접근할 수 있는 간편한 방법을 제공한다.

사후조건을 정의하기 위해서 Contract.Ensures 메서드를 제공한다. Reserve 메서드의 사후조건은 반환값이 Reservation 인스턴스가 null이어서는 안 된다는 것이다.  

```java
public Reservation Reserve(Customer customer, int audienceCount) {
    Contract.Requires(customer != null);
    Contract.Requires(audienceCount >= 1);
    Contract.Ensures(Contract.Result<Reservation>() != null);
    return new Reservation(customer, this, calculateFee(audienceCount), audienceCount);
}
```

Ensures 메서드 안에서 사용된 Contract.Result<T> 메서드가 바로 Reserve 메서드의 실행 결과에 접근할 수 있게 해주는 메서드다. 이 메서드는 제너릭 타입으로 메서드의 반환 타입에 대한 정보를 명시할 것을 요구한다.  

```java
public decimal Buy(Ticket ticket) {
    Contract.Requires(ticket != null);
    Contract.Ensures(Contract.Result<decimal>() >= 0);
    if(bag.invited) {
        bag.Ticket = ticket;
        return 0;
    } else {
        bag.Ticket = ticket;
        bag.MinusAmount(ticket.Fee);
        return ticket.Fee;
    }
}
```

다음과 같이 return이 복수 개인 메서드에서도 Ensures를 한 번만 기술하는 것으로도 처리가 된다.  

```java
public string Middle(string text) {
    Contract.Requires(text != null && text.Length >= 2);
    Contract.Ensures(Contract.Result<string>().Length < Contract.OldValue<string>(text).Length);
    text = text.Substring(1, text.Length - 2);
    return text.Trim();
}
```

다음과 같이 Contract.OldValue<T>를 이용하면 메서드 실행 전의 상태에 접근할 수 있어 text가 변경되어도 변경 이전의 값을 이용해 조건을 걸 수가 있다.  

<br/>

### 불변식

각 메서드마다 달라지는 사전조건과 사후조건과는 달리 불변식은 인스턴스 생명주기 전반에 걸쳐 지켜져야 하는 규칙을 명세한다.  
불변식은 다음과 같은 두 가지 특성을 가진다.  

* 불변식은 클래스의 모든 인스턴스가 생성된 후에 만족돼야 한다. 이것은 클래스에 정의된 모든 생성자는 불변식을 준수해야 한다는 것을 의미한다.
* 불변식은 클라이언트에 의해 호출 가능한 모든 메서드에 의해 준수돼야 한다. 메서드가 실행되는 중에는 객체의 상태가 불안정한 상태로 빠질 수 있기 때문에 불변식을 만족시킬 필요는 없지만 메서드 실행 전과 메서드 종료 후에는 항상 불변식을 만족하는 상태가 유지돼야 한다.

불변식은 클래스의 모든 메서드의 사전조건과 사후조건에 추가되는 공통의 조건으로 생각할 수 있다. Contract.Invariant 메서드를 이용해 불변식을 정의할 수 있다. 불변식은 생성자 실행 후, 메서드 실행 전, 메서드 실행 후에 호출돼야 한다. ContractInvariantMethod 애트리뷰트를 이용하면 일일이 호출할 필요 없이 불변식을 체크해야 하는 모든 지점에 자동으로 추가된다.  

```java
public class Screening {
    private Movie movie;
    private int sequence;
    private DateTime whenScreened;
    
    [ContractInvariantMethod]
    private void Invariant() {
        Contract.Invariant(movie != null);
        Contract.Invariant(Sequence >= 1);
        Contract.Invariant(whenScreened > DateTime.now);
    }
}
```

<br/>

## 계약에 의한 설계와 서브타이핑

계약에 의한 설계의 핵심은 클라이언트와 서버 사이의 견고한 협력을 위해 준수해야 하는 규약을 정의하는 것이다. 계약에 의한 설계는 클라이언트가 만족시켜야 하는 사전조건과 클라이언트의 관점에서 서버가 만족시켜야 하는 사후조건을 기술한다. 계약에 의한 설계와 리스코프 치환 원칙이 만나는 지점이 바로 이곳이다. 리스코프 치환 원칙의 규칙을 두 가지 종류로 세분화할 수 있다. 첫 번째 규칙은 협력에 참여하는 객체에 대한 기대를 표현하는 **계약 규칙**이고, 두 번째 규칙은 교체 가능한 타입과 관련된 **가변성 규칙**이다.  

**계약 규칙**은 슈퍼타입과 서브타입 사이의 사전조건, 사후조건, 불변식에 대해 서술할 수 있는 제약에 관한 규칙이다.

* 서브타입에 더 강력한 사전조건을 정의할 수 없다.
* 서브타입에 더 완화된 사후조건을 정의할 수 없다.
* 슈퍼타입의 불변식은 서브타입에서도 반드시 유지돼야 한다.

**가변성 규칙**은 파라미터와 리턴 타입의 변형과 관련된 규칙이다.

* 서브타입의 메서드 파라미터는 반공변성을 가져야 한다.
* 서브타입의 리턴 타입은 공변성을 가져야 한다.
* 서브타입은 슈퍼타입이 발생시키는 예외와 다른 타입의 예외를 발생시켜서는 안 된다.

자바의 assert를 사용해서 구현해보자.  

<br/>

## 계약 규칙

```java
public interface RatePolicy {
    Money calculateFee(List<Call> calls);
}
```

핸드폰 과금 시스템에서 RatePolicy의 하위 클래스들이 정말 이것의 서브타입인지 다시 말해서 리스코프 치환 원칙을 만족하는지 확인하기 위해서 RatePolicy 인터페이스의 구현 클래스들이 RatePolicy의 클라이언트인 Phone과 체결한 계약을 준수하는지 살펴봐야 한다.  

이해를 돕기 위해 요금 청구서를 발생하는 publishBill 메서드를 Phone에 추가한다. 청구서의 개념을 표현한 Bill 클래스는 요금 청구의 대상인 핸드폰과 통화 요금을 인스턴스 변수로 포함한다.  

```java
public class Bill {
    private Phone phone;
    private Money fee;
    
    public Bill(Phone phone, Money fee) {
        if(phone == null) {
            throw new IllegalArgumentException();
        }
        
        if(fee.isLessThan(Money.ZERO)) {
            throw new IlleaglArgumentException();
        }
        
        this.phone = phone;
        this.fee = fee;
    }
}

public class Phone {
    private RatePolicy ratePolicy;
    private List<Call> calls = new ArrayList<>();
    
    public Phone(RatePolicy ratePolicy) {
        this.ratePolicy = ratePolicy;
    }
    
    public void call(Call call) {
        calls.add(call);
    }
    
    public Bill publishBill() {
        return new Bill(this, ratePolicy.calculateFee(calls));
    }
}
```

publishBill 메서드에서 요금은 0원보다 커야 한다. 따라서 calculateFee의 사후조건을  
``assert result.isGreaterThanOrEqual(Money.ZERO);`` 과 같이 정의할 수 있다.  
calculateFee 메서드를 호출할 때에 calls가 null 이어서는 안된다. 따라서 calculateFee 오퍼레이션의 사전조건을 다음과 같이 정의한다.  
``assert calls != null;``   

RatePolicy 인터페이스를 구현하는클래스가 RatePolicy의 서브타입이 되기 위해서는 위에서 정의한 사전조건과 사후조건을 만족해야 한다. 먼저 기본 정책을 구현하는 추상 클래스인 BasicRatePolicy에 사전조건과 사후조건을 추가한다.  

```java
public abstract class BasicRatePolicy implements RatePolicy {
    @Override
    public Money calculateFee(List<Call> calls){
        // 사전조건
        assert calls != null;
        
        Money result = Money.ZERO;
        
        for(Call call : calls){
            result.plus(calculateCallFee(call));
        }
        
        // 사후조건
        assert result.isGreaterThanOrEqual(Money.ZERO);
        return result;
    }
    
    protected abstract Money calculateCallFee(Call call);
}
```

서브타입은 더 강력한 사전조건을 정의할 수 없다. 사전조건을 만족시키는 것은 클라이언트의 책임이다. 클라이언트인 Phone은 오직 RatePolicy 인터페이스만 알고 있기 때문에 RatePolicy가 null을 제외한 어떤 calls라도 받아들인다고 가정한다면 빈 List를 전달하더라도 문제가 발생하지 않는다고 예상할 것이다. 클라이언트의 관점에서 BasicRatePolicy는 RatePolicy를 대체할 수 없기 때문에 리스코프 치환 원칙을 위반한다. BasicRatePolicy는 RatePolicy의 서브타입이 아닌 것이다. 역으로 사전조건을 완화시키는 것은 리스코프 치환 원칙을 위반하지 않는다.  

서브타입에 더 완화된 사후조건을 정의할 수 없다. 사후조건을 완화한다는 것은 서버가 클라이언트에게 제공하겠다고 보장한 계약을 충족시켜주지 못한다는 것을 의미한다. 서버는 계약을 위반했기 때문에 이제 계약은 더 이상 유효하지 않다. 다시 말해서 서브타입이 아니라는 것이다. 역으로 사후조건을 강화시키는 것은 문제되지 않는다.  

불변식은 메서드가 실행되기 전과 후에 반드시 만족시켜야 하는 조건이다. 모든 객체는 객체가 생성된 직후부터 소멸될 때까지 불변식을 만족시켜야 한다. protected 인스턴스 변수를 가진 부모 클래스의 불변성은 자식 클래스에 의해 언제라도 쉽게 무너질 수 있따. 모든 인스턴스 변수의 가시성은 private으로 제한돼야 한다. 그렇다면 자식 클래스에서 인스턴스 변수의 상태를 변경하고 싶다면 어떻게 해야 할까? 부모 클래스에 protected 메서드를 제공하고 이 메서드를 통해 불변식을 체크하게 해야 한다.  

```java
public abstract class AdditionalRatePolicy implements RatePolicy {
    private RatePolicy next;
    
    public AdditionalRatePolicy(RatePolicy next) {
        changeNext(next);
    }
    
    protected void changeNext(RatePolicy next) {
        this.next = next;
        // 불변식
        assert next != null;
    }
}
```

<br/>

### 가변성 규칙

서브타입은 슈퍼타입이 발생시키는 예외와 다른 타입의 예외를 발생시켜서는 안 된다.  

서브타입의 리턴 타입은 공변성을 가져야 한다.  

* **공변성** : S와 T 사이의 서브타입 관계가 그대로 유지된다. 이 경우 해당 위치에서 서브타입인 S가 슈퍼타입인 T 대신 사용될 수 있다. 			  리스코프 치환 원칙이라고 생각하면 된다.
* **반공변성** : S와 T 사이의 서브타입 관계가 역전된다. 이 경우 해당 위치에서 슈퍼타입인 T가 서브타입인 S 대신 사용할 수있다.  
* **무공변성** : S와 T아이에는 아무런 관계도 존재하지 않는다. 

지금까지 살펴본 서브타이핑은 단순히 서브타입이 슈퍼타입의 모든 위치에서 대체 가능하다는 것이다.  
그러나 공변성과 반공변성의 영역으로 들어서기 위해서는 타입의 관계가 아니라 메서드의 리턴 타입과 파라미터 타입에 초점을 맞춰야 한다. 리턴 타입 공변성이란 메서드를 구현한 클래스의 타입 계층 방향과 리턴 타입의 타입 계층 방향이 동일한 경우를 가리킨다.

부모 클래스에서 구현된 메서드를 자식 클래스에서 오버라이딩할 때 파라미터 타입을 부모 클래스에서 사용한 파라미터의 슈퍼타입을 ㅗ지정할 수 있는 특성을 **파라미터 타입 반공변성**이라고 부른다. (자바는 지원하지 않는다)  

***