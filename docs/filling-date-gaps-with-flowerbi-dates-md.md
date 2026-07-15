# Filling Date Gaps in Time-Series Data with flowerbi-dates

## The Problem: Missing Time Periods in Query Results

Time-series chart queries often return data only for periods that have records. A monthly sales query, for example, returns nothing for months where no sales occurred. When rendered directly, these gaps produce a broken line chart — the x-axis jumps discontinuously or omits entire periods, misleading the viewer about the data's continuity.

The solution is to fill in those missing periods with synthetic records containing zero or null for the metric, creating a continuous sequence that renders as a proper time-series chart.

## Installation

The `flowerbi-dates` package is published on npm as `@flowerbi/dates`. Install it via your package manager:

```bash
npm install @flowerbi/dates
```

The package has a peer dependency on `moment` (`^2.29.4`), which you will also need to install.

## The `fillDates` Function

The core export is the `fillDates` function. It accepts an options object with the following parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `records` | `T[]` | The source records to fill gaps within |
| `getDate` | `(record: T) => FillDate` | Extracts a date value from each record |
| `fill` | `(dateText: string, record: T | undefined) => R` | Generates the output record for a given date |
| `type` | `FillDateType` (optional) | The date unit type; auto-detected if omitted |
| `min` | `FillDate` (optional) | Minimum date to generate; padded if set |
| `max` | `FillDate` (optional) | Maximum date to generate; padded if set |

The `FillDate` type accepts `Date | string | number | Moment`, giving flexibility in how dates are supplied. Numeric values are converted to strings before parsing to avoid being interpreted as milliseconds since 1970.

### Using the `fill` Callback

The `fill` callback is called for every output record in sequence. It receives:
- **`dateText`**: the formatted date string (e.g. `"Apr 2020"`)
- **`record`**: the original record from the input if a real record exists for that date, or **`undefined`** if this is a gap-fill entry

You can use this to assign a label to every record, and apply sensible defaults (such as zero) to gap entries while preserving real values from real records via the spread operator.

### The `min` and `max` Range

When you supply `min` and/or `max`, records are also generated before the earliest input record or after the latest input record to fully cover the desired range. The values are rounded down by the chosen `FillDateType`, so you don't need to specify an exact boundary date.

## Built-in Date Types

The `dateTypes` object exposes four standard `FillDateType` implementations, each defining three operations: rounding a date down to the nearest unit boundary, formatting it to a display string, and incrementing it by one unit.

### `dateTypes.days`
- Rounds to the start of the day
- Formats using moment's `"ll"` format (e.g. `"Apr 1, 2020"`)
- Increments by one day

### `dateTypes.months`
- Rounds to the start of the month
- Formats as `"MMM YYYY"` (e.g. `"Apr 2020"`)
- Increments by one month

### `dateTypes.quarters`
- Rounds to the start of the quarter
- Formats as a range like `"Apr-Jun 2020"` (first month, last month, year)
- Increments by three months

### `dateTypes.years`
- Rounds to the start of the year
- Formats as `"YYYY"` (e.g. `"2020"`)
- Increments by one year

## Automatic Type Detection with `detectDateType`

If you do not pass a `type` parameter, `fillDates` automatically calls `detectDateType` on the parsed dates from your records. The detection logic inspects the set of dates and selects the most specific unit that fits all of them:

1. If not all dates fall on the 1st of the month, it selects **days**.
2. Otherwise, if not all months are multiples of three (Jan, Apr, Jul, Oct), it selects **months**.
3. Otherwise, if not all dates are January 1st, it selects **quarters**.
4. Otherwise, it selects **years**.

You can always override detection by passing an explicit `type`.

## Concrete Example: Filling Monthly Sales Data

Suppose you query monthly sales totals and receive records for April, June, and July, but May is missing:

```typescript
import { fillDates, dateTypes } from '@flowerbi/dates';

const records = [
    { date: '2020-04-01', totalSales: 10 },
    { date: '2020-06-01', totalSales: 4 },
    { date: '2020-07-01', totalSales: 9 },
];

const filled = fillDates({
    records,
    type: dateTypes.months,
    getDate: (rec) => rec.date,
    fill: (label, rec) => ({
        label,
        totalSales: 0,
        ...rec,
    }),
});
```

Result:

```typescript
[
    { label: 'Apr 2020', date: '2020-04-01', totalSales: 10 },
    { label: 'May 2020', totalSales: 0 },
    { label: 'Jun 2020', date: '2020-06-01', totalSales: 4 },
    { label: 'Jul 2020', date: '2020-07-01', totalSales: 9 },
]
```

Notice how:
- Every output record gets a `label` property from the formatted date.
- May (the gap) has `totalSales: 0` and no `date` property (the spread of `rec` into gap entries produces nothing since `rec` is `undefined`).
- Real records keep their original `totalSales` value via `...rec` after the default `totalSales: 0`.

### Adding Range Padding

To extend the series with leading and trailing months that have no data, supply `min` and `max`:

```typescript
const filled = fillDates({
    records,
    type: dateTypes.months,
    getDate: (rec) => rec.date,
    fill: (label, rec) => ({
        label,
        totalSales: 0,
        ...rec,
    }),
    min: '2020-02-15',
    max: '2020-12-02',
});
```

This produces two leading gap months (Feb, Mar) before the first record, all the real months, and a trailing gap month (Dec) after the last record.

## The `FillDateType` Interface for Custom Date Handling

If the built-in date types do not meet your needs — for example you need weekly intervals, fiscal quarters, or custom formatting — implement the `FillDateType` interface yourself:

```typescript
interface FillDateType {
    /** Round the given date down to the nearest whole unit */
    round(d: Moment): Moment;
    /** Format the given date to a string */
    format(d: Moment): string;
    /** Increment the date by the unit. The given date will already be rounded down. */
    increment(d: Moment): Moment;
}
```

Pass your custom `FillDateType` as the `type` option to `fillDates`, and it will use your rounding, formatting, and incrementing logic throughout.

## Deprecated `smartDates`

The package also exports `smartDates` as a deprecated wrapper around `fillDates`. Prefer the `fillDates` options-object API for new code.
