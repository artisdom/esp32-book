## Writing Rust Code to Create an LED Fading Effect on ESP32

Now comes the fun part; let's dive into the coding!

### Generate project using esp-generate

You have done this step already in the quick start section. 

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 led-fader
```

This will open a screen asking you to select options.  In the latest esp-hal, ledc requires us to explicitly enable the unstable features. 

- So you select the option "Enable unstable HAL features."

Then save it by pressing "s" in the keyboard.


<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">Imports</b>
            <p>In the explanation, I won't include the required imports since they aren't particularly interesting or need much explanation. You can always refer to the "Full Code" section below or clone the project to cross-check.</p>
    </div>
</div>


## Auto generated code

When you generate a project with esp-generate, it sets up the basic structure, configures the CPU clock, and includes boilerplate code for peripherals, so you don't have to type these out every time. In this book, we will always create projects using esp-generate and build on top of that.

```rust
let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
let peripherals = esp_hal::init(config);
```

## LED Pin

Next, we take our desired GPIO from the peripherals instance. In this case, we're turning on the onboard LED of the Devkit, which is connected to GPIO 2.

```rust
let led = peripherals.GPIO2;
```

### PWM configuration
In this exercise, we will be using the low-speed PWM channel. First, we need to set the clock source. The esp-hal library defines the `LSGlobalClkSource` enum for the low-speed clock source, which currently has only one value: `APBClk`.

```rust
ledc.set_global_slow_clock(LSGlobalClkSource::APBClk);
```

Next, we configure the timer. Since we are using the low-speed PWM channel, we obviously need to use the low-speed timer. We also have to specify which low-speed timer to use (from 0 to 3).

```rust
let mut lstimer0 = ledc.timer::<LowSpeed>(timer::Number::Timer0);
```

We need to do a few more configurations before using the timer. We'll set the frequency to 24 kHz. For this frequency with the APB clock, the formula gives a maximum resolution of 12 bits and a minimum resolution of 2 bits. In the esp-hal, a 5-bit PWM resolution is used for this frequency, and we will use the same.

```rust
lstimer0
    .configure(timer::config::Config {
        duty: timer::config::Duty::Duty5Bit,
        clock_source: timer::LSClockSource::APBClk,
        frequency: Rate::from_khz(24),
    })
    .unwrap();
```
 
### PWM Channels

Next, we configure the PWM channel. We'll use channel0 and set it up with the selected timer and initial duty percentage "10%". Additionally, we'll set the pin configuration as PushPull.

```rust
let mut channel0 = ledc.channel(channel::Number::Channel0, led);
channel0
    .configure(channel::config::Config {
        timer: &lstimer0,
        duty_pct: 10,
        pin_config: channel::config::PinConfig::PushPull,
    })
    .unwrap();
```

### Fading 

The esp-hal has a function called start_duty_fade, which makes our job easier. Otherwise, we would have to manually increment and decrement the duty cycle in a loop at regular intervals. This function gradually changes from one duty cycle percentage to another. It also accepts a third parameter, which specifies how much time it should take to transition from one duty cycle to another.

```rust
channel0.start_duty_fade(0, 100, 1000).unwrap();
```

We will run this in a loop and use another function provided by the HAL, `is_duty_fade_running`; It returns boolean value whether the duty fade is complete or not.

```rust
while channel0.is_duty_fade_running() {}
```

## The full code

```rust
#![no_std]
#![no_main]

use esp_hal::clock::CpuClock;
use esp_hal::ledc::channel::ChannelIFace;
use esp_hal::ledc::timer::TimerIFace;
use esp_hal::ledc::{channel, timer, LSGlobalClkSource, Ledc, LowSpeed};
use esp_hal::main;
use esp_hal::time::Rate;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[main]
fn main() -> ! {
    // generator version: 0.3.1

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let led = peripherals.GPIO2;
    // let led = peripherals.GPIO5;

    let mut ledc = Ledc::new(peripherals.LEDC);
    ledc.set_global_slow_clock(LSGlobalClkSource::APBClk);
    let mut lstimer0 = ledc.timer::<LowSpeed>(timer::Number::Timer0);
    lstimer0
        .configure(timer::config::Config {
            duty: timer::config::Duty::Duty5Bit,
            clock_source: timer::LSClockSource::APBClk,
            frequency: Rate::from_khz(24),
        })
        .unwrap();

    let mut channel0 = ledc.channel(channel::Number::Channel0, led);
    channel0
        .configure(channel::config::Config {
            timer: &lstimer0,
            duty_pct: 10,
            pin_config: channel::config::PinConfig::PushPull,
        })
        .unwrap();

    loop {
        channel0.start_duty_fade(0, 100, 1000).unwrap();
        while channel0.is_duty_fade_running() {}
        channel0.start_duty_fade(100, 0, 1000).unwrap();
        while channel0.is_duty_fade_running() {}
    }
}

```


## Clone the existing project
You can also clone (or refer) project I created and navigate to the `led-fader` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/led-fader
```

## Flashing
Once you flash the code into the ESP32, you should see the fading effect on the onboard LED.

```rust
cargo run --release
```
