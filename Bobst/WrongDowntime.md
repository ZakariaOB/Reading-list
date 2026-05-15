# PM Wrong Downtime - Read Notes

This document explains the reproduced Rabat3 case in a way that can be read later without needing the whole discussion context.

It covers:

- what the user-visible issue is;
- what the data proves;
- how PM constructs downtime ranges from raw downtime events;
- why a machine can show `machineSpeed > 0` while availability shows downtime;
- why the `CreationDate ASC` / `DowntimesReader` suggestion is not enough;
- what fix should be implemented;
- what tests should prove before and after the fix.

The investigation data is under:

```text
C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\188_Sprint\PMWrongDowntime\Data
```

Main captured files:

```text
downTimePage.json
histDetail.json
```

The important reproduced equipment is:

```text
GENV24G004
```

The important raw ADX table is:

```text
PerformanceDB.Events
```

## 1. Short conclusion

PM can show a downtime while the machine speed is greater than zero because the machine speed and the downtime availability come from different interpretations of data.

Machine speed says:

```text
The machine is physically producing at this timestamp.
```

Availability/OEE says:

```text
The downtime event stream currently contains an open downtime range.
```

In the reproduced case, the raw downtime events contain a same-timestamp pair where `end` is ordered before `start`:

```text
Timestamp UTC: 2026-05-14 23:33:20

end    ShopFloorProcessDefect   source=machine   origin=automatic
start  ShopFloorProcessDefect   source=machine   origin=automatic
```

The downtime range builder is a state machine. It reads events in order. If it reads `end` then `start` at the same timestamp, it can leave a false downtime open:

```text
23:33:20 end   -> close or update previous downtime state
23:33:20 start -> open a new downtime at 23:33:20
no later end   -> keep downtime open until now / end of day
```

That false open range is visible in `downTimePage.json` as:

```text
05/15/2026 05:03:20 -> 05/15/2026 23:59:59
duration: 1136.67 minutes
category: 3, ShopFloorProcessDefect
```

This is the core issue.

## 2. Business symptom

Observed behavior:

```text
PM shows ongoing downtime.
The speed chart shows the machine is producing.
OEE / availability can become 0.
After production stops or after recalculation, the displayed downtime may disappear or be replaced by production.
```

Why this is possible:

```text
Speed is a telemetry/production signal.
Downtime is calculated from downtime start/end events.
If the downtime event stream is interpreted incorrectly, downtime can be shown even while telemetry speed is positive.
```

So the contradiction is not necessarily in the raw database data. The database can contain both the `start` and `end` events. The bug can be in the order in which PM interprets them.

## 3. Important reproduced evidence

### 3.1 The daily downtime page shows a huge false downtime on May 15

From `downTimePage.json`, May 15 contains 13 downtime occurrences, with one very large occurrence:

```text
05/15/2026 05:03:20 -> 05/15/2026 23:59:59
duration: 1136.67 minutes
category: 3
isDurationEqualToTotalDuration: true
```

Normal `ShopFloorProcessDefect` / category 3 downtimes in this dataset are usually much shorter, often around 20-40 minutes. This one runs almost the whole day.

That is the false downtime that explains why availability/OEE is wrong.

### 3.2 May 14 also contains suspicious zero-duration setup downtimes

From `downTimePage.json`, May 14 contains four zero-duration occurrences at exactly the same timestamp:

```text
05/14/2026 19:52:26 -> 05/14/2026 19:52:26  duration: 0  category: 5
05/14/2026 19:52:26 -> 05/14/2026 19:52:26  duration: 0  category: 5
05/14/2026 19:52:26 -> 05/14/2026 19:52:26  duration: 0  category: 5
05/14/2026 19:52:26 -> 05/14/2026 19:52:26  duration: 0  category: 5
```

Category 5 corresponds to `Setup` in the ADX results.

Zero-duration setup downtimes are allowed by the business behavior, but they are dangerous for the current calculation because timestamp alone cannot tell which event should be processed first.

A valid zero-duration downtime looks like this:

```text
T start setup
T end setup
```

But if the system reads it as:

```text
T end setup
T start setup
```

the state machine may open a false ongoing downtime.

### 3.3 Querying the wrong UTC window returned no data

The first raw ADX query searched around:

```text
2026-05-15T05:00:00Z .. 2026-05-15T05:10:00Z
```

It returned no rows.

That did not disprove the theory. The issue was a time conversion mismatch.

The JSON/UI daily timestamps are site/local converted, while ADX `Timestamp` is UTC. The false downtime start shown in the daily JSON is:

```text
05/15/2026 05:03:20
```

The matching UTC raw event timestamp is:

```text
2026-05-14T23:33:20Z
```

After converting the window and querying:

```text
2026-05-14T23:25:00Z .. 2026-05-14T23:45:00Z
```

ADX returned the relevant rows.

