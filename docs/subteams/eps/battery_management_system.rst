Battery Board
=============

:Status: Draft
:Project Lead: Donovan Woo
:Reviewers: TBD
:Last Updated: 2026-06-16
:Revision: v0.1

.. contents:: Table of Contents
   :depth: 3
   :local:

----

Overview
--------

This document provides comprehensive documentation for the EPS Battery Board, one of two PCBs
that together form the satellite's Electrical Power System (EPS). The companion board is the
MPPT/Power Distribution Board. The Battery Board is responsible for safely storing, protecting,
monitoring, and distributing power from the Li-Ion battery pack to the rest of the satellite.

This document covers:

- Battery topology and configuration rationale
- Cell-level protection and balancing architecture
- Power conditioning and switching circuitry
- IC selection with detailed justification
- Global label and schematic cross-reference mappings
- Charging methodology (CC/CV)
- Microcontroller pinout and interfacing
- PC104 bus integration
- Open risks, action items, and design change history

.. note::

   On this satellite, the EPS is split into two PCBs: the **Battery Board** (this document) and
   the **MPPT/Power Distribution Board**. This document assumes all other PCBs adhere to the
   interface standards defined here.

For the companion MPPT/PDB documentation, see the MPPT PCB document.

----

Battery Configuration
---------------------

Topology
~~~~~~~~

The battery pack uses a **2S4P** Li-Ion configuration:

- **2 series cells** → achieves the required bus voltage (~7.2 V nominal, ~8.4 V fully charged)
- **4 parallel cells per series group** → achieves the required capacity and discharge current

The battery cells used are the **NCR18650GA** (Panasonic).

Each parallel bank is treated as an independent group, with per-cell fusing to isolate individual
cell failures without taking down the whole bank.

.. list-table:: Battery Configuration Summary
   :header-rows: 1

   * - Parameter
     - Value
   * - Cell chemistry
     - Li-Ion
   * - Cell model
     - NCR18650GA
   * - Configuration
     - 2S4P
   * - Nominal voltage
     - ~7.2 V
   * - Max charge voltage
     - ~8.4 V
   * - Series groups
     - 2
   * - Parallel cells per group
     - 4
   * - Protection IC
     - BQ28Z610 (×2, split-pack)

Rationale for 2S4P
~~~~~~~~~~~~~~~~~~~

The pack was updated from 2S2P to 2S4P to handle increased current demands from the payload.
The split-pack architecture (two independent 2S2P strings each with a dedicated BQ28Z610)
was chosen over a cold-spare or series-FET configuration for the following reasons:

- **Real-time telemetry comparison** of string A vs. string B enables early fault detection,
  which is especially important given the payload's current draw.
- **No single point of failure** — partial failure of one string is survivable.
- **Reduced MOSFET heating** — each DSG/CHG FET pair handles only 50% of total current during
  normal operation. Since :math:`P = I^2 R`, halving the current reduces heat by a factor of 4.

Why 2S Protection Instead of Stacked 1S
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The original design used stacked 1S cell protection ICs (BQ2970/BQ29723). This was replaced with
the BQ28Z610 2S dedicated IC for the following reasons:

- **Floating ground risk**: if the bottom bank faults, the top bank loses its connection to
  ``PACK_N``, allowing uncontrolled current paths.
- **Doubled series resistance**: stacked 1S configurations require 4 MOSFETs in the main current
  path; the 2S IC uses only 2.
- **1S ICs were not designed to be stacked**: floating ground voltages cause sensing errors. If the bottom protection IC opens its FET the midpoint voltage is no longer anchored to anything and floats to whatever value the parasitic capacitance, leakage currents, and any stray conductive path pulls it. The top IC, whose VSS is MID, now has a completely undefined reference. Its voltage measurements become meaningless garbage.
  charge even at low state of charge.
- **Space efficiency**: the BQ28Z610 integrates IV monitoring, temperature sensing, and cell
  balancing into a single compact IC.
- **Redundancy**: if one 1S IC failed, all cells were unprotected. The 2S split-pack gives
  string-level redundancy.

----

----

IC Reference
------------

Cell-Level Protection — BQ28Z610
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:Datasheet: https://www.ti.com/lit/ds/symlink/bq28z610.pdf

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - What is it?
     - Function in this circuit
     - Limitations / Notes
   * - Dedicated 2S Li-Ion battery protection and fuel-gauge IC. Integrates cell balancing,
       current sensing, voltage sensing, temperature sensing, and state-of-health reporting.
     - Two instances used (split-pack). Each IC manages one 2S2P string.
       Protections include: cell OVP, cell UVP, charge overcurrent (OCC), discharge overcurrent
       (OCD), overload discharge, short-circuit in charge, overtemperature in charge/discharge,
       pre-charge timeout, fast-charge timeout.
       Cell balancing is passive (resistor + MOSFET bleed).
       Data output via I2C (``I2C_SDA``, ``I2C_SCK``) to the STM32 MCU.
     - I2C address is fixed at 0x55 — both ICs share the same address, requiring either a
       multiplexer or a second I2C peripheral on the MCU. Resolved by upgrading to the
       STM32F030C8T6.

.. note::

   The BQ28Z610 replaces the original BQ2970/BQ29723 1S ICs. See `Rationale for 2S Protection`_
   above for full justification.

**Why Passive Cell Balancing?**

Passive balancing bleeds excess energy as heat through resistors and MOSFETs.
Active balancing uses DC-DC converters and inductors to transfer energy between cells.

For this application, passive balancing is preferred because:

- Only 2 series cells → voltage drift between series groups is minimal.
- DC-DC converters introduce switching noise (EMI) and reduce efficiency.
- Passive components are simpler, more space-efficient, and have no additional failure modes.

**External Cell Balancing Schematic**

