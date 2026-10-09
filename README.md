# Intl Supported Time Zone Options

This proposal refines the default output of `Intl.supportedValuesOf("timeZone")` and `Intl.Locale.prototype.getTimeZones()` to return location-based primary time zone identifiers plus `"UTC"`, and introduces an `options` parameter on `Intl.supportedValuesOf` to allow callers to control which non-location time zones are included.

- Stage: 0
- Champion: Shane F. Carr (@sffc)
- Related ECMA-402 Issues: [#1103](https://github.com/tc39/ecma402/issues/1103), [#778](https://github.com/tc39/ecma402/issues/778), [#1052](https://github.com/tc39/ecma402/issues/1052)

## Background

ECMA-402 provides two APIs that enumerate supported time zone identifiers:

- `Intl.supportedValuesOf("timeZone")`, which returns all supported primary time zone identifiers, primarily to populate user-facing time zone pickers.
- `Intl.Locale.prototype.getTimeZones()`, which returns the primary time zone identifiers associated with a locale's region.

In the IANA Time Zone Database (TZDB) and Unicode Common Locale Data & Rules (CLDR), primary time zones may or may not be associated with a location:

- **Location-based time zones** are associated with a geographical region (an ISO 3166-1 Alpha-2 country code in TZDB `zone.tab` and CLDR `timezone.xml`), such as `"America/New_York"`, `"Europe/Berlin"`, and `"Asia/Tokyo"`. All of these identifiers start with a continent or ocean prefix (`Africa/`, `America/`, `Antarctica/`, `Arctic/`, `Asia/`, `Atlantic/`, `Australia/`, `Europe/`, `Indian/`, `Pacific/`).
- **Non-location time zones** are not associated with any country or region (and are assigned to the world region `"001"` in CLDR). These include `"UTC"`, fixed-offset POSIX zones (`"Etc/GMT+1"` through `"Etc/GMT+12"` and `"Etc/GMT-1"` through `"Etc/GMT-14"`), and legacy POSIX zones (`"EST5EDT"`, `"CST6CDT"`, `"MST7MDT"`, `"PST8PDT"`).

### Cross-Browser Divergence

Although ECMA-402 currently specifies that `Intl.supportedValuesOf("timeZone")` returns all `AvailablePrimaryTimeZoneIdentifiers()`, major browsers have never aligned on non-location time zones ([#778](https://github.com/tc39/ecma402/issues/778)):

```javascript
const REGIONS = /^(Africa|America|Antarctica|Arctic|Asia|Atlantic|Australia|Europe|Indian|Pacific)\//;
Intl.supportedValuesOf("timeZone").filter((tz) => !REGIONS.test(tz));
// Chrome: []
// Safari (< 26): ["UTC"]
// Safari (26+): ["Etc/GMT+1", ..., "Etc/GMT-14", "UTC"]
// Firefox (with TZDB 2026d): ["CST6CDT", "EST5EDT", "Etc/GMT+1", ..., "Etc/GMT-14", "MST7MDT", "PST8PDT", "UTC"]
```

| Time Zones | Examples | Chrome (V8) | Safari (< 26) | Safari (26+) | Firefox (SpiderMonkey) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Location-based primary zones | `"America/New_York"`, `"Europe/Paris"` | Included | Included | Included | Included |
| UTC | `"UTC"` | Omitted | Included | Included | Included |
| Fixed-offset `Etc/GMT*` zones | `"Etc/GMT+5"`, `"Etc/GMT-8"` | Omitted | Omitted | Included | Included |
| Legacy POSIX non-location zones | `"EST5EDT"`, `"CST6CDT"` | Omitted | Omitted | Omitted | Included (when primary) |

- **Chrome (V8)** ([`src/objects/intl-objects.cc`](https://github.com/v8/v8/blob/201a2ea99803a6067c7633235764c24f125c51a6/src/objects/intl-objects.cc#L2358-L2367)) queries ICU with `UCAL_ZONE_TYPE_CANONICAL_LOCATION`, omitting all non-location zones including `"UTC"` (which has led to developer bug reports requesting `"UTC"`, e.g. [proposal-intl-enumeration#47](https://github.com/tc39/proposal-intl-enumeration/issues/47)).
- **Safari (JavaScriptCore)** ([`Source/JavaScriptCore/runtime/IntlObject.cpp`](https://github.com/WebKit/WebKit/blob/f5430ef75d980c24083a2415fb63d085bf862285/Source/JavaScriptCore/runtime/IntlObject.cpp#L1961-L1974)) queries ICU with `UCAL_ZONE_TYPE_CANONICAL` and filters out identifiers without `'/'` other than `"UTC"` and `"GMT"`, omitting `"EST5EDT"`, `"CST6CDT"`, `"MST7MDT"`, and `"PST8PDT"`. Prior to [July 2025](https://github.com/WebKit/WebKit/commit/faba271b18), it also filtered out non-UTC `Etc/GMT*` zones, returning only location-based primary zones plus `"UTC"`.
- **Firefox (SpiderMonkey)** ([`js/src/builtin/intl/IntlObject.cpp`](https://github.com/mozilla-firefox/firefox/blob/2a67d084d49a62ba48c30ef12ee14372d7bf0a85/js/src/builtin/intl/IntlObject.cpp#L408-L435), [`js/src/builtin/intl/SharedIntlData.cpp`](https://github.com/mozilla-firefox/firefox/blob/2a67d084d49a62ba48c30ef12ee14372d7bf0a85/js/src/builtin/intl/SharedIntlData.cpp#L135-L233)) returns all primary time zone identifiers, including `"UTC"`, `"Etc/GMT*"`, and any primary POSIX zones.

A similar divergence exists in `Intl.Locale.prototype.getTimeZones()` ([#1052](https://github.com/tc39/ecma402/issues/1052)). Because CLDR assigns non-location time zones to the world region `"001"`, calling `new Intl.Locale("en-001").getTimeZones()` in V8 ([`src/objects/js-locale.cc`](https://github.com/v8/v8/blob/201a2ea99803a6067c7633235764c24f125c51a6/src/objects/js-locale.cc#L883-L889)) and JavaScriptCore ([`Source/JavaScriptCore/runtime/IntlLocale.cpp`](https://github.com/WebKit/WebKit/blob/f5430ef75d980c24083a2415fb63d085bf862285/Source/JavaScriptCore/runtime/IntlLocale.cpp#L950-L983)) returns non-location time zones (including `Etc/GMT*` and deprecated `SystemV/*` zones), whereas SpiderMonkey ([`js/src/builtin/intl/Locale.cpp`](https://github.com/mozilla-firefox/firefox/blob/2a67d084d49a62ba48c30ef12ee14372d7bf0a85/js/src/builtin/intl/Locale.cpp#L1634-L1672)) rejects non-country regions and returns `[]`:

```javascript
new Intl.Locale("en-001").getTimeZones();
// Chrome / Safari: ["Etc/GMT", "Etc/GMT+1", ..., "Etc/UTC", "SystemV/AST4", ...]
// Firefox: []
```

### Un-deprecation of `EST5EDT`, `CST6CDT`, `MST7MDT`, and `PST8PDT` in TZDB 2026d

In TZDB 2024b, the legacy POSIX zones `"EST5EDT"`, `"CST6CDT"`, `"MST7MDT"`, and `"PST8PDT"` were deprecated and converted into `Link`s pointing to `"America/New_York"`, `"America/Chicago"`, `"America/Denver"`, and `"America/Los_Angeles"`, respectively.

In TZDB 2026d, these four identifiers were un-merged back into separate `Zone` entries in the `backward` file ([#1103](https://github.com/tc39/ecma402/issues/1103)). Because several Canadian provinces (and potentially US states in the future) have legislated permanent Daylight Saving Time (DST) or standard time, TZDB separated the POSIX zones so they continue observing POSIX US DST rules (`USback`) indefinitely even if cities stop changing clocks. Following TZDB 2026d, CLDR 48 ([CLDR-19784](https://unicode-org.atlassian.net/browse/CLDR-19784)) un-deprecated `est5edt`, `cst6cdt`, `mst7mdt`, and `pst8pdt`, making them canonical non-location time zones again.

These legacy POSIX zones do not correspond to any civil authority, so users should not be using them. If `Intl.supportedValuesOf("timeZone")` includes all primary time zone identifiers by default, these zones will appear in UI time zone pickers, where users looking for `"EST"`, `"CST"`, `"MST"`, or `"PST"` may select them instead of their actual city time zone. If their jurisdiction later changes its DST rules, those users will remain on hard-coded POSIX transitions rather than local civil time.

Treating these four identifiers as non-primary aliases to city time zones in ECMA-402 is only a temporary workaround. It works only while their post-1970 transition rules coincide with a major city, and stops working as soon as jurisdictions change their DST rules and no city remains to which the POSIX zone can be aliased.

## Proposed Solution

### Default Behavior

By default, `Intl.supportedValuesOf("timeZone")` returns only location-based primary time zone identifiers plus `"UTC"`:

```javascript
const defaultZones = Intl.supportedValuesOf("timeZone");

defaultZones.includes("America/New_York"); // true
defaultZones.includes("UTC");              // true

defaultZones.includes("Etc/GMT+5");        // false
defaultZones.includes("EST5EDT");          // false
```

Non-location primary time zones remain valid, supported time zone identifiers in `Intl.DateTimeFormat` and `Temporal` (via `AvailableNamedTimeZoneIdentifiers`). They are only excluded from the default enumeration.

### Options Parameter on `Intl.supportedValuesOf`

`Intl.supportedValuesOf` is extended to accept an optional `options` object as its second parameter:

```javascript
Intl.supportedValuesOf(key, options)
```

The `options` argument is only read when `key` is `"timeZone"`. For other values of `key` (`"calendar"`, `"collation"`, `"currency"`, `"numberingSystem"`, `"unit"`), `options` is ignored.

When `key` is `"timeZone"`, `options` supports an `includeNonLocation` string enumeration option to control which non-location primary time zones are returned alongside location-based primary time zones:

- `"utc"` (default): Includes `"UTC"` in addition to location-based primary time zone identifiers.
- `"none"`: Excludes all non-location time zones, returning strictly location-based primary time zone identifiers (omitting `"UTC"`).
- `"offset"`: Includes `"UTC"` and fixed-offset `"Etc/GMT*"` primary time zone identifiers (`"Etc/GMT+1"` through `"Etc/GMT+12"` and `"Etc/GMT-1"` through `"Etc/GMT-14"`) in addition to location-based primary time zone identifiers.
- `"all"`: Includes all non-location primary time zone identifiers (such as `"UTC"`, `"Etc/GMT*"`, `"EST5EDT"`, `"CST6CDT"`, `"MST7MDT"`, and `"PST8PDT"`) in addition to location-based primary time zone identifiers.

```javascript
const locationOnlyZones = Intl.supportedValuesOf("timeZone", {
  includeNonLocation: "none",
});
locationOnlyZones.includes("UTC");       // false

const offsetZones = Intl.supportedValuesOf("timeZone", {
  includeNonLocation: "offset",
});
offsetZones.includes("Etc/GMT+5");       // true
offsetZones.includes("EST5EDT");         // false

const allPrimaryZones = Intl.supportedValuesOf("timeZone", {
  includeNonLocation: "all",
});
allPrimaryZones.includes("Etc/GMT+5");   // true
allPrimaryZones.includes("EST5EDT");     // true
```

**Open question:** How should we bikeshed the `includeNonLocation` option name and its values (`"none" | "utc" | "offset" | "all"`)?

### Alignment with `Intl.Locale.prototype.getTimeZones`

`Intl.Locale.prototype.getTimeZones()` (`TimeZonesOfLocale`) is updated to return only location-based primary time zone identifiers plus `"UTC"`. The `includeNonLocation` option is not currently proposed for `Intl.Locale.prototype.getTimeZones()`.

For a locale whose region is a country or territory code (such as `"US"` or `"FR"`), `getTimeZones()` returns the location-based primary time zone identifiers for that region. For a locale with the world region `"001"`, `getTimeZones()` returns `["UTC"]`:

```javascript
new Intl.Locale("en-US").getTimeZones();
// ["America/Adak", "America/Anchorage", "America/Boise", "America/Chicago", ...]

new Intl.Locale("en-001").getTimeZones();
// ["UTC"]
```

## Prior Art

This proposal aligns directly with how CLDR and ICU categorize time zones:

- **CLDR ([UTS #35](https://unicode.org/reports/tr35/tr35-dates.html#Time_Zone_Identifiers) / [`common/bcp47/timezone.xml`](https://github.com/unicode-org/cldr/blob/main/common/bcp47/timezone.xml))**: Every BCP 47 time zone identifier has an associated region code—either derived from the first two letters of a 5-letter short ID (e.g., `usnyc` → `US`), specified via an explicit `region` attribute (e.g., `jeruslm` → `IL`), or defaulting to `"001"` (World) for non-location zones (`utc`, `gmt`, `utce01`–`utce14`, `utcw01`–`utcw12`, `est5edt`, `cst6cdt`, `mst7mdt`, `pst8pdt`, `unk`).
- **ICU (`USystemTimeZoneType`)**: `TimeZone::createTimeZoneIDEnumeration` accepts a `USystemTimeZoneType` enum:
  - `UCAL_ZONE_TYPE_CANONICAL_LOCATION`: Canonical time zones associated with a specific region (excluding `"001"`), corresponding to `{ includeNonLocation: "none" }`.
  - `UCAL_ZONE_TYPE_CANONICAL`: All canonical time zones, including non-location (`"001"`) zones, corresponding to `{ includeNonLocation: "all" }`.

## Future Directions

Introducing an `options` bag on `Intl.supportedValuesOf` makes it straightforward to add further time zone enumeration options in future proposals if needed, such as:

- **Non-primary aliases (`Link` names)**: An option to include non-primary time zone identifiers such as `"US/Eastern"` or `"Asia/Calcutta"`, corresponding to ICU's `UCAL_ZONE_TYPE_ANY`.
- **Matching IANA TZDB `Zone`s**: An option to return only identifiers that are `Zone`s in the default build of the IANA Time Zone Database, excluding country-boundary `Link`s that CLDR elevates to separate canonical zones (such as `"Europe/Oslo"`, which TZDB links to `"Europe/Berlin"`).
