# Nepionic_Log

Structured logging library for TwinCAT 3. Modelled on the Java logging pattern (SLF4J / Logback): a lightweight `DefaultLogger` facade dispatches records through a central `LogManager` to one or more pluggable `LogAppender` sinks.

---

## Libraries

| Library | Description |
|---|---|
| `Nepionic_Log` | Core — logger facade, manager, ADS output window appender |
| `Nepionic_Log_OTel` | OTel ring-buffer appender — **deprecated**, superseded by `Nepionic_Log_Klein` |
| `Nepionic_Log_Klein` | Klein Event ring-buffer appender (consumed by the Klein collector's `ads` receiver) |
| `Nepionic_Log_Syslog` | UDP RFC 5424 syslog appender |

Add only the libraries you need. `Nepionic_Log_OTel`, `Nepionic_Log_Klein`, and `Nepionic_Log_Syslog` all depend on `Nepionic_Log`.

---

## Architecture

```
DefaultLogger           DefaultLogger           DefaultLogger
  .Info(msg)              .Error(msg)             .Debug(msg)
       │                       │                       │
       └───────────────────────┴───────────────────────┘
                               │
                         GetLogManager()
                         ┌─────────────┐
                         │  LogManager  │  ← process-global singleton
                         │  Level gate  │
                         │  Reentrancy  │
                         │    guard     │
                         └──────┬───────┘
       ┌────────────────┬───────┴────────┬────────────────┐
       ▼                ▼                ▼                ▼
AdsLogAppender  OTelLogAppender  KleinEventAppender  SyslogAppender
(ADS output win) (ring buffer,   (ring buffer,        (UDP syslog)
                  deprecated)     -> Klein Event)
```

`GetLogManager()` is a `FUNCTION` with a `VAR_STAT` instance — no GVL, no declaration needed.

---

## Quick start

### 1. Reference the libraries

In your TwinCAT project, add references to:
- `Nepionic_Log`
- `Nepionic_Log_Klein` *(optional — recommended for Klein-based collectors)*
- `Nepionic_Log_OTel` *(optional, deprecated — see above)*
- `Nepionic_Log_Syslog` *(optional)*

### 2. Declare appenders and loggers

```st
// In a PROGRAM or FB that runs at startup
VAR
    ads_appender   : Nepionic_Log.AdsLogAppender;
    klein_appender : Nepionic_Log_Klein.KleinEventAppender;
END_VAR
```

```st
// In any component FB
VAR
    log : Nepionic_Log.DefaultLogger('AxisX');
END_VAR
```

### 3. Register appenders once at startup

```st
// Called once — e.g. from FB_Init or a startup PROGRAM
Nepionic_Log.GetLogManager().Register(ads_appender);
Nepionic_Log.GetLogManager().Register(klein_appender);
```

### 4. Log from anywhere

```st
log.Debug('Cycle started');
log.Info('Homing complete');
log.Warning('Velocity limit approached');
log.Error('Drive fault detected');
log.Fatal('Safety circuit open');
```

That's it. No cyclic wiring required for `AdsLogAppender`, `OTelLogAppender`, or `KleinEventAppender`.

---

## DefaultLogger

### Declaration

```st
VAR
    log : DefaultLogger('MyComponent');
END_VAR
```

The string passed to `FB_Init` becomes the `source` field on every log record — use it to identify the component (e.g. `'AxisX'`, `'ConveyorBelt'`, `'SafetyController'`).

### Basic logging

```st
log.Debug('Detailed trace message');
log.Info('Normal operational message');
log.Warning('Something unexpected but recoverable');
log.Error('An error occurred');
log.Fatal('Unrecoverable failure');
```

### Structured attributes

Attach key/value pairs to a record before the terminal method. Up to 10 attributes per record; extras are silently ignored.

```st
log.WithAttribute('state', 'Homing')
   .WithAttribute('speed', '250')
   .WithAttribute('direction', 'Positive')
   .Info('Axis moving');
```

### Conditional logging

Emit a record only when a condition is true:

```st
log.OnCondition(limit_switch_active).Warning('Limit switch triggered');
```

`OnCondition` and `WithAttribute` can be chained together:

```st
log.OnCondition(error_active)
   .WithAttribute('error_code', INT_TO_STRING(drive_error))
   .Error('Drive fault');
```

---

## LogManager

`GetLogManager()` returns a `REFERENCE TO LogManager` — the process-global singleton.

### Set a global level floor

Drops all records below the specified level before any appender sees them:

```st
Nepionic_Log.GetLogManager().Level := Nepionic_Log.LogLevel.Information;
```

### Check registered appender count (diagnostic)

```st
count := Nepionic_Log.GetLogManager().Count;
```

Up to **8 appenders** can be registered. Extras are silently ignored.

---

## Appenders

### AdsLogAppender (`Nepionic_Log`)

Writes to the TwinCAT ADS output window via `AdsLogStr`. No cyclic call needed.

```st
VAR
    ads_appender : Nepionic_Log.AdsLogAppender;
END_VAR

// Optional: raise threshold to reduce noise in output window
ads_appender.Level := Nepionic_Log.LogLevel.Warning;

Nepionic_Log.GetLogManager().Register(ads_appender);
```

Output format: `[source] message  {key=value, key=value}`

---

### OTelLogAppender (`Nepionic_Log_OTel`) — deprecated

**Deprecated.** Superseded by `Nepionic_Log_Klein.KleinEventAppender` below. Kept for existing
consumers; new projects should reference `Nepionic_Log_Klein` instead.

Writes structured log entries into an owned `OTelLogRing` ring buffer, exposed as an ADS symbol and polled by the `otelcol-ads` Go collector. No cyclic call needed.

```st
VAR
    otel_appender : Nepionic_Log_OTel.OTelLogAppender;
END_VAR

Nepionic_Log.GetLogManager().Register(otel_appender);
```

Point the collector at the ring symbol. If the appender is declared in `MAIN`:

```yaml
# otelcol-ads config
receivers:
  adslogsreceiver:
    symbol: MAIN.otel_appender.Ring
```

All 10 structured attributes are preserved in the OTel log record.

---

### KleinEventAppender (`Nepionic_Log_Klein`)

Writes structured log entries into an owned `KleinEventRing` ring buffer — the same proven
seqlock wire layout as `OTelLogRing`, under new independently-addressable types — exposed as
an ADS symbol and read by [Klein's](https://github.com/siyka-au/klein) `ads` receiver
(`internal/receiver/ads/eventring.go`), which decodes each slot into a Klein `model.Event`. No
cyclic call needed.

```st
VAR
    klein_appender : Nepionic_Log_Klein.KleinEventAppender;
END_VAR

Nepionic_Log.GetLogManager().Register(klein_appender);
```

Point the Klein collector at the ring symbol. If the appender is declared in `MAIN`:

```yaml
# Klein ads receiver config
receivers:
  ads:
    event_rings:
      - symbol: MAIN.klein_appender.ring
```

Mapping onto the resulting `model.Event` (documented byte-for-byte in `KleinEventEntry.TcDUT`):

| Log record field | Klein `Event` field |
|---|---|
| `level` | `Severity` (`Debug`→2, `Information`→3, `Warning`→4, `Error`→5, `Fatal`→6) |
| `message` | `Message` |
| `source` | `Signal.Path`, and `Actor.ID` (`Actor.Type` is fixed to `"plc"`) |
| `attributes` (when any) | `To`, as a `StructValue{Fields: {key: value, ...}}` |

All 10 structured attributes are preserved. An entry with zero attributes leaves `Event.To` unset
rather than emitting an empty struct.

---

### SyslogAppender (`Nepionic_Log_Syslog`)

Sends RFC 5424 UDP syslog messages. **Requires a cyclic `Execute()` call** to drive the UDP socket state machine.

```st
VAR
    syslog_appender : Nepionic_Log_Syslog.SyslogAppender('192.168.1.50', 514);
END_VAR

// Startup
Nepionic_Log.GetLogManager().Register(syslog_appender);

// Every cycle
syslog_appender.Execute();
```

Optional properties:

```st
syslog_appender.AppName := 'MyPLC';
syslog_appender.Level   := Nepionic_Log.LogLevel.Warning;
```

---

## Level reference

| `LogLevel` | Value | Syslog severity | OTel severity | Klein `model.Severity` |
|---|---|---|---|---|
| `Debug` | 0 | 7 (Debug) | 0 | `SeverityDebug` (2) |
| `Information` | 1 | 6 (Informational) | 1 | `SeverityInfo` (3) |
| `Warning` | 2 | 4 (Warning) | 2 | `SeverityWarn` (4) |
| `Error` | 3 | 3 (Error) | 3 | `SeverityError` (5) |
| `Fatal` | 4 | 2 (Critical) | 4 | `SeverityFatal` (6) |

---

## Full startup example

```st
PROGRAM StartupLogger
VAR
    ads_appender    : Nepionic_Log.AdsLogAppender;
    klein_appender  : Nepionic_Log_Klein.KleinEventAppender;
    syslog_appender : Nepionic_Log_Syslog.SyslogAppender('10.0.0.50', 514);
    initialized     : BOOL;
END_VAR

IF NOT initialized THEN
    // Optional: raise global floor
    Nepionic_Log.GetLogManager().Level := Nepionic_Log.LogLevel.Debug;

    // Optional: per-appender thresholds
    ads_appender.Level  := Nepionic_Log.LogLevel.Information;
    syslog_appender.Level := Nepionic_Log.LogLevel.Warning;

    Nepionic_Log.GetLogManager().Register(ads_appender);
    Nepionic_Log.GetLogManager().Register(klein_appender);
    Nepionic_Log.GetLogManager().Register(syslog_appender);

    initialized := TRUE;
END_IF

// Drive syslog UDP state machine every cycle
syslog_appender.Execute();
```

```st
// In any component FB
VAR
    log : Nepionic_Log.DefaultLogger('ConveyorBelt');
END_VAR

log.Info('Belt started');
log.WithAttribute('speed_rpm', REAL_TO_STRING(actual_speed)).Debug('Speed reading');
log.OnCondition(jam_detected).Error('Belt jam detected');
```
