# FPGA Combo-Lock Safe System

## Project Overview

This project is a digital security system designed using **SystemVerilog** and deployed on a **DE1-SoC FPGA** board. It simulates a "hotel-style" safe where a user can program a custom password, lock the safe, and subsequently unlock it only by entering the matching combination.

The system features a hardware-based "Hint" mechanism that provides real-time feedback on incorrect guesses, utilizing the FPGA's LEDs to indicate how close the attempt is to the stored password.

## Hardware & Architecture

### Finite State Machine (FSM) Design
The core logic is driven by a **4-State Moore Machine**, implemented in the `fsm_synth.sv` module. Being a Moore machine, the outputs depend solely on the current state, ensuring stable and synchronous operation.

* **OPENSTATE**: The safe is unlocked and ready to accept a new password.
* **OPENHOLD**: Intermediate state to register the "Set Password" command.
* **LOCKEDSTATE**: The safe is secured; inputs are treated as guesses.
* **LOCKEDHOLD**: Intermediate state to verify the guess against the stored password.

### Logic Implementation
The design utilizes a hybrid of **Combinational** and **Sequential** logic to manage data flow:

* **Sequential Logic (Registers)**: Used for the FSM state memory and to store the user's `PASSWORD` and current `ATTEMPT`. These are updated on the positive edge of the 50MHz clock (`CLK50`).
* **Combinational Logic**: Used for next-state logic calculations and output generation. This includes the comparator logic that checks if `ATTEMPT == PASSWORD`.

### `synth.sv` vs. `gates.sv`
The project explores two methods of FSM implementation:
* **Behavioral (`fsm_synth.sv`)**: Uses high-level SystemVerilog constructs like `enum` and `case` statements. This is the primary module used in the top-level design as it is more readable and allows synthesis tools to optimize the logic layout.
* **Structural (`fsm_gates.sv`)**: An alternative implementation using manual Boolean logic equations (e.g., `(Q1 & ~Q0) | ...`) to define state transitions at the gate level.

## User Interaction & Hardware Interface

The system utilizes the DE1-SoC peripherals for a tactile user interface:

| Component | Function | Description |
| :--- | :--- | :--- |
| **Switches (SW 0-9)** | **Input** | Used to set the 10-bit password and input guesses while locked. |
| **Button (KEY 1)** | **Enter** | Acts as the "Enter" key. It transitions the FSM from Open to Locked (setting the password) and checks guesses when Locked. |
| **Button (KEY 0)** | **Reset** | Hard reset that returns the system to the Open state and clears stored data. |
| **HEX Displays** | **Status** | Visual feedback displaying "OPEN" or "LOCKED" (via encoded hex words like `48'hFC...`). |
| **LEDs (LEDR 0-3)** | **Hint** | Displays the "Hamming Distance" between the guess and the password (see below). |

## The Hint System (XOR Logic)

A unique feature of this safe is the feedback mechanism designed to aid debugging or guessing. The system calculates the bitwise difference between the user's input (`SW`) and the stored `PASSWORD` using an **XOR** operation:

```systemverilog
diff = SW ^ PASSWORD; // 1 means the bits are different, 0 means they match