
# 🚑 Smart Ambulance — Priority-Based Patient Telemetry on QNX Neutrino RTOS

> **Critical alerts never wait behind routine telemetry.**

Built for a **State Level Hackathon** (Team: *Dharani_aus [Surampalem]*) — Domain: **Healthcare / Emergency Medical Services**.

A real-time ambulance-to-hospital telemetry system built on **QNX Neutrino** native message passing. An ambulance transmitter classifies each patient reading as **CRITICAL** or **ROUTINE** and sends it to the hospital receiver, which processes critical alerts on a higher-priority thread so that a life-threatening reading is never delayed by routine data.

---

## 📌 Problem

An ambulance continuously streams heart rate, SpO₂, temperature, GPS and emergency severity. If routine updates are being processed when a critical reading arrives, the alert can be delayed — and the ER loses preparation time.

**Success metrics**

- 100% of critical alerts are served before routine work
- A running routine job yields within **100 ms** of a critical alert
- The interrupted routine job is acknowledged to the ambulance as `INTERRUPTED`

---

## 🧠 How It Works

### Triage rules (applied by the transmitter)

| Parameter   | CRITICAL if          | ROUTINE          |
|-------------|----------------------|------------------|
| Severity    | 4 or 5               | 1 to 3           |
| Heartbeat   | < 50 or > 130 BPM    | 50 to 130 BPM    |
| SpO₂        | < 90 %               | 90 % or more     |
| Temperature | > 39 °C or < 35 °C   | 35 to 39 °C      |
| GPS         | never                | always           |

Critical readings go to the **critical channel**; everything else goes to the **routine channel**.

### Architecture

```
 ┌────────────────────────┐                 ┌──────────────────────────────────────┐
 │  Ambulance Transmitter │                 │          Hospital Receiver           │
 │  (ambulance_transmitter)│                │          (hospital_receiver)         │
 │                        │   MsgSend       │                                      │
 │  - reads vitals        │ ──────────────► │  Critical channel → Critical thread  │
 │  - validates ranges    │   (critical)    │    SCHED_RR 30                       │
 │  - classifies readings │                 │    prints alert, critical_count++    │
 │                        │   MsgSend       │                                      │
 │                        │ ──────────────► │  Routine channel → Routine thread    │
 │                        │   (routine)     │    SCHED_RR 10                       │
 │                        │                 │    5 s job (50 × 100 ms), aborts if  │
 │  ◄──── MsgReply = ACK ─┼─────────────────┤    critical_count changes            │
 └────────────────────────┘                 └──────────────────────────────────────┘
                                                  shared critical_count (mutex)
```

### QNX features used

- **Native message passing:** `MsgSend`, `MsgReceive`, `MsgReply` — the reply doubles as the ACK
- **Two channels, one thread each:** critical traffic never queues behind routine traffic
- **`_NTO_CHF_FIXED_PRIORITY`** on both channels
- **`SCHED_RR`** thread priorities: critical = **30**, routine = **10**
- **Mutex-protected `critical_count`** polled by the routine thread every 100 ms

### Reply codes (returned to the ambulance)

| Code | Meaning                                  |
|------|------------------------------------------|
| `1`  | `CRITICAL` — critical alert processed    |
| `2`  | `ROUTINE` — routine data processed       |
| `3`  | `INTERRUPTED` — routine job aborted because a critical alert arrived |
| `-1` | `REJECTED` — message sent on the wrong priority channel |

### Resilience

- **Critical alert during a 5 s routine job:** the routine thread sees `critical_count` change within one 100 ms poll, aborts, and replies `INTERRUPTED`.
- **Multiple ambulances:** each message carries an `ambulance_id`; the hospital serves critical traffic first regardless of arrival order.
- **Bad data:** the transmitter range-checks every reading and discards invalid sets; the receiver rejects messages sent on the wrong channel.

---

## 📁 Repository Structure

```
.
├── ambulance_transmitter.c   # Ambulance side: input, validation, triage, MsgSend
├── hospital_receiver.c       # Hospital side: channels, priority threads, MsgReceive/MsgReply
├── QNX_HACKATHON.pptx        # Hackathon presentation
└── README.md
```

> Rename the `.c` files above to match your actual filenames.

---

## ⚙️ Requirements

- **QNX Neutrino RTOS** (QNX SDP 7.x / 8.0) — target or QEMU VM
- **QNX Momentics IDE** or the `qcc` command-line toolchain
- Permission to set `SCHED_RR` priorities (run as `root` or with the required privileges on QNX)

---

## 🔨 Build

```bash
qcc -o hospital_receiver hospital_receiver.c
qcc -o ambulance_transmitter ambulance_transmitter.c
```

---

## ▶️ Run

**1. Start the hospital receiver** (Terminal 1):

```bash
./hospital_receiver
```

It prints the values you need for the transmitter:

```
SMART AMBULANCE HOSPITAL RECEIVER
Hospital PID: <pid>
Critical Channel ID: <id>
Routine Channel ID: <id>
```

**2. Start an ambulance transmitter** (Terminal 2) and enter the values above when prompted:

```bash
./ambulance_transmitter
```

You will be asked for:

```
Ambulance ID, Hospital PID, Critical Channel ID, Routine Channel ID
Heartbeat (BPM), SpO2 (%), Temperature (C), Latitude, Longitude, Emergency Severity (1-5)
```

### Sample inputs

| Scenario | HR | SpO₂ | Temp | Severity | Result |
|----------|----|------|------|----------|--------|
| Stable patient | 80 | 98 | 36.8 | 2 | All readings sent as ROUTINE |
| Critical patient | 145 | 85 | 39.5 | 5 | EMERGENCY, HR, SpO₂, TEMPERATURE sent as CRITICAL |

### 🔬 Demonstrating the interrupt behaviour

1. Start `hospital_receiver`.
2. Start a **first** transmitter and send a stable (routine) reading — the receiver begins its 5 s routine job.
3. Start a **second** transmitter (different Ambulance ID) in another terminal and send a critical reading **while the routine job is running**.
4. The receiver prints `ROUTINE INTERRUPTED! Critical alert took priority.` and the first transmitter receives `ACK: Routine processing interrupted`.

---

## 📊 Performance

| Metric | Value |
|--------|-------|
| Critical vs routine thread priority (`SCHED_RR`) | 30 vs 10 |
| Max time for routine work to yield (design) | 100 ms |
| Measured critical alert latency | _TBD — fill in from QNX Momentics System Profiler_ |

<!-- Add your System Profiler screenshot here:
![QNX System Profiler](docs/profiler.png)
-->

---

## 🚀 Future Scope

- **Real sensors:** pulse-ox, temperature and GPS over UART or CAN replace console input
- **Network scale:** same messages over **QNX Qnet** to a remote hospital node, many ambulances
- **Event-driven design:** replace 100 ms polling with an event or pulse for faster interruption
- **Safety hardening:** priority-inheritance mutex, HAM auto-restart, CPU affinity, logging

---

## 🎓 Key Learning

A blocking `MsgReceive` server cannot be interrupted mid-job. The fix is **priority-separated channels and threads**, plus a mutex-protected counter that the low-priority job polls so it can yield to critical work.

> Every second counts: the right alert reaches the ER first, every time.

---

## ⚠️ Current Limitations

- Vitals are entered at the console — no GPIO or real sensor interface in this build
- CPU sets and affinity are not used
- Routine interruption uses 100 ms polling rather than events/pulses

---

## 👥 Team

**Dharani_aus [Surampalem]** — State Level Hackathon

_Add team member names here._

---

## 📄 License

Add a license of your choice (e.g. MIT) — or remove this section.