An external cell balancing schematic (``External_Cell_Balancing.sch``) was chosen over an
integrated balancing resistor, because the 4 parallel cells per bank require higher balancing
currents than an internal resistor can handle. Higher balancing current is necessary to
meaningfully alter the voltage of a large parallel bank.


Buck-Boost Converter — TPS63060
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:Datasheet: https://www.ti.com/lit/ds/symlink/tps63060.pdf

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - What is it?
     - Function in this circuit
     - Limitations / Notes
   * - Buck-boost switching regulator. Automatically transitions between buck and boost modes
       to maintain a regulated output regardless of whether input voltage is above or below
       the output setpoint.
     - Converts an MPPT charge input to the voltage required to charge power the watchdog and deployment timer suring launch.
       Feedback resistors R66 (560 kΩ) and R67 (100 kΩ) set the output voltage.
       Output capacitors C58, C59, C60 = 22 µF each for filtering
       Input capacitors C9, C10 = 22 µF; C11 = 0.1 µF.
       Inductor L1 = 2.2 µH. (NEEDS TO BE VERIFIED FOR SWITCHING DUTY CYCLE)
     - Input voltage range: 2.5 V – 12 V.
       Efficiency: up to 93%.
       Output current at 3.3 V (V\ :sub:`IN` < 10 V): 2 A (buck mode).
       Output current at 3.3 V (V\ :sub:`IN` > 4 V): 1.3 A (boost mode).

----

E-Fuse — TPS259472ARPWR
~~~~~~~~~~~~~~~~~~~~~~~~

:Datasheet/Source: https://www.digikey.com/en/products/detail/texas-instruments/TPS259472ARPWR/14124020

An electronic fuse (E-Fuse) was added to protect the entire battery pack output. This
supersedes relying solely on the high-side inhibit, which has no inherent protection circuitry
unless coded into the MCU (which introduces latency).

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - What is it?
     - Function in this circuit
     - Limitations / Notes
   * - Programmable electronic fuse IC.
     - Protects the full battery output path. Provides:

       - Back-to-back FETs for reverse polarity protection.
       - Dedicated analog load current monitor.
       - High voltage threshold for resilience to voltage spikes (e.g. from Electrodynamic
         Tether deployment).
       - Much faster response than PPTC fuses — prevents other subsystems from being damaged
         during a fault.
       - Programmable current limit without relying on MCU communication.
     - ``E_FUSE_EN`` signal must be **inverted** (the IC pulls this line low during a fault;
       it also does not have sufficient voltage to self-enable). Ensure proper level-shifting
       or inverter circuit on this line.

**Why not rely only on PPTC fuses?**

PPTC fuses have a slow thermal response time. The E-Fuse is significantly faster, protecting
downstream subsystems from transient fault currents before they cause damage.

----

Per-Cell PPTC Fuses
~~~~~~~~~~~~~~~~~~~~

Four PPTC (Positive Temperature Coefficient) fuses were added, one per cell in each parallel bank.

**Why add per-cell fuses?**

Each cell can develop an internal short. Without fusing, parallel cells will sink large currents
into the shorted cell to balance the parallel voltage, causing overheating and potential fire.
The per-cell fuse isolates the faulty cell, allowing the remaining cells in the bank to continue
operating.

**Why PPTC?**

- Very low voltage drop in normal operation.
- Automatically resettable — no manual intervention required.
- Single component; minimal board space impact.
- Slow response time is not detrimental in this application (the E-Fuse handles fast faults).

**Fuse Rating**

Each fuse was selected with a trip current of **4.5 A**. With 4 cells, this gives the pack
a total fault current budget of **18 A** before tripping.

----

Ideal Diodes — LM74800-Q1
~~~~~~~~~~~~~~~~~~~~~~~~~~

:Datasheet: https://www.ti.com/lit/ds/symlink/lm7480-q1.pdf

Ideal diodes were added between the outputs of the two 2S2P battery strings to prevent
back-feeding current from one string into the other when the strings are at different states
of charge.

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - What is it?
     - Function in this circuit
     - Limitations / Notes
   * - Active ideal diode controller (controls external N-channel MOSFETs to emulate a very
       low forward-voltage diode).
     - Placed at the outputs of each 2S2P string. Prevents reverse current flow between
       parallel strings with different voltages. Enables safe OR-ing of the two strings.
     - Operating temperature: −40 °C to +125 °C.
       Forward voltage drop: ~10.5 mV.
       Quiescent current: 35 µA.
       Reverse blocking response time: 0.5 µs.
       Not yet extensively flight-heritage tested in space.

**4-MOSFET Architecture (2 per Rail)**

To achieve true power path isolation and complete logical shutdown via the ``EN/UVLO`` pin,
each power rail uses two N-channel MOSFETs in a back-to-back common-source configuration:

A single MOSFET cannot fully isolate the rail because its parasitic body diode allows leakage
current even when the gate is off. With two back-to-back FETs, the body diodes face opposite
directions, providing a complete block in both forward and reverse directions.

- **MOSFET 1 (HGATE)**: disconnect switch — blocks forward leakage on shutdown.
- **MOSFET 2 (DGATE)**: active ideal diode — blocks reverse back-feeding from a
  higher-voltage parallel string.

**Open-Drain NMOS Enable Switch**

The MOSFET configuration above allows for the ideal diode to be controlled through the ''EN/UVLO'' pin. This allows the ideal diodes to serve as a high side inhibit rather than having a separate one, reducing part count and extra modes of failure that come with additional component count. The ``EN/UVLO`` pin is controlled by an open-drain inverter to allow digital logic control:

- Pull-up resistor (10 kΩ – 100 kΩ) between ``V_SNS`` and ``EN/UVLO`` → default ON state.
- Small-signal NMOS (e.g. 2N7002): Drain to ``EN/UVLO``, Source to GND, Gate driven by MCU.



