# Light it up
## embedded-hal

In this chapter we are going to make one of the many LEDs on the back of the micro:bit blink since this is
basically the "Hello World" of embedded programming. In order to get this task done we will use a set of
abstractions provided by the crate `embedded-hal`. `embedded-hal` is a crate which provides a set of traits
that describe behaviour of hardware, for example the [OutputPin trait] which allows us to turn a pin on or off.

In order to use these traits we have to implement them for the chip we are using. Luckily this has already been done
in our case in the [nrf51-hal]. Crates like this are commonly referred to as HALs (Hardware Abstraction Layer)
and allow us to use the same API to blink an LED and of course many more complex things accross all chips that implement
the `embedded-hal` traits.

This also enables people to write crates that only rely on the `embedded-hal` traits being implemented for certain
objects which in turn enables anyone with a chip that implements `embedded-hal` to use this library for themselves, despite
the other possibly not even knowing about the existence of the MCU the consumer of the library is using.

For example, a
person working on an embedded project might decide to implement a driver to show characters on a screen and writes a
library based on `embedded-hal` for the screen, once this library is published every chip that has a HAL library can be
made to control said screen easily since the HALs all expose the same API.

[OutputPin trait]: https://docs.rs/embedded-hal/0.2.4/embedded_hal/digital/v2/trait.OutputPin.html
[nrf51-hal]: https://crates.io/crates/nrf51-hal

## The micro:bit LEDs

On the back of the micro:bit you can see a 5x5 square of LEDs, usually called an LED matrix. This matrix alignment is
used so that instead of having to use 25 seperate pins to drive every single one of the LEDs, we can just use 10 (5+5) pins in
order to control which column and which row of our matrix lights up. However, the micro:bit team implemented this a
little differently. Their [schematic page] says that it is actually implemented as a 3x9 matrix but a few columns simply
remain unused.

In order to determine which pins we need to control to light up an LED we can check out
micro:bit's open source [schematic], linked on the same page. The very first sheet contains the LED matrix circuit which
is apparently connected to the pins named ROW1-3 and COL1-9. Further down on sheet 5 you can see that these pins
directly map to our MCU. For example, ROW1 is connected to P0.13.

> **NOTE**: The naming scheme of the NRF51 for its pins (P0.13) simply refers to port 0 (P0) pin 13. This is done
> because on MCUs with dozens or hundreds of pins you usually end up with multiple pins grouped up as ports for the sake of
> clarity. The NRF51, however, is so small that it only has one GPIO port (P0).

[schematic page]: https://tech.microbit.org/hardware/schematic/
[schematic]: https://github.com/bbcmicrobit/hardware/blob/master/V1.5/SCH_BBC-Microbit_V1.5.PDF

## Actually lighting it up!

The code required to light up an LED in the matrix is actually quite simple but it requires a bit of setup. First take
a look at it and then we can go through it step by step:

```rust
#![deny(unsafe_code)]
#![no_main]
#![no_std]

use cortex_m_rt::entry;
use panic_halt as _;
use nrf51_hal as hal;
use hal::prelude::*;

#[entry]
fn main() -> ! {
    let p = hal::pac::Peripherals::take().unwrap();

    let p0 = hal::gpio::p0::Parts::new(p.GPIO);
    let mut row1 = p0.p0_13.into_push_pull_output(hal::gpio::Level::Low);
    let mut col1 = p0.p0_04.into_push_pull_output(hal::gpio::Level::Low);

    row1.set_high().unwrap();

    loop {}
}
```

The first few lines until the main function just do some basic imports and setup we already looked at before.
However, the main function looks pretty different to what we have seen up to now.

The first line is related to how most HALs written in Rust work internally. Usually these crates rely on so-called
PACs (Peripheral Access Crates). A PAC is usually an autogenerated crate that provides some minimal abstractions
for all the peripherals our MCU has to offer. `let p = hal::pac::Peripherals::take().unwrap();` basically takes all
these peripherals from the PAC and binds them to a variable.

> **NOTE**: If you are wondering why we have to call `unwrap()` here, in theory it is possible for `take()` to be called
> more than once. This would lead to the peripherals being represented by two separate variables and thus lots of
> possible confusing behaviour because two variables modify the same resource. In order to avoid this, PACs are
> implemented in a way that it would panic if you tried to take the peripherals twice.

Once we got the peripherals, we assemble the GPIO port 0 from them with `let p0 = hal::gpio::p0::Parts::new(p.GPIO);` and
proceed to construct the `ROW1` and `COL1` pin using the two lines below, initialized as a switched-off
(`hal::gpio::Level::Low`) push-pull output pin (`into_push_pull_output`).

> **NOTE** If you don't know what push-pull means, don't worry about it, it's mostly irrelevant for us here, if you do
> want to figure it out, have a look [here](https://en.wikipedia.org/wiki/Push%E2%80%93pull_output).

Now we can finally light the LED connected to `ROW1`, `COL1` up by setting the `ROW1` pin to high (i.e. switching it on).
The reason we can leave `COL1` set to low is because of how the LED [matrix circuit works]. Furthermore, `embedded-hal` is
designed in a way that every operation on hardware can possibly return an error, even just toggling a pin on or off. Since
that is highly unlikely in our case, we can just `unwrap()` the result.

[matrix circuit works]: TODO ADD LINK, suggestion? @code reviewers

## Testing it

Testing our little program is quite simple. We simply have to run `cargo-embed` again, let it flash and just like before,
open our GDB and connect to the GDB stub:

```
$ gdb target/thumbv6m-none-eabi/debug/led-roulette
(gdb) target remote :1337
Remote debugging using :1337
cortex_m_rt::Reset () at ~/.cargo/registry/src/github.com-1ecc6299db9ec823/cortex-m-rt-0.6.12/src/lib.rs:489
489     pub unsafe extern "C" fn Reset() -> ! {
(gdb)
```

If we now let the program run via the GDB `continue` command, one of the LEDs on the back of the micro:bit should light
up.