### 3.4 ADX proves the bad order

The ADX query ordered by:

```text
Timestamp ASC, creationDate ASC
```

It returned these rows around the false downtime:

```text
Timestamp UTC          creationDate UTC                downtimeType  category                 source   origin
2026-05-14 23:26:43    2026-05-14 23:26:53.1649678     end           ShopFloorProcessDefect   machine  automatic
2026-05-14 23:26:43    2026-05-14 23:26:53.1651262     start         ShopFloorProcessDefect   machine  automatic
2026-05-14 23:26:43    2026-05-14 23:26:53.1654008     end           ShopFloorProcessDefect   machine  automatic
2026-05-14 23:26:43    2026-05-14 23:26:53.1655376     start         Setup                    machine  automatic

2026-05-14 23:33:20    2026-05-14 23:33:23.2503305     end           ShopFloorProcessDefect   machine  automatic
2026-05-14 23:33:20    2026-05-14 23:33:23.2506955     start         ShopFloorProcessDefect   machine  automatic
```

The important pair is:

```text
2026-05-14 23:33:20 end   ShopFloorProcessDefect machine automatic
2026-05-14 23:33:20 start ShopFloorProcessDefect machine automatic
```

This exact timestamp maps to the false daily downtime start:

```text
UTC:        2026-05-14 23:33:20
Daily JSON: 2026-05-15 05:03:20
```

The event stream says `end` first, then `start`, for the same timestamp and same logical downtime identity.

This is the smoking gun.

## 4. How PM downtime is constructed

This section explains the simplified but accurate construction path.

### 4.1 Raw events

PM stores downtime events as raw rows. A downtime event has fields like:

```text
Timestamp
CreationDate
DowntimeType: start or end
Category
Source
Origin
Reason
ReasonCode
CustomReasonCode
CorrelationId
Deleted
```

The important distinction is:

```text
Timestamp    = when the event happened on the machine timeline
CreationDate = when the event row was created/ingested/recorded
```

For downtime calculation, `Timestamp` is the business timeline. `CreationDate` is only a tie-breaker. It does not always represent the business order.

### 4.2 Repository fetches downtime events

The hot path in `Backend/Repositories/HotEventRepository.cs` fetches downtime events with:

```text
ORDER BY c.timestamp ASC, c.creationDate ASC
```

Then it sorts again in memory:

```csharp
downtimesResult = downtimesResult
	.OrderBy(_ => _.Timestamp)
	.ThenBy(_ => _.CreationDate)
	.ToList();
```

The warm path in `Backend/Repositories/WarmEventRepository.cs` does the same idea with ADX:

```text
order by Timestamp asc, todatetime(EventData.creationDate) asc
```

Then it also sorts again in memory:

```csharp
downtimesResult = downtimesResult
	.OrderBy(_ => _.Timestamp)
	.ThenBy(_ => _.CreationDate)
	.ToList();
```

Then the repository splits events by source:

```text
machine downtime events
user downtime events
intelligent downtime events
```

In the reproduced case, the bad rows are:

```text
source = machine
origin = automatic
```

### 4.3 DowntimeService prepares the stream

In `Backend/Services/ManagementServices/DowntimeManagement/DowntimeService.cs`, `PopulateWithDataByCategory(...)` does this:

```text
1. Load downtime and job events.
2. Extract jobs.
3. Generate machine downtime occurrences.
4. Generate user downtime occurrences.
5. Generate intelligent downtime occurrences.
6. Merge machine, intelligent, and user downtime occurrences.
7. Split by shifts if required.
```

The key method is:

```text
DowntimeService.GenerateDowntimeOccurences(...)
```

This method does several important things:

```text
1. Removes deleted/update shadow events.
2. Finds a first downtime before the requested range.
3. Extends the query range for OEE by one day before and after.
4. Filters events.
5. Sorts by Timestamp only.
6. Groups by CorrelationId and flattens the groups.
7. Removes consecutive duplicate end events.
8. Calls DowntimeExtension.GenerateDowntimeOccurrence(...).
```

The current filter is suspicious:

```csharp
var downtimes = dbDowntimes
	.Where(dt => dt.Timestamp >= startPeriod || dt.Timestamp <= endPeriod)
	.OrderBy(_ => _.Timestamp)
	.ToList();
```

For a normal interval where `startPeriod < endPeriod`, almost every timestamp satisfies:

```text
timestamp >= start OR timestamp <= end
```

Usually interval logic is:

```text
timestamp >= start AND timestamp <= end
```

However, this `OR` is probably not the main root cause of the reproduced case. The root cause is the same-timestamp `end,start` ordering. Still, the `OR` can amplify the problem by keeping more events in the state machine than expected.

### 4.4 The occurrence builder is a state machine

The main occurrence builder is in:

```text
Backend/Services/ManagementServices/DowntimeManagement/Helper/DowntimeExtension.cs
```