.. list-table:: Enable Switch Logic
   :header-rows: 1

   * - Control Input (Gate)
     - NMOS State
     - EN/UVLO Voltage
     - Power Path Status
   * - Logic HIGH (3.3 V / 5 V)
     - ON (closed)
     - 0 V (clamped to GND)
     - **Shutdown** — back-to-back FETs isolate load; output drops to ~0 V
   * - Logic LOW (0 V)
     - OFF (open)
     - V\ :sub:`SNS` (pulled high)
     - **Active OR-ing** — 1.4 ms soft-start ramp; FETs fully enhanced
**Ideal Diode Placement: High-Side vs. Low-Side**

The ideal diodes are placed on the **positive (high-side) rail**, not in the ground return path.

Placing switching elements on the high-side preserves a continuous, unbroken ground plane
shared across all subsystems. This is critical for I2C signal integrity — if the ideal diode
were in the low-side (ground) path, the ground reference of the battery string would sit at
a different potential than the system ground whenever the switch was in transition. Any current
seeking to return to ground could find an alternative path through I2C data lines or other
shared signal cables, potentially damaging downstream ICs.

High-side placement also provides natural back-feeding protection: if one string is at a
higher voltage than the other, the ideal diode blocks reverse current on the positive rail
before it can reach the lower-voltage source, with the ground plane remaining undisturbed
throughout.

.. warning::

   **PCB Layout Critical Rules for LM74800-Q1:**

   - The **Thermal Pad (Pin 13 / RTN)** must be completely isolated from the main GND plane.
     It must sit on a standalone copper island tied strictly to the ``RTN`` net to prevent
     destroying the IC's ESD substrate during a reverse-battery fault.
   - Add **10 Ω – 47 Ω series gate resistors** on the power MOSFETs to damp high-frequency
     switching transients and prevent parasitic ringing.

----

Timers — LTC6995HS6-1#TRMPBF (Deployment & Watchdog)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:Datasheet: https://www.analog.com/media/en/technical-documentation/datasheets/LTC6995-6695-1-6695-2.pdf

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - What is it?
     - Function in this circuit
     - Limitations / Notes
   * - Silicon oscillator IC designed for long-duration timing events (seconds range).
       Generates a 50% duty cycle square wave.
       Frequency set by a voltage divider on the ``DIV`` pin (two selectable settings via
       connector pins).
       Has a hardware reset feature.
     - Two instances:

       **Watchdog Timer**: monitors the MCU. Oscillates at 2.5 s with a 1.25 s delay from the
       last received PPS signal from the OBC. If PPS is not received in time, the watchdog
       resets the MCU.

       **Deployment Timer**: ensures ``EN_D1`` is only activated after the satellite has
       completed deployment. The timer output connects to the low-side inhibit ground path;
       nothing can be activated during deployment. The ANDed output of watchdog + deployment
       timer (in ``power_control_RBF.sch``) provides dual-redundant safety to prevent false
       positive activations.
     - Supply voltage: max 6 V.
       Operating temperature: −40 °C to +125 °C.

----

High-Side Inhibit — TPS24750 (Under Revision)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:Datasheet: https://www.ti.com/lit/ds/symlink/tps24750.pdf

High-side inhibits cut power *before* the load, isolating the load from the source (battery pack)
when commanded off. This prevents shorts-to-ground faults from propagating.

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - What is it?
     - Function in this circuit
     - Limitations / Notes
   * - Positive hot-swap controller IC. Provides inrush current control, undervoltage lockout
       (UVLO), and load overvoltage protection.
       Timer capacitor sets the output voltage rise time (see datasheet p. 29).
       C = 0.1 µF bypass capacitor from drain to GND.
     - Isolates the battery pack from the load when off.
       Activates when ``EN_D1`` goes high — this signal is the ANDed output of the watchdog
       and deployment timer (from ``power_control_RBF.sch``).
       Output (``PCM_IN``) feeds into the Power Conditioning Module (a separate subsystem).
     - Bus operating range: 2.5 V – 18 V.

----

Low-Side Inhibit — NTJD1155L (Under Revision)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:Datasheet: https://www.onsemi.com/pdf/datasheet/ntjd1155l-d.pdf

Low-side inhibits cut the *ground* connection of the load from the source. Placed between the
load and ``PACK_N`` (pack negative terminal). This is a mandatory launch safety requirement to
prevent hazardous operation during launch.

.. list-table::
   :header-rows: 1
   :widths: 30 40 30

   * - What is it?
     - Function in this circuit
     - Limitations / Notes
   * - Dual P+N channel MOSFET load switch. The load connects directly to the positive rail;
       the ground return path is switched.
       Pull-up resistor R: 10 kΩ – 1 MΩ.
     - Controlled by ``EN_D3`` (from the deployment timer in ``power_control_RBF.sch``).
       When the deployment timer activates, the circuit becomes connected to ground, allowing
       current to flow.
     - Supply line voltage: 1.8 V – 8 V.
       Max current: ±1.3 A. **This component is under review** — the entire battery discharge
       current passes through this device; the package is too small to handle the required
       current.

.. warning::

   **NTJD1155L is being replaced.** The entire battery current return path runs through this
   device, exceeding its 1.3 A rating. Candidate replacements:

   - **FDC6318P** (dual P-channel): larger footprint; requires a NOT gate to invert the
     enable signal since it is P-channel rather than N+P.
   - **2× discrete MOSFETs (P + N)**: requires a gate resistor and space verification.

   Additionally, the **ferrite bead** connected to the negative battery terminal has strict
   current limits and must be reviewed if the low-side switch is on the same net.

**Why Disconnect Ground During Launch?**

- Intense vibrations can cause electrostatic discharge events.
- There is an increased risk of stray currents with the ground connected during launch.
- This is a **mandatory launch requirement** per CubeSat standards.

----

Microcontroller — STM32F030C8T6
--------------------------------

