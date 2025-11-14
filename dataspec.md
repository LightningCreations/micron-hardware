
# Micron CPU Datasheet

## Pinout

* 5VCC (I): 5V Logic Supply Input
* GND: Reference Ground for 5VCC
* A0-23 (O): Low Order 20 bits of Address Bus
* A24-26/W0-2 (O): For Primary Bus Transfers, next 3 bits of Address Bus. For I/O transfers, lower 3 bits of the transfer width. For Coprocessor Transfers, reserved.
* A27-29/W3-4/N0-1 (O): For Primary Bus Transfers, top 2 bits of Address Bus. For I/O transfers, top 2 bits of transfer width. For Coprocessor Transfers, target coprocessor number.
* BE (IO): Bus Enabled
* BA (I): Bus Acknowledge
* W (O): Bus Transfer Write Control
* IO (O): Primary/Secondary Bus Control
* SC (O): Bus Designation
* D0-31 (IO): Data Bus
* RESET (I): Reset Line
* EX0-2 (I): External Exception Trigger
* IRQ0-5 (I): Interrupt Request Trigger
* S0-1 (O): CPU Status
* CLK (I): Internal Clock
* COE0-3 (O): Coprocessor Enabled
* COP0-3 (I): Coprocessor Present
* CXE (O): Coprocessor Execute Enable
* LOCK (IO): Bus Lock

91 Total Pins

## Bus Transfer

(Note: In Table below, BE=1 refers to when the CPU sets it. Other devices setting BE prevents the CPU from performming bus transfers until BE falls low)

| BE | W | IO | SC | Address | Width | Notes                |
|----|---|----|----|---------|-------|----------------------|
| 0  | x | x  | x  | N/A     | N/A   | No transfer occuring |
| 1  | 0 | 0  | 0  | A0-29*4 | 32    | Main Memory Read     |
| 1  | 1 | 0  | 0  | A0-29*4 | 32    | Main Memory Write    |
| 1  | 0 | 0  | 1  | A0-29*4 | 32    | Memory Read (Instruction Fetch) |
| 1  | 1 | 0  | 1  | A0-29*4 | 32    | Not produced/Treat as Write |
| 1  | 0 | 1  | 0  | A0-19   | W0-4  | I/O Port Read |
| 1  | 1 | 1  | 0  | A0-19   | W0-4  | I/O Port Write |
| 1  | 0 | 1  | 1  | N0-1:A0-4* | 32  | Coprocessor Register Read |
| 1  | 1 | 1  | 1  | N0-1:A0-5* | 32  | Coprocessor Register Write |

The lower Width bits of the Data bus contain the salient data, other bits are Connected to a Pull Down Resistor.

BE is an IO pin that controls whether or not the Bus is in use. When not set to 1 by the CPU, it is connected to a pull down resistor. 

When BE is not set by the CPU, all other bits are unconnected, except for Data, which is connected to a pull down resistor.

The BA signal is used to indicate when read operations are completed. 
Write operations are expected to be recieved no later than the 4th cycle since BE became High.

### Bus Transfer Protocol

The following timing protocol is used for data transfer:
* Cycle 0(RE): CPU sets BE=1,
* Cycle 0(FE): Address lines, Data lines (for writes), W, IO, and SC become available
* Cycle 1(FE): Data and Address, W, IO, and SC lines latched by downstream,
* Cycle 2(RE): Data, IO, and SC lines become unavailable (may not contain salient values),
* Cycle N(RE) (W=0): Downstream sets BA=1
* Cycle N(FE) (W=0): CPU latches Data from downstream.
* Cycle N(FE): Earliest Cycle for CPU to set BE=0.
* Cycle N+1(RE) (W=0, IO=0): CPU May set W=1 to perform a succesive write. If so, downstream releases the Data lines.
* Cycle N+1(FE) (initial W=0, IO=0, W=1): Data lines for succesive write are made available.
* Cycle N+2(FE) (init W=0, IO=0, W=1): New Data lines latched by downstream
* Cycle N+k(FE): CPU unsets BE=1,  

