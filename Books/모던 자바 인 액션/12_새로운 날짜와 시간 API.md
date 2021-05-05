# 새로운 날짜와 시간 API

## LocalDate, LocalTime, Instant, Duration, Period 클래스

### LocalDate와 LocalTime 사용

```java
        LocalDate date = LocalDate.of(2017, 9, 21);//2017-09-21
        int year = date.getYear();//2017
        Month month = date.getMonth();//SEPTEMBER
        int day = date.getDayOfMonth();//21
        DayOfWeek dow = date.getDayOfWeek();//THURSDAY
        int len = date.lengthOfMonth();//31
        boolean leap = date.isLeapYear();//false(윤년이 아님)
```

팩토리 메서드 ``now``는 시스템 시계의 정보를 이용해서 현재 날짜 정보를 얻는다.  

``LocalDate today = Lodate.now();``  

get 메서드에 TemporalField를 전달해서 정보를 얻는 방법도 있다. TemporalField는 시간 관련 객체에서 어떤 필드의 값에 접근할지 정의하는 인터페이스다. 열거자 ChronoField는 TemporalField 인터페이스를 정의하므로 다음 코드에서 보여주는 것처럼 ChronoField의 열거자 요소를 이용해서 워하는 정보를 쉽게 얻을 수 있다.  

```java
        int year = date.get(ChronoField.YEAR);//2017
        int month = date.get(ChronoField.MONTH_OF_YEAR);//9
        int day = date.get(ChronoField.DAY_OF_MONTH);//21
```

이 코드들은 내장 메서드 ``getYear();``, ``getMonthValue();``, ``getDayOfMonth();`` 로 가독성을 높일 수 있다.  

13:45:20 같은 시간은 LocalTime 클래스로 표현할 수 있다.  

```java
        LocalTime time = LocalTime.of(13, 45, 20);
        int hour = time.getHour();//13
        int minute = time.getMinute();//45
        int second = time.getSecond();//20
```

날짜와 시간 문자열로 LocalDate와 LocalTime의 인스턴스를 만드는 방법도 있다.  

```java
        LocalDate date = LocalDate.parse("2017-09-21");
        LocalTime time = LocalTime.parse("13:45:20");
```

### 날짜와 시간 조합

LocalDateTime은 LocalDate와 LocalTime을 쌍으로 갖는 복합 클래스다.  

```java
        LocalDateTime.of(2017, 9, 21, 13, 45, 20);	//숫자로 해도 되고 밑에처럼 해도 됨
        LocalDateTime dt1 = LocalDateTime.of(2017, Month.SEPTEMBER, 21, 13, 45, 20);
        LocalDateTime dt2 = LocalDateTime.of(date, time);
        LocalDateTime dt3 = date.atTime(13, 45, 20);
        LocalDateTime dt4 = date.atTime(time);
        LocalDateTime dt5 = time.atDate(date);
```

LocalDateTime에서 ``toLocalDate``나 ``toLocalTime`` 메서드로 LocalDate나 LocalTime 인스턴스를 추출할 수 있다.  

```java
LocalDate date1 = dt1.toLocalDate();//2017-09-21
LocalTime time1 = dt2.toLocalTime();//13:45:20
```

### Instant 클래스 : 기계의 날짜와 시간

사람은 보통 주, 날짜, 시간, 분으로 날짜와 시간을 계산한다. 하지만 기계에서는 이와 같은 단위로 시간을 표현하기가 어렵다.  
기계의 관점에서는 연속된 시간에서 특정 지점을 하나의 큰 수로 표현하는 것이 가장 자연스러운 시간 표현 방법이다.  
``java.time.Instant`` 클래스에서는 이와 같은 기계적인 관점에서 시간을 표현한다. 즉, Instant 클래스는 유닉스 에포크 시간을 기준으로 특정 지점까지의 시간을 초로 표현한다.  