Method:

```text
GenerateDowntimeOccurrence(...)
```

It starts by sorting the input again:

```csharp
downtimes = downtimes.OrderBy(x => x.Timestamp).ToList();
```

This is critical.

It sorts by `Timestamp` only. If multiple events have the same timestamp, their relative order is whatever came from the previous layer. The method does not repair or define semantic order inside equal timestamps.

Then it loops over the events:

```text
for each downtime event:
	if type is start:
		ProcessStartDowntimeOccurrences(...)
	if type is end:
		GenerateDowntimeOccurrenceWhenTypeIsEnd(...)
```

The mental model is:

```text
start -> open a downtime occurrence
end   -> close the latest opened downtime occurrence
```

This model works only if the stream order is correct.

### 4.5 What happens when the current event is `start`

In `ProcessStartDowntimeOccurrences(...)`, the code creates a new `DowntimeEntity` with:

```text
StartTimestamp = current event timestamp
OriginalStartTimestamp = current event timestamp
Category = current event category
Source = current event source
Origin = current event origin
ReasonCode = current event reason code
CorrelationId = current event correlation id, if not empty
```

If the `start` is the last downtime event in the list, the code sets an artificial end timestamp:

```text
if current start is last event:
	end = now, if it is a live/current day case
	or end = end of day, if it is a historical day case
```

This is why a false open downtime becomes visible as:

```text
start -> now
```

or:

```text
start -> 23:59:59
```

In the reproduced `downTimePage.json`, the false occurrence is historical/day-bounded:

```text
05:03:20 -> 23:59:59
```

### 4.6 What happens when the current event is `end`

In `GenerateDowntimeOccurrenceWhenTypeIsEnd(...)`, an `end` event can do two things:

1. If it is the first event in the list, it creates a downtime from either:

```text
firstDowntimeBeforeDate -> current end timestamp
```

or:

```text
start of day -> current end timestamp
```

2. If there is already a downtime occurrence, it updates the last occurrence end timestamp:

```text
last occurrence EndTimestamp = current end timestamp
```

That means an `end` can close something, but it does not prevent a following same-timestamp `start` from opening a new downtime.

### 4.7 Why `end` then `start` creates a false open downtime

For a normal downtime:

```text
10:00 start
10:05 end
```

The state machine builds:

```text
10:00 -> 10:05
```

For a valid zero-duration downtime:

```text
10:00 start
10:00 end
```

The state machine builds:

```text
10:00 -> 10:00
```

But if the valid zero-duration pair is read in the opposite order:

```text
10:00 end
10:00 start
```

The state machine behaves like:

```text
10:00 end   -> close/update previous state
10:00 start -> open a new downtime at 10:00
no later end -> keep it open until now/end-of-day
```

That is exactly what happened in the reproduced Rabat3 case.

## 5. Why the machine can be producing while downtime is shown

The speed chart and availability chart are not the same calculation.

The speed chart can show:

```text
Speed = 600 meters/min
Target = 700 meters/min
```

This means production telemetry exists and the machine is physically moving/producing.

The availability chart can still show downtime because it uses downtime occurrence ranges.

If the occurrence builder creates this false range:

```text
05:03:20 -> 23:59:59 ShopFloorProcessDefect
```

then PM treats that whole period as unavailable, even if speed data inside the same period is positive.

So the contradiction is:

```text
Telemetry path: machine is producing.
Downtime path: downtime range is open.
```

The bug is in the downtime path interpretation.

## 6. Why the two views are useful

The two views are useful because they expose the same inconsistency from different angles.

### 6.1 `downTimePage.json`

This file shows day-level downtime occurrences.

It reveals:

```text
May 15 total downtime = 1347.23 minutes
May 15 occurrences = 13
Last May 15 occurrence = 05:03:20 -> 23:59:59, 1136.67 minutes
```

That is abnormal because one occurrence consumes almost the whole day.

### 6.2 `histDetail.json`

This file shows detailed timeline information for a day window:

```text
equipmentId: GENV24G004
startDate: 2026-05-13T18:30:00Z
endDate:   2026-05-14T18:30:00Z
```

It contains jobs, machine production data, production status, shifts, and downtime information for the detailed timeline page.

This file helped identify that timestamps must be handled carefully because the API/UI view and ADX raw event view are not displayed in the same time basis.

### 6.3 What the difference tells us

The difference between the views is not just a frontend display problem.

It tells us:

```text
The backend can construct a downtime range that conflicts with production/speed data.
Different endpoints/views may use different windows or post-processing.
The root problem remains the raw downtime event interpretation.
```

## 7. Why `CreationDate ASC` is not enough

A suggested fix was to change `DowntimesReader` from:

```csharp
.ThenByDescending(e => e.CreationDate)
```

to:

```csharp
.ThenBy(e => e.CreationDate)
```

