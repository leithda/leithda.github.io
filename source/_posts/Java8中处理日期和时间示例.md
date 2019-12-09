---
title: Java8中处理日期和时间示例
categories:
  - Java
  - Basic
tags:
  - 日期
author: 长歌
abbrlink: 2517593741
date: 2019-10-14 00:00:00
---

Java8 中新增了几个关于日期和时间处理的类，主要有`LocalDate`、`LocalTime`等。可以更好的进行关于日期和时间的处理。
<!-- More -->


### 主要类
- `java.time`包下时间相关类

| 类名            | 说明                                    |
| --------------- | --------------------------------------- |
| `Instant`       | 时间戳                                  |
| `Duuration`     | 持续时间，时间差                        |
| `LocalDate`     | 只包含日期,比如 2019-01-01              |
| `LocalTime`     | 只包含事件,比如 23:45:56                |
| `LocalDateTime` | 包含日期和时间,比如 2001-02-03 04:05:06 |
| `Period`        | 时间段                                  |
| `ZoneOffset`    | 时区偏移量，比如  + 8:00                |
| `ZonedDateTime` | 带时区的时间                            |
| `Clock`         | 时钟                                    |

- `java.time.format`包

| 类                  | 说明       |
| ------------------- | ---------- |
| `DateTimeFormatter` | 时间格式化 |

 

### 获取今天的日期
```java
LocalDate todayDate = LocalDate.now();
System.out.println("今天的日期是: "  +  todayDate);
```

### 指定日期，进行相应操作
```java
LocalDate oneDay = LocalDate.parse("2019-10-12");

// 取2019年10月的第1天
LocalDate firstDay = oneDay.with(TemporalAdjusters.firstDayOfMonth());
System.out.println(firstDay);

// 取2019年10月的第1天
firstDay = oneDay.withDayOfMonth(1);
System.out.println(firstDay);

// 取2019年10月的最后1天
LocalDate lastDay = oneDay.with(TemporalAdjusters.lastDayOfMonth());
System.out.println(lastDay);

// 当前日期 + 1天
LocalDate tomorrow = oneDay.plusDays(1);
System.out.println("Tomorrow is " + tomorrow);

// 判断是否是闰年
boolean isLeapYear = tomorrow.isLeapYear();
System.out.println("今年[" + oneDay.getYear() + "]是否是闰年:" + isLeapYear);
```

### 生日检查或者账单日检查
```java
LocalDate birtyday = LocalDate.of(1993, 11, 16);
MonthDay birthdayMD = MonthDay.of(birtyday.getMonth(), birtyday.getDayOfMonth());
MonthDay today = MonthDay.from(LocalDate.of(2019, 11, 16));
System.out.println("今天"  +  today  +  "是否是["  +  birtyday  +  "]出生的人的生日: "  +  today.equals(birthdayMD));
```
- 校验月 + 日可以使用`MonthDay`类，相应的还有一个类`YearMonth`

### 获取当前时间
```java
// 获取当前时间
LocalTime nowTime = LocalTime.now();
System.out.println("当前时间: " + nowTime);

// 不显示毫秒
nowTime = LocalTime.now().withNano(0);
System.out.println("当前时间(无毫秒): " + nowTime);


// 指定时间
LocalTime time = LocalTime.of(17,35,32);
System.out.println("指定时间: " + time);
time = LocalTime.parse("18:00:02");
System.out.println("指定时间: " + time);


// 当前时间增加2小时
LocalTime nowTimePlus2Hours = nowTime.plusHours(2);
System.out.println("当前时间增加2小时: " + nowTimePlus2Hours);
// 或者
nowTimePlus2Hours = nowTime.plus(-2, ChronoUnit.HOURS);
System.out.println("当前时间减少2小时: " + nowTimePlus2Hours);
```

### 日期前后比较
```java
LocalDate nowDate = LocalDate.now();

LocalDate specifyDate = LocalDate.of(2018,10,12);

System.out.println("今天[" + nowDate + "]是否在[" + specifyDate + "]后面: " + nowDate.isAfter(specifyDate));
// 今天[2019-10-12]是否在[2018-10-12]后面: true
```

