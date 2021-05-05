# 타입 계층의 구현
이번 장에서 염두에 두어야 할 2가지 사항
* 타입 계층은 동일한 메세지에 대한 행동 호환성을 전제로 하기 때문에 이번 장에서 언급하는 모든 방법은 타입 계층을 구현하는 방법인 동시에 다형성을 구현하는 방법이기도 하다.
* 이번 장에서 제시하는 방법을 이용해 타입과 타입 계층을 구현한다고 해서 서브타이핑 관계가 보장되는 것은 아니다. 리스코프 치환 원칙을 준수해야 한다.

서브클래싱: 코드를 재사용할 목적으로 상속하는 경우. 구현 상속/클래스 상속이라고도 부름. 자식 클래스의 인스턴스가 부모 클래스 인스턴스 대체 불가  
서브타이핑: 타입계층을 구성하기 위해 상속을 사용하는 경우, 인터페이스 상속이라고도 부름. 자식 클래스의 인스턴스가 부모 클래스 인스턴스 대체 가능

***
## 클래스를 이용한 타입 계층 구현
타입은 객체의 퍼블릭 인터페이스를 가리키기 때문에 클래스는 객체의 타입과 구현을 동시에 정의하는 것과 같다.  
클래스를 **사용자 정의 타입** 이라고 부르는 이유다.  

```java
public class Phone{
    private Money amount;
    private Duration seconds;
    private List<Call> calls = new ArrayList<>();

    public Phone(Money amount, Duration seconds) {
        this.amount = amount;
        this.seconds = seconds;
    }
    public Money calculateFee(){
        Money result = Money.ZERO;

        for (Call call : calls) {
            result = result.plus(amount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
        }
        return result;
    }
}
```
다음은 10장에서 구현한 Phone 클래스이다.  
calculateFee 메서드가 Phone의 퍼블릭 인터페이스를 구성한다.  
타입은 퍼블릭 인터페이스를 의미하기 때문에 Phone 클래스는 Phone 타입을 구현한다고 말할 수 있다.  
Phone은 calculateFee 메세지에 응답할 수 있는 타입을 선언하는 동시에 객체 구현을 정의하고 있는 것이다.  

Phone과 퍼블릭 인터페이스는 동일하지만 다른 방식으로 구현해야 하는 객체가 필요한 경우,  
즉 구현은 다르지만 Phone과 동일한 타입으로 분류되는 객체가 필요할 경우,  
**퍼블릭 인터페이스는 유지하면서 새로운 구현을 가진 객체를 추가할 수 있는 가장 간단한 방법은 상속이다.**  

```java
public class NightlyDiscountPhone extends Phone{
    private static final int LATE_NIGHT_HOUR=22;
    
    private Money nightlyAmount;

    public NightlyDiscountPhone(Money nightlyAmount, Money regularAmount, Duration seconds) {
        super(regularAmount, seconds);
        this.nightlyAmount = nightlyAmount;
    }
    
    @Override
    public Money calculateFee(){ ... }
}
```
상속을 이용하면 타입 계층을 쉽게 구현할 수 있지만, **자식 클래스를 부모 클래스의 구현에 강하게 결합**시키기 때문에 지양해야한다.  
가급적 추상 클래스를 상속받거나 인터페이스를 구현하는 방법을 지향해야한다.  