This is probably a reasonable cleanup, but it does not fix the reproduced case.

Why?

The ADX query already used:

```text
Timestamp ASC, creationDate ASC
```

And it still returned:

```text
23:33:20 end   ShopFloorProcessDefect creation 23:33:23.2503305
23:33:20 start ShopFloorProcessDefect creation 23:33:23.2506955
```

The `end` really was created slightly before the `start`.

So sorting by creation date ascending preserves the bad order:

```text
end before start
```

The issue is not only that `CreationDate` is descending somewhere. The deeper issue is that `CreationDate` is not a safe semantic tie-breaker for same-timestamp downtime events.

## 8. Why `DowntimesReader` is also suspicious but not the main fix

`Backend/Services/Tools/DowntimesReader.cs` currently sorts with:

```csharp
.OrderBy(e => e.Timestamp)
.ThenByDescending(e => e.DowntimeType)
.ThenByDescending(e => e.CreationDate)
```

The enum is:

```csharp
public enum DowntimeType
{
	start,
	end,
}
```

So numerically:

```text
start = 0
end   = 1
```

`ThenByDescending(e => e.DowntimeType)` puts:

```text
end before start
```

That might be intentional for some back-to-back downtime boundary cases, but it is dangerous for zero-duration same-logical-downtime pairs.

Changing only creation date order gives:

```text
Timestamp ASC
DowntimeType DESC
CreationDate ASC
```

It still puts `end` before `start` for the same timestamp.

So the `DowntimesReader` fix can be part of cleanup, but it is not enough for the reproduced Rabat3 issue.

## 9. The real issue

The real issue is this:

```text
PM has no deterministic semantic ordering rule for same-timestamp downtime events.
```

Current code relies on combinations of:

```text
Timestamp
CreationDate
DowntimeType
CorrelationId grouping
Repository-specific ordering
Stable LINQ sorting side effects
```

But the business meaning is:

```text
Some same-timestamp start/end pairs are the same downtime and should produce a zero-duration range.
Some same-timestamp end/start pairs are different downtimes and should preserve a real boundary.
```

Those two cases need different ordering.

### 9.1 Same logical downtime pair

Example:

```text
T end   ShopFloorProcessDefect machine automatic
T start ShopFloorProcessDefect machine automatic
```

If these two events represent the same logical downtime pair, PM must interpret them as:

```text
T start
T end
```

Expected result:

```text
T -> T
no open downtime
```

### 9.2 Different logical downtime boundary

Example:

```text
T end   ShopFloorProcessDefect machine automatic
T start Setup                  machine automatic
```

This may be a real transition:

```text
one downtime ends
another downtime starts
```

For this case, `end` before `start` may be correct.

This is why we cannot globally sort all `start` events before all `end` events.

## 10. Suggested fix

### 10.1 Fix principle

Create one shared semantic ordering rule for downtime occurrence generation.

The rule must run before the state machine builds ranges.

It should be used in or before:

```text
DowntimeService.GenerateDowntimeOccurences(...)
DowntimeExtension.GenerateDowntimeOccurrence(...)
```

The safest place is probably in `DowntimeService.GenerateDowntimeOccurences(...)`, before the duplicate consecutive `end` cleanup and before calling `GenerateDowntimeOccurrence(...)`.

Why before duplicate `end` cleanup?

Because the reproduced cluster has patterns like:

```text
end SFPD
start SFPD
end SFPD
start Setup
```

If semantic ordering moves same-logical-pair starts before ends, duplicate ends can become adjacent and be removed by the existing duplicate-end cleanup.

### 10.2 Logical identity for pairing

Use `CorrelationId` if it exists and is not empty.

But in the reproduced data, `correlationId` is empty. So the fix must have a fallback logical identity.

Recommended fallback identity:

```text
Timestamp
Source
Origin
Category
Reason
ReasonCode
CustomReasonCode
Fault, if used as automatic reason
JobId, if relevant and stable
```

The minimal identity for the reproduced case is:

```text
Timestamp
Source = machine
Origin = automatic
Category = ShopFloorProcessDefect
```

But using only category/source/origin may be too broad. Include reason fields when available.

### 10.3 Proposed ordering behavior

For events with different timestamps:

```text
Timestamp ASC
```

For events with the same timestamp:

```text
1. Group events by logical identity.
2. Inside a logical identity group that has both start and end:
	   start before end
	   then CreationDate ASC as tie-breaker
3. For logical identity groups that are truly different:
	   preserve deterministic boundary behavior
	   end-only groups can remain before start-only groups
	   CreationDate ASC can be used as a final tie-breaker
```

In plain English:

```text
Same timestamp and same downtime meaning:
	start must be read before end.

Same timestamp but different downtime meaning:
	do not blindly reorder into start before end.
```

### 10.4 Important: do not globally sort all starts before ends

