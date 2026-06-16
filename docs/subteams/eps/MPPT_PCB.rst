MPPT PCB
========

:Status: Draft
:Project Lead: Jaron (23wbjx@queensu.ca)
:Reviewers: Name, Name
:Last Updated: 2026-06-16
:Revision: v0.1

Overview
--------

This document sets out to document every element of the Maximum Power Point Tracker (MPPT) and Power Distribution PCB. The point of this board is to safely obtain the maximum power from the solar panels, and convert it to several logic levels which other boards will use. It will also gather telermetry for the power being given to each board. It will include the following:

- An explanation of MPPT
- The Scope of this project
- A system diagram explaining the high level workings
- The boards that it will interface with
- Key equirements made for this system, along with a link to all of them
- A detailed explanation behind every page in the schematic
- A Power Budget for the board
- A detailed pinout
- A link to the Bill of Materials
- The Layout & Stackup chosen
- And a deepdive into the layout and decisions made

The goal for this is to have everything here so in a few years time, this can continue to be a useful resource for the team.

For added clarity, on this Satellite, the EPS is split into two PCBs, a battery board and the MPPT/PDB.


Working Principle
------------------

A Maximum Power Point Tracker (MPPT) extracts the maximum possible
power from a solar panel regardless of changing conditions like
sunlight intensity or temperature.

A solar panel's output voltage and current are not independent. For
any given lighting condition, the panel has a current-voltage (I-V)
curve, and at any point on that curve, power is simply
:math:`P = V \times I`. Because the curve is nonlinear, power is *not*
constant across the curve: it rises, peaks, then falls. That peak is
the Maximum Power Point (MPP), and its location shifts as sunlight and
temperature change. An MPPT's job is to continuously find and sit at
that peak.

To do this, the panel is connected directly to a DC-DC converter
(typically a buck converter) instead of directly to the battery. The
buck converter's output voltage is set by its duty cycle:

.. math::

    V_{out} = D \times V_{in}

Where:

:math:`V_{out}` is the regulated output voltage.

:math:`V_{in}` is the supply input voltage.

:math:`D` is the duty cycle (:math:`0 \le D \le 1`).

In our case, :math:`V_{out}` is fixed by the battery bus voltage, and
:math:`V_{in}` is the solar panel's voltage. Since :math:`V_{out}` is
fixed, the controller's choice of duty cycle :math:`D` directly sets
what :math:`V_{in}` must be, meaning the MPPT controller can pull the
panel's operating voltage up or down by adjusting :math:`D`, sweeping
the panel across its I-V curve. The controller continuously adjusts
:math:`D`, measures the resulting power, and converges on the duty
cycle that lands the panel voltage at the MPP.

For more detail, see the `Wikipedia article on MPPT
<https://en.wikipedia.org/wiki/Maximum_power_point_tracking>`_.

Scope & Assumptions
--------------------

This will cover as much as I can on the topic of this PCB, but assumes all other PCBs adhere to the standards set, and will take these notes into account to integrate well with one another.

System Architecture
--------------------

Here is a flow chart which captures the main logic of the PCB:

.. mermaid::

   %%{init: {
   'theme': 'neutral',
   'themeVariables': {
   'background': '#ffffff',
   'primaryColor': '#ffffff'
   }
   }}%%
   graph TD
   %% Nodes Definitions
   Solar[Solar Array]
   MPPT["4 x MPPT Converter"]
   Batteries[Batteries]
   PowerLines[Battery Board]

   BuckSolar["Local Buck Converter<br>(3.3V)"]
   IdealDiode[Ideal Diode]
   BuckExt1["Back to this board<br>3V3 External Buck"]
   BuckExt2["Back to this board<br>5V External Buck"]

   STM32[STM32 Microcontroller]
   Watchdog[External Watchdog]
   I_V_IN[I_IN, V_IN Telemetry]
   OBC[OBC]
   OBC_TEL[OBC Current Sense 3V3]
   ADCS[ADCS]
   ADCS_TEL1[ADCS Current Sense 3V3]
   ADCS_TEL2[ADCS Current Sense 5V]

   %% Main Power Path (Soft Rust/Coral)
   Solar --> MPPT
   MPPT --> Batteries
   Batteries --> PowerLines
   Batteries --> IdealDiode
   Solar --> IdealDiode
   IdealDiode --> BuckSolar
   PowerLines --> BuckExt1
   PowerLines --> BuckExt2

   linkStyle 0,1,2,3,4,5,6,7 stroke:#d98a6c,stroke-width:2.5px;

   %% Control & Housekeeping (Muted Slate Blue)
   Watchdog --> STM32
   Solar -.-> I_V_IN
   I_V_IN --> STM32
   BuckSolar --> STM32
   BuckSolar --> Watchdog

   linkStyle 8,9,10,11,12 stroke:#5b7085,stroke-width:2px;

   %% Subsystem Delivery & Telemetry (Soft Mauve/Rose)
   BuckExt1 --> OBC_TEL
   BuckExt1 --> ADCS_TEL1
   BuckExt2 --> ADCS_TEL2
   OBC_TEL --> OBC
   ADCS_TEL1 --> ADCS
   ADCS_TEL2 --> ADCS

   linkStyle 13,14,15,16,17,18 stroke:#a67585,stroke-width:2px;

   %% Force Legend to the Bottom using Invisible Links
   OBC ~~~ L1
   ADCS ~~~ L5

   %% Diagram Legend Box
   subgraph Legend ["— DIAGRAM LEGEND —"]
      direction LR
      L1[Primary Power Path] -->| | L2[ ]
      L3[Control & Housekeeping] -->| | L4[ ]
      L5[Subsystem Delivery / Tel] -->| | L6[ ]
   end

   %% Style Legend Nodes to hide the dummy targets
   style L2 width:0px,height:0px,fill:#fff,stroke:#fff;
   style L4 width:0px,height:0px,fill:#fff,stroke:#fff;
   style L6 width:0px,height:0px,fill:#fff,stroke:#fff;
   style Legend fill:#fafafa,stroke:#ddd,stroke-width:1px;

   %% Apply colors to legend lines
   linkStyle 21 stroke:#d98a6c,stroke-width:2.5px;
   linkStyle 22 stroke:#5b7085,stroke-width:2px;
   linkStyle 23 stroke:#a67585,stroke-width:2px;


Interfaces
----------

Here is an explicit list of every boundary this PCB will cross. This is the
main section for other subteams. It will allow you to know how this board will interface, if it is treated as a black box with inputs and outputs.

.. role:: raw-html(raw)
   :format: html

.. list-table::
   :header-rows: 1

   * - Interface
     - Connects To
     - Type
     - Notes
   * - Power input
     - EPS — 7.2V main batteryrail
     - Electrical
     - :raw-html:`<span style="background-color: #ffff00;">Still need to decide on current limits and conection</span>`
   * - CAN BUS x2 
     - OBC — CAN
     - Data
     - Parralell CAN lines for main communication
   * - Solar Input — ~ 5-20V
     - Solar Board
     - Power
     - Using Harwin Datamate J-Tek series right-angle versions for Connector
   * - ADCS Out x2
     - ADCS — 3V3, 5V
     - Power
     - Gives 3V3 and 5V lines to the board
   * - OBC Out 
     - OBC — 3V3
     - Power
     - Gives power to OBC
   * - Shutdown Power lines x2
     - OBC
     - GPIO 
     - Can control when ADCS gets power (along with maybe itself?)
   * - EPS Thermisters x4
     - EPS Battery Board
     - Analog 
     - :raw-html:`<span style="background-color: #ffff00;">Still need to decide connector, to be placed next to battery</span>`

Hardware Detail
----------------

Schematic
~~~~~~~~~

Embed or link the schematic (PDF/PNG export from your EDA tool, kept
in the repo under ``_static/`` or a ``hardware/`` folder so it's
version-controlled alongside this page). Walk through the design by
functional block, not net-by-net — power input, regulation, sensor
interface, protection, etc.

.. image:: /_static/[subteam]/[board_name]_schematic.png
   :alt: [Board Name] schematic
   :width: 100%

Briefly explain the purpose of each functional block and any
non-obvious design choice (why this topology, why this protection
circuit, etc.). Link to the source schematic file (KiCad/Altium/Eagle
project) in References.

Power Budget
~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Rail
     - Voltage
     - Max Current
     - Source
     - Notes
   * - [e.g. 3.3V]
     - [3.3V ± 5%]
     - [500 mA]
     - [EPS regulated bus]
     - [Worst-case load condition]

