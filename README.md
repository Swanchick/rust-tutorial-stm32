# Who Presses First — STM32 with Rust

### NUCLEO-L452RE-P | STM32L452RE

> This tutorial was written and tested on **Linux Fedora**.

---

## Prerequisites

Before starting, make sure you have the following installed:

- [Rust](https://rustup.rs/) (nightly toolchain recommended)
- [probe-rs](https://probe.rs/) — for flashing and debugging over USB

### Installing probe-rs

```sh
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/probe-rs/probe-rs/releases/latest/download/probe-rs-tools-installer.sh | sh
```

After installation, verify it works:

```sh
probe-rs --version
```

### Adding the cross-compilation target

The STM32L452 uses a Cortex-M4F core, which maps to the `thumbv7em-none-eabihf` target:

```sh
rustup target add thumbv7em-none-eabihf
```

---

## Project Setup

### 1. Initialize the project

```sh
cargo init stm32-button
```

### 2. Enter the project directory

```sh
cd stm32-button
```

### 3. Add dependencies

```sh
cargo add cortex-m -F critical-section-single-core
cargo add cortex-m-rt
cargo add defmt defmt-rtt
cargo add panic-probe -F print-defmt
cargo add stm32l4xx-hal -F rt -F stm32l452
```

### 4. Configure the toolchain

#### i. Create the `.cargo` directory

```sh
mkdir .cargo
```

#### ii. Create `.cargo/config.toml`

```toml
[target.thumbv7em-none-eabihf]
runner = "probe-rs run --chip STM32L452RETx"
rustflags = [
  "-C", "link-arg=-Tlink.x",
  "-C", "link-arg=--nmagic",
  "-C", "link-arg=-Tdefmt.x",
]

[build]
target = "thumbv7em-none-eabihf"

[env]
DEFMT_LOG = "trace"
```

#### iii. Create `memory.x` in the project root

This file tells the linker the exact memory layout of the STM32L452RE — **512K Flash, 160K RAM**:

```
MEMORY
{
  /* K = KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   : ORIGIN = 0x20000000, LENGTH = 160K
}
```

### 5. Configure the release profile

Add the following to the end of `Cargo.toml`:

```toml
[features]
default = [
    "defmt-default",
]

defmt-default = []

[profile.release]
debug = 2
```

This keeps debug symbols in release builds, which `probe-rs` and `defmt` need to decode logs correctly.

### 6. Project structure

```
stm32-button/
├── .cargo/
│   └── config.toml
├── src/
│   └── main.rs
├── Cargo.toml
└── memory.x
```

### 7. Build and flash

Connect your NUCLEO board over USB, then run:

```sh
cargo run --release
```

`probe-rs` will flash the binary and immediately start streaming `defmt` logs to your terminal.

---

## Project 1 — LED Blink

The LED blink project is the embedded equivalent of "Hello World". It introduces the full peripheral initialization chain and the core GPIO API, making it the right starting point for understanding how Rust models hardware access on the STM32.

```rust
#![no_std]
#![no_main]

use defmt::*;
use defmt_rtt as _;
use panic_probe as _;
use cortex_m_rt::entry;

use cortex_m::Peripherals as CpuPeripherals;
use stm32l4xx_hal as hal;
use hal::{delay::Delay, pac::Peripherals as McuPeripherals, prelude::*};

#[entry]
fn main() -> ! {
    let cpu_peripherals = CpuPeripherals::take().unwrap();
    let mcu_peripherals = McuPeripherals::take().unwrap();

    let mut flash = mcu_peripherals.FLASH.constrain();
    let mut rcc = mcu_peripherals.RCC.constrain();
    let mut pwr = mcu_peripherals.PWR.constrain(&mut rcc.apb1r1);

    let clocks = rcc.cfgr.freeze(&mut flash.acr, &mut pwr);

    let mut delay = Delay::new(cpu_peripherals.SYST, clocks);

    let mut gpioa = mcu_peripherals.GPIOA.split(&mut rcc.ahb2);

    let mut led = gpioa
        .pa8
        .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);

    loop {
        delay.delay_ms(1000u32);

        debug!("Hello World");
        info!("Hi!");
        led.toggle();
    }
}
```

### `#![no_std]` and `#![no_main]`

These two attributes tell the Rust compiler that this is a bare-metal program — there is no operating system underneath. `no_std` disables the standard library, which relies on OS services like heap allocation and file I/O. `no_main` disables the default entry point convention, because on a microcontroller the startup sequence is handled by the runtime crate (`cortex-m-rt`), not by the OS.

### Panic and logging handlers

```rust
use defmt_rtt as _;
use panic_probe as _;
```

These two lines pull in side-effect-only crates. `panic_probe` installs a panic handler that reports panics over the debug probe instead of silently halting. `defmt_rtt` sets up the RTT (Real-Time Transfer) transport that `defmt` uses to send log messages to your terminal via `probe-rs`. Without them the program won't link.

### Taking peripherals

```rust
let cpu_peripherals = CpuPeripherals::take().unwrap();
let mcu_peripherals = McuPeripherals::take().unwrap();
```

On a microcontroller, peripherals are globally unique hardware resources. The HAL enforces this at the type level — `take()` returns `Some(...)` exactly once and `None` on every subsequent call. This makes it impossible to accidentally have two parts of your code driving the same peripheral at the same time. There is no equivalent guarantee in C; it relies entirely on developer discipline.

### Clock and power initialization

```rust
let mut flash = mcu_peripherals.FLASH.constrain();
let mut rcc   = mcu_peripherals.RCC.constrain();
let mut pwr   = mcu_peripherals.PWR.constrain(&mut rcc.apb1r1);

let clocks = rcc.cfgr.freeze(&mut flash.acr, &mut pwr);
```

`constrain()` converts the raw peripheral register block into a higher-level HAL type that exposes a safe API. The RCC (Reset and Clock Control) peripheral controls clock sources and frequencies for the entire chip. Calling `.freeze()` locks the clock configuration in place — after this point the `clocks` value carries the actual frequencies as types, and the HAL uses them to calculate correct timer and delay values at compile time where possible. The FLASH and PWR peripherals need to be present because changing clocks requires adjusting flash wait states and the power domain simultaneously.

### Delay

```rust
let mut delay = Delay::new(cpu_peripherals.SYST, clocks);
```

`Delay` uses the SysTick timer — a 24-bit countdown timer built into the Cortex-M core — to produce accurate blocking delays. It needs the resolved `clocks` value so it can calculate how many ticks correspond to a given number of milliseconds.

### GPIO setup

```rust
let mut gpioa = mcu_peripherals.GPIOA.split(&mut rcc.ahb2);

let mut led = gpioa
    .pa8
    .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);
```

`.split()` consumes the raw GPIOA peripheral and splits it into individual pin types, enabling the clock on the AHB2 bus in the process. Each pin is its own type in the HAL, and the mode — input, output, alternate function — is encoded in the type itself. Calling `into_push_pull_output()` physically writes the MODER and OTYPER registers and changes the pin's type so the compiler will only allow output operations on it from this point forward. You cannot call `.set_high()` on a pin that is still in input mode — that's a compile error, not a runtime error.

### Main loop

```rust
loop {
    delay.delay_ms(1000u32);
    debug!("Hello World");
    info!("Hi!");
    led.toggle();
}
```

The loop waits one second, emits log messages at two different severity levels via `defmt`, then toggles the LED. `debug!` and `info!` are zero-overhead logging macros — they transmit a short integer ID over RTT rather than a formatted string, and `probe-rs` reconstructs the human-readable message on the host side using the debug symbols in the binary.

---

## Project 2 — Button Interrupt (EXTI)

This project implements the actual "who presses first" game. Two buttons are wired to GPIO pins and configured to fire hardware interrupts on a falling edge (button press). A state machine tracks the game phase, and the interrupt handler determines the winner. The key challenge here is safely sharing mutable state between the main loop and the interrupt handler.

```rust
#![no_main]
#![no_std]

use core::{
    cell::RefCell,
    ops::{Deref, DerefMut},
};

use defmt_rtt as _;
use panic_probe as _;

use cortex_m::Peripherals as CpuPeripherals;
use defmt::*;
use hal::{
    delay::Delay,
    flash::FlashExt,
    gpio::{Edge, Input, PA12, PA15, PullUp},
    interrupt,
    pac::NVIC,
    prelude::*,
    pwr::PwrExt,
    rcc::RccExt,
};
use cortex_m_rt::entry;
use cortex_m::interrupt::{Mutex, free};
use stm32l4xx_hal as hal;
use stm32l4xx_hal::pac::Peripherals as McuPeripherals;

static GAME_PINS: Mutex<RefCell<Option<GamePins>>> = Mutex::new(RefCell::new(None));

enum Player {
    Left,
    Right,
}

enum GameState {
    Prepare,
    Press,
    Win(Player),
}

struct GamePins {
    left_button: PA12<Input<PullUp>>,
    right_button: PA15<Input<PullUp>>,
    state: GameState,
}

#[entry]
fn main() -> ! {
    let cpu = CpuPeripherals::take().unwrap();
    let mut mcu = McuPeripherals::take().unwrap();
    mcu.RCC.apb2enr.write(|w| w.syscfgen().set_bit());

    let mut flash = mcu.FLASH.constrain();
    let mut rcc = mcu.RCC.constrain();
    let mut pwr = mcu.PWR.constrain(&mut rcc.apb1r1);

    let clocks = rcc
        .cfgr
        .hclk(48.MHz())
        .sysclk(80.MHz())
        .pclk1(24.MHz())
        .pclk2(24.MHz())
        .freeze(&mut flash.acr, &mut pwr);

    let mut gpioa = mcu.GPIOA.split(&mut rcc.ahb2);

    let mut left_button = gpioa
        .pa12
        .into_pull_up_input(&mut gpioa.moder, &mut gpioa.pupdr);

    left_button.make_interrupt_source(&mut mcu.SYSCFG, &mut rcc.apb2);
    left_button.enable_interrupt(&mut mcu.EXTI);
    left_button.trigger_on_edge(&mut mcu.EXTI, Edge::Falling);

    let mut right_button = gpioa
        .pa15
        .into_pull_up_input(&mut gpioa.moder, &mut gpioa.pupdr);

    right_button.make_interrupt_source(&mut mcu.SYSCFG, &mut rcc.apb2);
    right_button.enable_interrupt(&mut mcu.EXTI);
    right_button.trigger_on_edge(&mut mcu.EXTI, Edge::Falling);

    unsafe {
        NVIC::unmask(hal::interrupt::EXTI15_10);
    }

    let mut blue_led = gpioa
        .pa8
        .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);

    let mut red_led = gpioa
        .pa11
        .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);

    free(|cs| {
        let game_pins = GamePins {
            left_button,
            right_button,
            state: GameState::Prepare,
        };

        GAME_PINS.borrow(cs).replace(Some(game_pins));
    });

    info!("Hello World!");

    let mut delay = Delay::new(cpu.SYST, clocks);
    let mut counter = 0;

    loop {
        free(|cs| {
            let mut game_pins_ref = GAME_PINS.borrow(cs).borrow_mut();
            if let Some(game_pins) = game_pins_ref.deref_mut() {
                match &game_pins.state {
                    GameState::Prepare => {
                        if counter == 0 {
                            blue_led.set_low();
                            red_led.set_high();
                        }

                        blue_led.toggle();
                        red_led.toggle();

                        delay.delay_ms(100u32);

                        if counter >= 50 {
                            counter = 0;
                            blue_led.set_low();
                            red_led.set_low();
                            game_pins.state = GameState::Press;
                        }

                        counter += 1;
                    }
                    GameState::Press => {}
                    GameState::Win(Player::Left) => {
                        blue_led.toggle();
                        delay.delay_ms(100u32);

                        if counter >= 20 {
                            counter = 0;
                            blue_led.set_low();
                            red_led.set_high();
                            game_pins.state = GameState::Prepare;
                        }

                        counter += 1;
                    }
                    GameState::Win(Player::Right) => {
                        red_led.toggle();
                        delay.delay_ms(100u32);

                        if counter >= 20 {
                            counter = 0;
                            blue_led.set_low();
                            red_led.set_high();
                            game_pins.state = GameState::Prepare;
                        }

                        counter += 1;
                    }
                }
            }
        });
    }
}

#[interrupt]
fn EXTI15_10() {
    free(|cs| {
        let mut game_pins_ref = GAME_PINS.borrow(cs).borrow_mut();

        if let Some(game_pins) = game_pins_ref.deref_mut() {
            if !matches!(game_pins.state, GameState::Press) {
                game_pins.left_button.clear_interrupt_pending_bit();
                game_pins.right_button.clear_interrupt_pending_bit();
                return;
            }

            let left_fire = game_pins.left_button.check_interrupt();
            let right_fire = game_pins.right_button.check_interrupt();

            if left_fire {
                game_pins.state = GameState::Win(Player::Left);
                info!("Left won!");
                game_pins.left_button.clear_interrupt_pending_bit();
            }

            if right_fire {
                game_pins.state = GameState::Win(Player::Right);
                info!("Right won!");
                game_pins.right_button.clear_interrupt_pending_bit();
            }
        }
    });
}
```

### State machine

```rust
enum Player { Left, Right }

enum GameState {
    Prepare,
    Press,
    Win(Player),
}
```

The game logic is modeled as an explicit state machine using Rust enums. `Prepare` is the countdown phase where LEDs blink to build anticipation. `Press` is the active window where button presses are valid. `Win(Player)` carries the result. Using an enum with data (`Win(Player)`) means the state and the outcome are stored together — there is no separate "winner" variable that could get out of sync with the state.

### Sharing state between main and interrupt: `Mutex<RefCell<Option<T>>>`

```rust
static GAME_PINS: Mutex<RefCell<Option<GamePins>>> = Mutex::new(RefCell::new(None));
```

This pattern is the standard way to share mutable data between the main loop and an interrupt handler in bare-metal Rust, and it is worth understanding each layer:

- **`static`** — the value lives for the entire program lifetime, which is required because interrupt handlers have no stack frame to borrow from.
- **`Mutex`** — this is `cortex_m::interrupt::Mutex`, not the standard library one. It grants access only inside a `free()` critical section, which temporarily disables interrupts. This prevents the interrupt handler from firing mid-access and corrupting the data.
- **`RefCell`** — provides interior mutability, allowing mutable borrows of the contents at runtime even though the outer `Mutex` is behind a shared reference. This is safe here because the `Mutex` ensures only one context can hold the `RefCell` borrow at a time.
- **`Option`** — the static must be initialized at compile time, but the `GamePins` struct can only be constructed at runtime after peripherals are taken. `Option` solves this by starting as `None` and being filled in during `main`.

### Enabling SYSCFG and configuring EXTI

```rust
mcu.RCC.apb2enr.write(|w| w.syscfgen().set_bit());
```

The SYSCFG peripheral must be explicitly clocked before it can be used. It is responsible for routing GPIO lines to the EXTI interrupt controller — without it, `make_interrupt_source()` would write to registers with no effect.

```rust
left_button.make_interrupt_source(&mut mcu.SYSCFG, &mut rcc.apb2);
left_button.enable_interrupt(&mut mcu.EXTI);
left_button.trigger_on_edge(&mut mcu.EXTI, Edge::Falling);
```

These three calls configure the full interrupt pipeline for a pin. `make_interrupt_source` wires the GPIOA pin to the EXTI line via SYSCFG. `enable_interrupt` unmasks that line in the EXTI controller. `trigger_on_edge(Falling)` configures the edge detector — falling edge means the interrupt fires the moment the button is pressed, as the pin goes from high to low due to the pull-up resistor.

### Unmasking the NVIC

```rust
unsafe {
    NVIC::unmask(hal::interrupt::EXTI15_10);
}
```

The NVIC (Nested Vectored Interrupt Controller) is the Cortex-M component that manages interrupt priorities and enables/disables individual interrupt lines at the CPU level. Even after configuring EXTI, the CPU will not actually respond to the interrupt until the corresponding NVIC line is unmasked. This is `unsafe` because unmasking an interrupt whose handler accesses uninitialized data would be undefined behavior — the programmer is responsible for ensuring the handler is ready before unmasking.

### The interrupt handler

```rust
#[interrupt]
fn EXTI15_10() {
    free(|cs| {
        ...
        if !matches!(game_pins.state, GameState::Press) {
            game_pins.left_button.clear_interrupt_pending_bit();
            game_pins.right_button.clear_interrupt_pending_bit();
            return;
        }
        ...
    });
}
```

`EXTI15_10` fires for any of the EXTI lines 10 through 15, which covers both PA12 and PA15. The handler first checks whether the game is in the `Press` state — if not, it clears the pending bits and returns immediately, effectively ignoring presses during the countdown or after a winner has been decided. Clearing the pending bit is mandatory: if it is not cleared before the handler returns, the interrupt will fire again immediately in an infinite loop.

---

## Project 3 — Timer Interrupts

This project demonstrates hardware timer interrupts — a fundamental mechanism for running periodic tasks independently of the main loop. Two timers fire at different frequencies, each toggling its own LED, while the main loop independently blinks a third LED using a blocking delay. All three run concurrently.

```rust
#![no_main]
#![no_std]

use core::{cell::RefCell, ops::DerefMut};

use defmt_rtt as _;
use panic_probe as _;

use cortex_m::{
    interrupt::{Mutex, free},
    peripheral::{NVIC, Peripherals as CpuPeripherals},
};
use defmt::*;

use hal::{
    delay::Delay,
    flash::FlashExt,
    gpio::{Output, PA8, PA11, PushPull},
    interrupt,
    pac::{Peripherals as McuPeripherals, TIM2, TIM6},
    prelude::*,
    pwr::PwrExt,
    rcc::RccExt,
    timer::{Event, Timer},
};
use cortex_m_rt::entry;
use stm32l4xx_hal as hal;

trait LedToggle {
    fn toggle(&mut self);
}

struct BlueTimer {
    led: PA11<Output<PushPull>>,
    timer: Timer<TIM2>,
}

impl LedToggle for BlueTimer {
    fn toggle(&mut self) {
        self.led.toggle();
        self.timer.clear_interrupt(Event::TimeOut);
    }
}

struct GreenTimer {
    led: PA8<Output<PushPull>>,
    timer: Timer<TIM6>,
}

impl LedToggle for GreenTimer {
    fn toggle(&mut self) {
        self.led.toggle();
        self.timer.clear_interrupt(Event::TimeOut);
    }
}

static BLUE_TIMER: Mutex<RefCell<Option<BlueTimer>>> = Mutex::new(RefCell::new(None));
static GREEN_TIMER: Mutex<RefCell<Option<GreenTimer>>> = Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let cpu = CpuPeripherals::take().unwrap();
    let mcu = McuPeripherals::take().unwrap();
    mcu.RCC.apb2enr.write(|w| w.syscfgen().set_bit());

    let mut flash = mcu.FLASH.constrain();
    let mut rcc = mcu.RCC.constrain();
    let mut pwr = mcu.PWR.constrain(&mut rcc.apb1r1);

    let clocks = rcc.cfgr.freeze(&mut flash.acr, &mut pwr);

    unsafe {
        NVIC::unmask(hal::stm32::Interrupt::TIM2);
        NVIC::unmask(hal::stm32::Interrupt::TIM6_DACUNDER);
    }

    let mut tim2 = Timer::tim2(mcu.TIM2, 10.Hz(), clocks, &mut rcc.apb1r1);
    tim2.listen(Event::TimeOut);

    let mut tim6 = Timer::tim6(mcu.TIM6, 20.Hz(), clocks, &mut rcc.apb1r1);
    tim6.listen(Event::TimeOut);

    let mut gpioa = mcu.GPIOA.split(&mut rcc.ahb2);

    let blue_led = gpioa
        .pa11
        .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);

    let green_led = gpioa
        .pa8
        .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);

    free(|cs| {
        BLUE_TIMER.borrow(cs).replace(Some(BlueTimer {
            led: blue_led,
            timer: tim2,
        }));

        GREEN_TIMER.borrow(cs).replace(Some(GreenTimer {
            led: green_led,
            timer: tim6,
        }))
    });

    info!("Hello World!");

    let mut delay = Delay::new(cpu.SYST, clocks);

    let mut red_led = gpioa
        .pa12
        .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);

    loop {
        red_led.toggle();
        delay.delay_ms(1000u32);
    }
}

#[interrupt]
fn TIM2() {
    free(|cs| {
        let mut blue_timer_ref = BLUE_TIMER.borrow(cs).borrow_mut();

        if let Some(blue_timer) = blue_timer_ref.deref_mut() {
            blue_timer.toggle();
        }
    });
}

#[interrupt]
fn TIM6_DACUNDER() {
    free(|cs| {
        let mut green_timer_ref = GREEN_TIMER.borrow(cs).borrow_mut();

        if let Some(green_timer) = green_timer_ref.deref_mut() {
            green_timer.toggle();
        }
    });
}
```

### `LedToggle` trait

```rust
trait LedToggle {
    fn toggle(&mut self);
}
```

Each timer-LED pair has a different concrete type — `BlueTimer` owns a `Timer<TIM2>` and `GreenTimer` owns a `Timer<TIM6>`. These are different types at the HAL level because TIM2 and TIM6 are different hardware peripherals. The `LedToggle` trait provides a unified interface for the toggle-and-clear operation, keeping the interrupt handlers identical in structure regardless of which hardware they're driving. This is the kind of abstraction that would require function pointers or callbacks in C; in Rust it is a zero-cost trait.

### Pairing the LED and timer in a struct

```rust
struct BlueTimer {
    led: PA11<Output<PushPull>>,
    timer: Timer<TIM2>,
}

impl LedToggle for BlueTimer {
    fn toggle(&mut self) {
        self.led.toggle();
        self.timer.clear_interrupt(Event::TimeOut);
    }
}
```

Bundling the LED pin and the timer together in a single struct is a deliberate design choice. The interrupt handler for TIM2 needs both — it must toggle the LED and clear the timer's interrupt flag before returning. Keeping them together means the handler can operate through a single mutex access, and it is impossible to forget to clear the flag because the struct encapsulates both operations as one.

### Timer configuration

```rust
let mut tim2 = Timer::tim2(mcu.TIM2, 10.Hz(), clocks, &mut rcc.apb1r1);
tim2.listen(Event::TimeOut);

let mut tim6 = Timer::tim6(mcu.TIM6, 20.Hz(), clocks, &mut rcc.apb1r1);
tim6.listen(Event::TimeOut);
```

`Timer::tim2()` configures TIM2 to fire at 10 Hz — ten times per second. The HAL calculates the correct prescaler and auto-reload register values from the `clocks` token automatically. `.listen(Event::TimeOut)` enables the timer's update interrupt, which fires every time the counter overflows. TIM6 is set to 20 Hz, so it fires twice as often — the green LED blinks at double the rate of the blue one.

### `TIM6_DACUNDER` interrupt name

```rust
#[interrupt]
fn TIM6_DACUNDER() { ... }
```

On the STM32L452, TIM6 shares its interrupt vector with the DAC underrun event — they are physically wired to the same interrupt line in hardware. This is why the handler is named `TIM6_DACUNDER` rather than just `TIM6`. The HAL reflects the actual hardware interrupt table, so the name must match exactly or the handler will never be called.

### Concurrent execution

The main loop runs its own 1 Hz red LED blink using a blocking `delay_ms`. Meanwhile, TIM2 and TIM6 fire asynchronously and toggle their LEDs through the interrupt handlers. All three LEDs blink at independent rates — 1 Hz, 10 Hz, and 20 Hz — without any of them interfering with the others. This is the core demonstration: hardware timers allow truly independent periodic tasks without any scheduling infrastructure.

---

## Project 4 — UART and ADC

This project reads an analog voltage from a potentiometer using the ADC (Analog-to-Digital Converter) and transmits the result over UART to a host machine. It introduces two new peripheral categories: analog signal acquisition and serial communication.

```rust
#![no_std]
#![no_main]

use core::fmt::Write;

use defmt_rtt as _;
use panic_probe as _;

use cortex_m::Peripherals as CpuPeripherals;
use cortex_m_rt::entry;
use stm32l4xx_hal as hal;
use stm32l4xx_hal::pac::Peripherals as McuPeripherals;
use hal::{
    adc::ADC,
    delay::Delay,
    flash::FlashExt,
    prelude::*,
    pwr::PwrExt,
    rcc::RccExt,
    serial::Serial,
};

#[entry]
fn main() -> ! {
    let cpu = CpuPeripherals::take().unwrap();
    let mcu = McuPeripherals::take().unwrap();

    let mut flash = mcu.FLASH.constrain();
    let mut rcc = mcu.RCC.constrain();
    let mut pwr = mcu.PWR.constrain(&mut rcc.apb1r1);

    let clocks = rcc
        .cfgr
        .sysclk(80.MHz())
        .pclk1(80.MHz())
        .pclk2(80.MHz())
        .freeze(&mut flash.acr, &mut pwr);

    let mut gpioa = mcu.GPIOA.split(&mut rcc.ahb2);

    let mut analog = gpioa.pa0.into_analog(&mut gpioa.moder, &mut gpioa.pupdr);

    let tx = gpioa
        .pa2
        .into_alternate::<7>(&mut gpioa.moder, &mut gpioa.otyper, &mut gpioa.afrl);

    let rx = gpioa
        .pa3
        .into_alternate::<7>(&mut gpioa.moder, &mut gpioa.otyper, &mut gpioa.afrl);

    let mut delay = Delay::new(cpu.SYST, clocks);

    let mut adc = ADC::new(
        mcu.ADC1,
        mcu.ADC_COMMON,
        &mut rcc.ahb2,
        &mut rcc.ccipr,
        &mut delay,
    );

    let serial = Serial::usart2(mcu.USART2, (tx, rx), 115_200.bps(), clocks, &mut rcc.apb1r1);
    let (mut tx, _) = serial.split();

    loop {
        let value = adc.read(&mut analog);
        if let Ok(value) = value {
            let voltage = (value as f32) * 3.3 / 4095.0;
            writeln!(tx, "Potentiometer value: {}\r", voltage).ok();
            delay.delay_ms(500u32);
        }
    }
}
```

### Analog pin configuration

```rust
let mut analog = gpioa.pa0.into_analog(&mut gpioa.moder, &mut gpioa.pupdr);
```

`into_analog()` configures PA0 in analog mode by writing the MODER register and disabling the pull-up/pull-down resistors via PUPDR. In analog mode the pin is disconnected from the digital logic and connected directly to the ADC input. Leaving pull resistors enabled would skew the reading, so the HAL handles disabling them as part of the mode transition.

### Alternate function GPIO

```rust
let tx = gpioa
    .pa2
    .into_alternate::<7>(&mut gpioa.moder, &mut gpioa.otyper, &mut gpioa.afrl);

let rx = gpioa
    .pa3
    .into_alternate::<7>(&mut gpioa.moder, &mut gpioa.otyper, &mut gpioa.afrl);
```

GPIO pins on STM32 can be connected to internal peripherals through alternate function (AF) routing. The `<7>` const generic selects AF7, which maps PA2 and PA3 to USART2's TX and RX lines on the STM32L452 — this mapping is fixed in hardware and documented in the datasheet's alternate function table. The HAL encodes the alternate function number as a compile-time constant, so selecting the wrong AF number for a pin produces a type error rather than silent misbehaviour.

### ADC initialization

```rust
let mut adc = ADC::new(
    mcu.ADC1,
    mcu.ADC_COMMON,
    &mut rcc.ahb2,
    &mut rcc.ccipr,
    &mut delay,
);
```

The ADC requires a startup calibration sequence that takes a few microseconds — this is why `delay` must be passed in during construction. `ADC_COMMON` is the shared configuration register block that controls the clock source for all ADC instances on the chip. `rcc.ccipr` is the clock independent peripheral clock register, which selects the ADC's independent clock domain. The HAL handles all of this internally; in STM32Cube this would be several screens of CubeMX configuration followed by generated initialization code.

### USART setup and split

```rust
let serial = Serial::usart2(mcu.USART2, (tx, rx), 115_200.bps(), clocks, &mut rcc.apb1r1);
let (mut tx, _) = serial.split();
```

`Serial::usart2()` takes ownership of the USART2 peripheral and the two configured pins, validates that the baud rate is achievable at the current clock frequency, and configures the peripheral registers. `.split()` consumes the `Serial` instance and returns the TX and RX halves as independent types. Since this project only transmits, the RX half is discarded with `_`. Splitting into independent halves allows TX and RX to be passed to different parts of a program without carrying unnecessary access to the other direction.

### Writing over UART with the `Write` trait

```rust
use core::fmt::Write;
...
writeln!(tx, "Potentiometer value: {}\r", voltage).ok();
```

`core::fmt::Write` is a trait from the core library that provides formatted output via the `write!` and `writeln!` macros. The HAL implements this trait for the TX half, so any type that can accept formatted text — including a UART transmitter — plugs into the same macro system used everywhere in Rust. The `\r` is necessary because most serial terminals expect `\r\n` line endings. `.ok()` discards the `Result` — in a real application you would want to handle transmission errors.

### ADC reading and voltage conversion

```rust
let value = adc.read(&mut analog);
if let Ok(value) = value {
    let voltage = (value as f32) * 3.3 / 4095.0;
    ...
}
```

`adc.read()` performs a single conversion and returns a `Result<u16, _>`. The raw value is a 12-bit integer ranging from 0 to 4095, corresponding to 0V to 3.3V (the reference voltage). The conversion to voltage is a simple linear scaling: `value / 4095 * 3.3`. The result is sent over UART every 500 ms, producing a continuous stream of voltage readings readable with any serial monitor.

---

> Made by Kyryl Lebedenko