This would fix the reproduced same-pair bug but could break real back-to-back downtimes.

Example of a real boundary:

```text
09:00 start Downtime A
10:00 end   Downtime A
10:00 start Downtime B
11:00 end   Downtime B
```

At `10:00`, the correct order is:

```text
end A
start B
```

If we globally force `start before end`, we could create an overlap or merge/confuse the two downtimes.

So the fix must be same-logical-identity aware.

### 10.5 Review the interval filter separately

This line should be reviewed:

```csharp
dt.Timestamp >= startPeriod || dt.Timestamp <= endPeriod
```

It probably should be:

```csharp
dt.Timestamp >= startPeriod && dt.Timestamp <= endPeriod
```

But this should be tested separately because the method intentionally extends the OEE window by one day before/after.

This is not the primary reproduced root cause. The primary root cause is equal-timestamp event ordering.

## 11. Suggested implementation shape

This is intentionally pseudocode, not final code.

```csharp
private static List<DbDowntime> OrderDowntimesForOccurrenceGeneration(IEnumerable<DbDowntime> downtimes)
{
	return downtimes
		.Where(d => d != null)
		.GroupBy(d => d.Timestamp)
		.OrderBy(g => g.Key)
		.SelectMany(OrderSameTimestampDowntimes)
		.ToList();
}

private static IEnumerable<DbDowntime> OrderSameTimestampDowntimes(IEnumerable<DbDowntime> sameTimestampDowntimes)
{
	// 1. Build logical keys.
	// 2. If a logical group contains start and end, return start before end.
	// 3. If groups are different logical downtimes, keep deterministic boundary order.
	// 4. Use CreationDate only as a final tie-breaker.
}
```

A possible logical key:

```text
if CorrelationId is not empty:
	CorrelationId
else:
	Source + Origin + Category + Reason + ReasonCode + CustomReasonCode + Fault
```

For the reproduced pair:

```text
end   ShopFloorProcessDefect machine automatic
start ShopFloorProcessDefect machine automatic
```

The ordering helper should output:

```text
start ShopFloorProcessDefect machine automatic
end   ShopFloorProcessDefect machine automatic
```

Then the state machine produces:

```text
05:03:20 -> 05:03:20
```

and no false open downtime.

### 11.1 Concrete code fix location

Because Rabat3 is an extract from production, the fix should be implemented close to the occurrence generation path and protected by focused tests.

Recommended new file:

```text
Backend/Services/ManagementServices/DowntimeManagement/Helper/DowntimeEventOrderingHelper.cs
```

Why a new helper file:

```text
The ordering rule is business logic for downtime events.
It should not be hidden inside a repository query.
It should be reusable from both DowntimeService and DowntimeExtension.
The local codebase convention prefers focused helper files for focused helper classes.
```

Recommended first call site:

```text
Backend/Services/ManagementServices/DowntimeManagement/DowntimeService.cs
method: GenerateDowntimeOccurences(...)
```

Put the semantic ordering after the date filtering and before this block:

```text
remove consecutive duplicate end events
```

Reason:

```text
The duplicate-end cleanup should run after the same-timestamp semantic order is fixed.
Otherwise it may remove or keep events based on the wrong accidental order.
```

Recommended second call site:

```text
Backend/Services/ManagementServices/DowntimeManagement/Helper/DowntimeExtension.cs
method: GenerateDowntimeOccurrence(...)
```

Replace the timestamp-only sort there too, as a defensive measure. Some tests and code paths call `GenerateDowntimeOccurrence(...)` directly, bypassing `DowntimeService.GenerateDowntimeOccurences(...)`.

### 11.2 Suggested helper code

This is the concrete shape I would start with. It may need small style adjustments when implemented, but the algorithm is the important part.