:Replaces: STM32F030F4P6
:Reason for Upgrade: The BQ28Z610 has a fixed I2C address (0x55). With two ICs in the
   split-pack design, the original STM32 lacked a second I2C peripheral to address them
   independently. A multiplexer was considered but rejected in favour of upgrading the MCU.

**Why Upgrade Rather than Add a Multiplexer?**

- One fewer component that can fail.
- Two independent I2C ports provide full electrical isolation between the two BQ28Z610 strings.
- Standard I2C drivers can be used directly, rather than custom multiplexer timing code
  (which is prone to timing errors).
- Additional benefits: more RAM for data and calculations, extra GPIO for test LEDs, more
  thermistor inputs, essentially no additional power consumption versus the F4P6 variant.

Crystal Oscillator
~~~~~~~~~~~~~~~~~~~

An external quartz crystal is connected to pin **PF1** to serve as the MCU clock source.
External quartz crystals maintain frequency stability across a wide temperature range,
unlike internal RC oscillators, which is important in the thermal cycling environment of orbit.

.. note::

   Crystal load capacitance:
   :math:`C = (\text{Load Capacitance} - \text{Stray Capacitance}) \times 2`

   Stray capacitance is typically 3–5 pF. See AN2867 (STM Oscillator Design Guide).

Pinout
~~~~~~

.. list-table::
   :header-rows: 1

   * - Pin
     - Signal
     - Direction
     - Notes
   * - PA4, PA5
     - Battery heater control
     - Output
     - Gate drive voltage to heater MOSFETs (located in root schematic)
   * - PA6
     - ``BAT-TEMP``
     - Input
     - Encoded battery temperature from the temperature sensor (root schematic)
   * - PA9, PA10
     - ``I2C_SDA``, ``I2C_SCK``
     - I2C
     - Battery voltage & current data from INA219 (``VI_Monitor.sch``)
   * - PB1
     - PC104 bus signal
     - Output
     - Leads to the PC104 bus connector
   * - PF1
     - Crystal oscillator
     - Clock
     - External 16 MHz quartz crystal
   * - PA11, PA12
     - Available
     - TBD
     - Reserved for future use

Power Supply (Under Revision)
~~~~~~~~~~~~

The MCU takes power from the 3.3 V bus (though perhaps it may be better if sourced from the MPPT). A 100 Ω / 100 MHz ferrite bead (FB7)
is placed in line on the 3.3 V supply for noise filtering. Decoupling capacitors are placed
on ``3V3BUS`` and ``3V3A`` rails per STM32 recommended layout.

----

PC104 Bus Connector
-------------------

The PC104 stackthrough connector (``J?``) connects the EPS Battery Board to the rest of the
satellite subsystems. It carries both regulated and unregulated voltage buses, I2C telemetry,
and control signals.

.. list-table::
   :header-rows: 1
   :widths: 20 25 15 40

   * - Net / Signal
     - Connector Side
     - Direction
     - Notes
   * - ``BAT_INT``
     - H1 (female / top)
     - Out
     - Unregulated battery bus
   * - ``EPS_INT``
     - H1
     - Out
     - EPS internal bus
   * - ``5VBUS``
     - H1 / H2
     - Out
     - Regulated 5 V bus
   * - ``3V3BUS``
     - H1 / H2
     - Out
     - Regulated 3.3 V bus
   * - ``5V_USB_CHG``
     - H1
     - In
     - USB charge input
   * - ``PCM_IN``
     - H1 / H2
     - In/Out
     - Power conditioning module interface
   * - ``BCR_OUT``
     - H1 / H2
     - Out
     - Battery charge regulator output to bus
   * - ``I2C_SDA``, ``I2C_SCK``
     - H1
     - Bidirectional
     - I2C to communicate with rest of satellite
   * - GND
     - H1 / H2
     - —
     - Multiple ground pins distributed across connector

.. note::

   Approximate maximum current per connector: ~3 A (needs verification).

   **Open question:** What are ``EPS_INT`` and ``BAT_INT`` exactly? These need to be formally
   defined in coordination with the MPPT/PDB and OBC subteams.

   Also needed: RBF (Remove Before Flight) pins and deployment switch connections — confirm
   PC104 standard with COMMS and OBC.

**I2C Signal Integrity**

Zener diodes with resistors on both ends of the I2C lines (``I2C_SDA``, ``I2C_SCK``) have been
added for ESD protection and to suppress transients that could corrupt communication.

----

Charging Architecture
---------------------

Overview
~~~~~~~~

Charging is **handled on the MPPT/Solar board**, not the Battery Board. The Battery Board
participates in charging only through its protection ICs (which gate the charge FETs).

The EPS operates in two charging modes:

MPPT Mode (Constant-Current Phase)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:When: Battery voltage is below the End-of-Charge (EoC) threshold.
:How: The MPPT algorithm operates the solar panel at its Maximum Power Point for maximum
      power transfer. This delivers constant current to the battery as voltage rises.

EoC Mode (Constant-Voltage Phase)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:When: Battery voltage reaches the EoC threshold (~8.4 V for 2S).
:How: The solar array operating point is shifted *away* from the MPP. The battery is held at
      constant voltage while current tapers to near-zero. Excess solar power is dissipated
      as heat on the array.

Charging Phases in Detail
~~~~~~~~~~~~~~~~~~~~~~~~~~

**Pre-Charge**

A low-current phase used when the battery has been drained below the CC charge threshold
(approximately 2.5 V per cell). Pre-charge brings cells up to the point where normal CC
charging can commence safely.

**Thermal Regulation**

At the start of CC charging, the large difference between initial and final cell voltages
causes high power dissipation and heat generation. Once the cell voltage rises sufficiently,
the current no longer generates as much heat and the thermal regulation phase ends.

**Constant-Current (CC)**