(N is the number of cycles before downstream sets BA in the case of a read, and is 3 for a write, k is the number of cycles after the earliest point the CPU unsets BE=1)

If, during an initial Memory Read, the CPU performs an immediate write with the same Bus Enable, k is a minimum of 2. Otherwise k is a minimum of 0.

### Coprocessor Transfer Address

Coprocessor registers are identified by A0-4. The Coprocessor to access is identified by N0-1. 

As a special case for Writes, An address of 100000 (A5=1, A0-4=0) accesses the 6 Coprocessor Control Bits (reachable from register sys1/copctl). The top 26 bits of the Data Bus are 0. 
A5 is not used for reads, and any other values of A0-4 with A5=1 are invalid.

### LOCK and External BE

The BE pin is IO to allow multiple devices (including coprocessors and other Micron CPUs) to share the same bus for basic operations without complicated coherency logic. While BE is brought high externally, the Micron CPU will not attempt to perform any bus accesses or coprocessor instruction dispatches. Instead, the CPU will wait on them until BE becomes low.

The LOCK pin is a specialized pin for supporting atomic memory accesses. When LOCK is high, the Micron CPU will not attempt to perform any memory accesses, other than instruction fetches. The LOCK pin does not control I/O port accesses or coprocessor register accesses. It also does not block instruction fetches. LOCK is an output pin because it may be used as one in a future chip. Current Micron CPUs do not generate LOCK signals.

### Primitive Read-Modify-Writes

When reading main memory (IO=0, W=0), after BA becomes High, the CPU may set W=1 at the immediately following rising edge (Cycle N+1). If so, it places data to write on the bus at the falling edge of that cycle. 

## Co-processor Instruction Execution

When CXE=1, The Identified coprocessor is expected to fetch an instruction from the data bus and execute it. 
D0-23 contains the payload, D24-31 contains the function, and N0-1 contains the Co-processor Number.

CXE=1 is exclusive with BE=1, but is not exclusive with LOCK=1. BE will be low when the CPU is performing a Co-processor Instruction Execution,  and the CPU will not attempt a bus operation until at least the rising edge of the cycle following CXE=1 becoming low. 

To simplify intermediate processing, the W, IO, and SC bits will all be 1 at the same time as CXE=1. 
Thus intermediate bus controllers and logic need only pass CXE=1 through to Coprocessors, and can OR it together with BE for internal logic.

Note that CXE is NOT an IO pin, unlike BE. Thus, if multiple micron CPUs are sharing the same bus without appropiate intermediate arbitration, a CXE=1 indication from any of the Micron CPUs must be passed through as an external BE=1 signal to each other CPU.

Co-processor Instruction Execution is performed in the same manner as a write operation.

## Exceptions and IRQs

External Exception Trigger:

| EX | Vector | Notes |
|----|--------|-------|
| 0  | -      | No Exception |
| 1  | EX[1]  | For external MMU to raise Bus Error |
| 2  | EX[15] | Non-maskable Interrupt Request |
| 3  | -      | Reserved |
| 4-7| EX[4-7]| Co-processor 0-3 Error |

Interrupt Request:

* 0: No IRQ 
* 1-15: Reserved
* 16-63: Trigger Vector number 16-63

An Exception or IRQ request occurs by placing a non-zero value on the bus before the rising edge of a clock pulse.

## Status

| S | Status | Notes |
|---|--------|-------|
| 0 | Normal | Running |
| 1 | Waiting | Waiting on Co-processor |
| 2 | Halted | Stopped by Halt Instruction (Waiting for Interrupt) |
| 3 | Stopped | Stopped by STOP instruction (Waiting for Reset) |

## Reset

Bring RESET High for Minimum of 2 Cycles to trigger a reset. S0-1=11 will indicate when the CPU is no longer busy, RESET brought low any time after this indication will trigger CPU Start up.