팩토리 메서드 ``ofEpochSecond``에 초를 넘겨줘서 Instant 클래스 인스턴스를 만들 수 있다. Instant 클래스는 나노초(10억분의 1초)의 정밀도를 제공한다. 또한 오버로드된 ``ofEpochSecond`` 메서드 버전에서는 두 번째 인수를 이용해서 나노초 단위로 시간을 보정할 수 있다.  

```java
        Instant instant = Instant.ofEpochSecond(3);
        Instant instant1 = Instant.ofEpochSecond(3, 0);
        Instant.ofEpochSecond(2, 1_000_000_000);//2초 이후의 1억 나노초(1초)
        Instant.ofEpochSecond(4, -1_000_000_000);//4초 이전의 1억 나노초(1초)
```

Instant 클래스도 사람이 확인할 수 있도록 시간을 표시해주는 정적 팩토리 메서드 ``now``를 제공한다.  
하지만 Instant는 기계 전용의 유틸리티다. 초와 나노초 정보를 포함한다. 사람이 읽을 수 있는 시간 정보를 제공하지 않는다.  

``int day = Instant.now().get(ChronoField.DAY_OF_MONTH);``  
위 코드는 ``java.time.temporal.UnsupportedTemporalTypeException : Unsupported field: DayOfMonth``  
다음과 같은 예외를 일으킨다.  

### Duration과 Period 정의

```java
        Duration d1 = Duration.between(time1, time2);
        Duration d2 = Duration.between(dateTime1, dateTime2);
        Duration d3 = Duration.between(instant1, instan2);

        LocalDateTime dateTime1 = LocalDateTime.of(2021, 1, 25, 13, 29);
        LocalDateTime dateTime2 = LocalDateTime.of(2021, 1, 26, 13, 30, 30);
        Duration durationDate = Duration.between(dateTime1, dateTime2);
        System.out.println(durationDate);
//PT24H1M30S
```

각 2개의 LocalTime, LocalDateTime, Instant로 Duration을 만들 수 있다. (LocalDate로는 안됨)  
LocalDateTime은 사람, Instant는 기계가 사용하도록 만들어진 클래스로 두 인스턴스는 서로 혼합될 수 없다.  

LocalDate을 표현할 때에는 Period를 이용한다.  

```java
        Period period = Period.between(
                LocalDate.of(2021, 1, 1), LocalDate.of(2021, 1, 24));
        System.out.println(period);
//P23D
```

```java
        Duration threeMinutes = Duration.ofMinutes(3);//PT3M
        //Duration.of(3, ChronoUnit.MINUTES);

        Period tenDays = Period.ofDays(10);//P10D
        Period threeWeeks = Period.ofWeeks(3);//P21D
        Period twoYearsSizMonthsOneDay = Period.of(2, 6, 1);//P2Y6M1D
```

위와 같이 두 시간 객체를 사용하지 않고도 Duration과 Period 클래스를 만들 수 있다.  

| 메서드       | 정적   | 설명                                                         |
| ------------ | ------ | ------------------------------------------------------------ |
| between      | 네     | 두 시간 사이의 간격을 생성함                                 |
| from         | 네     | 시간 단우로 간격을 생성함                                    |
| of           | 네     | 주어진 구성 요소에서 간격 인스턴스를 생성함                  |
| parse        | 네     | 문자열을 파싱해서 간격 인스턴스를 생성함                     |
| addTo        | 아니오 | 현재값의 복사본을 생성한 다음에 지정된 Temporal 객체에 추가함 |
| get          | 아니오 | 현재 간격 정보값을 읽음                                      |
| isNegative   | 아니오 | 간격이 음수인지 확인함                                       |
| isZero       | 아니오 | 간격이 0인지 확인함                                          |
| minus        | 아니오 | 현재값에서 주어진 시간을 뺀 복사본을 생성함                  |
| multipliedBy | 아니오 | 현재값에 주어진 값을 곱한 복사본을 생성함                    |
| pneagated    | 아니오 | 주어진 값의 부호를 반전한 복사본을 생성함                    |
| plus         | 아니오 | 현재값에 주어진 시간을 더한 복사본을 생성함                  |
| subtractFrom | 아니오 | 지정된 Temporal 객체에서 간격을 뺌                           |