```csharp
using PerformanceManagement.DataObjects;
using PerformanceManagement.DataObjects.DowntimeEntities;
using DbDowntime = PerformanceManagement.DataObjects.DowntimeEntities.Downtime;

namespace PerformanceManagement.Services.ManagementServices.DowntimeManagement.Helper;

internal static class DowntimeEventOrderingHelper
{
	public static List<DbDowntime> OrderForOccurrenceGeneration(this IEnumerable<DbDowntime> downtimes)
	{
		return downtimes?
				   .Where(downtime => downtime != null)
				   .GroupBy(downtime => downtime.Timestamp)
				   .OrderBy(group => group.Key)
				   .SelectMany(OrderSameTimestampDowntimes)
				   .ToList()
			   ?? new List<DbDowntime>();
	}

	private static IEnumerable<DbDowntime> OrderSameTimestampDowntimes(IEnumerable<DbDowntime> downtimes)
	{
		return downtimes
			.GroupBy(GetLogicalIdentity)
			.OrderBy(GetLogicalGroupRank)
			.ThenBy(group => group.Min(downtime => downtime.CreationDate))
			.SelectMany(OrderLogicalGroup);
	}

	private static IEnumerable<DbDowntime> OrderLogicalGroup(IEnumerable<DbDowntime> downtimes)
	{
		var group = downtimes.ToList();
		var hasStart = group.Any(downtime => downtime.DowntimeType == DowntimeType.start);
		var hasEnd = group.Any(downtime => downtime.DowntimeType == DowntimeType.end);

		if (hasStart && hasEnd)
		{
			return group
				.OrderBy(downtime => downtime.DowntimeType == DowntimeType.start ? 0 : 1)
				.ThenBy(downtime => downtime.CreationDate);
		}

		return group
			.OrderBy(downtime => downtime.DowntimeType == DowntimeType.end ? 0 : 1)
			.ThenBy(downtime => downtime.CreationDate);
	}

	private static int GetLogicalGroupRank(IGrouping<DowntimeLogicalIdentity, DbDowntime> group)
	{
		var hasStart = group.Any(downtime => downtime.DowntimeType == DowntimeType.start);
		var hasEnd = group.Any(downtime => downtime.DowntimeType == DowntimeType.end);

		if (hasStart && hasEnd)
		{
			return 0;
		}

		if (hasEnd)
		{
			return 1;
		}

		return 2;
	}

	private static DowntimeLogicalIdentity GetLogicalIdentity(DbDowntime downtime)
	{
		if (downtime.CorrelationId.HasValue && downtime.CorrelationId.Value != Guid.Empty)
		{
			return new DowntimeLogicalIdentity(
				downtime.CorrelationId,
				default,
				default,
				default,
				string.Empty,
				string.Empty,
				string.Empty,
				string.Empty);
		}

		return new DowntimeLogicalIdentity(
			null,
			downtime.Source,
			downtime.Origin,
			downtime.Category,
			Normalize(downtime.Reason),
			Normalize(downtime.ReasonCode),
			Normalize(downtime.CustomReasonCode),
			Normalize(downtime.Fault));
	}

	private static string Normalize(string value)
	{
		return value?.Trim() ?? string.Empty;
	}

	private readonly record struct DowntimeLogicalIdentity(
		Guid? CorrelationId,
		MachineEventSource Source,
		DowntimeOrigin Origin,
		DowntimeCategory? Category,
		string Reason,
		string ReasonCode,
		string CustomReasonCode,
		string Fault);
}
```

### 11.3 Why the helper orders groups this way

The helper works in two levels.

First, it groups by `Timestamp`:

```text
Different timestamps are easy: oldest timestamp first.
```

Then, inside one timestamp, it groups by logical downtime identity:

```text
CorrelationId if present.
Otherwise Source + Origin + Category + Reason + ReasonCode + CustomReasonCode + Fault.
```

For a logical group containing both `start` and `end`, it forces:

```text
start before end
```

That fixes Rabat3:

```text
input from ADX:
	end   ShopFloorProcessDefect machine automatic
	start ShopFloorProcessDefect machine automatic

ordered for occurrence generation:
	start ShopFloorProcessDefect machine automatic
	end   ShopFloorProcessDefect machine automatic
```

For logical groups that are different and contain only an `end` or only a `start`, it keeps boundary-friendly behavior:

```text
end-only groups before start-only groups
```

That protects legitimate back-to-back cases:

```text
T end   Downtime A
T start Downtime B
```

### 11.4 Change in DowntimeService

Current code:

```csharp
var downtimes = dbDowntimes.Where(dt => dt.Timestamp >= startPeriod || dt.Timestamp <= endPeriod).OrderBy(_ => _.Timestamp).ToList();
downtimes.GroupBy(d => d.CorrelationId!).ForEach(d => downtimebyCorrelation.AddRange([.. d]));

downtimes = downtimebyCorrelation.Count != 0 ? downtimebyCorrelation : downtimes;
```

Suggested replacement for the first targeted fix:

```csharp
var downtimes = dbDowntimes
	.Where(dt => dt.Timestamp >= startPeriod || dt.Timestamp <= endPeriod)
	.OrderForOccurrenceGeneration();
```

Important notes:

```text
Do not change the OR filter in the same first fix unless tests explicitly cover it.
The OR filter is suspicious, but it is a separate behavior change.
The Rabat3 root cause is same-timestamp ordering, so keep the first fix targeted.
```

This also removes the old `CorrelationId` grouping/flattening from this method. That old grouping is risky because it can disturb chronological order across different correlation ids and does not help when `CorrelationId` is empty, as in Rabat3.

### 11.5 Change in DowntimeExtension

Current code in `GenerateDowntimeOccurrence(...)`:

```csharp
downtimes = downtimes.OrderBy(x => x.Timestamp).ToList();
```

Suggested replacement:

```csharp
downtimes = downtimes.OrderForOccurrenceGeneration();
```

Why this second change matters:

```text
DowntimeService should order correctly before duplicate-end cleanup.
DowntimeExtension should also defend itself because tests and other code paths can call GenerateDowntimeOccurrence directly.
```

### 11.6 What this does to the Rabat3 production extract

Raw ADX order for the false long downtime:

```text
2026-05-14 23:33:20 end   ShopFloorProcessDefect machine automatic
2026-05-14 23:33:20 start ShopFloorProcessDefect machine automatic
```

Order after the helper:

```text
2026-05-14 23:33:20 start ShopFloorProcessDefect machine automatic
2026-05-14 23:33:20 end   ShopFloorProcessDefect machine automatic
```

Occurrence result after the state machine:

```text
2026-05-15 05:03:20 -> 2026-05-15 05:03:20 local
```

Instead of:

```text
2026-05-15 05:03:20 -> 2026-05-15 23:59:59 local
```

That removes the false 1136-minute downtime while preserving the raw events.

## 12. Tests to add before fixing

Add tests first. The most important tests should be close to:

```text
Backend/Services.Tests/ManagementServices/DowntimeTest/DowntimeExtensionTest.cs
```

or wherever `DowntimeService.GenerateDowntimeOccurences(...)` behavior can be tested with realistic ordering.

### Test 1: exact reproduced pair

Input:

```text
T end   ShopFloorProcessDefect source=machine origin=automatic creation=C1
T start ShopFloorProcessDefect source=machine origin=automatic creation=C2
```

Expected:

```text
No occurrence from T to end-of-day.
Either no occurrence or a zero-duration occurrence T -> T, depending on expected product behavior.
```

This is the must-have test.

### Test 2: same timestamp setup zero-duration pair

Input:

```text
T end   Setup source=machine origin=automatic creation=C1
T start Setup source=machine origin=automatic creation=C2
```

Expected:

```text
No false ongoing downtime.
```

### Test 3: multiple same-timestamp generated pairs

Input similar to the ADX cluster:

```text
T end   ShopFloorProcessDefect machine automatic
T start ShopFloorProcessDefect machine automatic
T end   ShopFloorProcessDefect machine automatic
T start Setup                  machine automatic
```

Expected:

```text
No false ShopFloorProcessDefect downtime from T to end-of-day.
Setup start behavior remains correct according to following events.
Duplicate ends are handled deterministically.
```

### Test 4: legitimate back-to-back different downtimes

Input:

```text
T0 start ShopFloorProcessDefect
T1 end   ShopFloorProcessDefect
T1 start Setup
T2 end   Setup
```

Expected:

```text
ShopFloorProcessDefect: T0 -> T1
Setup:                  T1 -> T2
```

This test protects against the naive `start before end for everything` fix.

### Test 5: correlation id present

Input:

```text
T end   CorrelationId=A
T start CorrelationId=A
T end   CorrelationId=B
T start CorrelationId=B
```

Expected:

```text
Events are paired by correlation id.
No false ongoing downtime.
```

### Test 6: correlation id missing

Input:

```text
T end   same source/origin/category/reason fields, no correlation id
T start same source/origin/category/reason fields, no correlation id
```

Expected:

```text
Fallback logical key is used.
No false ongoing downtime.
```

This test is important because the reproduced Rabat3 case has empty `correlationId`.

### Test 7: repository order is not enough

Input already sorted as repository returns it:

```text
Timestamp ASC, CreationDate ASC
T end creation C1
T start creation C2
```

Expected:

```text
The occurrence generation still fixes the semantic order.
```

This protects against the false belief that `CreationDate ASC` solves the issue.

## 13. ADX queries used for proof

### 13.1 Query around false downtime start

```kusto
Events
| where EquipmentId == "GENV24G004"
| where EventType == "DowntimeEvent"
| where Timestamp between (datetime('2026-05-14T23:25:00Z') .. datetime('2026-05-14T23:45:00Z'))
| extend downtimeType = tostring(EventData.downtimeType)
| extend creationDate = todatetime(EventData.creationDate)
| extend category = tostring(EventData.category)
| extend source = tostring(EventData.source)
| extend origin = tostring(EventData.origin)
| extend reason = tostring(EventData.reason)
| extend reasonCode = tostring(EventData.reasonCode)
| extend customReasonCode = tostring(EventData.customReasonCode)
| extend correlationId = tostring(EventData.correlationId)
| project Timestamp, creationDate, downtimeType, category, source, origin, reason, reasonCode, customReasonCode, correlationId
| order by Timestamp asc, creationDate asc
```

Important result:

```text
2026-05-14 23:33:20 end   ShopFloorProcessDefect machine automatic
2026-05-14 23:33:20 start ShopFloorProcessDefect machine automatic
```

### 13.2 Query to find all same-timestamp start/end collisions