Current is held constant at a fixed rate while voltage rises freely, up to the maximum cell
voltage (~4.1 V per cell for a 3.7 V nominal cell). A feedback loop monitors the duty cycle
of the DC-DC converter to prevent overcurrent as the battery voltage rises.

**Constant-Voltage (CV)**

The circuit holds the output at a constant voltage while current tapers naturally to near-zero
as the battery approaches full charge.



Power Distribution Module (PDM) (Under Revision)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Downstream of the PCM. Provides:

- Commandable load switching (OBC can turn individual outputs on/off via I2C)
- Individual LCLs per output

Emergency / Safe Mode Power Strategy (Under Revision)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In SOS/safe mode, the following subsystems must remain operational (per advisor recommendation):

- **ADCS** — attitude determination and control
- **EPS** — power management
- **COMMS** — communications

Combined power requirement in SOS mode: ~17 W. This figure was determined in April 2025, and may be subject to change.

All other subsystems should be commandably shed. Inrush current for each subsystem must be
accounted for when designing the LCL trip thresholds.

----

Interfaces
----------

.. list-table::
   :header-rows: 1

   * - Interface
     - Connects To
     - Type
     - Notes
   * - Battery pack output
     - High-side inhibit → PCM
     - Power
     - Main battery bus. MOSFETs must be rated at ≥ 2× battery voltage due to spikes from
       Electrodynamic Tether deployment.
   * - ``BCR_OUT``
     - Battery pack input / PC104 bus
     - Power
     - Output of TPS63060; feeds pack and bus simultaneously.
   * - I2C (``I2C_SDA`` / ``I2C_SCK``)
     - STM32 ↔ INA219, BQ28Z610 ×2
     - Data
     - Internal telemetry bus. Zener ESD protection on both lines.
   * - PC104 bus
     - OBC, ADCS, COMMS, and other subsystems
     - Power + Data
     - Stackthrough connector; carries regulated buses and I2C telemetry.
   * - ``EN_D1``
     - High-side inhibit enable
     - GPIO
     - ANDed output of watchdog timer + deployment timer. Active high.
   * - ``EN_D3``
     - Low-side inhibit (PACK_N switch)
     - GPIO
     - Deployment timer output. Enables ground return path post-deployment.
   * - Battery thermistors
     - STM32 PA6 (``BAT-TEMP``)
     - Analog
     - Temperature monitoring for thermal runaway prevention.
   * - PPS from OBC
     - Watchdog timer input
     - GPIO
     - If PPS not received within 1.25 s, watchdog resets MCU.
   * - Battery heater
     - STM32 PA4, PA5 → heater MOSFETs
     - GPIO / Power
     - MCU controls heater gate voltage to maintain battery temperature in eclipse.

----

Signal Integrity & EMI Mitigation
-----------------------------------

The following measures have been implemented to ensure accurate telemetry and reliable
communication in the electromagnetically noisy switching environment:

- **Low-pass filter capacitors** on the BQ28Z610's ``VC1``, ``VC2``, ``SRP``, and ``SRN`` pins
  to reduce EMI coupling into the ADCs used for voltage and current sensing.
- **Zener diodes with series resistors** on both I2C lines (``I2C_SDA``, ``I2C_SCK``) for
  ESD suppression and transient protection.
- **10 Ω – 47 Ω series gate resistors** on power MOSFETs to damp switching-induced ringing.
- **Ferrite bead** (100 Ω @ 100 MHz) on the MCU 3.3 V supply rail.
- **Decoupling capacitors** on all IC supply pins per each IC's datasheet recommendation.

.. note::

   The ferrite bead on ``PACK_N`` has strict current limits. Verify that the bead's rated
   current is sufficient for the full battery discharge current before finalising layout.

----

Component Change Log
--------------------

.. list-table::
   :header-rows: 1

   * - Old Component
     - New Component
     - Reason
   * - BQ2970 / BQ29723 (1S cell protection ×4)
     - BQ28Z610 (2S protection ×2)
     - 2S IC integrates protection, balancing, IV monitoring, and temperature sensing.
       Eliminates floating ground risk and stacked-1S limitations. See `Cell-Level Protection`_
       for full rationale.
   * - STM32F030F4P6
     - STM32F030C8T6
     - Two BQ28Z610 ICs share I2C address 0x55; the F4P6 only has one I2C peripheral.
       C8T6 has two independent I2C peripherals and additional GPIO/RAM.
   * - No per-cell fusing
     - PPTC fuses (one per cell, trip at 4.5 A)
     - Isolate individual shorted cells without taking down parallel bank.
   * - No E-Fuse
     - TPS259472ARPWR
     - Fast electronic protection for the full pack output; faster than PPTC alone.
   * - No ideal diodes
     - LM74800-Q1 (×2)
     - Prevent back-feeding between the two 2S2P strings.
   * - NTJD1155L (low-side inhibit)
     - Under review (FDC6318P or 2× discrete MOSFETs)
     - Original device cannot handle the full battery return current (max 1.3 A rating).
   * - No series cell balancing IC
     - BQ28Z610 (integrated passive balancing)
     - Prevents individual cell overcharge; extends pack lifespan.


----

Open Risks & TBDs
------------------