위에서 살펴본 모든 클래스는 불변이다.  

<br/>

## 날짜 조정, 파싱, 포매팅

``withAttribute`` 메서드로 기존의 LocalDate를 바꾼 버전을 직접 간단하게 만들 수 있다.  

절대적인 방식으로 LocalDate의 속성을 바꾸는 방법은 아래와 같다.

```java
        LocalDate date1 = LocalDate.of(2017, 9, 21);//2017-09-21
        LocalDate date2 = date1.withYear(2011);//2011-09-21
        LocalDate date3 = date2.withDayOfMonth(25);//2011-09-25
        LocalDate date4 = date3.with(ChronoField.MONTH_OF_YEAR, 2);//2011-02-25, withMonth()로도 가능
```

Temporal 인터페이스(날짜와 시간 API의 모든 클래스가 구현한다) 는 LocalDate, LocalTime, LocalDateTime, Instant 처럼 특정 시간을 정의한다. get과 with 메서드로 Temporal 객체의 필드값을 읽거나 고칠 수 있다. 어떤 Temporal 객체가 지정된 필드를 지원하지 않으면 ``UnsupportedTemporalTypeException``이 발생한다.

상대적인 방식으로 LocalDate의 속성을 바꾸는 방법은 아래와 같다.  

```java
        LocalDate date1 = LocalDate.of(2017, 9, 21);//2017-09-21
        LocalDate date2 = date1.plusWeeks(1);//2017-09-28
        LocalDate date3 = date2.minusYears(6);//2011-09-28
        LocalDate date4 = date3.plus(6, ChronoUnit.MONTHS);//2012-03-28
```

특정 시점을 표현하는 날짜 시간 클래스의 공통 메서드는 다음과 같다.  

| 메서드   | 정적   | 설명                                                         |
| -------- | ------ | ------------------------------------------------------------ |
| from     | 예     | 주어진 Temporal 객체를 이용해서 클래스의 인스턴스를 생성함   |
| now      | 예     | 시스템 기계로 Temporal 객체를 생성함                         |
| of       | 예     | 주어진 구성 요소에서 Temporal 객체의 인스턴스를 생성함       |
| parse    | 예     | 문자열을 파싱해서 Temporal 객체를 생성함                     |
| atOffset | 아니오 | 시간대 오프셋과 Temporal 객체를 합침                         |
| atZone   | 아니오 | 시간대 오프셋과 Temporal 객체를 합침                         |
| format   | 아니오 | 지정된 포매터를 이용해서 Temporal 객체를 문자열로 변환함(Instant는 지원하지 않음) |
| get      | 아니오 | Temporal 객체의 상태를 읽음                                  |
| minus    | 아니오 | 특정 시간을 뺀 Temporal 객체의 복사본을 생성함               |
| plus     | 아니오 | 특정 시간을 더한 Temporal 객체의 복사본을 생성함             |
| with     | 아니오 | 일부 상태를 바꾼 Temporal 객체의 복사본을 생성함             |

### TemporalAdjusters 사용하기

때로는 다음 주 일요일, 돌아오는 평일, 어떤 달의 마지막 날 등 좀 더 복잡한 날짜 조정 기능이 필요할 때가 있다.  
이때는 오버로드된 버전의 with 메서드에 좀 더 다양한 동작을 수행할 수 있도록 하는 기능을 ``TemporalAdjusters``를 전달하는 방법으로 문제를 해결할 수 있다.  

```java
        LocalDate date1 = LocalDate.of(2021, 1, 26);//2021-01-26
        LocalDate date2 = date1.with(nextOrSame(DayOfWeek.SUNDAY));//2021-01-31
        LocalDate date3 = date2.with(lastDayOfMonth());//2021-01-31
```