```kusto
let equipmentId = "GENV24G004";
let startUtc = datetime('2026-05-14T00:00:00Z');
let endUtc = datetime('2026-05-15T06:00:00Z');
Events
| where EquipmentId == equipmentId
| where EventType == "DowntimeEvent"
| where Timestamp between (startUtc .. endUtc)
| extend downtimeType = tostring(EventData.downtimeType)
| extend creationDate = todatetime(EventData.creationDate)
| extend source = tostring(EventData.source)
| extend origin = tostring(EventData.origin)
| extend category = tostring(EventData.category)
| extend reason = tostring(EventData.reason)
| extend reasonCode = tostring(EventData.reasonCode)
| extend customReasonCode = tostring(EventData.customReasonCode)
| extend correlationId = tostring(EventData.correlationId)
| summarize
	starts = countif(downtimeType == "start"),
	ends = countif(downtimeType == "end"),
	firstStartCreation = minif(creationDate, downtimeType == "start"),
	firstEndCreation = minif(creationDate, downtimeType == "end"),
	rows = make_list(pack(
		"type", downtimeType,
		"creationDate", creationDate,
		"source", source,
		"origin", origin,
		"category", category,
		"reason", reason,
		"reasonCode", reasonCode,
		"customReasonCode", customReasonCode,
		"correlationId", correlationId))
  by Timestamp, source, origin, category, reason, reasonCode, customReasonCode, correlationId
| where starts > 0 and ends > 0
| extend endCreatedBeforeStart = firstEndCreation < firstStartCreation
| order by Timestamp asc
```

Purpose:

```text
Find same-timestamp start/end collisions and prove whether creationDate can put end before start.
```

### 13.3 Query speed / production at same time

The exact telemetry table/field may differ, but the goal is:

```kusto
let equipmentId = "GENV24G004";
let startUtc = datetime('2026-05-14T23:25:00Z');
let endUtc = datetime('2026-05-14T23:45:00Z');
Telemetry
| where EquipmentId == equipmentId
| where Timestamp between (startUtc .. endUtc)
| where Name in ("machinespeed", "machineSpeed", "speed")
| project Timestamp, Name, Value
| order by Timestamp asc
```

Purpose:

```text
Show positive speed during a period that PM also marks as downtime.
```

## 14. How to explain the story simply

Use this story when explaining the bug to someone else:

```text
The database is not necessarily missing the downtime stop event.
The stop event can be present.

The problem is that the downtime engine reads same-timestamp events like a stream.
For one reproduced machine, ADX returns an `end` before a matching `start` at the exact same timestamp.

Because the occurrence builder reads `end` first, then `start`, it opens a new downtime at that timestamp.
No later matching end is found in the selected day, so PM stretches that downtime until 23:59:59.

That creates a fake downtime of 1136 minutes.
The speed chart can still show 600 m/min because speed comes from another data path.
Availability/OEE becomes wrong because it trusts the fake downtime range.
```

One-line story:

```text
Correct raw events + wrong same-timestamp order = false open downtime.
```

## 15. What not to do

Do not assume:

```text
The downtime stop event is missing.
```

The reproduced ADX query shows the event exists.

Do not assume:

```text
CreationDate ASC fixes the issue.
```

The reproduced ADX query already uses creation date ascending and still returns `end` before `start`.

Do not apply:

```text
Always sort start before end.
```

That can break legitimate back-to-back downtimes where one downtime ends and a different one starts at the same timestamp.

Do not rely only on:

```text
CorrelationId
```

The reproduced data has empty correlation ids.

Do not fix only:

```text
DowntimesReader.CreationDate descending
```

It is related cleanup, but not sufficient for this reproduced case.

## 16. Recommended next action

1. Add a failing unit test reproducing the exact Rabat3 pair:

```text
T end   ShopFloorProcessDefect machine automatic
T start ShopFloorProcessDefect machine automatic
```

Expected:

```text
No T -> 23:59:59 downtime.
```

2. Add a semantic same-timestamp ordering helper before occurrence generation.

3. Re-run the new tests and existing downtime tests.

4. Review the `OR` interval filter in a separate commit/test path.

5. Optionally align `DowntimesReader` creation date direction, but do not treat that as the root fix.

## 17. Final working conclusion

The Rabat3 reproduction confirms the suspected event ordering bug.

The strongest proof is this pair:

```text
2026-05-14 23:33:20 end   ShopFloorProcessDefect machine automatic
2026-05-14 23:33:20 start ShopFloorProcessDefect machine automatic
```

It maps directly to the false downtime:

```text
05/15/2026 05:03:20 -> 05/15/2026 23:59:59
```

The fix should make same-timestamp ordering semantic:

```text
same timestamp + same logical downtime identity:
	start before end

same timestamp + different logical downtime identity:
	preserve real boundary behavior
```

That is the safest route to remove the false open downtime without breaking legitimate back-to-back downtime transitions.