### 处理不同时区的时间
```java
// 查看当前的时区
ZoneId defaultZone = ZoneId.systemDefault();
System.out.println("当前时区: " + defaultZone);   // Asia/Chongqing

// 查看美国纽约当前的时间
ZoneId america = ZoneId.of("America/New_York");
LocalDateTime shanghaiTime = LocalDateTime.now();
LocalDateTime americaDateTime = LocalDateTime.now(america);
System.out.println("当前时区时间: " + shanghaiTime);    // 2019-10-12T17:26:55.351
System.out.println("纽约时区时间: " + americaDateTime); // 2019-10-12T05:26:55.351

// 带时区的时间
ZonedDateTime americaZoneDateTime = ZonedDateTime.now(america);
System.out.println("带时区的时间： " + americaZoneDateTime); // 2019-10-12T05:26:55.360-04:00[America/New_York]
```

### 比较两个日期之间时间差
```java
LocalDate nowDate = LocalDate.now();
LocalDate specifyDate = LocalDate.of(1993,11,16);

Period period = Period.between(specifyDate,nowDate);
System.out.println("当前时间[" + nowDate + "],纪念日[" + specifyDate + "]");   // 当前时间[2019-10-12],纪念日[1993-11-16]
System.out.println("相差天数" + period.getDays());    // 相差天数26
System.out.println("相差月数" + period.getMonths());  // 相差月数10
System.out.println("相差实际天数" + specifyDate.until(nowDate,ChronoUnit.DAYS));    //  相差实际天数9461
System.out.println("相差实际月数" + specifyDate.until(nowDate,ChronoUnit.MONTHS));  // 相差实际月数310
System.out.println("相差实际年数" + specifyDate.until(nowDate,ChronoUnit.YEARS));   // 相差实际年数25
```
- `Period`类比较的是相对差，11-16到下个月12号之间一共有26天，2019年11月到下一年的10月之间一共10个月
- 如果需要实际差可以使用until方法，指定参数获取

### 日期时间格式解析、格式化
```java
String specifyDate = "19931116";

LocalDate formateed = LocalDate.parse(specifyDate, DateTimeFormatter.BASIC_ISO_DATE);

System.out.println("格式化后日期" + formateed);
```

### 时间类与Date类的相互转化
```java
// Date 与 Instant 的相互转化
Instant instant = Instant.now();
Date date = Date.from(instant);
System.out.println("Instant["+instant+"] -> Date["+date+"]");

Instant date2Instant = date.toInstant();
System.out.println("Date["+date+" -> Instant["+instant+"]");

// Date 转为 LocalDateTime
date = new Date();
LocalDateTime date2LocalDateTime = LocalDateTime.ofInstant(date.toInstant(),ZoneId.systemDefault());
System.out.println("Date["+date+"] -> LocalDateTime["+date2LocalDateTime);

// LocalDateTime 转为 Date
LocalDateTime nowDateTime = LocalDateTime.now();
instant = nowDateTime.atZone(ZoneId.systemDefault()).toInstant();
Date localDateTime2Date = Date.from(instant);
System.out.println("LocalDateTime["+nowDateTime+"] -> Date["+localDateTime2Date+"]");

// LocalDate 转为 Date
// 因为LocalDate不包含时间，转化为Date时，时间默认 00:00:00
LocalDate nowDate = LocalDate.now();
instant = nowDate.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant();
Date localDate2Date = Date.from(instant);
System.out.println("LocalDate["+nowDate+"] -> Date["+localDate2Date+"]");
```

