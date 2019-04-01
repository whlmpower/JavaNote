## Java 8 中的日期和时间

### 1.LocalDate 和 LocalTime

该类的实例是一个不可变对象

```java
LocalDate date = LocalDate.of(2014, 3, 31);
int year = date.getYear();
LocalDate today = LocalDate.now();
```

同样可以传递一个TemporalField参数给get方法拿到同样的信息，chronoField枚举实现了这一接口。

```Java
int year = date.get(ChronoField.YEAR);
```

```Java
LocalDate date = LocalDate.parse("2019-03-31");
```

通过LocalDate传递一个时间对象，或者向LocalTime传递一个日期对象，从而创建一个LocalDateTime对象。

也可以从LocalDateTime中提取LocalDate或者LocalTime组件

```Java
LocalDateTime dt1 = date.atTime(13, 45, 20);
LocalTime time1 = dt1.toLocalTime();
```

### 2.机器的日期和时间格式

java.time.Instant 类对时间建模的方式，以Unix元年时间经历的秒数。

```Java
Instant.ofEpochSecond(3); // 2s之后在加上100w纳秒(1s)
```

Instant类设计初衷便于机器使用。now方法返回当前时刻的时间戳。

```Java
int day = Instant.now().get(ChronoField.DAY_OF_MONTH);

```

上述代码会抛出异常，因为无法处理我们非常容易理解的时间单位。

两个Instant对象之间的duration。

```Java
Duration d1 = Duration.between(time1, time2);
Duration d1 = Duration.between(dateTime1, dateTime2);
Duration d1 = Duration.between(instant1, instant2);
```

LocalDateTime 和Instant是为了不同目的设计的，一个是为了便于阅读使用，一个是为了便于机器处理。两类之间不能创建Duration，否则会触发DateTimeException异常。Duration以秒和纳秒来衡量时间的长短，不能仅想between方法传递一个LocalDate对象做参数。如果需要以年 月  日的方式对多个时间单位建模，可以使用Period类。

```Java
Period tenDays = Period.between(LocalDate.of(2014, 3, 8), LocalDate.of(2018,3,5));
```

![1554039163728](.\image\%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1554039163728.png)

### 3.操纵、解析和格式化日期

创建它的一个修改版，withAttribute方法会创建对象的一个副本，并按照需要进行属性修改。

```Java
LocalDate date1 = LocalDate.of(2014, 3, 18);
LocalDate date2 = date1.withYear(2011);
```

使用get 和 with方法，可以将Temporal对象值的读取和修改分开。

声明式的访问方式操纵LocalDate对象

```Java
LocalDate date1 = LocalDate.of(2014,3,5);
LocalDate date2 = date1.plusWeeks(1);
LocalDate date4 = date1.plus(6, ChronoUnit.MONTHS);
```

表示时间点的日期时间类提供了大量通用的方法

![1554039755535](.\image\%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1554039755535.png)

### 使用TemporalAdjuster

复杂操纵，比如将日期调整到下个周日，下个工作日。可以选择重载版本的with方法，向其传递一个提供了更多定制化选择的TemporalAdjuster对象，更加灵活地处理日期。

```Java
LocalDate date1 = LocalDate.of(2018,2,3);
LocalDate date2 = date1.with(nextOrSame(DayOfWeek.SUNDAY));
```

![1554040025942](.\image\%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5C1554040025942.png)

自定义Temporal-Adjuster 

### 4.解析日期时间对象

```Java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
LocalDate date1 = LocalDate.of(2014, 3, 18);
String formattedDate = date1.format(formatter);
LocalDate date2 = localDate.parse(formattedDate, formatter);
```

### 5.不同时区

一旦得到了ZoneId对象，可以将它与LocalDate、LocalDateTime或者Instant对象整合起来，构造出ZoneDateTime实例。

```Java
ZoneId romeZone = ZoneId.of("Europe/Rome");
LocalDate date = LocalDate.of(2014, Month.MARCH, 18);
ZoneDateTime zdt1 = date.atStartOfDay(romeZone);

LocalDateTime dateTime = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45);
ZonedDateTime zdt2 = date.atZone(romeZone);
Instant instantFromDateTime = dateTime.toInstant(romeZone);//转换为Instant

Instant instant = Instant.now();
LocalDateTime timeFromInstant = LocalDateTime.ofInstant(instant, romeZone);//转换为LocalDateTime

Instant instant = Instant.now();
ZonedDateTime zdt3 = instant.atZone(romeZone);
```

固定偏差计算时区

```Java
LocalDateTime dateTime = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45);
ZoneOffset newYorkset = ZoneOffset.of("-05:00");
OffsetDateTime dateTimeInNewYork = offsetDateTime.of(dateTime ,newYorkOffset);
```

