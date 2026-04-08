# Nepionic_Log

Structured logging library for TwinCAT 3. Modelled on the Java logging pattern (SLF4J / Logback): a lightweight `DefaultLogger` facade dispatches records through a central `LogManager` to one or more pluggable `LogAppender` sinks.

---

## Libraries

| Library | Description |
|---|---|
| `Nepionic_Log` | Core — logger facade, manager, ADS output window appender |
| `Nepionic_Log_OTel` | OTel ring-buffer appender (consumed by `otelcol-ads` Go collector) |
| `Nepionic_Log_Syslog` | UDP RFC 5424 syslog appender |

Add only the libraries you need. `Nepionic_Log_OTel` and `Nepionic_Log_Syslog` both depend on `Nepionic_Log`.

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
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
       AdsLogAppender   OTelLogAppender   SyslogAppender
     (ADS output window) (ring buffer)    (UDP syslog)
```

`GetLogManager()` is a `FUNCTION` with a `VAR_STAT` instance — no GVL, no declaration needed.

---

## Quick start

### 1. Reference the libraries

In your TwinCAT project, add references to:
- `Nepionic_Log`
- `Nepionic_Log_OTel` *(optional)*
- `Nepionic_Log_Syslog` *(optional)*

### 2. Declare appenders and loggers

```st
// In a PROGRAM or FB that runs at startup
VAR
    ads_appender  : Nepionic_Log.AdsLogAppender;
    otel_appender : Nepionic_Log_OTel.OTelLogAppender;
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
Nepionic_Log.GetLogManager().Register(otel_appender);
```

### 4. Log from anywhere

```st
log.Debug('Cycle started');
log.Info('Homing complete');
log.Warning('Velocity limit approached');
log.Error('Drive fault detected');
log.Fatal('Safety circuit open');
```

That's it. No cyclic wiring required for `AdsLogAppender` or `OTelLogAppender`.

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

### OTelLogAppender (`Nepionic_Log_OTel`)

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

| `LogLevel` | Value | Syslog severity | OTel severity |
|---|---|---|---|
| `Debug` | 0 | 7 (Debug) | 0 |
| `Information` | 1 | 6 (Informational) | 1 |
| `Warning` | 2 | 4 (Warning) | 2 |
| `Error` | 3 | 3 (Error) | 3 |
| `Fatal` | 4 | 2 (Critical) | 4 |

---

## Full startup example

```st
PROGRAM StartupLogger
VAR
    ads_appender    : Nepionic_Log.AdsLogAppender;
    otel_appender   : Nepionic_Log_OTel.OTelLogAppender;
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
    Nepionic_Log.GetLogManager().Register(otel_appender);
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
