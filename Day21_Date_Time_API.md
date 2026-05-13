# Day 21 — Java 8 Date and Time API
**Level:** Beginner → Intermediate | **Java Version:** 8+

---

## 🌟 The Legacy Problem

Before Java 8, date and time handling in Java was notoriously bad.
- `java.util.Date` and `java.util.Calendar` were **mutable** (not thread-safe!).
- Months were zero-indexed (January = 0), but days were one-indexed.
- Formatting required `SimpleDateFormat`, which is also not thread-safe and caused silent concurrency bugs.

Java 8 introduced the `java.time` package (inspired by Joda-Time) to fix all this.
**Key Features of new API:** Immutable, Thread-safe, Clear naming, Fluent API.

---

## 📅 The Core Classes

### 1. LocalDate (Date only: YYYY-MM-DD)
Used when time zones and hours don't matter (e.g., birthdays, holidays).

```java
import java.time.LocalDate;
import java.time.Month;
import java.time.temporal.ChronoUnit;

// Current Date
LocalDate today = LocalDate.now();
System.out.println(today); // e.g., 2024-05-15

// Specific Date
LocalDate birthday = LocalDate.of(1995, Month.AUGUST, 25);
LocalDate parsed = LocalDate.parse("2024-12-31");

// Manipulation (Returns NEW objects due to immutability!)
LocalDate nextWeek = today.plusDays(7);
LocalDate lastMonth = today.minusMonths(1);
LocalDate firstDay = today.withDayOfMonth(1); // Modifies the day field

// Inspection
boolean isBefore = birthday.isBefore(today); // true
boolean isLeap = today.isLeapYear();

// Calculate gap
long daysBetween = ChronoUnit.DAYS.between(birthday, today);
```

### 2. LocalTime (Time only: HH:mm:ss)
Used when date doesn't matter (e.g., store opening hours).

```java
import java.time.LocalTime;

LocalTime now = LocalTime.now();
LocalTime lunchTime = LocalTime.of(13, 30); // 13:30 (1:30 PM)
LocalTime endOfDay = LocalTime.parse("17:00:00");

LocalTime later = now.plusHours(2).plusMinutes(15);
int hour = now.getHour();
```

### 3. LocalDateTime (Date + Time)
Used when you need both, but without specific geographical time zone rules (e.g., logging events on a local server).

```java
import java.time.LocalDateTime;

LocalDateTime now = LocalDateTime.now();
LocalDateTime meeting = LocalDateTime.of(2024, Month.JUNE, 1, 10, 30);
// Or combine LocalDate and LocalTime
LocalDateTime combined = LocalDateTime.of(today, lunchTime);

System.out.println(meeting); // 2024-06-01T10:30
```

---

## 🌍 Time Zones (`ZonedDateTime`)

If your app operates across multiple countries, use `ZonedDateTime`. It handles daylight saving time (DST) shifts automatically.

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;

// All available zones
// ZoneId.getAvailableZoneIds().forEach(System.out::println);

ZoneId istZone = ZoneId.of("Asia/Kolkata");
ZonedDateTime currentIst = ZonedDateTime.now(istZone);
System.out.println("IST: " + currentIst); // 2024-05-15T10:00:00.000+05:30[Asia/Kolkata]

// Convert local time to a specific zone
LocalDateTime localMeeting = LocalDateTime.of(2024, 6, 1, 9, 0); // 9 AM
ZonedDateTime nyMeeting = ZonedDateTime.of(localMeeting, ZoneId.of("America/New_York"));

// What time is the NY meeting in IST?
ZonedDateTime meetingInIst = nyMeeting.withZoneSameInstant(istZone);
System.out.println("Meeting in IST: " + meetingInIst);
```

---

## ⏱️ Machine Time (`Instant`)

`Instant` represents a specific moment on the timeline, measured in nanoseconds since the Unix Epoch (Jan 1, 1970 UTC). It's essentially UTC time.
*Best for: Timestamps, database storage, comparing exact moments.*

```java
import java.time.Instant;
import java.time.Duration;

Instant start = Instant.now(); // Always UTC

// ... do some heavy processing ...
for(int i=0; i<1000000; i++) {}

Instant end = Instant.now();

// Calculate duration
Duration duration = Duration.between(start, end);
System.out.println("Millis taken: " + duration.toMillis());
```

---

## 📏 Period vs Duration

Both measure an amount of time, but on different scales.

### `Period` (Date-based)
Measures time in Years, Months, and Days.
```java
import java.time.Period;