.. list-table::
   :header-rows: 1

   * - Risk / TBD
     - Owner
     - Target Resolution
   * - Low-side inhibit (NTJD1155L) replacement not finalised — FDC6318P vs. 2× discrete
       MOSFETs. Verify current rating, package size, and gate drive requirements.
     - TBD
     - Before PCB layout freeze
   * - Ferrite bead on ``PACK_N`` current rating — must be verified against peak discharge
       current.
     - TBD
     - Before PCB layout freeze
   * - PC104 bus connector current limit (~3 A assumed) — needs formal verification with
       standard and OBC/COMMS subteams.
     - TBD
     - Coordinator review
   * - ``EPS_INT`` and ``BAT_INT`` net definitions — purpose and routing not yet formally
       documented.
     - TBD
     - Schematic review
   * - ``E_FUSE_EN`` signal inversion and level-shifting circuit — component values not yet
       chosen.
     - TBD
     - Schematic update
   * - MOSFET ratings — all MOSFETs in the main power path must be rated at ≥ 2× battery
       voltage to withstand spikes from Electrodynamic Tether deployment.
     - TBD
     - Component selection review
   * - Inrush current limits — LCL trip thresholds for each subsystem not yet defined.
     - TBD
     - System-level power budget
   * - BQ28Z610 filter capacitor values (``VC1``, ``VC2``, ``SRP``, ``SRN``) — values not
       yet chosen. Document rationale when selected.
     - TBD
     - Schematic update
   * - Charging implementation — optocoupler vs. optoemulator vs. buck + op-amp feedback.
       Finalise approach considering radiation environment.
     - TBD
     - Architecture review
   * - What systems remain powered in SOS mode — formal power budget for safe mode not yet
       finalised. Advisor recommends ADCS + EPS + COMMS (17 W).
     - TBD
     - System-level review
   * - Death-of-discharge scenario — behaviour and recovery if battery fully depletes not
       yet defined.
     - TBD
     - Firmware + hardware review
   * - CAN bus changes on PC104 — flagged in schematic comment (DW1). Confirm final CAN
       routing.
     - TBD
     - OBC coordination

----

Action Items
------------

Hardware

- [ ] Select replacement for NTJD1155L low-side inhibit — compare FDC6318P (dual P-channel,
      requires signal inversion) against 2× discrete P+N MOSFETs. Verify rated current covers
      full battery discharge path through ``PACK_N``. Check whether a NOT gate or inverter
      is needed for gate drive.
- [ ] Verify ferrite bead on ``PACK_N`` is rated for full battery discharge current.
- [ ] Verify all MOSFETs in the main power path are rated ≥ 2× battery voltage to withstand
      transient spikes from Electrodynamic Tether deployment.
- [ ] Choose discrete component values for E-Fuse (TPS259472ARPWR) circuit — set current
      limit threshold and design the ``E_FUSE_EN`` inversion and level-shifting circuit.
- [ ] Choose filter capacitor values for BQ28Z610 ``VC1``, ``VC2``, ``SRP``, and ``SRN``
      pins. Document rationale for chosen values.
- [ ] Confirm inductor L1 = 2.2 µH is appropriate for TPS63060 switching duty cycle at
      the chosen output voltage and load range.
- [ ] Resolve low-side inhibit ground disconnect — confirm whether the low-side inhibit
      should be on the high-current ``PACK_N`` path, or whether the ferrite bead on that
      net must be replaced or removed.

Schematic & Design

- [ ] Define ``EPS_INT`` and ``BAT_INT`` net names formally — document purpose and routing
      in coordination with MPPT/PDB subteam.
- [ ] Determine whether MCU 3.3 V supply should be sourced from the Battery Board local
      regulator or from the MPPT board. Document rationale and update schematic accordingly.
- [ ] Review MPPT and Battery Board schematics together and map potential failure modes
      across the boundary.
- [ ] Confirm CAN bus routing change on PC104 (flagged in schematic comment DW1) with OBC
      subteam.
- [ ] Confirm PC104 bus connector current limit and pin assignments with COMMS and OBC
      subteams. Clarify RBF and deployment switch pin locations.
- [ ] Confirm whether batteries can be mounted below the PCB and coordinate with Mechanical and MPPT (for kelvin and temperature sensing).

Firmware & Testing

- [ ] Define which subsystems remain powered in SOS/safe mode and formalise the 17 W
      power budget. Confirm figures are still current (originally set April 2025).
- [ ] Define behaviour and recovery procedure for death-of-discharge (battery fully depleted
      below pre-charge threshold).
- [ ] Define inrush current profile for each subsystem and set LCL trip thresholds
      accordingly.
- [ ] Determine what needs to be programmed on the STM32 — list firmware modules required
      (BQ28Z610 I2C polling, heater control, watchdog PPS handling, telemetry reporting).
- [ ] Measure STM32F030C8T6 current consumption and add to power budget.

Procurement

- [ ] Generate KiCad Bill of Materials.
- [ ] Purchase test stock: BQ29737 ICs, CSD16406Q3 MOSFETs, 330 Ω resistors, 2.2 kΩ
      resistors, 0.1 µF capacitors.
- [ ] Verify PPTC fuse trip current is correct for the series-connected cell pairs —
      confirm 4.5 A per cell is appropriate given the 2S4P topology and expected peak
      discharge current.

----

Traceability (V-Model)
------------------------

Requirements
~~~~~~~~~~~~

.. req:: Battery pack bus voltage
   :id: REQ_EPS_001
   :status: draft

   The battery pack shall provide a nominal bus voltage of 7.2 V in a 2S Li-Ion
   configuration, with a maximum charge voltage not exceeding 8.4 V.

.. req:: Cell-level fault protection
   :id: REQ_EPS_002
   :status: draft

   Each Li-Ion cell shall be protected against overvoltage (OVP), undervoltage (UVP),
   overcurrent in charge (OCC), overcurrent in discharge (OCD), overload in discharge,
   short circuit in charge, and overtemperature in charge and discharge.

.. req:: Cell balancing
   :id: REQ_EPS_003
   :status: draft

   The battery pack shall balance series cells during charging to prevent individual cell
   overvoltage and to extend pack lifespan.

.. req:: Launch safety inhibit
   :id: REQ_EPS_004
   :status: draft

   The battery pack shall be electrically inhibited from supplying current to any load
   during launch and until satellite deployment is confirmed, in accordance with CubeSat
   launch provider requirements.

.. req:: Pack-level overcurrent protection
   :id: REQ_EPS_005
   :status: draft

   The battery pack output shall be protected against overcurrent and reverse polarity
   conditions before current reaches any downstream subsystem.

