# Split latency investigation — right half lag after hours of use

Symptom: after a few hours of continuous use the right half's keystrokes land
late relative to the left, so cross-half words come out reordered (`off` →
`fof`). Order within each half is preserved. Restarting the right half clears
it. Interference, range, bonding and firmware-version mismatch were already
ruled out.

## What the repo actually contained

The diagnostic grep from the brief returns almost nothing:

```
boards/shields/eyelash_sofle/eyelash_sofle_central_dongle.conf:18:CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
config/eyelash_sofle.conf:5:CONFIG_ZMK_SLEEP=y
config/eyelash_sofle.conf:21:CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
```

No `PERIPHERAL_PREF_*`, no `CONN_INTERVAL`, no `BT_BUF_*`, no
`BT_GAP_AUTO_UPDATE_CONN_PARAMS`, no `BT_CONN_PARAM_UPDATE_TIMEOUT`. The split
link runs entirely on upstream ZMK defaults. Per the brief, that is the
informative outcome: nothing was tuned, so the defaults are the whole story.

There is also no pointing device in devicetree on either half. `CONFIG_INPUT=y`
and `CONFIG_ZMK_POINTING=y` on the peripherals only enable the mouse behavior
plumbing; mouse movement comes from keymap behaviors that run on the central.
The right half sources key events only, at human typing rates.

## The defaults, and the number they imply

From `cormoran/zmk@v0.3-branch+dya`, `app/src/split/bluetooth/Kconfig`:

| Symbol | Default | Meaning |
| --- | --- | --- |
| `ZMK_SPLIT_BLE_PREF_INT` | 6 | 6 × 1.25 ms = **7.5 ms** connection interval |
| `ZMK_SPLIT_BLE_PREF_LATENCY` | 30 | peripheral may skip up to **30** connection events |
| `ZMK_SPLIT_BLE_PREF_TIMEOUT` | 400 | 4 s supervision timeout |

Worst-case peripheral → central delivery time:

```
interval × (latency + 1) = 7.5 ms × 31 = 232.5 ms
```

The user's unverified "feels like ~100 ms" sits inside that envelope, and the
ceiling is over twice it. This is the first hard number the investigation has.

## Correction to H1 as stated

H1 was "connection parameters get renegotiated upward for power saving and
never come back down." **That specific mechanism cannot happen in this
firmware.** Two findings rule it out:

1. The central sets the parameters exactly once, when it opens the link
   (`app/src/split/bluetooth/central.c:975`):

   ```c
   struct bt_le_conn_param *param =
       BT_LE_CONN_PARAM(CONFIG_ZMK_SPLIT_BLE_PREF_INT, CONFIG_ZMK_SPLIT_BLE_PREF_INT,
                        CONFIG_ZMK_SPLIT_BLE_PREF_LATENCY, CONFIG_ZMK_SPLIT_BLE_PREF_TIMEOUT);
   err = bt_conn_le_create(addr, BT_CONN_LE_CREATE_CONN, param, &slot->conn);
   ```

   Min and max interval are both `PREF_INT`, so there is no range to drift
   within, and there is no other `bt_conn_le_param_update` call anywhere on the
   split path.