***
## 인터페이스를 이용한 타입 계층 구현  
![capture1](https://user-images.githubusercontent.com/45073750/95649152-0884b400-0b17-11eb-806a-f0895e964c52.PNG)
다음과 같은 구조를 구현하고자 할 때에, Explosion은 Displayable 과 Effect 클래스를 동시에 상속받을 수 없다.  
받을 수 있었다고 해도 결합으로 인한 문제가 생겼을 것이다. 이때에는 인터페이스를 사용한다.  

```java
public interface GameObject{
    String getName();
}

public interface Displayable extends GameObject{
    Point getPosition();
    void update(Graphics graphics);
}

public interface Collidable extends Displayable{
    boolean collideWith(Collidable other);
}

public class Player implements Collidable{
    @Override
    public String getName() {...}

    @Override
    public Point getPosition() {...}

    @Override
    public void update(Graphics graphics) {...}

    @Override
    public boolean collideWith(Collidable other) {...}
}

public class Explosion implements Displayable, Effect{...}
```
* 여러 클래스가 동일한 타입을 구현할 수 있다.
* 하나의 클래스가 여러 타입을 구현할 수 있다.

타입은 동일한 퍼블릭 인터페이스를 가진 객체들의 범주,  
클래스는 타입에 속하는 객체들을 구현하기 위한 구현 메커니즘  

객체지향에서 중요한 것은 협력 안에서 객체가 제공하는 행동  
따라서 중요한 것은 클래스 자체가 아니라 **타입**  

***
## 추상 클래스를 이용한 타입 계층 구현
클래스 상속을 이용해 구현을 공유하면서도 결합도로 인한 부작용을 피하는 방법으로, **추상 클래스**를 이용하는 방법이 있다.  

```java
public abstract class DiscountPolicy{
    private List<DiscountCondition> conditions = new ArrayList<>();

    public DiscountPolicy(DiscountCondition... conditions) {
        this.conditions=Arrays.asList(conditions);
    }

    public Money calculateDiscountAmount(Screening screening) {
        for (DiscountCondition each : conditions) {
            if(each.isSatisfiedBy(screening)){
                return getDiscountAMount(screening);
            }
        }
        return screening.getMovieFee();
    }
    
    abstract protected Money getDiscountAMount(Screening screening);
}

public class AmountDiscountPolicy extends DiscountPolicy{
    @Override
    protected Money getDiscountAmount(Screening screening) {
        return discountAmount;
    }
}

public class PercentDiscountPolicy extends DiscountPolicy{
    @Override
    protected Money getDiscountAmount(Screening screening) {
        return screening.getMovieFee().times(percent);
    }
}
```
구체 클래스를 상속받는 방법과의 차이점  
1. **추상화 정도가 다르다.**  
앞의 예제인 Phone 에서는 calculateFee 메서드가 구체적인 내부 구현에 강하게 결합되어 Phone의 내부 구현 변경에 따라 자식 클래스인 NightlyDiscountPhone도 함께 변경할 가능성이 높다. 반면, 위의 경우 내부 구현이 아닌 추상메서드의 시그니처에만 의존한다. 자식 클래스는 DiscountPolicy의 내부구현을 알 필요가 없다.
getDiscountAmount 메서드를 오버라이딩해야 한다는 사실만 알면 된다. 부모,자식 클래스 모두 추상 메서드인 getDiscountAmount에 의존함으로서 **의존성 역전 원칙**을 따른다.
2. **상속을 사용하는 의도가 다르다.**  
Phone은 상속을 염두하지 않고 설계되었으며, 확장에 대한 준비가 되어있지 않다.  
DiscountPolicy는 처음부터 상속을 염두에 두고 설계되었으며, 추상 메서드를 통해 상속 계층을 쉽게 확장할 수 있게하고 결합도로 인한 부작용을 방지했다.  
***
## 추상 클래스와 인터페이스 결합하기
클래스를 이용하여 타입을 구현할 경우 앞의 예제의 Explosion과 같은 상속 문제가 발생할 수 있다.  
인터페이스만 사용할 경우에는 구현 코드를 포함시킬 수 없기 때문에 중복 코드를 제거하기가 어렵다.  
=> 인터페이스를 이용해 타입을 정의하고 특정 상속 계층에 국한된 코드를 공유할 필요가 있을 경우에는 추상 클래스를 이용해 코드 중복을 방지  

이러한 형태로 추상 클래스를 사용하는 방식을 **골격 구현 추상 클래스** 라고 부른다.  

```java
public interface DiscountPolicy{
    Money calculateDiscountAmount(Screening screening);
}

public abstract class DefaultDiscountPolicy implements DiscountPolicy {
    private List<DiscountCondition> conditions = new ArrayList<>();

    public DiscountPolicy(DiscountCondition... conditions) {
        this.conditions=Arrays.asList(conditions);
    }

    public Money calculateDiscountAmount(Screening screening) {
        for (DiscountCondition each : conditions) {
            if(each.isSatisfiedBy(screening)){
                return getDiscountAMount(screening);
            }
        }
        return screening.getMovieFee();
    }

    abstract protected Money getDiscountAMount(Screening screening);    
}

public class AmountDiscountPolicy extends DefaultDiscountPolicy{...}
public class PercentDiscountPolicy extends DefaultDiscountPolicy{...}
```
인터페이스와 추상 클래스를 함께 사용하는 방법은 추상 클래스만 사용하는 방법과 비교해 2가지 장점이 있다.  
1. 다양한 구현 방법이 필요할 경우 새로운 추상 클래스를 추가해서 쉽게 해결 가능하다.
2. 이미 부모 클래스가 존재하는 클래스라고 하더라도 인터페이스를 추가함으로써 새로운 타입으로 쉽게 확장할 수 있다.

**상속 계층에 얽매이지 않기 위해 인터페이스로 정의하고, 추상 클래스로 기본 구현을 제공해서 중복 코드를 제거하자.**  

***
## 덕 타이핑 사용하기
>어떤 새가 오리처럼 걷고, 오리처럼 헤엄치며, 오리처럼 꽥꽥 소리를 낸다면 나는 이 새를 오리라고 부를 것이다. -James Whitcom Riley  

어떤 대상의 '행동'이 오리와 같다면 그것을 오리라는 타입으로 취급해도 무방하다는 것이다.  
즉, 객체가 어떤 인터페이스에 정의된 행동을 수행할 수만 있다면 그 객체를 해당 타입으로 분류해도 문제가 없다는 것.  

자바와 같은 정적 타입 언어에서는 지원하지 않는 방법이다.(C# 제외)  
```java
public interface Employee{
    Money calculatePay(double taxRate);
}

public class SalariedEmployee{
    private String name;
    private Money basePay;
    public SalariedEmployee(...){...}
    
    public Money calculatePay(double taxRate){
        return basePay.minus(basePay.times(taxRate));
    }
}

public class HourlyEmployee{
    private String name;
    private Money basePay;
    private int timeCrd;
    public HourlyEmployee(...){...}

    public Money calculatePay(double taxRate){
        return basePay.times(timeCard).minus(basePay.times(taxRate));
    }
}
```
다음과 같은 상황에서 SalariedEmployee와 HourlyEmployee 클래스는 Employee 인터페이스에 정의된 calculatePay 오퍼레이션과 동일한 시그니처를 가진 퍼블릭 메서드를 포함하고 있다. 때문에 동일한 타입으로 취급할 수 있다고 예상할 수 있지만, 정적 타입 언어에서는 동일한 타입으로 취급하기 위해서는 동일하게 선언돼 있어야만 한다.  

인터페이스가 클래스보다 더 유연한 설계를 가능하게 해주는 이유는 클래스가 정의하는 구현이라는 컨텍스트에 독립적인 코드 작성을 가능케 해주기 때문, 덕 타이핑은 더 나아가서 메서드의 시그니처만 동일하면 명시적인 타입 선언이라는 컨텍스트를 제거할 수 있다. 클래스나 인터페이스에 대한 의존성을 메세지에 대한 의존성으로 대체한다.  

결과적으로 **코드는 낮은 결합도를 유지하고 변경에 유연하게 대응**할 수 있게된다.  

***
## 믹스인과 타입 계층
**믹스인**은 객체를 생성할 때 코드 일부를 섞어 넣을 수 있도록 만들어진 일종의 추상 서브클래스다.  
믹스인을 사용하는 목적은 다양한 객체 구현 안에서 동일한 **'행동'**을 **중복 코드 없이 재사용**할 수 있게 만드는 것이다.  

믹스인을 통해 코드를 재사용하는 객체들은 동일한 행동을 공유하게 된다. 즉 믹스인된 객체들은 동일한 퍼블릭 인터페이스를 공유하게 되는 것이고,  타입은 퍼블릭 인터페이스와 관련이 있기 때문에 대부분의 믹스인을 구현하는 기법들은 타입을 정의하는 것으로 볼 수 있다.  

스칼라 언어에서 예를 보면,
```java
trait Ordered[A] extends Any with Comparable[A]{
        def compare(that: A): Int
        def < (that: A): Boolean = (this compare that) < 0
        def > (that: A): Boolean = (this compare that) >0
        def <= (that: A): Boolean = (this compare that) <=0
        def >= (that: A): Boolean = (this compare that) >=0
        def compareTo(that: A): Int = compare(that)
}

case class Money(amount: Long) extends Ordered[Money]{
        def + (that: Money): Money = Money(this.amount + that.amount)
        def - (that: Money): Money = Money(this.amount - that.amount)
        def compare(that: Money): Int = (this.amount - that.amount).toInt
}
```
Ordered라는 트레이트를 추가하고 싶은 클래스에 믹스인 하였다.  
이제 Money는 Ordered를 요구하는 모든 위치에서 Ordered를 대체할 수 있다. 이것은 리스코프 치환 원칙을 만족하기 때문에 Money는 Ordered타입으로 분류될 수 있다.  
Money는 +,-만을 가진 간결한 인터페이스에서 풍부한 인터페이스가 되었다.  

자바에도 **디폴트 메서드**라는 것이 있다.  
인터페이스에 디폴트 메서드가 구현돼 있다면 추상 클래스를 통해 해당 인터페이스를 구현하는 방법을 사용하지 않아도 된다.  
디폴트 메서드를 사용하면 추상 클래스가 제공하는 코드 재사용성이라는 혜택을 그대로 누리면서 상속 계층에 얽매이지 않는 인터페이스의 장점을 가진다.  

```java
public interface DiscountPolicy{
    default Money calculateDiscountAmount(Screening screening){
        ofr(DiscountCondition each : getConditions()){
            if(each.isSatisfiedBy(screening)){
                return getDiscountAmount(screening);
            }
        }
        return screening.getMovieFee();
    }
    
    List<DiscountCondition> getConditions();
    Money getDiscountAmount(Screening screening);
}
```
아까 골격 구현 추상 클래스 방법을 이용했던 것을 ``default``를 이용해서 해결하였다.  
그런데 다시보면 기존의 인터페이스에는 없었던 getConditions와 getDiscountAmount가 생긴 것을 볼 수가 있다.  
디폴트 메서드에서 해당되는 2개의 메서드를 사용했기 때문이다. 이것은 클래스에서 해당 메서드의 구현을 제공해야 한다는 것을 명시한 것이다.  
문제는, 인터페이스에 정의됐기 때문에 클래스 안에서 public으로 구현해야 된다는 것이다.  
이것은 외부에 노출할 필요가 없는 메서드를 불필요하게 퍼블릭 인터페이스에 추가하는 결과를 낳게 된다.  
getConditions메서드의 경우에 외부에 DiscountConditions 목록을 공개하게 되므로 더욱 심각해진다.  
이것은 설계의 제 1원칙인 **캡슐화**를 약화시킨다.  

```java
public class AmountDiscountPolicy implements DiscountPolicy{
    private Money discountAmount;
    private List<DiscountCondition> conditions = new ArrayList<>();
    
    public AmountDiscountPolicy(Money discountAmount, DiscountCondition... conditions){
        this.discountAmount = discountAmount;
        this.conditions = conditions;
    }
    
    @Override
    public List<DiscountCondition> getConditions(){
        return conditions;
    }

    @Override
    public Money getDiscountAmount(Screening screening) {
        return discountAmount;
    }
}

public class PercentDiscountPolicy implements DiscountPolicy{
    private double percent;
    private List<DiscountCondition> conditions = new ArrayList<>();

    public PercentDiscountPolicy(Money discountAmount, DiscountCondition... conditions){
        this.percent = percent;
        this.conditions = Arrays.asList(conditions);
    }

    @Override
    public List<DiscountCondition> getConditions(){
        return conditions;
    }

    @Override
    public Money getDiscountAmount(Screening screening) {
        return screening.getMovieFee().times(percent);
    }
}
```
구현내용을 보면 두 Policy 클래스 사이의 코드 중복도 완벽하게 제거해주지 못하는 것을 볼 수가 있다.  

이러한 문제들이 존재하는 이유는 자바 8에 디폴트 메서드를 추가한 이유가 인터페이스로 추상클래스의 역할을 대체하려는 것이 아니기 때문이다.  
디폴트 메서드가 추가된 이유는 기존에 널리 사용되고 있는 인터페이스에 새로운 오퍼레이션을 추가할 경우에 발생하는 하위 호환성 문제를 해결하기 위해서지 추상 클래스를 제거하기 위한 것이 아니다. 따라서 타입을 정의하기 위해 디폴트 메서드를 사용할 생각이라면 그 한계를 명확하게 알아야 한다.  

***
