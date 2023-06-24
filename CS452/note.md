instructor: Martin Karsten 

MC4063 TTh 1-2:20

https://student.cs.uwaterloo.ca/~cs452/W23/

<details>
<summary>ex</summary>
4 1 32 5
</details>

- [Week 1. Jan 10](#week-1-jan-10)
- [Week 2. Jan 17](#week-2-jan-17)
- [Week 3. Jan 24](#week-3-jan-24)
- [Week 4. Jan 31](#week-4-jan-31)
- [Week 5. Feb 7](#week-5-feb-7)
- [Week 6. Feb 14](#week-6-feb-14)
- [Week 9. Mar 7](#week-9-mar-7)
- [Week 11. Mar 21](#week-11-mar-21)


# Week 1. Jan 10

```
     +-----+                                linux.student.cs
     |train|                      network     +-------+   network
     |track|                   +--------------+ TFTP  +---------+
     +-----+                   |              +-------+         |
                               |                                |
                               |                                |
+----+------+-----+         +--+---+                        +---+--+
|PWR | 6021 | 6051+---------+ ARM  +---------+              | DEV  |
+----+------+-----+  COM1   +------+  COM2   |              +------+
      Marklin               grey box         |
                                             |
                                             |
                                             |
                                             | /dev/ttySx
                                          +--+--+
                                          | OPS |
                                          +-----+
                                          track PC
```

raspberry runs without MLU, unaligned access to memory will not work.

memory layout of running program
```
-------- 0xffff
 stack v
--------
 heap  ^
--------
 data
--------
 txt
--------
-------- 0
```

elf sections
* text
* data: initialized data in the code
* bss: zero-inited data

_freestanding_ environment (in contrast to hosted)
* has no OS under the hood
* no heap memory
* no libc; it is part of compiler

memory setup:
* physical memory
* device registers: 'magic' addresses to communicate with devices
  * if we write bytes to the magic addresses, they are not written to memory but to devices.
  * BCM section 1.2.4
  * in 'low peripheral mode', please replace `0x7e000000` with `0xfe000000`

system timer (ch10) (vs arm timer):
* has frequency 54Mhz (arm time has dynamic freq)
* 7 control registers starting from `0x3000`
* registers have Clo, Chi
  * only compare low part because time is 32bit wide

big polling loop (a0):
```c
for (;;) {
    if (c1) a1();
    if (c2) a2();
}
```

serial communication
* pi board has GPIO
* serial hat connected to pins 18-21
* we program the pins to connect to SPI master etc
* delay initialization to runtime; flexible
* ch5 and page78
```
                GPIO
                +--+         +--+
                |  +-------> +--+ SPI master
                |  |
+-----+         |  |         +--+
|ser. +------>  | 18+------->---+
|hat  |         | ..
+-----+------>  | 21 +------>+--+
                |  |         +--+
                |  |
                |  +-------> +--+
                +--+         +--+
```

SP1 (serial peripheral interface) (section 2.3 and ch9)
* synchronous interface

```
                31             0
                +--+---+---+---+
                |  | 1 | 2 | 3 |  +----> send end
                +^-+---+---+---+
               count
               bits

                +--+---+---+---+
recv end  +---> |  | 1 | 2 | 3 |
                +--+---+---+---+
                         1   2
                             1
```

UART:
* a characteristic device (within arm): 1 byte at a time
* asynchronous
* FIFO buffer
  * do not worry about performance implications for now
* has register
  * TX/RX level regs (whether can send/receive data)
  * TX/RX hold regs
* not BCM, but SCI6 on top of pi
* SCI6 section 8, addressing table 34

UART configuration:
* start and stop bits
* parity bit (a redundant bit for detecting error)
* channel 1: 115200 baud (impulses/s)
  * communicate with terminal
  * 8 bits per byte, no start bit, no parity, 1 stop bit
* channel 2: 2400 baud
  * with trains
  * 8 bits per byte, no parity, no start bit, 2 stop bits
  * so 2400/10 = 240 bytes per second

train and track commands:
* speed: 2 bytes
  * speed: 0-14; 15 for reversing; plus 16 for lights
  * train number
* turnouts: 2 bytes
  * 0x21 straight; 0x22 curved
  * train number
  * please use 0x20 to wait
* sensor:
  * reset 0xc0
  * half-duplex

__eg.__ this code is bad; do not use other than SPI code
```c
for (;;) {
    while (!c1) a1();
    while (!c2) a2();
}
```

# Week 2. Jan 17
problems:
* no SIMD instruction
* `-q0 -mgeneral-regs-only`
* ldur/stur create unaligned accesses and crash; use `-mstrict-align` (sometimes no working with c++)

bounties:
* fix toolchain
* give compact examples of the problem
* get redboot to work

## multitasking

what is problem in the polling loop?
```c
for (;;) {
    if (c1) a1();
    if (c2) a2();
}
```
even if only `c1` is active, we still need to check everything.

improvement?
```c
for (;;) {
    if (c1) a1();
    if (c2) a2();
    if (c1) a1();  // c1 is important, let's check again
    ...
}
```
this does not solve the problem.

what if we split big task:
```c
for (;;) {
    if (c1) a1();
    if (c2) {a2_1(); a2_half_done = 1;}
    if (c1) a1();
    if (a2_half_done) {a2_2();}
}
```
this is too complex, not reusable.

concurrency:
* manage independent activities.
* separation of concerns
* runtime management
* multitasking (kernel + tasks) is one way to deal with concurrency

key concerns of task: use stack; because need to handle recursion (unknown number of push/pops).

canonical approach of kernel code:
```c
void kmain() {
  initialize();  // includes starting the first user task
  for (;;) {
    currtask = schedule();
    request = activate(currtask);
    handle(request);
  }
}
```

kernel:
* create and manage tasks
* hardware protection mechanisms
  * uses kernel mode
    * kernel mode is usually configured to be non-interruptable
    * in user mode we want interrupts
  * has memory protection
* task communication and synchronization are exclusively done by message passing

why not shared memory for communication? it uses locks that are for OS.

entry/exit kernel-user-level tasks
* software interrupts (via syscalls)
* hardware interrupts

task stack has:
* scheduling state: ready/blocked/active
  * blocked: might be waiting for IO
  * need a queue for scheduling and blocking
    * priority queue + round robin for equal priority
* execution state: which line of code/what are values for variables
* task descriptor

memory:
* no shared memory: no global vars, static symbol in theory
  * it is ok to use static gv for singleton (const)
* primarily on stack
* no heap
* might still need some dynamic mem allocation
  * _slab allocation_: slabs of memory + free list
  * free list is still a linked list => use _intrusive linkage_: embed next pointer within the data structure

software development principle: KISS
* knuth: no premature optimization

## context switch

ARMv8:
* execution state: 32bit (historical) or 64bit
* execution level:
  * 0: user program
  * 1: kernel
  * 2: hypervisor (default boot mode)
  * 3: secure

register file:
* `x0..x30`: general purpose; 32bit
* `x31 = xzr`: zero
* `r15 = rpc`: program counter
* `r13 = rsp`: tack pointer register bank (4 exec levels => 4 regs)
* pstate: processor state
  * exec state
  * whether interrupts are taking place
  * condition codes N,C,Z,V
* many system control register banks (section 4.3)
  * MRS/read
  * MSR/write
  * SPSR (saved processor state; last pstate after doing exec level switch)
  * ELR (link register)
  * ESR (status of system)
  * VBAR

ARM instructions:
* 32bit RISC instructions
* SIMD instructions (we do not use)
* unaligned memory (we do not use)

what is context switch? changing stack (pointer), register file and pc.

questions:
* do we use same stack?
* is it synchronous? (switch on our own or something happens)
* is it symmetric? (same procedure when switching from A to B as B to A?) typically not
* subroutine call: same stack, synchronous, asymmetric
* coroutine: different stacks, synchronous, symmetric
* multitasking: plus a scheduler

kernel design: 01N
* 1: use single kernel stack with state (recommended)
* 0: no stack, 'subroutine' have global/static state (ugly)
* N: one stack per user task - multithread kernel (complicated)

application binary interface (ABI)
* registers: structure and hardware roles
  * `x0-x7` for procedure arguments
  * `x8`: struct return values (caller provides memory)
  * `x9-x15`: local register
  * `x16-x17`: reserved for linker
  * `x18`: platform, TLS
  * `x19-x28`: global registers
  * `x29`: frame pointer
  * `x30`: link register (ret address)
* rules:
  * `x0-x18` are caller-saved (callee-owned)
  * `x19-...` are callee-saved
  * pstate, condflags are undefined
  * stack pointer needs to be aligned to multiple of 16

subroutine
* `b dst` jump
* `bl dst` jump & link
* `ret` pc:=lr

_stack switch_: a simple form of context switch
```c
// abi rules apply (x0-x18 already saved)
void StackSwitch(char* newSP, char** oldSP);
```
1. save `x19-x30` (push to stack)
2. save SP to to memory location `*x1`
3. load SP from `x0`
4. restore `x19-x30` (pop)
5. return

mode switch:
* SP is banked; SPSd = 1
* system call: synchronous
  * `svc N => handler();` (N from ESR)
  * ABI rules apply
  * asymmetric
* interrupt
  * ABI rules do not apply
  * pstate is relevant

how to implement `handler()` for syscall:
* exception vector (sec 10.4)
  * VBAR_EL1: 4 groups of 4 vectors, each vector is 128 bytes
    * (initialize to something that crash, and only work on needed)
    * groups: at 64 bit mode
    * vectors: synchronous, IRQ
    * if handler is short enough, can directly write to entry

# Week 3. Jan 24
useful registers in SVC mode switch:
* ELR_EL1 stores PC
* ESR_EL1 stores the N in `svc`
* SPSR_EL1 stores process state
* PC is the first address of exception handler
* SPSel bit: 0/1; 0 means sp is EL0; 1 means sp is EL1 banks
  * can use SP_EL0/1/... to get sp directly

__eg.__ how to mix C code and assembler?

main.c:
```c
extern int adder(int, int);

int main(void) {...}
```

adder.c:
```arm
.text
.align 2  // align everything to 2^2=4 bytes (instr size)
          // or .balign 4
.global	adder
.type adder, %function
adder:
  add x2, x1, x0
  mov x0, x2
  bx lr  // or ret
```

__eg.__ combine assembler with C code.
```c
/* https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html */
unsigned int val;
// read via asm
asm volatile("mrs %x0, cpsr" : "=r"(val));
printf("cpsr: 0x%x\n\r", val);
// write via asm
asm volatile("msr cpsr, %x0" :: "r"(val));
```

pi initialization:
* power, reset
  * all memory reset
  * hardware/devices reset

kernel start:
* EL1
* stack
* IRQ disabled (more on this later)
* K1: set up syscalls
* later: set up IRQs, timer, uart, reset the track
* create initial task, push to ready queue, and start main loop

task exit: what if user program forgets to call `Exit()`? wrap the user function:
```c
void task_start(void (*task_function)()) {
    task_function();
    Exit();
}
```

scheduling:
* initial preparation and time slicing
* after kernel entry and corresponding processing: ready or block

## message passing

|sender|receiver||
|:-:|:-:|-:|
|SEND_WAIT|RECEIVE_WAIT|
||mailbox: queue of senders|when ready, sender put itself to queue
||rendezvous|copy message
|REPLY_WAIT||<- decouple in time/space
|2nd rendezvous||copy response

message:
* byte memory; use union/type casting for type safety

synchronization:
* no locks, cvs, semaphores required
  * because there is no shared memory
* what about devices: use one task for device

we can build an async producer/consumer pattern on top of the sync message passing mechanism.

single-producer-single-consumer
* two options:
  * producer sends data => push
  * consumer sends request => pull
  * sender needs to block

multiple-producer-single-consumer
* push communication is good
* pull communication: needs to wait for multiple producers, which one to pull from?

single-producer-multiple-consumer
* pull communication is good
* push communication is not good

multiple-producer-multiple-consumer
* add a buffer task
  * MPSC as the single consumer
  * SPMC as the single producer

server pattern:
* task provides service
* server never calls `Send()` => server is either busy computing or available on request (receive)
* it is a loop:
  1. receive request
  2. process request
  3. (delayed) response

name resolution:
* always has implicit/default starting point (_closure mechanism_)
* have name string => tid integer
* resolution then communication

# Week 4. Jan 31

compiler optimization:
* instruction reordering
* loop unrolling
* constant folding
* remove code without side effects
* register caching
* eliminate repeated reads/writes
* compiler understands data dependencies
  * but only single-threaded

C/C++ volatile:
```c
int addr = 0xfe000000 + 0x3000 + 0x04;
volatile unsigned *d = (...)addr;
unsigned val = *d;
```

timing/measurements:

__eg.__ what is problem with this measurement?
```c
start = clock();
...
stop = clock();
duration = stop - start;
```
* duration of `clock()` call counts
* background noise
* inaccuracy and granularity

improvement: do many operations and measure total time, then take average. this has problems too
* compiler optimization may comes in the way: use `volatile`
* loop execution (counter, branch)
* N needs to be big

periodic action with period X:

__eg.__ what is problem with this?
```c
for (;;) {
    sleep(X);
    action();
}
```
* sleep may drift and accumulate errors
* action itself takes time => actually not sleeping for X

an improvement:
```c
for (;;) {
    target += X;  // never accumulates error
    sleep(target - clock());
    action();
}
```

CPU caches:
* L1data: 128kb
* L1intruction: 192kb
* L2 unified: 2MB
* branch predictors
* TLB
* write buffer: mediate fast cpu and slow memory

cache modes (section b2.7)
* non cachable: for devices
* write-through
* write-back
* don't have to mess with these

cache setup:
* SCTLR_EL1: controls both EL0 and EL1

cache maintainance:
* CTR_ELO: cache information
* CSSIDR_EL1 | CSSELR_EL1: cache sizes
* clean cache
* invalidate cache
* disable -> disable: noop
* enable -> disable: clean
* enable -> enable: clean
* enable -> disable -> enable: invalidate

## interrupts

interrupts vs polling
* kernel wakes user tasks
* asynchronous => high overhead (switching contexts)
* best of both worlds: interrupt mitigation (NAPI)

interrupt handling/scheduling:
* return-to-task
  * eg: record clock ticks and return to task
  * may be more complex: time slicing
* preemptive
  * always reschedule
  * full context switch

stack-switch:
* pure stack switch => save caller-saved registers (x10-x30)
* pure irq switch => save x0-x18

interrupt hardware
* `device -- interrupt controller -- CPU`
* enabling/disabling cpu interrupt flags => ignore them
* GIC-400 is GICv2 (vs arm legacy controller)
* base address: `0xff840000`
* GIC reference ch 4
* bootloader disables interrupts via DAIF set (section 652,10.1)
* GIC default: all interrupts disabled
  * need to set up GIC
    * GICC control, GICD distribution
  * set up pstate usev

edge interrupt vs level interrupt
* clock tick creates level interrupt to interrupt handler. interrupt handler gives it to cpu. the cpu needs to respond

GIC registers
* GICD-ISENABLER => enable (bitmask)
* GICD-ITAREGISTER => reroute to core 0
* GICC-IAR => retrieve interrupt IRQ#
* GICC-EOIR => confirm to GIC that interrupt is processed

how to get interrupt number (IRQ#):
* timers: (video core) VC 0-3 (broadcom 6.2.4)
* VC interrupt => plus 96

handler routine:
* set up exception vector
* EL1, registers disabled
* on EL1 stack
* start context switch

# Week 5. Feb 7
timer interrupt:
* compare registers: C0 - C3
  * C0 and C2 are for video core, so use C1 or C3
* confirm: use CS register
* the timer device is actually level signal, because without CS register, we get interrupt immediately (we would not need CS if it were CS).

AwaitEvent():
* matchmaking, similar to SSR
* what if event happens but the task that should be waiting on it is not available => system is overloaded
* what if N tasks waiting for same interrupt => perhaps do not do that
* kernel delivers one task to notifier and notifier asserts

interrupt handling:
* user task executing -> interrupt -> handler -> kernel determines interrupt -> process & confirm -> push to ready queue
* after confirm, we can have a loop to handle another interrupt if we care about interrupts have same priority and happen at same time
  * not caring would also work => more context switches

clock server:
```c
// same standard server loop
for (;;) {
    req = Receive();
    process(req);
    if (...) {
        Reply(req);
    }
}
```
the blocking on AwaitEvent is delegated to the clock notifier:
```c
for (;;) {
    AwaitEvent();
    Send(clock_server);
}
```

idle:
* no task is ready to execute
* should see ~90% of time to be idling
* can do
  * diagnosis (compute time)
  * maintenance (scrub task stacks)
  * save -> call `wfi` to go to low power mode (`wfe` also includes interprocessor interrupts) -> device interrupt happens
    * there may be spurious wake-ups => need to check clock tick
    * waking up from this mode may be slower than busy wait

## software design
problem complexity vs. solution complexity

objectives:
* complexity
  * modularization
  * smaller tasks
* correctness
  * deadlock avoidance
    * avoid circular send
    * avoid circular receive
    * do not mix send and receive in same task
* performance/efficiency/throughput
* timeliness/latency
  * scheduling priorities
  * low utilization

design patterns:
* task types: client, server, worker
* worker types:
  * notifier
    ```c
    for (;;) {
        AwaitEvent();
        Send();
    }
    ```
  * courier
    ```c
    for (;;) {
        Send(server1, req, msg);  // pull from server 1
        Send(server2, msg);       // receive from server 2
    }
    ```

debugging:
* program model <=> implementation
* environment (HW/SW): interface <=> internals

||model|implementation|
|:-:|:-:|:-:|
|program|design/logic|code|
|environment|interface & docs|internals

debugging strategy
* logging (print)
* declaring (assert)
* unit testing
* inspection
* interactive debugging: not easily accessible right now
* no experiment can prove something
* build model: gather data, compare model and data, repeat

timing problems
* realtime: timing <=> correctness
* non-determinism/concurrency: timing => functionality

race condition/data races:
* concurrent access to resource
* potential problem if any one access is a mutation
  * leads to multiple outcomes
* real problem if a least one outcome is incorrect
  * coordination

# Week 6. Feb 14
### uart
```
cpu --- spi --- uart (tx/rx) --- marklin
    gpio
```

sources of uart interrupts:
* transmit TX fifo is not full
* receive RX fifo is not empty
* receive timeout RX fifo not empty below trigger level for some time (4 byte times hardcoded)
* trigger levels => level signal
* CTS -> flow control
  * this is edge signal

uart IRQ control:
* interrupt is controlled by IER register (section 8)
* cpu only gets one interrupt via gpio
* read IIR register to query and confirm

uart fifo:
* hardware FIFO (64 bytes) reduces interrupt rates (with interrupt mitigation) and absorbs small bursts of data
* enable/disable both RX/TX in FCR[0] 
* triggers FCR
  * EFR[4] => MCR[2] => TLR

operation:
* naive: interrupt -> action
* better: interrupt -> check status -> action -> repeat
* best: check status -> action or wait for interrupt -> repeat 

__eg.__ when writing to TX fifo:
1. write TX
2. check TXLVL
3. if not ready, enable TX interrupt, otherwise go back to step 1
4. [interrupt]
5. disable TX interrupt
6. go to step 1

task structure:
* uart server(s)
* notifier to handle interrupts/events
* read/write using syscalls

SPI:
* need atomicity of 2-byte UART operations
* implement in SPI interactions in kernel and expose via UART-specific system calls

GPIO:
* gpio banks: 0-27, 28-45, 46-57
* gpio pin for interrupts: 24
* gpio interrupt for VC is 49
* add 96 = 145
* set up for pin 24:
  * set function GPIO_INPUT, resistor GPIO_NONE
  * add interrupt by setting Bit 24 in GPLEN0

flow control:
* not overwhelming the marklin box
* UART's 'TX ready' status only means the communication channel is ready.
* Clear-To-Send (CTS) signalled on separate wire when the receiver is able to receive and process the next byte
* before sending, check TX up and CTS actually transitions from low to up

## train
train velocity differs at each level of speed, and differs in each environment => cannot predict state of the track. have to measure the speed.

data representation of velocity:
* velocity = distance / time
* time: 32 bits at 1ns =>. ~71 minutes
* 10ms ticks
* longest shortest path: 10500 mm
* no floating point
  * use fixed-point representation with scale by power of 2 (shift is faster)
  * avoid integer rounding `(a/(b/c)) = a*c/b = c*(a/b)`

server types:
* proprietor
  * control exclusive access to a resource
  * node internal buffering, serial execution
  * sync or async execution
  * example: track server -> uart server
    ```c
    for (;;) {
        Receive();
        syncProcess();
        Reply();
        asyncProcess();
    }
    ```
* administrator
  * general server
  * must not send
  * buffer and park requests
  * use worker for communication
  * client communication

suggestion:
* build small independent control loops
  * localized location estimate vs. route-based location estimate 
* higher concurrency and lower complexity via smaller tasks?

# Week 9. Mar 7
## train modelling

experiments and data:
* determine minimum speed (per segment) to avoid getting stuck
* stop from lower speed is more accurate

measurement:
* use tight polling loop (no kernel)
* sensor -> timestamp
* measurement errors:
  * signal delivery (marklin processing) => constant error cancel out when subtracting timestamps
  * delivery to software (polling loop) => is variable, use statistical modelling
  * processing in software => small, ignore

uncertainty:
* assume actual sensor times are uniformly distributed across duration of polling loop
* time interval between two sensors in sum of uniform distributions
  * general case (N > 2): irwin-hall distribution
  * N = 2: triangular distribution
  * sample mean/variance are good estimates of real mean/variance
  * can also use min/max to estimate midpoint
* velocity (dist/time) is nonlinear
  * compute average time first, then speed; no other way around
* we will not see perfect triangular distribution, but bimodal (or trimodal in corner cases)
  * frequency too low (70m)
* recommendation: keep experiment raw data as much as possible

__eg.__ if we have dist=100, t1=10, t2=20: then tavg=15, so v=100/15=6.67.

dynamic calibration: use _exponentially weighted moving average_ (EWMA)
* formula: `c <- c * (1 - a) + d * a`
* c: current time estimate; d: next data sample; a: weighting factor
* no need to store array of samples
* with appropriate choice of a ($\alpha=\frac{1}{2^n}$), can use bit shift
* can use similar approximation for standard deviation

acceleration:
* velocity is finite <=> movement must be continuous
* acceleration is finite <=> velocity must be continuous
* simplified kinematic model: assume constant acceleration
  * approximate as average velocity during acceleration interval
  * or approximate as velocity step change in the middle of the interval
  * both ok if somewhat lower-quality location estimate is acceptable during acceleration
* measurement:
  * similar to velocity
  * measure time between two sensors
  * change speed level at 1st sensor
  * then compute acceleration based on known estimates for velocities
    * velocities are known by measuring avg time and constant dist
    * first detect whether train has reached target velocity at 2nd sensor?
    * avg velocity during acceleration interval is `v_avg = (v1 + v2) / 2`

__eg.__ acceleration completes before 2nd sensor hit (`d/t > v_avg`)
* split d=d1+d2 into two segments (d1: acceleration, d2: stable velocity)
* split t into corresponding segments
* `v_avg = d1 / t1`
* `v2 = d2 / t2`
* can solve for d1, d2, t1, t2
* acceleration is `(v2 - v1) / t1`

```
   v
   ^
v2 |   +---------+-
   |  /          |
va | /           |
   |/            |
v1 +-------------+--> t
       t1        t2
```

__eg.__ acceleration does not complete before 2nd sensor hit (`d/t < v_avg`)
* actual average velocity during acceleration is `v_s = d / t`
* velocity at 2nd sensor hit is `v_r = 2(v_s - v1) + v1`
* acceleration is `(v_r - v1) / t`

```
   v
vr ^   +
   |  /|
vs | / |
   |/  |
v1 +---+-----------> t
       t
```

measuring stop distance:
* send stop command when sensor is triggered
* measure stop distance manually

measuring stop time:
* compute using acceleration model
* manual binary search
* can use stop time + velocity to compute distance

start:
* special case of acceleration
* measured based on known estimates for stop time/distance
* stop at known location, then start
* measure time to sensor
* can compute acceleration

short moves:
* stop before reaching target velocity => dynamic calibration
* manual experiments with different (short) times between start and stop

### latency

latency analysis:
* program as graph => computation & termination vs. control/service loop
* control/service loop: throughput vs latency
  * timing of edges: busy vs block
  * throughput: compare avg busy cost of paths to offered load
  * latency: consider both busy and blocked cost

average latency: no balancing between lower and higher latencies
* service loop: outliers change mean => potentially skews utility
  * late response is useless
* control loop: mean masks outliers => potentially hides catastrophic problem
  * how often a train enters critical track section matters more than how much it overshoots
* analysis must work with latency distribution
  * service loop: higher order percentile
  * control loop: worst case latency
* rarely relevant


worst-case latency: WCET
* identify critical paths or worst-case execution path (WCEP)
* uncertainty:
  * busy (memory/cpu pipelining/caching)
  * block (contributes to latency but not throughput)
  * internal loops (bounds)
* queueing:
  * standing queues (contributes to latency but not throughput)
  * theory: latency increases as arrivals approach capacity
* alternative model: _critical instant_
  * assume all events happen at same time; all tasks ready
  * trace priority execution

automatic analysis:
* static analysis: np-hard
* hybrid: measure single feasible paths (SFP) and analyze combination
* record loop counts, recursion depths

train control:
* preemptive scheduling: shortcut to loop start
* small tasks: shorter graph edges
* critical path is sensor event -> action -> effect
* preemption: keep uninterruptible kernel execution short
* priorities: support resource management via scheduling

task priorities:
* critical path is sensor -> uart -> supervisor -> engineer -> track -> uart -> command
* priority determines worst-case latency
* priority is not as critical when utilization is low
* low-level tasks: use higher priority and/or buffer
  * low priority could lose interrupts or input

options for train control command loop:
* single-bank sensor queries - budget for train commands
* add uncertainty to sensor timing based on # of previous train commands
* ignore additional error, because it is small?

# Week 11. Mar 21
TC:
* add graceful exit command that stops everything
* light up trains
* start all trains from exit?

context switch stress testing:
* tasks switch to each other
* task do expensive computations involving all registers

### train control
scenarios:
* travel at constant speed => calibrate velocities
* travel at target velocity (avoid collision) => adjust speed
* travel in cohort => adjust speed but there are two trains to choose from

sensor attributions => error margin for prediction
* expected time T ± error margin
* handle one error: either turnout or sensor
* add redundancy by listening to multiple sensors

what if unexpected errors (timeouts) occur:
* good strategies: stop trains and print errors
* reality: run as much as possible?
* build independent subsystems

deadlock: wait for random time an restart?

path-finding:
* online
  * compute path using track/reserved node
  * single-node dijkstra doable online (computing part + tracing part)
* offline: determine loop-free alternative path with loop threshold

### design approach
decomposing problems:
* travel server:
  * manage travel plans with timing
  * query for sensor attribution
* sensor server
  * sends notification to train
* route server
* reservation server
  * reservations must be continuous
  * do not worry about timing
* train task
  * source/dest: obtain route from route server
  * file travel plan
  * drive train
    * reserve/release track segments
    * set train speeds
    * switch reserved turnouts
    * handle sensor and timeouts

### priority/interrupts
interrupts:
* want to avoid interrupts
  * but always want clock/timer interrupts
  * uart tx: always try sending first, enable interrupt if cannot
    * can set ui things to have lower priority
  * uart rx: response needed
    * principle: should finish requests
* want to avoid buffering in kernel

priority inversion
* lower priority task indirectly keeps higher priority task from running
* can become latency problem
* can become correctness problem is system is fully loaded
* use dynamic priority

# Week 12. Mar 28
## real-time cpu scheduling
special-purpose vs general-purpose systems:
* special-purpose: hardware, os, application co-design
  * control/VR
  * predictability
* general-purpose: hardware, os, application decoupled
  * performance, throughput and liveness

types of realtime:
* 'hard' RT: 'no delay', no deadline misses
* 'firm' RT: rare deadline misses
* 'near' RT: some variation and misses are ok
* 'soft' RT: statistical distribution of delay is ok, some degradation is ok

efficiency vs latency tradeoff:
* also modeled as pipeline (buffer/queue) vs. run-to-completion

_cyclic executive_ (polling loop):
* target should be larger than max iteration time if all flags are true to prevent overheat
* limited monolith
```cpp
for (;;) {
    t = current_time();
    if (1) ...
    ...
    sleep(t + target - current_time());
}
```

real-time task scheduling:
* task destruction: periodic vs aperiodic
* admission control: set of feasible
* selection mechanism: scheduler

liu/layland (1973):
  * rate-monotonic (RM) scheduling 
    * full and immediate preemption
    * periodic tasks
    * deadline = around time of next request
    * static role-monotonic priority => task priority accord to actual frequency
    * admission test: if total utilization < 70%, then scheduler will guarantee all deadlines
  * earliest deadline first (ddf):
    * get optimal admission if util < 100%
    * algos exist to bring down complexity of sorting
  * only works for one processor

multiprocessor scheduling: np-hard

linux objectives:
* hardware interrupts arbitrated by interrupt controller service
* synchronization => memory allocation (priority inversion)
* work conservation
* time-slicing preemption for fairness
* interrupt mitigation

# Week 13. Apr 4
## multi-core programming
parallelism:
* unknown and nondeterministic execution ordering
* asynchronous memory access
* need: atomicity and memory ordering
* want: critical section (ACI)

without atomic ops, use _dekker's algo_:
```c
// shared state
int turn = 0;
int wantIn[2] = {false, false};  // bool

// thread init
int me = 0;
int you = 1;

// thread critical section
wantIn[me] = true;
while (wantIn[you]) {
    if (turn != me) {
        wantIn[me] = false;
        while (turn != me) { /* wait */ }
        wantIn[me] = true;
    }
}
// critical code ...
turn = you;
wantIn[me] = false;
```

with atomic RMW operations (atomic increment, test-set etc.), use _ticket lock_:
```c
// shared state
int ticket = 0;
int serving = 0;

// thread critical section
int myTicket = ticket++;
while (myTicket != serving) { /* wait */ }
// critical code
serving++;
```

### memory ordering
__eg.__
```c
// shared state
int a = 0, b = 0;

// thread 1
a = 1;
print(b);

// thread 2
b = 1;
print(a);
```

outcome of 00 is possible because of delayed writes.

_TSO (total store order)_:
* classic memory model
* writes can be deferred before reads
* write conflicts are never reordered
* (permits 00 in example)

_memory barriers_ (fences): special instructions to enforce memory ordering
* blocks pipelines and turn TSO to total order
```c
// thread critical section
wantIn[me] = true;
fence;
while (wantIn[you]) {
    if (turn != me) {
        wantIn[me] = false;
        while (turn != me) { /* wait */ }
        wantIn[me] = true;
        fence;
    }
}
```

_sequential consistency_:
* processor order - loads and stores
* some total order - interleaving
* SC = TSO + fences
* (cannot have 00 in example)

ticket lock:
```c
// thread critical section
int myTicket = ticket++;
while (myTicket != serving) { /* wait */ }
fence;
// critical code
fence;
serving++;
```

x86 atomics have fence semantics.

volatile:
* C volatile forbids compiler optimization/reordering
* java volatile ensures sequential consistency

we use atomic instructions to ensure parallelism and fence for memory ordering.
