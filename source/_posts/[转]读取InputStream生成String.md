---
title: '[转]读取InputStream生成String'
comments: false
toc: false
date: 2019-07-30 13:47:54
categories: java
tags:
---

## 1,使用IOUtils.toString（Apache Utils）

``` java
String result = IOUtils.toString(inputStream, StandardCharsets.UTF_8);
```

## 2,使用CharStreams（guava）

``` java
String result = CharStreams.toString(new InputStreamReader(inputStream, Charsets.UTF_8));
```

## 3,使用Scanner（JDK）

``` java
Scanner s = new Scanner(inputStream).useDelimiter("\A");
String result = s.hasNext() ? s.next() : "";
```

## 4,5 使用Stream Api（Java 8）

``` java
String result = new BufferedReader(new InputStreamReader(inputStream)).lines().collect(Collectors.joining("\n"));
String result = new BufferedReader(new InputStreamReader(inputStream)).lines().parallel().collect(Collectors.joining("\n"));  //并行
```

> 会将将不同的换行符（如\r\n）转换为\n

## 6,使用InputStreamReader和StringBuilder（JDK）

``` java
final int bufferSize = 1024;
final char[] buffer = new char[bufferSize];
final StringBuilder out = new StringBuilder();
Reader in = new InputStreamReader(inputStream, "UTF-8");
for (; ;) {
    int rsz = in.read(buffer, 0, buffer.length);
    if (rsz < 0) break;
    out.append(buffer, 0, rsz);
}
return out.toString();
```

## 7,使用StringWriter和IOUtils.copy（Apache Commons）

``` java
StringWriter writer = new StringWriter();
IOUtils.copy(inputStream, writer, "UTF-8");
return writer.toString();
```

## 8,使用ByteArrayOutputStream和inputStream.read（JDK）

``` java
ByteArrayOutputStream result = new ByteArrayOutputStream();
byte[] buffer = new byte[1024];
int length;
while ((length = inputStream.read(buffer)) != -1) result.write(buffer, 0, length);
// StandardCharsets. UTF_8.name() > JDK 7
return result.toString("UTF-8");
```

## 9,使用BufferedReader（JDK）

``` java
String newLine = System.getProperty("line.separator");
BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
StringBuilder result = new StringBuilder();
String line; boolean flag = false;
while ((line = reader.readLine()) != null) {
    result.append(flag? newLine: "").append(line);
    flag = true;
}
return result.toString();
```

> 会将不同的换行符（如\n\r）转换为line.separator系统属性（例如，在Windows中为“\ r \ n”）

## 10,使用BufferedInputStream和ByteArrayOutputStream（JDK）

``` java
BufferedInputStream bis = new BufferedInputStream(inputStream);
ByteArrayOutputStream buf = new ByteArrayOutputStream();
int result = bis.read();
while(result != -1) {
    buf.write((byte) result);
    result = bis.read();
}
// StandardCharsets.UTF_8.name() > JDK 7
return buf.toString("UTF-8");
```

## 11,使用inputStream.read（）和StringBuilder （JDK）

``` java
int ch;
StringBuilder sb = new StringBuilder();
while((ch = inputStream.read()) != -1) sb.append((char)ch);
reset();
return sb.toString();
```

> 存在Unicode问题，例如使用文本（只能使用非Unicode文本正常工作）

## 性能测试

### 对于小型String（长度=175）

``` java
        Benchmark                               Mode  Cnt   Score   Error  Units
 8. ByteArrayOutputStream and read (JDK)        avgt   10   1,343 ± 0,028  us/op
 6. InputStreamReader and StringBuilder (JDK)   avgt   10   6,980 ± 0,404  us/op
10. BufferedInputStream, ByteArrayOutputStream  avgt   10   7,437 ± 0,735  us/op
11. InputStream.read() and StringBuilder (JDK)  avgt   10   8,977 ± 0,328  us/op
 7. StringWriter and IOUtils.copy (Apache)      avgt   10  10,613 ± 0,599  us/op
 1. IOUtils.toString (Apache Utils)             avgt   10  10,605 ± 0,527  us/op
 3. Scanner (JDK)                               avgt   10  12,083 ± 0,293  us/op
 2. CharStreams (guava)                         avgt   10  12,999 ± 0,514  us/op
 4. Stream Api (Java 8)                         avgt   10  15,811 ± 0,605  us/op
 9. BufferedReader (JDK)                        avgt   10  16,038 ± 0,711  us/op
 5. parallel Stream Api (Java 8)                avgt   10  21,544 ± 0,583  us/op
```

### 对于大String（长=50100）

``` java
               Benchmark                        Mode  Cnt   Score        Error  Units
 8. ByteArrayOutputStream and read (JDK)        avgt   10   200,715 ±   18,103  us/op
 1. IOUtils.toString (Apache Utils)             avgt   10   300,019 ±    8,751  us/op
 6. InputStreamReader and StringBuilder (JDK)   avgt   10   347,616 ±  130,348  us/op
 7. StringWriter and IOUtils.copy (Apache)      avgt   10   352,791 ±  105,337  us/op
 2. CharStreams (guava)                         avgt   10   420,137 ±   59,877  us/op
 9. BufferedReader (JDK)                        avgt   10   632,028 ±   17,002  us/op
 5. parallel Stream Api (Java 8)                avgt   10   662,999 ±   46,199  us/op
 4. Stream Api (Java 8)                         avgt   10   701,269 ±   82,296  us/op
10. BufferedInputStream, ByteArrayOutputStream  avgt   10   740,837 ±    5,613  us/op
 3. Scanner (JDK)                               avgt   10   751,417 ±   62,026  us/op
11. InputStream.read() and StringBuilder (JDK)  avgt   10  2919,350 ± 1101,942  us/op
```