TemporalAdjuster 클래스의 팩토리 메서드는 다음과 같다.  

| 메서드              | 설명                                                         |
| ------------------- | ------------------------------------------------------------ |
| dayOfWeekInMonth    | 서수 요일에 해당하는 날짜를 반환하는 TemporalAdjuster를 반환함(몇 째주, DayOfWeek 을 인수로 받음) |
| firstDayOfMonth     | 현재 달의 첫 번째 날짜를 반환하는 TemporalAdjuster를 반환함  |
| firstDayOfNextMonth | 다음 달의 첫 번째 날짜를 반환하는 TemporalAdjuster를 반환함  |
| firstDayOfNextYear  | 내년의 첫 번째 날짜를 반환하는 TemporalAdjuster를 반환함     |
| firstDayOfYear      | 올해의 첫 번째 날짜를 반환하는 TemporalAdjuster를 반환함     |
| firstInMonth        | 현재 달의 첫 번째 요일에 해당하는 날짜를 반환하는 TemporalAdjuster를 반환함(DayOfWeek를 인수로 받음) |
| lastDayOfMonth      | 현재 달의 마지막 날짜를 반환하는 TemporalAdjuster를 반환함   |
| lasyDayOfNextMonth  | 다음 달의 마지막 날짜를 반환하는 TemporalAdjuster를 반환함   |
| lastDayOfNextYear   | 내년의 마지막 날짜를 반환하는 TemporalAdjuster를 반환함      |
| lastDayOfYear       | 올해의 마지막 날짜를 반환하는 TemporalAdjuster를 반환함      |
| lastInMonth         | 현재 달의 마지막 요일에 해당하는 날짜를 반환하는 TemporalAdjuster를 반환함(DayOfWeek를 인수로 받음) |
| next previous       | 현재 달에서 현재 날짜 이후로 지정한 요일이 처음으로 나타나는 날짜를 반환하는 TemporalAdjuster를 반환함(DayOfWeek를 인수로 받음) |
| nextOrSame          | 현재 날짜 이후로 지정한 요일이 처음/이전으로 나타나는 날짜를 반환하는 TemporalAdjuster를 반환함(현재 날짜도 포함, DayOfWeek를 인수로 받음) |
| previousOrSame      | 현재 날짜 이전에 지정한 요일이 처음/이전으로 나타나는 날짜를 반화하는 TemporalAdjuster를 반환함(현재 날짜도 포함, DayOfWeek를 인수로 받음) |

TemporalAdjuster 인터페이스는 하나의 메서드만 정의한 함수형 인터페이스이므로 쉽게 커스텀 TemporalAdjuster 구현을 만들 수 있다.  

```java
@FunctionalInterface
public interface TemporalAdjuster {
    Temporal adjustInto(Temporal temporal);
}
```

날짜를 하루씩 다음날로 바꾸는 이때 토요일과 일요일은 건너뛰는 코드를 구현하고자 한다.  

```java
public class NextWorkingDay implements TemporalAdjuster {
    @Override
    public Temporal adjustInto(Temporal temporal) {
        //temporal.get(ChronoField.DAY_OF_WEEK)로 월~금을 1~7로 받아옴
        DayOfWeek dow = DayOfWeek.of(temporal.get(ChronoField.DAY_OF_WEEK));
        int dayToAdd = 1;
        if(dow == DayOfWeek.FRIDAY) dayToAdd = 3;
        else if (dow == DayOfWeek.SATURDAY) dayToAdd = 2;
        return temporal.plus(dayToAdd, ChronoUnit.DAYS);
    }
}
```

람다 표현식으로 바꿀 수도 있다.  