.. req:: Per-cell fault isolation
   :id: REQ_EPS_006
   :status: draft

   A single shorted cell shall be isolatable from its parallel bank without interrupting
   power delivery from the remaining cells.

.. req:: Inter-string isolation
   :id: REQ_EPS_007
   :status: draft

   The two 2S2P battery strings shall be electrically isolated from one another to prevent
   reverse current flow from a higher-voltage string into a lower-voltage string.

.. req:: Battery telemetry
   :id: REQ_EPS_008
   :status: draft

   The EPS shall measure and report battery pack voltage, current, temperature, and state
   of charge for each string independently over I2C to the STM32 MCU.

.. req:: Thermal protection
   :id: REQ_EPS_009
   :status: draft

   The EPS shall monitor battery temperature and shall activate a heater circuit to
   maintain battery temperature within the safe operating range during eclipse.

.. req:: Watchdog reset
   :id: REQ_EPS_010
   :status: draft

   The MCU shall be reset by an external hardware watchdog if a PPS signal from the OBC
   is not received within 1.25 s, without requiring MCU firmware intervention.

.. req:: Safe mode power continuity
   :id: REQ_EPS_011
   :status: draft

   In safe mode, the EPS shall maintain power to ADCS, EPS, and COMMS subsystems. Total
   power budget for safe mode shall not exceed 17 W (figure subject to revision).

Design Specifications
~~~~~~~~~~~~~~~~~~~~~~

.. spec:: 2S4P split-pack battery topology
   :id: SPEC_EPS_001
   :satisfies: REQ_EPS_001, REQ_EPS_003, REQ_EPS_007

   Two independent 2S2P battery strings using NCR18650GA cells. Each string is managed by
   a dedicated BQ28Z610 2S protection IC with integrated passive cell balancing (external
   balancing schematic ``External_Cell_Balancing.sch``). LM74800-Q1 ideal diode controllers
   with back-to-back N-channel MOSFETs are placed at each string output to prevent
   reverse current flow between strings. This topology eliminates the floating ground
   failure mode of stacked 1S ICs and halves the current through each FET pair, reducing
   :math:`I^2R` losses by a factor of four versus a single-string design.

.. spec:: BQ28Z610 cell-level protection
   :id: SPEC_EPS_002
   :satisfies: REQ_EPS_002, REQ_EPS_003, REQ_EPS_008

   Two BQ28Z610 ICs, one per 2S2P string, provide OVP, UVP, OCC, OCD, overload discharge,
   short-circuit-in-charge, and overtemperature protection for charge and discharge. Each IC
   uses two independent ADCs to sample cell voltage and current simultaneously, providing
   accurate state-of-health telemetry. Passive cell balancing is handled by an external
   balancing schematic to support the higher balancing currents required by the 4-cell
   parallel banks. Low-pass filter capacitors on ``VC1``, ``VC2``, ``SRP``, and ``SRN``
   reduce EMI coupling into the ADCs. Both ICs communicate over I2C (fixed address 0x55)
   using two independent I2C peripherals on the STM32F030C8T6.

.. spec:: Per-cell PPTC fusing
   :id: SPEC_EPS_003
   :satisfies: REQ_EPS_006

   One PPTC fuse rated at 4.5 A trip current is placed in series with each cell. An
   internal cell short causes parallel cells to sink excessive current into the faulted
   cell; the PPTC fuse trips and isolates the shorted cell, allowing the remaining cells
   in the bank to continue operating. PPTC fuses are automatically resettable and
   introduce a negligible voltage drop under normal conditions.

.. spec:: E-Fuse pack output protection
   :id: SPEC_EPS_004
   :satisfies: REQ_EPS_005

   A TPS259472ARPWR electronic fuse is placed at the battery pack output. It provides
   back-to-back FET reverse polarity protection, a dedicated analog current monitor,
   and a programmable fast-trip current limit that does not rely on MCU communication.
   The E-Fuse response is significantly faster than the PPTC fuses, protecting downstream
   subsystems from transient fault currents. The ``E_FUSE_EN`` signal requires inversion
   and level-shifting before connection to the MCU (component values TBD).

.. spec:: Deployment timer and launch inhibit
   :id: SPEC_EPS_005
   :satisfies: REQ_EPS_004

   An LTC6995HS6-1 silicon oscillator deployment timer controls the low-side inhibit
   (PACK_N ground path switch), preventing any battery current from flowing until the
   satellite has completed deployment. A second LTC6995 instance acts as the watchdog
   timer. The high-side inhibit (TPS24750) is gated by the ANDed output of both timers
   (``EN_D1`` in ``power_control_RBF.sch``), providing dual-redundant safety against
   false positive activation.

.. spec:: STM32F030C8T6 microcontroller
   :id: SPEC_EPS_006
   :satisfies: REQ_EPS_008, REQ_EPS_009, REQ_EPS_010

   The STM32F030C8T6 MCU provides two independent I2C peripherals to poll both BQ28Z610
   ICs at their shared address (0x55) without a multiplexer. It receives battery
   temperature via PA6, controls heater MOSFETs via PA4 and PA5, and receives pack
   voltage and current from the INA219 via PA9/PA10. An external 16 MHz quartz crystal
   on PF1 provides a stable clock reference across the thermal cycling range of orbit.
   An external hardware watchdog (LTC6995) resets the MCU if PPS from the OBC is not
   received within 1.25 s.

.. spec:: Safe mode load shedding
   :id: SPEC_EPS_007
   :satisfies: REQ_EPS_011

   The PDM provides commandable load switching via I2C from the OBC, with individual
   LCLs per output. In safe mode, all non-essential loads are shed and power is
   maintained only to ADCS, EPS, and COMMS. The 17 W safe mode power figure must be
   validated against the final power budget before PDM LCL thresholds are set.