Component Selection
~~~~~~~~~~~~~~~~~~~~

For any part chosen for a non-obvious reason (radiation tolerance,
temperature rating, lead time, cost, footprint constraint), record the
decision and alternatives considered. This is what future cohorts need
when a part goes obsolete or fails and they have to requalify a
replacement.

.. list-table::
   :header-rows: 1

   * - Component
     - Part Number
     - Reason Selected
     - Alternatives Considered
   * - [e.g. Voltage regulator]
     - [MPN]
     - [Tolerance / rad-hardness / availability]
     - [Other parts evaluated and why rejected]

Pinout / Connector Mapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Pin
     - Signal
     - Direction
     - Connects To
     - Notes
   * - [1]
     - [3V3]
     - [Power in]
     - [EPS J3]
     - [Max 500mA]

Bill of Materials (BOM)
~~~~~~~~~~~~~~~~~~~~~~~~

Link to the maintained BOM file (spreadsheet or exported CSV) rather
than duplicating it in prose — this should live as a version-controlled
file so it can be diffed across revisions.

- Full BOM: [link to file in repo]
- BOM revision matching this document revision: [rev]

Layout & Stackup
~~~~~~~~~~~~~~~~~

Note layer count, key layout decisions (impedance-controlled traces,
ground plane strategy, thermal relief, keep-out zones), and link to the
board outline/mechanical fit drawing if relevant to Mech subteam
coordination.

Bring-Up & Debug Procedure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Step-by-step first-power-on procedure: what to check before applying
power, what to measure first, expected nominal values, and known
failure signatures from past bring-ups (cross-reference Lessons
Learned entries below where applicable).

#. [Step — e.g. "Verify no shorts between 3V3 and GND with multimeter
   before powering on"]
#. [Step — e.g. "Apply power at current-limited 100mA, confirm 3.3V
   rail within ±5%"]
#. [Step]

Errata
~~~~~~

Known issues with the current board revision that are accepted/not yet
fixed (distinct from Lessons Learned, which documents issues already
resolved). Update or remove entries as new revisions fix them.

- [Issue] — affects revision [X] — [workaround if any]

Traceability (V-Model)
------------------------

Requirements
~~~~~~~~~~~~

Define the requirements this project must satisfy. Use Sphinx-Needs
``req`` blocks so these appear in the global traceability matrix.

.. req:: [Requirement Title]
   :id: REQ_[SUBTEAM]_[NNN]
   :status: draft

   The subsystem shall [testable, measurable requirement statement].

Design Specifications
~~~~~~~~~~~~~~~~~~~~~~

Define the design that satisfies the requirement(s) above.

.. spec:: [Specification Title]
   :id: SPEC_[SUBTEAM]_[NNN]
   :satisfies: REQ_[SUBTEAM]_[NNN]

   Describe the implementation decision and why it satisfies the
   requirement.

Test Cases & Verification
~~~~~~~~~~~~~~~~~~~~~~~~~~

Define how the specification is verified. Link results/evidence once
testing occurs (test report, data file, photo of setup).

.. test:: [Test Case Title]
   :id: TEST_[SUBTEAM]_[NNN]
   :verifies: SPEC_[SUBTEAM]_[NNN]

   Describe the test procedure, conditions, and pass/fail criteria.

Traceability Matrix
~~~~~~~~~~~~~~~~~~~~

.. needtable::
   :filter: "[SUBTEAM]" in id
   :columns: id, title, type, satisfies, verifies

Open Risks & TBDs
--------------------

Forward-looking. Known unknowns, untested edge cases, or design
decisions still pending — distinct from Lessons Learned below, which
is retrospective. Remove items once resolved (move resolution to
Lessons Learned).

- [Risk/TBD item] — [owner] — [target resolution date]

Lessons Learned Log
----------------------

Append-only. Add an entry at the end of every prototyping or testing
phase. Never delete past entries — this is the institutional memory
that prevents repeated mistakes across cohorts.

[YYYY-MM-DD] (Issue #[N])
~~~~~~~~~~~~~~~~~~~~~~~~~~

:What Failed: [Observed problem]
:Why It Failed: [Root cause]
:Resolution: [Fix applied or follow-up action]

References
-----------

Datasheets, external standards, vendor docs, prior revisions of this
page, or related pages in other subteams.

- [Title](link or path)