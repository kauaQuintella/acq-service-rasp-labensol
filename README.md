# acquisition service oscilloscope

**Responsibilities:** Connect and configure the oscilloscope, run measurement cycles, record data to SQLite, generate an optional mirror CSV, log events, and respond to commands. It has no knowledge of HTTP or web browsers.

## Internal Components

* **Instrument:** Encapsulates PyVISA and SCPI commands.
* **Acquisition Loop:** Runs in its own dedicated thread.
* **Storage:** A single SQLite writer fed by a queue, plus the CSV mirror.
* **Command Channel:** The entry point for FastAPI.

## States

**DISCONNECTED → CONNECTING → READY ⇄ MEASURING → STOPPING → READY**, featuring an **ERROR** state and automatic reconnection with exponential backoff (increasing wait time). Each state accepts only specific commands; any other command returns **INVALID_STATE**.

## Measurement Cycle (Improved Version)

1. Check for a stop request.
2. Trigger the acquisition.
3. Wait for completion, checking for a stop request every 0.1 s until timeout.
4. On timeout: Log the **NO_TRIGGER** event, cancel the acquisition, and restart.
5. Capture a timestamp at the end of the acquisition.
6. Read each configured measurement (source + type) and, on the first iteration, the unit.
7. Enqueue for writing, log the entry, and pause for `interval_s`.

## Failures

In the event of a communication error with the instrument, the experiment is interrupted and the service reconnects automatically without resuming the experiment on its own (the professor decides whether to resume). Upon startup, any experiment left in the `in_progress` state due to a power outage or crash is marked as interrupted.

## Configuration Security

Only sources (`CH1`–`CH4`) and measurement types from a predefined, fixed list are accepted; unknown fields are strictly rejected. When configurations originate from the web interface, this prevents anyone from sending arbitrary SCPI commands to the instrument.