2. Split peripherals ship with `BT_GAP_AUTO_UPDATE_CONN_PARAMS` defaulted to
   `n` (`app/src/split/bluetooth/Kconfig`, comment: "Allow central to specify
   connection parameters"), so the peripheral never requests a change either.

Nothing drifts. The 232.5 ms allowance is present from the moment the link
comes up.

## Revised hypothesis (H1′) — the latency allowance stops being transparent

Slave latency is an *allowance to skip*, not a fixed delay. A peripheral with
pending TX data is supposed to wake at the next anchor point, which is why
`LATENCY=30` normally costs nothing on keypress. The delay only materialises
when the peripheral's link layer stops promptly breaking out of latency.

That reframing fits every observation, where the original H1 fits only some:

- **Step change to a fixed offset**, not gradual growth — the offset is the
  latency allowance, which is a constant.
- **Right half only** — see below.
- **Cleared by restarting the half** — a reconnect rebuilds link-layer state.
- **Prevented by aggressive idle/sleep** — the 30 s idle timeout forces
  frequent teardown/reconnect, which is the same reset the restart performs.
  This is why *more* sleep helped, which was the opposite of the initial guess.

### Why the right half specifically

Both links are opened with identical parameters, so the asymmetry is not in
configuration. The plausible source is scheduling: the dongle holds two
peripheral connections at the same 7.5 ms interval, and the Zephyr link layer
must fit both sets of connection events into that window. The half that
connects second gets the less favourable anchor placement and is the one that
loses events to contention. Combined with a 30-event skip allowance, that
produces a large one-sided delay. This is the least-verified part of the chain
and is the thing the fix below tests directly.

## Why H2 is weak here

`app/src/split/bluetooth/service.c` does not behave the way a backing-up queue
would need to:

- The queue is `ZMK_SPLIT_BLE_PERIPHERAL_POSITION_QUEUE_SIZE` = **10** entries
  and, on overflow, `send_position_state()` pops the oldest and retries. A
  backlog therefore cannot grow — it is bounded at 10 and sheds from the front.
- The payload is a full key-position bitmap snapshot, not a per-key delta, so a
  dropped entry is mostly self-healing; the newest snapshot is authoritative.
- `send_position_state_callback()` drains with `bt_gatt_notify()` and only
  `LOG_DBG`s failures. A congested TX path therefore **drops** events; it does
  not queue them up for late delivery.

So H2 predicts lost or stuck keys and progressive degradation. The reported
symptom is neither — keys all arrive, in order, uniformly late. H2 is not
excluded, but it is a poor fit.

## The change made

`boards/shields/eyelash_sofle/eyelash_sofle_central_dongle.conf`:

```
CONFIG_ZMK_SPLIT_BLE_PREF_INT=6
CONFIG_ZMK_SPLIT_BLE_PREF_LATENCY=0
CONFIG_ZMK_SPLIT_BLE_PREF_TIMEOUT=400
```

This bounds the worst case at one connection interval (7.5 ms) instead of
232.5 ms, and is fully Bluetooth-spec-compliant.

**Note this goes in the dongle's conf, not the halves'.** The brief assumed BLE
tuning for the halves would live in the peripheral shield `.conf` files. It
does not: `ZMK_SPLIT_BLE_PREF_*` are guarded by `if ZMK_SPLIT_ROLE_CENTRAL`, so
they only take effect on the central, and since peripherals have
`BT_GAP_AUTO_UPDATE_CONN_PARAMS=n` the central's values are the only ones that
matter. Setting `BT_PERIPHERAL_PREF_*` on the halves would be inert for the
split link.

Cost: the peripheral radio wakes every 7.5 ms rather than every ~232 ms when
idle. That is a real battery hit, but modest next to the RGB underglow
(`BRT_MAX=90`, on at start) and backlight (`BRT_START=100`) this board already
runs continuously. If battery life turns out to matter more, `LATENCY=3`
(worst case 30 ms) is a reasonable middle ground.

The non-spec-compliant sub-7.5 ms controller settings from ZMK issue #3381 are
left commented out in the same file. Their usual caveat — Zephyr-to-Zephyr only
— *is* satisfied here (dongle and halves are all Zephyr; the dongle reaches the
Mac over USB HID, not BLE), but they are unnecessary to fix a 232 ms problem and
would apply to the dongle's BLE output profiles if those are ever used.

## Correction to the measurement plan

The brief proposed enabling USB logging on the dongle and watching for
connection parameter update events. **That will not work**, for two reasons:

1. ZMK's split central registers no `le_param_updated` callback. The only place
   connection parameters are logged is the *peripheral*
   (`app/src/split/bluetooth/peripheral.c:129`), so the log has to come from the
   right half, not the dongle.
2. `le_param_updated` fires only on *change*, and per the finding above nothing
   ever changes the parameters. Even from the peripheral it would print once at
   connect and then stay silent, whether or not the bug is present.

More fundamentally, no log can show what H1′ actually predicts: whether the
link layer is exercising its skip allowance is controller-internal state, not
something the host stack reports.

A `eyelash_sofle_peripheral_right_debug` target was still added to `build.yaml`
(right half + `zmk-usb-logging` snippet; ZMK's `ZMK_LOG_LEVEL` already defaults
to 4/DBG so no extra cmake-args are needed). It is genuinely useful for
confirming the negotiated parameters at connect and for seeing the
`"Position state message queue full"` and `"Error notifying"` warnings that
would indicate H2 — but it will not, on its own, confirm H1′.

## Tests, in order of value

1. **Flash the change and use it normally for a day.** The mechanism is
   identified and the fix is a three-line config change, so this is both faster
   and more decisive than instrumenting. Reset the DYA Studio idle/sleep
   timeouts to something long first, otherwise the existing workaround masks
   the result.
2. **The constant-use test**, still the cleanest hypothesis separator, and worth
   running on the *unfixed* firmware if you want the diagnosis confirmed rather
   than just the symptom gone. Under sustained typing or gaming the halves never
   idle, so the sleep-based workaround should stop applying. Lag returning ⇒
   accumulation during connected uptime (H1′). Lag scaling with typing volume
   rather than appearing as a step ⇒ H2.
3. **Get a real number.** Compare the measured steady-state offset against
   232.5 ms and against multiples of 7.5 ms. A value near the ceiling, or a
   clean multiple of the interval, is strong confirmation of H1′.
4. If the lag survives `LATENCY=0`, the scheduling-contention theory is the next
   thing to attack: raise `ZMK_SPLIT_BLE_PREF_INT` to 12 (15 ms) so two
   connections fit the window comfortably, keeping `LATENCY=0` — worst case
   15 ms, with half the radio wakeups.

## Answered: what DYA Studio writes

The brief flagged this as unknown. It writes **ZMK core settings**, not
anything vendor-specific. `cormoran/zmk-module-settings-rpc` calls
`zmk_activity_set_idle_ms()` / `zmk_activity_set_sleep_ms()`
(`src/studio/settings_rpc_handler.c`), and ZMK persists those to NVS under the
`activity/s` key (`app/src/activity.c`, `settings_save_one("activity/s", …)`).
The module also relays the settings to the peripherals
(`src/events/activity_settings_changed.c`), so one change in DYA Studio applies
across all three devices.

Consequence: the stored NVS value **overrides** `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT`
in `config/eyelash_sofle.conf`. Reflashing will not restore the compile-time
value; only a settings reset will.

Unrelated but worth noting while in that file: the comment there says "go to
sleep after one hour (1*60*60*1000ms)" but the value is `7200000`, which is two
hours. The comment's own arithmetic gives 3600000. Left alone, since the
runtime setting supersedes it anyway.