## 封装日期工具类
```java
// 节假日处理方式枚举类型
public enum HolidayType {
    NEXT,   // 向后顺延
    PREVIOUS,   // 向前顺延
    NO  // 不处理
}

// 日期工具类
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-14
 * Description: 日期工具类
 */
public final class DateUtils {

    // 默认日期格式为 yyyyMMdd

    public static final String DATE_PATTERN_DEFAULT = "yyyyMMdd";

    public static final String TIME_PATTERN_DEFAULT = "HH:mm:ss";

    public static final String DATETIME_PATTERN_DEFAULT = "yyyyMMdd HH:mm:ss";

    /**
     * 节假日列表，一般应该通过数据库进行配置(TODO 以优化为月日形式,后续优化)
     */
    private static final List<String> holidayList = new ArrayList<String>(Arrays.asList("20190101", "20190501", "20191001","20191002","20191003"));

    private DateUtils() {
    }

    /**
     * 获取当前日期
     *
     * @param pattern 日期格式
     * @return 当前日期
     */
    public static String getCurrentDate(String pattern) {
        String sCurrnetDate;
        try {
            String dateFormatPattern = pattern == null ? DATE_PATTERN_DEFAULT : pattern;
            LocalDate currentDate = LocalDate.now();
            sCurrnetDate = currentDate.format(DateTimeFormatter.ofPattern(dateFormatPattern));
        } catch (Exception e) {
            throw new RuntimeException("日期格式错误!");
        }
        return sCurrnetDate;
    }

    /**
     * 获取当前日期
     *
     * @return 当前日期
     */
    public static String getCurrentDate() {
        return getCurrentDate(null);
    }

    /**
     * 获取当前时间
     *
     * @param pattern 时间格式
     * @return 当前时间
     */
    public static String getCurrentTime(String pattern) {
        String sCurrentTime;
        try {
            String timeFormatPattern = pattern == null ? TIME_PATTERN_DEFAULT : pattern;
            LocalTime currentTime = LocalTime.now();
            sCurrentTime = currentTime.format(DateTimeFormatter.ofPattern(timeFormatPattern));
        } catch (Exception e) {
            throw new RuntimeException("时间格式错误");
        }
        return sCurrentTime;
    }

    /**
     * 获取当前时间
     *
     * @return 当前时间
     */
    public static String getCurrentTime() {
        return getCurrentTime(null);
    }

    /**
     * 获取当前日期+时间
     *
     * @param pattern 日期+时间 格式
     * @return 日期+时间
     */
    public static String getCurrentDateTime(String pattern) {
        String sCurrentDateTime;
        try {
            String dateTimeFormatPattern = pattern == null ? DATETIME_PATTERN_DEFAULT : pattern;
            LocalDateTime currentDateTime = LocalDateTime.now();
            sCurrentDateTime = currentDateTime.format(DateTimeFormatter.ofPattern(dateTimeFormatPattern));
        } catch (Exception e) {
            throw new RuntimeException("日期时间格式错误");
        }
        return sCurrentDateTime;
    }

    /**
     * 获取当前日期+时间
     *
     * @return 日期+时间
     */
    public static String getCurrentDateTime() {
        return getCurrentDateTime(null);
    }

    /**
     * 计算日期 +N 天
     *
     * @param oneDate   日期
     * @param daysCount 天数
     * @return 结果日期
     */
    public static String plusDays(String oneDate, int daysCount) {
        LocalDate oneLocalDate = covStringToDate(oneDate, DATE_PATTERN_DEFAULT);
        LocalDate resultDate = oneLocalDate.plusDays(daysCount);
        return covDateToString(resultDate, DATE_PATTERN_DEFAULT);
    }

    /**
     * 计算日期 +1 天
     *
     * @param oneDate 日期
     * @return 结果日期
     */
    public static String addDay(String oneDate) {
        return plusDays(oneDate, 1);
    }

    /**
     * 计算日期 +N 周
     *
     * @param oneDate    日期
     * @param weeksCount 周数
     * @return 结果日期
     */
    public static String plusWeek(String oneDate, int weeksCount) {
        LocalDate oneLocalDate = covStringToDate(oneDate, DATE_PATTERN_DEFAULT);
        LocalDate resultDate = oneLocalDate.plusWeeks(weeksCount);
        return covDateToString(resultDate, DATE_PATTERN_DEFAULT);
    }

    /**
     * 计算日期 +N 月
     *
     * @param oneDate     日期
     * @param monthsCount 月数
     * @return 结果日期
     */
    public static String plusMonths(String oneDate, int monthsCount) {
        LocalDate oneLocalDate = covStringToDate(oneDate, DATE_PATTERN_DEFAULT);
        LocalDate resultDate = oneLocalDate.plusMonths(monthsCount);
        return covDateToString(resultDate, DATE_PATTERN_DEFAULT);
    }

    /**
     * 计算日期 +N 年
     *
     * @param oneDate    日期
     * @param yearsCount 年数
     * @return 结果日期
     */
    public static String plusYear(String oneDate, int yearsCount) {
        LocalDate oneLocalDate = covStringToDate(oneDate, DATE_PATTERN_DEFAULT);
        LocalDate resultDate = oneLocalDate.plusYears(yearsCount);
        return covDateToString(resultDate, DATE_PATTERN_DEFAULT);
    }

    /**
     * 根据期限进行日期计算
     *
     * @param oneDate     当前日期
     * @param sTerm       期限
     *                    单位:D-天 W-周 M-月 Y-年
     *                    支持反向计算(输入负数期限)
     * @param holidayType 节假日处理方式
     * @return 结果日期
     */
    public static String calDateByTerm(String oneDate, String sTerm, HolidayType holidayType) {
        LocalDate oneLocalDate = covStringToDate(oneDate, DATE_PATTERN_DEFAULT);

        // 期限数量
        int iTerm = 0;
        if (sTerm.substring(0, 1).equals("-")) {
            iTerm = -1 * Integer.parseInt(sTerm.substring(1, sTerm.length() - 1));
        } else {
            iTerm = Integer.parseInt(sTerm.substring(0, sTerm.length() - 1));
        }

        LocalDate resultLocalDate;
        String sResultDate;
        // 期限单位
        switch (sTerm.charAt(sTerm.length() - 1)) {
            case 'D':
                resultLocalDate = oneLocalDate.plusDays(iTerm);
                break;
            case 'W':
                resultLocalDate = oneLocalDate.plusWeeks(iTerm);
                break;
            case 'M':
                resultLocalDate = oneLocalDate.plusMonths(iTerm);
                break;
            case 'Y':
                resultLocalDate = oneLocalDate.plusYears(iTerm);
                break;
            default:
                System.out.println("无对应期限类型，返回原日期");
                return oneDate;
        }

        // 节假日处理
        sResultDate = covDateToString(resultLocalDate, DATE_PATTERN_DEFAULT);
        if (holidayList.contains(sResultDate)) {
            switch (holidayType) {
                case NO:
                    break;
                case NEXT:
                    while(holidayList.contains(sResultDate)){
                        resultLocalDate = resultLocalDate.plusDays(1);
                        sResultDate = covDateToString(resultLocalDate, DATE_PATTERN_DEFAULT);
                    }
                    break;
                case PREVIOUS:
                    while(holidayList.contains(sResultDate)){
                        resultLocalDate = resultLocalDate.plusDays(-1);
                        sResultDate = covDateToString(resultLocalDate, DATE_PATTERN_DEFAULT);
                    }
                    break;
            }
        }
        return sResultDate;
    }

    /**
     * 根据频率计算下一日期
     * @param oneDate 日期
     * @param freq 频率
     * @param holidayType   节假日处理方式
     * @return  结果日期
     */
    public static String calDateByFreq(String oneDate,String freq,HolidayType holidayType){
        // TODO 后续完善
        return "";
    }

    /**
     * 判断日期是否合法
     * @param oneDate 日期
     * @return  是否
     */
    public static boolean isDate(String oneDate){
        try{
            LocalDate localDate = LocalDate.parse(oneDate,DateTimeFormatter.BASIC_ISO_DATE);
        }catch (Exception e){
            return false;
        }
        return true;
    }

    /**
     * 字符串日期转换为LocalDate日期
     *
     * @param date    字符串日期
     * @param pattern 日期格式
     * @return LocalDate日期
     */
    private static LocalDate covStringToDate(String date, String pattern) {

        return LocalDate.parse(date, DateTimeFormatter.ofPattern(pattern));
    }

    /**
     * LocalDate日期转换为String日期
     *
     * @param date    LocalDate 日期
     * @param pattern 日期格式
     * @return 字符串日期
     */
    private static String covDateToString(LocalDate date, String pattern) {
        return date.format(DateTimeFormatter.ofPattern(pattern));
    }
}
```
- 简版本日期处理工具类，后续有时间进行完善