Test Cases & Verification
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. test:: Battery Voltage Rail
   :id: TEST_EPS_001
   :verifies: SPEC_EPS_001

   Apply a calibrated DC load to the battery pack output and verify bus voltage is within
   7.0 V – 8.4 V under all expected load conditions. Measure with a calibrated multimeter.
   Pass criterion: voltage within ±2% of expected value at each load step.

.. test:: OVP/UVP Fault Activation
   :id: TEST_EPS_002
   :verifies: SPEC_EPS_001

   Drive a single cell above the BQ28Z610 OVP threshold and verify that the charge FET is
   disabled within the IC's specified fault response time. Repeat for UVP.

.. test:: Launch Inhibit
   :id: TEST_EPS_003
   :verifies: SPEC_EPS_003

   With deployment timer in the pre-deployment (inhibit) state, verify that no voltage appears
   at any load output. Simulate deployment switch activation and confirm power is enabled
   within the timer's specified delay.

.. test:: Telemetry Accuracy
   :id: TEST_EPS_004
   :verifies: SPEC_EPS_004

   Compare BQ28Z610 reported current and voltage against calibrated bench measurements across
   a range of charge/discharge currents. Pass criterion: ≤ 1% error on current, ≤ 0.5% on
   voltage.

----

Bring-Up & Debug Procedure
----------------------------

#. **Pre-power checks**: Verify no short circuit between ``PACK_P``/``PACK_N`` and GND using
   a multimeter in continuity mode.
#. **Verify PPTC fuse placement**: Confirm each PPTC fuse is in series with its respective
   cell before connecting the battery pack.
#. **Inhibit state check**: With deployment timer in inhibit state, confirm that ``EN_D1``
   and ``EN_D3`` are logic LOW and no output voltage is present on any load rail.
#. **Apply power at current limit**: Connect bench supply at 3.3 V, 100 mA current limit.
   Confirm STM32 powers up and crystal oscillator starts (measure PF1 for 16 MHz clock).
#. **I2C communication**: Scan I2C bus (using STM32 or a logic analyser) and confirm BQ28Z610
   responds at address 0x55 on both I2C peripherals.
#. **Deployment timer simulation**: Simulate deployment switch activation. Verify that the
   low-side inhibit and high-side inhibit enable in sequence.
#. **Battery pack connection**: With all protection verified, connect battery pack. Monitor
   bus voltage and confirm it is within expected range (~7.2 V – 8.4 V).
#. **Charge cycle test**: Initiate a charge cycle and verify CC and CV phases transition
   correctly, and that the BQ28Z610 reports state-of-charge progression.
#. **Fault injection**: Force an overvoltage or overcurrent condition and confirm the
   BQ28Z610 opens the appropriate FET within the rated response time.

----

Errata
------

- No known errata for v0.1. Update this section as issues are discovered and accepted
  without fix for the current revision.

----

Lessons Learned Log
--------------------

Append-only. Add an entry after each prototyping or testing phase.

[2026-03-11] (Issue #1)
~~~~~~~~~~~~~~~~~~~~~~~~

:What Failed: Stacked 1S cell protection ICs (BQ2970) created a floating ground risk when
              the bottom bank entered a fault state.
:Why It Failed: 1S ICs were not designed to be stacked in series. The top bank lost its
                connection to PACK_N, allowing uncontrolled current paths.
:Resolution: Replaced with dedicated 2S BQ28Z610 ICs in a split-pack (2S2P × 2) topology.

[2026-03-12] (Issue #2)
~~~~~~~~~~~~~~~~~~~~~~~~

:What Failed: STM32F030F4P6 had only one I2C peripheral, insufficient for two BQ28Z610 ICs
              at the same fixed address (0x55).
:Why It Failed: IC address is hardwired; cannot be changed. A multiplexer was initially
                considered but added complexity.
:Resolution: Upgraded to STM32F030C8T6, which has two independent I2C peripherals.

----

References
----------

- BQ28Z610 Datasheet: https://www.ti.com/lit/ds/symlink/bq28z610.pdf
- BQ2970 Datasheet: https://www.ti.com/lit/ds/symlink/bq2970.pdf
- INA219 Datasheet: https://www.ti.com/lit/ds/symlink/ina219.pdf
- TPS63060 Datasheet: https://www.ti.com/lit/ds/symlink/tps63060.pdf
- LTC6995 Datasheet: https://www.analog.com/media/en/technical-documentation/datasheets/LTC6995-6695-1-6695-2.pdf
- TPS24750 Datasheet: https://www.ti.com/lit/ds/symlink/tps24750.pdf
- NTJD1155L Datasheet: https://www.onsemi.com/pdf/datasheet/ntjd1155l-d.pdf
- FDC6318P Datasheet (replacement candidate): https://www.onsemi.com/pdf/datasheet/fdc6318pd.pdf
- LM74800-Q1 Datasheet: https://www.ti.com/lit/ds/symlink/lm7480-q1.pdf
- TPS259472ARPWR (E-Fuse): https://www.digikey.com/en/products/detail/texas-instruments/TPS259472ARPWR/14124020
- BQ25887RGET (2S Charger): https://www.digikey.ca/en/products/detail/texas-instruments/BQ25887RGET/10270216
- CC-CV with op-amps: https://www.ti.com/lit/ab/slla619/slla619.pdf
- BQ25887 Application Notes: https://www.ti.com/lit/an/slua938/slua938.pdf
- NASA CubeSat 101: https://www.nasa.gov/wp-content/uploads/2017/03/nasa_csli_cubesat_101_508.pdf
- STM32 AN2867 Oscillator Design Guide
- Clyde Space EPS Manual (cc-cv charging reference, Figure 9-1, p. 28)
- LCL reference: https://www.3d-plus.com/products/space-radiation-tolerant-latch-up-current-limiter-lcl-protection/
- Sensitron LCL: https://www.sensitron.com/data_sheets/5100.pdf
