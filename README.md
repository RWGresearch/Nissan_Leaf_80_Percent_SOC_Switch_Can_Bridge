# Nissan_Leaf_80_Percent_SOC_Switch_Can_Bridge

Firmware for the STM32F105 **2-port CAN-bridge** used in Nissan LEAF battery upgrades. It adds a
hardware switch that sets where AC charging stops: **100 %, 80 % or 60 %**.

This is a modified version of the CANBRIDGE-2port firmware from
**[dalathegreat/Nissan-LEAF-Battery-Upgrade](https://github.com/dalathegreat/Nissan-LEAF-Battery-Upgrade)**,
licensed GPL-3.0. All credit for the original bridge firmware, flasher and install guide goes to
that project. Everything except the charge limit works exactly as in the original. Read the
upstream README for hardware, install and wiring of the bridge itself.

---

## What changed

The full change history is at the top of [`source/Src/can-bridge-firmware.c`](source/Src/can-bridge-firmware.c).

| Rev  | Change |
|------|--------|
| 0001 | Original unmodified firmware (baseline) |
| 0002 | **Selectable AC charge limit** read from input pins Inp1 and Inp2 |
| 0003 | Comments only: PCB silkscreen labels for the input pins |
| 0004 | Comments only: confirmed working on hardware; explained the board lock (see [Flashing](#flashing)) |

### How the charge limit works

- The bridge reads the battery's **real SOC** from CAN message `0x55B`, in 0.1 % steps.
- When the SOC reaches the selected limit **during an AC charge** (`0x1F2` charge request =
  "Normal Charge"), the bridge changes the battery's `0x1DB` message to the car. It sets
  **`LB_Full_CHARGE_flag` = 1** and **`LB_Failsafe_Status` = 2 ("Charging Mode Stop Request")**,
  then recalculates the CRC.
- The car ends the charge. **It won't charge again until the charge cable is unplugged and
  plugged back in.**
- **100 % (switch in the centre):** the bridge does nothing extra. The battery's own data goes
  to the car as in the original firmware, and the battery decides when it's full.
- **Only AC charging is affected.** Driving, idle and DC quick charging (CHAdeMO) are never
  touched. The QC station manages its own stop.
- **No stop/start cycling:** once the limit is hit it stays latched. The latch clears when SOC
  falls 3 % below the setpoint (e.g. below 77 % on the 80 % setting), when the switch goes back
  to the centre (100 %), or when the car wakes up again.
- **Lowest limit wins.** The original firmware's support for the car's own 80 % charge setting
  (`0x1F2` `TargetCharge_SOC`) is kept. If both are active, whichever limit is lower stops the
  charge. The switch can always lower the limit, but it can't force 100 % if the car itself
  asks for 80 %.
- **The dash may show a slightly higher %** when charging stops. The limit uses the battery's
  real SOC, not the rescaled dash SOC. On an AZE0 car the dash maps real 5–95 % to 0–100 %, so
  a real 80 % shows as about 83 %.
- The switch is read once a second, so you can change it while the car is on. No reboot needed.

---

## Switch wiring

Use a **3-position ON-OFF-ON (SPDT, centre off) toggle switch**:

| Switch position | Connects | Inp1 (unmarked) | Inp2 (W222) | Charge stops at |
|---|---|---|---|---|
| **Up**     | Inp1 → GND  | **Low** | High | **80 %** |
| **Centre** | nothing     | High | High | **100 %** (normal) |
| **Down**   | Inp2 → GND  | High | **Low** | **60 %** |

```
   3-position ON-OFF-ON switch (3 terminals)

   Inp1 (unmarked) ──── outer terminal A  -> lever UP    = 80 %
   board GND ────────── centre terminal      (common)
   Inp2 (W222) ──────── outer terminal B  -> lever DOWN  = 60 %

   lever CENTRE = nothing connected = 100 %
```

- The **centre (common) terminal goes to the board's ground (GND)**. The two outer terminals go
  to Inp1 and Inp2.
- On most toggle switches, pushing the lever **up** connects the **bottom** outer terminal.
  Check with a meter and swap the two outer wires if up and down are the wrong way round.
- The inputs have internal pull-ups, so no resistors are needed. Open = high = inactive.
  Grounded = low = active.
- If both inputs are somehow grounded at once, the bridge uses 60 % (the lower limit wins).
- Leaving the switch out entirely (both inputs open) = 100 % = original behaviour.

---

## Input pins and PCB labels

The board's silkscreen labels don't match the firmware names. This table maps them:

| Board label  | Firmware name | MCU pin | Use in this firmware |
|--------------|---------------|---------|----------------------|
| *(unmarked)* | Inp1          | PC12    | Pull to GND = **80 %** charge stop |
| **W222**     | Inp2          | PC11    | Pull to GND = **60 %** charge stop |
| **W116**     | Inp3          | PC10    | Not used (input, pulled up) |
| **BMW**      | Inp4          | PA15    | Not used (input, pulled up) |

All four pins are configured as inputs with internal pull-ups (`source/Src/gpio.c`). Nothing
needs to be connected to Inp3 or Inp4.

---

## Flashing

A ready-built file is in [`source/output/canbridgeWith_60_80_Switch_Input.srec`](source/output/).
It's the build tested on hardware. Later revisions only changed comments, so the code is the same.

**Flash it with `BridgeFlasher.exe` from the
[upstream repo](https://github.com/dalathegreat/Nissan-LEAF-Battery-Upgrade), not with ST-Link
directly.** Follow the upstream README for the ST-Link setup and wiring. BridgeFlasher needs the
**CLI** version of the ST-Link tool (`ST-LINK_CLI.exe`, part of STM32 ST-LINK Utility).

**Why it has to be BridgeFlasher (board lock):** the firmware checks the STM32's factory unique
chip ID at boot and keeps resetting if it doesn't match the 12 bytes stored in the firmware
(`au8_lock` in `source/Src/main.c`). Those 12 bytes in the source and the `.srec` are only a
**placeholder**. BridgeFlasher does three things:
1. Removes read protection.
2. Reads the target chip's unique ID from address `0x1FFFF7E8`.
3. Replaces the placeholder in the `.srec` with that ID, then flashes it.

So each board ends up with firmware matched to its own chip, and one `.srec` works on any number
of boards. **Don't change the `au8_lock` placeholder bytes**, or BridgeFlasher won't find them
("Failed to find replacement bytes"). If you flash the `.srec` straight from ST-Link, the bridge
will reset forever on boot.

---

## Building from source

- **IDE:** Keil µVision 5 (MDK-ARM). Open `source/MDK-ARM/jcan.uvprojx`.
- **Target:** STM32F105RC, STM32CubeF1 HAL (included under `source/Drivers/`, ST's own licences).
- A post-build step (`fromelf`) writes `source/output/canbridge.srec`. Flash that with
  BridgeFlasher as described above.

---

## Licence and disclaimer

GPL-3.0. See [`LICENSE`](LICENSE). This is a modified version of
[dalathegreat/Nissan-LEAF-Battery-Upgrade](https://github.com/dalathegreat/Nissan-LEAF-Battery-Upgrade).
The STM32 HAL/CMSIS files under `source/Drivers/` are © STMicroelectronics / Arm under their own
licences (included in those folders).

This modifies high-voltage battery messages on a vehicle CAN bus. It comes with **no warranty**,
and you use it at your own risk.
