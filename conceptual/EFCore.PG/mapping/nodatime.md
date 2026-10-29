# Date/Time Mapping with NodaTime

## What is NodaTime

By default, [the PostgreSQL date/time types](https://www.postgresql.org/docs/current/static/datatype-datetime.html) are mapped to the built-in .NET types (`DateTime`, `TimeSpan`). Unfortunately, these built-in types are flawed in many ways. The [NodaTime library](http://nodatime.org/) was created to solve many of these problems, and if your application handles dates and times in anything but the most basic way, you should consider using it. To learn more [read this blog post by Jon Skeet](http://blog.nodatime.org/2011/08/what-wrong-with-datetime-anyway.html).

Beyond NodaTime's general advantages, some specific advantages NodaTime for PostgreSQL date/time mapping include:

* NodaTime defines some types which are missing from the BCL, such as `LocalDate`, `LocalTime`, and `OffsetTime`. These cleanly correspond to PostgreSQL `date`, `time` and `timetz`.
* `Period` is much more suitable for mapping PostgreSQL `interval` than `TimeSpan`.
* NodaTime types can fully represent PostgreSQL's microsecond precision, and can represent dates outside the BCL's date limit (1AD-9999AD).

## Setup

To set up the NodaTime plugin, add the [Npgsql.EntityFrameworkCore.PostgreSQL.NodaTime nuget](https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL.NodaTime) to your project. Then, configure the NodaTime plugin as follows:

### [EF 9.0, with a connection string](#tab/ef9-with-connection-string)

If you're passing a connection string to `UseNpgsql`, simply add the `UseNodaTime` call as follows:

```csharp
builder.Services.AddDbContext<MyContext>(options => options.UseNpgsql(
    "<connection string>",
    o => o.UseNodaTime()));
```

This configures all aspects of Npgsql to use the NodaTime plugin - both at the EF and the lower-level Npgsql layer.

### [With an external NpgsqlDataSource](#tab/with-datasource)

If you're creating an external NpgsqlDataSource and passing it to `UseNpgsql`, you must call `UseNodaTime` on your NpgsqlDataSourceBuilder independently of the EF-level setup:

```csharp
var dataSourceBuilder = new NpgsqlDataSourceBuilder("<connection string>");
dataSourceBuilder.UseNodaTime();
var dataSource = dataSourceBuilder.Build();

builder.Services.AddDbContext<MyContext>(options => options.UseNpgsql(
    dataSource,
    o => o.UseNodaTime()));
```

### [Older EF versions, with a connection string](#tab/legacy-with-connection-string)

```csharp
// Configure UseNodaTime at the ADO.NET level.
// This code must be placed at the beginning of your application, before any other Npgsql API is called; an appropriate place for this is in the static constructor on your DbContext class:
static MyDbContext()
    => NpgsqlConnection.GlobalTypeMapper.UseNodaTime();

// Then, when configuring EF Core with UseNpgsql(), call UseNodaTime():
builder.Services.AddDbContext<MyContext>(options =>
    options.UseNpgsql("<connection string>", o => o.UseNodaTime()));
```

***

The above sets up all the necessary mappings and operation translators. You can now use NodaTime types as regular properties in your entities, and even perform some operations:

```csharp
public class Post
{
    public int Id { get; set; }
    public string Name { get; set; }
    public Instant CreationTime { get; set; }
}

var recentPosts = context.Posts.Where(p => p.CreationTime > someInstant);
```

## Operation translation

The provider knows how to translate many members and methods on mapped NodaTime types. For example, the following query will be translated to SQL and evaluated server-side:

```csharp
// Get all events which occurred on a Monday
var mondayEvents = context.Events.Where(p => p.SomeDate.DayOfWeek == DayOfWeek.Monday);

// Get all events which occurred before the year 2000
var oldEvents = context.Events.Where(p => p.SomeDate.Year < 2000);
```

Following is the list of supported NodaTime translations; If an operation you need is missing, please open an issue to request for it.

Starting with version 11.0, `EF.Functions.DateTrunc()` supports `LocalDateTime`, `Instant`, `ZonedDateTime`, `Period` and `Duration`. Its optional time zone argument is only supported for `Instant` and `ZonedDateTime` mapped to `timestamp with time zone`; without that argument, PostgreSQL uses the session time zone for these values.

.NET                                                                                   | SQL                                                                                                                                        | Notes
-------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | -----------
SystemClock.Instance.GetCurrentInstant()                                               | [now()](https://www.postgresql.org/docs/current/functions-datetime.html)                                                                   |
EF.Functions.DateTrunc("month", value)                                                 | [date_trunc('month', value)](https://www.postgresql.org/docs/current/functions-datetime.html)                                              | Added in 11.0
EF.Functions.DateTrunc("month", instant, "Europe/Berlin")                              | [date_trunc('month', instant, 'Europe/Berlin')](https://www.postgresql.org/docs/current/functions-datetime.html)                           | Added in 11.0
LocalDateTime.Date                                                                     | [date_trunc('day', timestamp)](https://www.postgresql.org/docs/current/functions-datetime.html)                                            |
LocalDateTime.Second (also LocalTime, ZonedDateTime)                                   | [date_part('second', timestamp)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                     |
LocalDateTime.Millisecond (also LocalTime, ZonedDateTime)                              | [floor(date_part('milliseconds', timestamp))::INT % 1000](https://www.postgresql.org/docs/current/functions-datetime.html)                 | Added in 11.0
LocalDateTime.Minute (also LocalTime, ZonedDateTime)                                   | [date_part('minute', timestamp)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                     |
LocalDateTime.Hour (also LocalTime, ZonedDateTime)                                     | [date_part('hour', timestamp)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                       |
LocalDateTime.Day, (also LocalDate, ZonedDateTime)                                     | [date_part('day', timestamp)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                        |
LocalDateTime.Month (also LocalDate, ZonedDateTime)                                    | [date_part('month', timestamp)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                      |
LocalDateTime.Year (also LocalDate, ZonedDateTime)                                     | [date_part('year', timestamp)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                       |
LocalDateTime.DayOfWeek (also LocalDate, ZonedDateTime)                                | [floor(date_part('dow', timestamp))::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                 |
LocalDateTime.DayOfYear (also LocalDate, ZonedDateTime)                                | [date_part('doy', timestamp)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                        |
OffsetDateTime.Year (also the other ZonedDateTime members above)                       | [date_part('year', timestamptz AT TIME ZONE 'UTC')::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                  | Added in 11.0
LocalDate.At(time)                                                                     | [date + time](https://www.postgresql.org/docs/current/functions-datetime.html)                                                             | Added in 10.0
LocalDate.AtMidnight()                                                                 | [date + '00:00:00'](https://www.postgresql.org/docs/current/functions-datetime.html)                                                       | Added in 10.0
Period.Seconds                                                                         | [trunc(date_part('second', interval))::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                               | Truncated towards zero like NodaTime since 11.0
Period.Milliseconds                                                                    | [trunc(date_part('milliseconds', interval))::INT % 1000](https://www.postgresql.org/docs/current/functions-datetime.html)                  | Added in 11.0
Period.Minutes                                                                         | [date_part('minute', interval)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                      |
Period.Hours                                                                           | [date_part('hour', interval)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                        |
Period.Days                                                                            | [date_part('day', interval)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                         |
Period.Months                                                                          | [date_part('month', interval)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                       |
Period.Years                                                                           | [date_part('year', interval)::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                                        |
Period.FromSeconds                                                                     | [make_interval(seconds => int)](https://www.postgresql.org/docs/current/functions-datetime.html)                                           |
Period.FromMinutes                                                                     | [make_interval(minutes => int)](https://www.postgresql.org/docs/current/functions-datetime.html)                                           |
Period.FromHours                                                                       | [make_interval(hours => int)](https://www.postgresql.org/docs/current/functions-datetime.html)                                             |
Period.FromDays                                                                        | [make_interval(days => int)](https://www.postgresql.org/docs/current/functions-datetime.html)                                              |
Period.FromWeeks                                                                       | [make_interval(weeks => int)](https://www.postgresql.org/docs/current/functions-datetime.html)                                             |
Period.FromMonths                                                                      | [make_interval(months => int)](https://www.postgresql.org/docs/current/functions-datetime.html)                                            |
Period.FromYears                                                                       | [make_interval(years => int)](https://www.postgresql.org/docs/current/functions-datetime.html)                                             |
Duration.Days                                                                          | [date_part('day', justify_hours(interval))::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                          | Changed in 11.0
Duration.Hours                                                                         | [date_part('hour', justify_hours(interval))::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                         | Changed in 11.0
Duration.Minutes                                                                       | [date_part('minute', justify_hours(interval))::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                       | Changed in 11.0
Duration.Seconds                                                                       | [trunc(date_part('second', justify_hours(interval)))::INT](https://www.postgresql.org/docs/current/functions-datetime.html)                | Changed in 11.0
Duration.Milliseconds                                                                  | [trunc(date_part('millisecond', justify_hours(interval)))::INT % 1000](https://www.postgresql.org/docs/current/functions-datetime.html)    | Added in 11.0
Duration.TotalMilliseconds                                                             | [date_part('epoch', interval) / 0.001](https://www.postgresql.org/docs/current/functions-datetime.html)                                    |
Duration.TotalSeconds                                                                  | [date_part('epoch', interval)](https://www.postgresql.org/docs/current/functions-datetime.html)                                            |
Duration.TotalMinutes                                                                  | [date_part('epoch', interval) / 60.0](https://www.postgresql.org/docs/current/functions-datetime.html)                                     |
Duration.TotalDays                                                                     | [date_part('epoch', interval) / 86400.0](https://www.postgresql.org/docs/current/functions-datetime.html)                                  |
Duration.TotalHours                                                                    | [date_part('epoch', interval) / 3600.0](https://www.postgresql.org/docs/current/functions-datetime.html)                                   |
Duration.BclCompatibleTicks                                                            | [(extract(epoch FROM interval) \* 10000000)::bigint](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-EXTRACT) | Added in 11.0
Duration.FromHours(value) (also FromDays, FromMinutes, FromSeconds, FromMilliseconds)  | [value \* INTERVAL '01:00:00'](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)                   | Added in 11.0
instant + duration (also ZonedDateTime, OffsetDateTime, -)                             | [(instant AT TIME ZONE 'UTC' + interval) AT TIME ZONE 'UTC'](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE) | Computed in UTC since 11.0
Plus(other), Minus(other)                                                              | [value + other, value - other](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)                   | Added in 11.0; also the static Add() and Subtract(), see below
Duration.Negate(duration)                                                              | [-duration](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)                                      | Added in 11.0
duration \* number (also number \* duration, duration / number, Multiply(), Divide())  | [duration * number](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)                               | Added in 11.0
duration1 / duration2 (also Divide())                                                  | [date_part('epoch', duration1) / date_part('epoch', duration2)](https://www.postgresql.org/docs/current/functions-datetime.html)            | Added in 11.0
LocalDate.PlusDays(days)                                                               | [date + days](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)                                    | Added in 11.0
LocalDate.PlusWeeks(weeks)                                                             | [date + weeks * 7](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)                               | Added in 11.0
LocalDate.PlusMonths(months) (also PlusYears)                                          | [CAST(date::timestamp + make_interval(months => months) AS date)](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE) | Added in 11.0
LocalDateTime.PlusMonths(months) (also PlusYears, PlusWeeks, PlusDays)                 | [timestamp + make_interval(months => months)](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)    | Added in 11.0
LocalDateTime.PlusHours(hours) (also LocalTime, PlusMinutes, PlusSeconds)              | [timestamp + make_interval(hours => hours)](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE)      | Added in 11.0
LocalDateTime.PlusMilliseconds(milliseconds) (also LocalTime)                          | [timestamp + milliseconds * INTERVAL '1 millisecond'](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE) | Added in 11.0
ZonedDateTime.PlusHours(hours) (also PlusMinutes, PlusSeconds, PlusMilliseconds)       | [(timestamptz AT TIME ZONE 'UTC' + make_interval(hours => hours)) AT TIME ZONE 'UTC'](https://www.postgresql.org/docs/current/functions-datetime.html#OPERATORS-DATETIME-TABLE) | Added in 11.0
ZonedDateTime.LocalDateTime                                                            | [timestamptz AT TIME ZONE 'UTC'](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT)           |
DateInterval.Length                                                                    | [upper(daterange) - lower(daterange)](https://www.postgresql.org/docs/current/functions-range.html)                                        |
DateInterval.Start                                                                     | [lower(daterange)](https://www.postgresql.org/docs/current/functions-range.html)                                                           |
DateInterval.End                                                                       | [upper(daterange) - 1](https://www.postgresql.org/docs/current/functions-range.html)                                                       |
DateInterval.Contains(LocalDate)                                                       | [daterange @> date](https://www.postgresql.org/docs/current/functions-range.html)                                                          |
DateInterval.Contains(DateInterval)                                                    | [daterange @> daterange](https://www.postgresql.org/docs/current/functions-range.html)                                                     |
DateInterval.Intersection(DateInterval)                                                | [CASE WHEN daterange1 && daterange2 THEN daterange1 * daterange2 END](https://www.postgresql.org/docs/current/functions-range.html)        | NULL when the intervals don't overlap, like NodaTime, since 11.0 ([#4083](https://github.com/npgsql/efcore.pg/issues/4083))
DateInterval.Union(DateInterval)                                                       | [CASE WHEN daterange1 && daterange2 OR daterange1 -\|- daterange2 THEN daterange1 + daterange2 END](https://www.postgresql.org/docs/current/functions-range.html) | NULL when the intervals neither overlap nor are adjacent, like NodaTime, since 11.0 ([#4083](https://github.com/npgsql/efcore.pg/issues/4083))
Instant.InZone(DateTimeZoneProviders.Tzdb["Europe/Berlin"]).LocalDateTime              | [timestamptz AT TIME ZONE 'Europe/Berlin'](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT) |
LocalDateTime.InZoneLeniently(DateTimeZoneProviders.Tzdb["Europe/Berlin"]).ToInstant() | [timestamp AT TIME ZONE 'Europe/Berlin'](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT)   |
ZonedDateTime.ToInstant                                                                | No PG operation (.NET-side conversion from ZonedDateTime to Instant only)                                                                  |
Instant.InZone(DateTimeZoneProviders.Tzdb["Europe/Berlin"]).ToInstant()                | No PG operation (the instant itself)                                                                                                       | Fixed in 11.0 ([#4085](https://github.com/npgsql/efcore.pg/issues/4085))
Instant.InUtc                                                                          | No PG operation (.NET-side conversion from Instant to ZonedDateTime only)                                                                  |
Instant.ToDateTimeUtc                                                                  | No PG operation (.NET-side conversion from Instant to UTC DateTime only)                                                                   |
Instant.ToUnixTimeSeconds() (also ToUnixTimeMilliseconds)                              | [floor(extract(epoch FROM timestamptz))::bigint](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-EXTRACT) | Added in 11.0
Instant.FromUnixTimeSeconds(seconds) (also FromUnixTimeMilliseconds)                   | [to_timestamp(seconds)](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-TABLE)                          | Added in 11.0
Instant.Max(instant1, instant2) (also Min)                                             | [GREATEST(instant1, instant2)](https://www.postgresql.org/docs/current/functions-conditional.html#FUNCTIONS-GREATEST-LEAST)                | Added in 11.0; Min() as LEAST()
LocalDateTime.InUtc()                                                                  | [timestamp AT TIME ZONE 'UTC'](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT)             | Added in 11.0
OffsetDateTime.ToInstant()                                                             | No PG operation (both are mapped to timestamptz)                                                                                           | Added in 11.0
LocalDateTime.ToDateTimeUnspecified()                                                  | No PG operation (timestamp without time zone is unchanged)                                                                                 | Added in 11.0
LocalDate.ToDateOnly() (also LocalTime.ToTimeOnly())                                   | No PG operation (both are mapped to the same type)                                                                                         | Added in 11.0
Duration.ToTimeSpan() (also Duration.FromTimeSpan())                                   | No PG operation (both are mapped to interval)                                                                                              | Added in 11.0
EF.Functions.Sum(periods)                                                              | [sum(periods)](https://www.postgresql.org/docs/current/functions-aggregate.html#FUNCTIONS-AGGREGATE-TABLE)                                 | See [Aggregate functions](translations.md#aggregate-functions).
EF.Functions.Sum(durations)                                                            | [sum(durations)](https://www.postgresql.org/docs/current/functions-aggregate.html#FUNCTIONS-AGGREGATE-TABLE)                               | See [Aggregate functions](translations.md#aggregate-functions).
EF.Functions.Average(periods)                                                          | [avg(durations)](https://www.postgresql.org/docs/current/functions-aggregate.html#FUNCTIONS-AGGREGATE-TABLE)                               | See [Aggregate functions](translations.md#aggregate-functions).
EF.Functions.Average(durations)                                                        | [avg(durations)](https://www.postgresql.org/docs/current/functions-aggregate.html#FUNCTIONS-AGGREGATE-TABLE)                               | See [Aggregate functions](translations.md#aggregate-functions).

In addition to the above, most arithmetic operators are also translated (e.g. LocalDate + Period).

Starting with version 11.0, NodaTime's fluent arithmetic methods are also translated:

* `Plus()` and `Minus()`, as well as the static `Add()` and `Subtract()`, are translated like the `+` and `-` operators for `Duration`, `Instant`, `ZonedDateTime`, `OffsetDateTime`, `LocalDateTime`, `LocalDate`, `LocalTime` and `Period`, and `Duration.Multiply()` and `Duration.Divide()` like the `*` and `/` operators (see below for where PostgreSQL's results differ).
* Arithmetic on `instant.InZone(zone)` (e.g. `PlusHours()` or `Plus(duration)`) is exact, like NodaTime's, and the result keeps that time zone, e.g. for `ToInstant()` or `Hour`.
* `PlusWeeks()` adds exactly 7 days per week, and `PlusMonths()` and `PlusYears()` keep the day of the month unless it's past the end of the resulting month, in which case the last day of that month is used (e.g. 2018-01-31 plus one month is 2018-02-28), like NodaTime.
* `LocalTime` arithmetic wraps around midnight, like NodaTime.
* `PlusTicks()` and `PlusNanoseconds()` aren't translated, since PostgreSQL's resolution is one microsecond.

The fluent methods are translated like the corresponding operators, so where PostgreSQL computes a result differently from NodaTime for an operator, it does so for the method too:

* Adding or subtracting a `Period` with both years and months to a `LocalDate` or `LocalDateTime`: NodaTime adds the years and then the months, clamping to the end of the month in between, whereas PostgreSQL adds them together as months (e.g. 2016-02-29 plus `P1Y1M` is 2017-03-28 in NodaTime but 2017-03-29 in PostgreSQL).
* `Period` arithmetic: PostgreSQL computes an `interval`, which Npgsql normalizes when reading it (e.g. `P3D` minus `PT2H` is read back as `P2DT22H` rather than `P3DT-2H`).
* The difference between two `LocalDate` values is computed with [`age()`](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-TABLE), whose month-end handling differs from NodaTime's (e.g. 2018-04-30 minus 2018-03-31 is `P30D` in PostgreSQL but `P1M` in NodaTime).
* The difference between two `LocalDateTime` values is computed with timestamp subtraction, which returns days and time but never months or years (e.g. 2018-04-20T10:31:33 minus 2017-04-20T10:36:33 is `P364DT23H55M` in PostgreSQL but `P11M30DT23H55M` in NodaTime).

Starting with version 11.0, adding or subtracting a `Duration` to an `Instant`, `ZonedDateTime` or `OffsetDateTime` is computed in UTC. This is because PostgreSQL's `timestamp with time zone + interval` adds the interval's days in the session time zone, so that adding a day across a daylight saving time transition would add 23 or 25 hours, rather than the 24 hours added by NodaTime ([#4087](https://github.com/npgsql/efcore.pg/issues/4087)). Similarly, the `Duration` components (`Days`, `Hours`, etc.) are now truncated towards zero like in NodaTime, and are also correct for durations computed in SQL, whose day and time fields can have different signs ([#4088](https://github.com/npgsql/efcore.pg/issues/4088)).