```java
date = date.with(temporal -> {
            DayOfWeek dow = DayOfWeek.of(temporal.get(ChronoField.DAY_OF_WEEK));
        int dayToAdd = 1;
        if(dow == DayOfWeek.FRIDAY) dayToAdd = 3;
        else if (dow == DayOfWeek.SATURDAY) dayToAdd = 2;
        return temporal.plus(dayToAdd, ChronoUnits.DAYS);
});
```

만일 TemporalAdjuster를 람다 표현식으로 정의하고 싶다면 ``UnaryOperator<LocalDate>``를 인수로 받는 TemporalAdjuster 클래스의 정적 팩토리 메서드 ``ofDateAdjuster``를 사용하는 것이 좋다.  

```java
        TemporalAdjuster nextWorkingDay = TemporalAdjusters.ofDateAdjuster(
                temporal -> {
                    DayOfWeek dow = DayOfWeek.of(temporal.get(ChronoField.DAY_OF_WEEK));
                    int dayToAdd = 1;
                    if(dow == DayOfWeek.FRIDAY) dayToAdd = 3;
                    else if (dow == DayOfWeek.SATURDAY) dayToAdd = 2;
                    return temporal.plus(dayToAdd, ChronoUnit.DAYS);
                });
        date = date.with(nextWorkingDay);
```

### 날짜와 시간 객체 출력과 파싱

```java
        LocalDate date = LocalDate.of(2021, 1, 26);
        String s1 = date.format(DateTimeFormatter.BASIC_ISO_DATE);//20210126
        String s2 = date.format(DateTimeFormatter.ISO_LOCAL_DATE);//2021-01-26

		//다음과 같이 문자열을 파싱해서 만들 수도 있다.
        LocalDate date1 = LocalDate.parse("20210126", DateTimeFormatter.BASIC_ISO_DATE);
        LocalDate date2 = LocalDate.parse("2021-01-26", DateTimeFormatter.ISO_LOCAL_DATE);
```

DateTimeFormatter 클래스는 특정 패턴으로 포매터를 만들 수 있는 정적 팩토리 메서드도 제공한다.  

```java
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
        LocalDate date1 = LocalDate.of(2021, 1, 26);//2021-01-26
        String formattedDate = date1.format(formatter);//26/01/2021
        LocalDate date2 = LocalDate.parse(formattedDate, formatter);//2021-01-26
```

LocalDate의 format 메서드는 요청 형식의 패턴에 해당하는 문자열을 생성한다. 그리고 정적 메서드 parse는 같은 포매터를 적용해서 생성된 문자열을 파싱함으로써 다시 날짜를 생성한다. 다음 코드에서 보여주는 것처럼 ``ofPattern``메서드도 Locale로 포매터를 만들 수 있도록 오버로드된 메서드를 제공한다.  

```java
        DateTimeFormatter italianFormatter = DateTimeFormatter.ofPattern("d. MMMM yyyy", Locale.ITALIAN);
        LocalDate date1 = LocalDate.of(2021, 1, 26);//2021-01-26
        String formattedDate = date1.format(italianFormatter);//26. gennaio 2021
        LocalDate date2 = LocalDate.parse(formattedDate, italianFormatter);//2021-01-26
```

``DateTimeFormatterBuilder`` 클래스로 복합적인 포매터를 정의해서 좀 더 세부적으로 포매터를 제어할 수 있다.  
즉, DateTimeFormatterBuilder 클래스로대소문자를 구분하는 파싱, 관대한 규칙을 적용하는 파싱, 해딩, 포매터의 선택사항 등을 활용할 수 있다.  

```java
        DateTimeFormatter italianFormatter = new DateTimeFormatterBuilder()
                .appendText(ChronoField.DAY_OF_MONTH)
                .appendLiteral(". ")
                .appendText(ChronoField.MONTH_OF_YEAR)
                .appendLiteral(" ")
                .appendText(ChronoField.YEAR)
                .parseCaseInsensitive()
                .toFormatter(Locale.ITALIAN);
```

<br/>

## 다양한 시간대와 캘린더 활용 방법

