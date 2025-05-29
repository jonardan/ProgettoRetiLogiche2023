# Progetto di Reti Logiche: Hardware Component for Memory Interface

This project describes the VHDL implementation of a hardware component designed to interface with memory, receiving serial inputs to determine memory locations for data retrieval and specifying the output port for display. 

## Table of Contents

- [Introduction](#introduction)
- [Component Interface](#component-interface)
- [Expected Component Behavior](#expected-component-behavior)
- [Development Process and Design Choices](#development-process-and-design-choices)
- [Finite State Machine (FSM) Description](#finite-state-machine-fsm-description)
- [Testing](#testing)
- [Synthesis Results](#synthesis-results)
- [Team](#team)

## Introduction

The objective of this year's Logic Networks Project was to create a VHDL description of a hardware component. This component interfaces with a memory, receiving instructions from various serial inputs regarding the memory location from which to retrieve data and the specific output port to display it. 

## Component Interface

The `project_reti_logiche` component features five inputs: `clk`, `reset`, `w`, `start`, and `mem_data`.

* `w`: A 1-bit signal containing information about the output port (one of four available) and the memory address from which to fetch data.
* `start`: Goes high when `w` contains useful information.
* `clk`: Regulates the circuit on rising edges. 
* `reset`: Functions asynchronously to the clock.
* `mem_data`: An 8-bit parallel input that provides data from memory.

The component has seven outputs: `z0`, `z1`, `z2`, `z3`, `done`, `mem_en`, and `mem_addr`.
* `z0`, `z1`, `z2`, `z3`: These are the four 8-bit outputs where data from memory should appear when the `done` signal is asserted.
* `done`: An internally generated signal that is asserted when outputs are activated.
* `mem_en`: A signal to be asserted to enable communication with memory.
* `mem_addr`: A 16-bit signal indicating the memory address to be read. 

An additional output, `mem_we`, is ignored and constantly set to zero, as it is used to enable memory writing, an action not required by the specification.

## Expected Component Behavior

In the reset state, the system must show 0 on both outputs and `done`.  Once `start` is set to '1', data read from the serial input `w` on rising clock edges is organized as follows: the first 2 bits indicate the output port (00 -> z0, 01 -> z1, 10 -> z2, 11 -> z3), and the subsequent bits (0 to 16) indicate the least significant part of the 16-bit memory address from which to retrieve data. Any more significant bits not specified by the serial input are set to 0.

The `start` signal is guaranteed to remain high for a minimum of 2 and a maximum of 18 clock cycles, and low for at least 21 clock cycles.  This allows 20 cycles for the circuit to query memory and await a response (guaranteed within less than 1 clock cycle), and 1 cycle to display the received data on the correct port simultaneously with the `done` signal.  If data is ready before 20 cycles, the `done` signal can be asserted early, but must remain high for only one clock cycle. A reset is guaranteed before `start` rises for the first time, but not for subsequent operations; however, receiving a reset at any point requires re-initialization of the entire module. 

When `done` is set to 0, all outputs must be 0. When `done` is set to 1, the chosen port must be overwritten with the memory output value for a single clock cycle, while other ports display their last observed output.

## Development Process and Design Choices

The development involved three distinct design iterations. Initially, the team used multiple interconnected modules to parallelize the serial input signal, read memory, and direct messages to the correct output port. Ultimately, a more streamlined solution was adopted, implementing a Mealy machine that performs all operations within a single module. A behavioral design strategy was chosen, focusing on high-level component functionality and simplifying the debugging process. 

The implemented Mealy machine operates sequentially. The first two clock cycles are dedicated to reading the correct output port address. Subsequently, the machine proceeds to read the memory address from which to retrieve data.  During this process, the `o_mem_en` signal is constantly set to '1', meaning the memory will read even unnecessary cells. However, the message is only directed to the requested port on the rising edge of the clock after the `i_start` signal is detected as 0, which occurs when the correct address reading is complete. Simultaneously, the `o_done` signal is set to '1' for a single clock cycle and then returns to 0 along with all other output ports.

## Finite State Machine (FSM) Description

The module implements a Mealy machine consisting of four states: A (reset), B, C, and D.

* **Transition from A (reset) to B, and B to C**: The FSM assigns the bits of the output port address to the `out_port` vector using concatenation. During the B to C transition, the `o_mem_en` port is also set to '1' to enable memory. This design choice allows for the correct message to be output in a single clock cycle.
* **Self-loop in state C**: The address of the memory cell to be read (initialized to 0 during reset) is decoded using concatenation.  Since `o_mem_en` is already '1', the memory outputs the message at the intermediate address during each cycle in state C.  However, as the address might not be complete in state C, the message is only reported to the correct port during the transition from state D back to the reset state.
* **Transition from C to D**: The machine transitions from state C to state D only when '0' is read on the `w` input port, indicating that the useful input has been fully read. This ensures the correct memory address is written to the address vector. 
* **Transition from D to reset**: The `i_done` signal is set to '1', signaling the completion of the memory read operation. The message read from memory is assigned to the correct port for a single clock cycle, after which all outputs are reset in the subsequent transition.  The correct output port was determined in the first two transitions by reading the 2-bit address. 

## Testing

Extensive testing was conducted to ensure the component's correct operation.  The eight provided testbenches were used to verify the Mealy machine's behavior in various scenarios, confirming that the `o_done` signal and port outputs behave as specified. Due to the chosen architecture, the `o_done` signal and output messages are reported on the ports after only one clock cycle from the detection of the `i_start` signal as 0. 
### Limit Case Testing

In addition to the provided testbenches, a custom simulation was written to test the component in extreme situations, such as when only the two bits of the output port address are detected, or when a 16-bit memory address is set. These tests successfully confirmed the component's ability to function correctly and reliably in all cases. 
## Synthesis Results

The developed component underwent both Behavioral and Post-Synthesis Functional simulations, successfully passing all testbenches in both cases.  As previously noted, the output is correctly sent to the specified port after one clock cycle from the moment the `i_start` signal is detected as 0. Further reducing this timing is not possible, as the signal read from memory cannot be instantaneously propagated to the output on the rising edge where `i_start` is seen as 0. 

The Design Timing Summary shows a positive Worst Negative Slack (WNS), indicating that the synthesized circuit meets the time constraints with a certain margin. A positive WNS also makes the circuit more robust in suboptimal operating conditions. 

**Design Timing Summary**

| Metric                  | Setup      | Hold       | Pulse Width |
| :---------------------- | :--------- | :--------- | :---------- |
| Worst Negative Slack (WNS) | 97.513 ns  |            |             |
| Worst Hold Slack (WHS)  |            | 0.139 ns   |             |
| Worst Pulse Width Slack (WPWS) |            |            | 49.500 ns   |
| Total Negative Slack (TNS) | 0.000 ns   |            |             |
| Total Hold Slack (THS)  |            | 0.000 ns   |             |
| Total Pulse Width Negative Slack (TPWS) |            |            | 0.000 ns    |
| Number of Failing Endpoints | 0          | 0          | 0           |
| Total Number of Endpoints | 197        | 197        | 102         |

All user-specified timing constraints are met. 

## Team

* Andrea Torti (10730470)
* Jonatan Sciaky (10741047) 
