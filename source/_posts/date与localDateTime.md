---
title: date与localDateTime
comments: false
toc: false
date: 2019-07-30 10:25:59
categories: java
tags:
---

Java 8中 `java.util.Date` 类新增了两个方法，分别是from(Instant instant)和toInstant()方法

```java
// Obtains an instance of Date from an Instant object.
public static Date from(Instant instant) {
    try {
        return new Date(instant.toEpochMilli());
    } catch (ArithmeticException ex) {
        throw new IllegalArgumentException(ex);
    }
}
// Converts this Date object to an Instant.
public Instant toInstant() {
    return Instant.ofEpochMilli(getTime());
}
```

通过Instant当媒介，然后通过Instant来使Date与LocalDateTime互相转换
<!--more-->

## java.util.Date --> java.time.LocalDateTime

```java
public void UDateToLocalDateTime() {
    java.util.Date date = new java.util.Date();
    Instant instant = date.toInstant();
    ZoneId zone = ZoneId.systemDefault();
    LocalDateTime localDateTime = LocalDateTime.ofInstant(instant, zone);
    // LocalDate localDate = localDateTime.toLocalDate();
    // LocalTime localTime = localDateTime.toLocalTime();
}
```

## java.time.LocalDateTime --> java.util.Date

```java

public void LocalDateTimeToUdate(LocalDateTime localDateTime) {
    // LocalDateTime localDateTime = LocalDateTime.now();
    ZoneId zone = ZoneId.systemDefault();
    Instant instant = localDateTime.atZone(zone).toInstant();
    java.util.Date date = Date.from(instant);
}

public void LocalDateToUdate(LocalDate localDate) {
    // LocalDate localDate = LocalDate.now();
    ZoneId zone = ZoneId.systemDefault();
    Instant instant = localDate.atStartOfDay().atZone(zone).toInstant();
    java.util.Date date = Date.from(instant);
}

public void LocalTimeToUdate(LocalTime localTime) {
    // LocalTime localTime = LocalTime.now();
    LocalDate localDate = LocalDate.now();
    LocalDateTime localDateTime = LocalDateTime.of(localDate, localTime);
    ZoneId zone = ZoneId.systemDefault();
    Instant instant = localDateTime.atZone(zone).toInstant();
    java.util.Date date = Date.from(instant);
}
```

## java.util.Date <--> long

```java
public void DateToLong(Object obj){
    long l,int i;
    if(obj instanceof java.sql.Date) l = ((java.sql.Date)obj).getTime();
    if(obj instanceof java.sql.Time) l = ((java.sql.Time)obj).getTime();
    if(obj instanceof java.sql.Timestamp) {
        l = ((java.sql.Timestamp)obj).toInstant().getEpochSecond();
        i = ((java.sql.Timestamp)obj).toInstant().getNano();
    }
}

public void LongToDate(long l,int i){
    java.sql.Date date = new java.sql.Date(l);
    java.sql.Time time = new java.sql.Time(l);
    java.sql.Timestamp timestamp = Timestamp.from(Instant.ofEpochSecond(l, i));
}
```