새로운 날짜와 시간 API의 큰 편리함 중 하나는 시간대를 간단하게 처리할 수 있다는 점이다.  
기존의 java.util.TimeZone을 대체할 수 있는 ``java.time.ZoneId`` 클래스가 새롭게 등장했다. 새로운 클래스를 이용하면 서머타임같은 복잡한 사항이 자동으로 처리된다. 날짜와 시간 API에서 제공하는 다른 클래스와 마찬가지로 ZoneId는 불변 클래스다.  

### 시간대 사용하기

표준 시간이 같은 지역을 묶어서 **시간대** 규칙 집합을 정의한다. ZoneRules 클래스에는 약 40개 정도의 시간대가 있다. ZoneId의 getRules()를 이용해서 해당 시간대의 규정을 획득할 수 있다. 다음처럼 지역 ID로 특정 ZoneId를 구분한다.  

```java
ZoneId romeZone = ZoneId.of("Europe/Rome");
```

지역 ID는 '{지역}/{도시}' 형식으로 이루어지며 IANA Time Zone Database에서 제공하는 지역 집합 정보를 사용한다.  
다음 코드에서 보여주는 것처럼 ZoneId의 새로우 메서드인 toZoneId로 기존의 TimeZone 객체를 ZoneId 객체로 변환할 수 있다.  

```java
ZoneId zoneId = TImeZonde.getDefault().toZoneId();
```

ZoneId 객체를 얻은 다음에는 LocalDate, LocalDateTime, Instant를 이용해서 ZoneDateTime 인스턴스로 변환할 수 있다.  
ZonedDateTime은 지정한 시간대에 상대적인 시점을 표현한다.  

```java
        ZoneId romeZone = ZoneId.of("Europe/Rome");

        LocalDate date = LocalDate.of(2021, 1, 26);
        ZonedDateTime zdt1 = date.atStartOfDay(romeZone);
        LocalDateTime dateTime = LocalDateTime.of(2021, 1, 26, 18, 13, 45);
        ZonedDateTime zdt2 = dateTime.atZone(romeZone);
        Instant instant = Instant.now();
        ZonedDateTime zdt3 = instant.atZone(romeZone);
        System.out.println(zdt1);
        System.out.println(zdt2);
        System.out.println(zdt3);
/*
2021-01-26T00:00+01:00[Europe/Rome]
2021-01-26T18:13:45+01:00[Europe/Rome]
2021-01-26T10:06:08.909849800+01:00[Europe/Rome]
*/
```

LocalDate + LocalTime = LocalDateTime  
LocalDateTime + ZoneId = ZonedDateTime 이라고 생각하면 된다.  

### UTC/Greenwich 기준의 고정 오프셋

때로는 UTC(협정 세계시)/GMT(그리니치 표준시)를 기준으로 시간대를 표현하기도 한다. 예를 들어 '뉴욕은 런던보다 5시간 느리다'라고 표현할 수 있다. ZoneId의 서브클래스인 ZoneOffset클래스로 런던의 그리니치 0도 자오선과 시간값의 차이를 표현할 수 있다.  

```java
ZoneOffset newYorkOffset = ZoneOffset.of("-05:00");
```

실제로 미국 동부 표준시의 오프셋값은 -05:00 이다. 하지만 위 예제에서 정의한 ZoneOffset으로는 서머타임을 제대로 처리할 수 없으므로 권장하지 않는 방식이다. ZoneOffset은 ZoneId이므로 위의 romeZone 예제처럼 ZoneOffset을 사용할 수 있다. 또한 ISO-8601 캘린더 시스템에서 정의하는 UTC/GMT와 오프셋으로 날짜와 시간을 표현하는 OffsetDateTime을 만드는 방법도 있다.  

```java
LocalDateTime dateTime = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45);
OffsetDateTime dateTimeInNewYork = OffsetDateTime.of(date, newYorkOffset);
```

***