LocalDate dob = LocalDate.of(1995, 8, 25);
LocalDate today = LocalDate.now();

Period age = Period.between(dob, today);
System.out.printf("Age: %d years, %d months, %d days%n",
        age.getYears(), age.getMonths(), age.getDays());
```

### `Duration` (Time-based)
Measures time in Hours, Minutes, Seconds, and Nanoseconds.
```java
LocalTime start = LocalTime.of(9, 0);
LocalTime end = LocalTime.of(17, 30);

Duration workDay = Duration.between(start, end);
System.out.println("Hours worked: " + workDay.toHours()); // 8
```

---

## 🎨 Formatting and Parsing

Use `DateTimeFormatter`. **It IS thread-safe!**

```java
import java.time.format.DateTimeFormatter;

LocalDateTime now = LocalDateTime.now();

// 1. Formatting (Object to String)
DateTimeFormatter customFormatter = DateTimeFormatter.ofPattern("dd-MM-yyyy HH:mm");
String formatted = now.format(customFormatter);
System.out.println(formatted); // 15-05-2024 14:30

// 2. Parsing (String to Object)
String input = "25/12/2024";
DateTimeFormatter parser = DateTimeFormatter.ofPattern("dd/MM/yyyy");
LocalDate xmas = LocalDate.parse(input, parser);
```

---

## ❓ Interview Questions & Answers

**Q1. Why was the new Date and Time API introduced in Java 8? What was wrong with `java.util.Date`?**
> The old API was: 1) **Mutable**, making it unsafe in multi-threaded environments. 2) **Poorly designed** (e.g., months were 0-indexed, `Date` actually contained time as well). 3) **Formatting issues** (`SimpleDateFormat` was not thread-safe, leading to severe production bugs). The new `java.time` API is immutable, thread-safe, domain-driven (`LocalDate` vs `LocalTime`), and handles time zones elegantly.

**Q2. What is the difference between `LocalDateTime` and `ZonedDateTime`?**
> `LocalDateTime` contains date and time but lacks any concept of a time zone. It cannot represent an exact moment on the global timeline (9 AM in New York is different from 9 AM in Tokyo). `ZonedDateTime` includes date, time, and specific time zone rules (handling offsets and Daylight Saving Time). Use `ZonedDateTime` when communicating times across different geographical regions.

**Q3. What is the difference between `Instant` and `LocalDateTime`?**
> `Instant` represents an exact, absolute point on the timeline in UTC (seconds since 1970). It is universally unambiguous. `LocalDateTime` is relative to the observer's clock and has no time zone context. For backend timestamps and database storage, always use `Instant`.

**Q4. What is the difference between `Period` and `Duration`?**
> Both measure an amount of time. `Period` measures date-based time (Years, Months, Days). It respects daylight savings and varying month lengths. `Duration` measures time-based time (Hours, Minutes, Seconds, Nanos) strictly in exact seconds. You cannot use `Duration.between()` on two `LocalDate` objects (it throws an exception because dates lack time precision).

**Q5. Is `DateTimeFormatter` thread-safe?**
> Yes. Unlike the old `SimpleDateFormat`, the new `DateTimeFormatter` is completely immutable and thread-safe. You can safely declare it as a `public static final` constant and use it across multiple threads simultaneously.

---

## ✅ Best Practices

1. **Use `Instant` for saving timestamps to the database.** It standardizes all times to UTC, preventing timezone offset bugs when servers move or daylight saving hits.
2. **Never use `java.util.Date` or `java.util.Calendar` in new code.** If interacting with legacy code, use the conversion methods: `date.toInstant()` or `Date.from(instant)`.
3. **Declare `DateTimeFormatter` as a static constant.**
4. **Use `LocalDate` instead of strings** to pass dates around in your application methods.

---

## 🛠️ Hands-on Practice

1. Write a method that takes a `LocalDate` (someone's birthday) and returns the number of days remaining until their *next* birthday in the current year.
2. Parse this string: `"2024-11-20 18:30:00"` into a `LocalDateTime` using a custom `DateTimeFormatter`.
3. Create a `ZonedDateTime` representing a flight departing London at 10:00 AM local time. If the flight takes 8 hours, calculate the exact arrival `ZonedDateTime` in New York (America/New_York).
4. Measure the exact execution time of a sorting algorithm in milliseconds using `Instant` and `Duration`.
