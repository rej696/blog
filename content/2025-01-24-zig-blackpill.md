---
date: 2025-01-24
tags: update, zig, blackpill, embedded, RTOS
---

# Baremetal Zig (Blackpill, RTOS)

## Investigating Embedded Zig in my Dissertation
During my university dissertation, I investigated using Zig for building and
writing baremetal embedded software for the STM32 Blackpill development board.

At the time, I was trying to use `build.zig` as a build system for the
baremetal C project I had already written, as well as integrate some Zig code
into the project.

Thanks to some friendly people on [Ziggit](https://ziggit.dev/), and some
helpful [example repos](https://github.com/haydenridd/stm32-zig-porting-guide),
I was able to get a working build.zig file that could compile both zig and C
for the STM32, link them together correctly with my hand written linkerscript,
and generate a binary file for flashing to the target hardware (and running
on my emulator, see [my dissertation project](https://github.com/rej696/dissertation).

However, I quickly dropped this as I had to focus on completing my project in
the time frame.

## Coming back to Zig
Now I have finished my dissertation, I'm interested in learning some more about
zig, and additionally wanted to finish
[Miro Samek's video series on RTOS](https://www.youtube.com/playlist?list=PLPW8O6W-1chyrd_Msnn4LD6LBs2slJITs),
as I am trying to learn more about RTOSes (and FreeRTOS) for a project at work.

I had made a lot of changes to my stm32 blackpill project through my
dissertation since playing with zig, so I spent a bit of time merging the zig
blackpill and dissertation project together.

I intend to keep extending this project in the future in
[this repo](https://github.com/rej696/zig-blackpill/tree/main),
implementing the RTOS in the video course, and playing around with zig.

## Things I learned about Zig and C in embedded so far
As I mentioned, I came up against a bunch of tricky issues to try and get my
project to compile with `build.zig`. I could have given up to use a pure zig
project like [microzig](https://github.com/ZigEmbeddedGroup/microzig). However,
the main reason I am interested in zig is it's ability to be used as a "drop
in" :eyes: C compiler and C interop. This _could_ make it waaaay more useful for
embedded than any other fancy new languages, as so many vendor toolchains and
libraries are written in C, and being able to make use of those without having
to rewrite anything could make it actually viable to use professionally (in my
opinion).

Something I think projects like embedded rust and microzig miss is that while
chips like rp2040 and STM32's are great (and used alot), and having a good
experience for writing software for them is great, there are so many different
microcontrollers around. For a language to be adopted in embedded it really
needs to be available on almost all of them, for the effort and cost of
committing to the language to be worthwhile. Not all of these chips are Arm
Cortex-M, for example I am currently working on a project that uses an Atmel
ARM Cortex-A5, and making heavy use of the vendor supplied board support
package and FreeRTOS port.

Anyway TLDR; C integration is very important, and thats why I wanted to see how
zig worked.

### Linker Issues
One of the hardest problems I had with using `build.zig` rather than a
`makefile` and `arm-none-eabi-gcc`, was that I couldn't figure out how to give
commands/options to the linker through zig. It took a lot of searching, reading
documentation and forums, and I'm still not entirely sure how I ended up with a
working `build.zig`... :sweat_smile:

One of the issues was that the LLVM linker `LLD` seemed to behave slightly (but
not that much) differently to the GCC `LD` linker. Notably, GCC seemed to be
able to read my linker script, and figure out that the memory sections in RAM
should be `NOLOAD`. However, `LLD` was not marking my `.stack` and `.bss`
sections as `NOLOAD`, and so `objcopy` was pulling in these RAM sections into
the binary file for programming the STM32, along with the `.text` and other
sections in flash. This was resulting in a huge binary file which was mostly
zeros, which I couldn't use to program the board!


### Figuring out the Zig Target
When you are compiling embedded software, you have to be quite specific to the
compiler to tell it what target you are compiling for. The STM32 on my
blackpill dev board is an Arm Cortex-M4, with an FPU (floating point unit).

I can tell `arm-none-eabi-gcc` what features are available (and therefore how to compile my code) with the following set of flags:
`-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16`.
These flags say "compile for cortex-m4", "use thumb mode instructions", "use
hardware floating point registers for float arguments" and "use floating point
instructions as defined by fpv4-sp-d16". See [this article] for more
explanation.

Using `build.zig`, these flags are supplied using a struct called `std.Target.Query`, where you can specify the architecture, cpu model etc.:
```zig
const target_default: std.Target.Query = .{
    .cpu_arch = .thumb,
    .os_tag = .freestanding,
    .abi = .eabi,
    .cpu_model = std.Target.Query.CpuModel{ .explicit = &std.Target.arm.cpu.cortex_m4 },
    .cpu_features_add = std.Target.arm.featureSet(
        &[_]std.Target.arm.Feature{
            std.Target.arm.Feature.vfp4d16sp,
        },
    ),
};
```

You can see (with hindsight) that these match up. However, if you look closer,
you can see that the flag `-mfpu=fpv4-sp-d16` matches, but isn't identical to
`std.Target.arm.Feature.vfp4d16sp`. I found trying to translate these flags was
a bit of a nightmare.

Luckily, since I worked through this last year, someone has written what looks
like a great tool/library called [gatz](https://github.com/haydenridd/gcc-arm-to-zig)
which should be able to translate the zig target structure from your GCC
compiler flags. I'm excited to try it out!


### NewLib
Another annoying thing about using `build.zig` over `arm-none-eabi-gcc` for
compiling C code at least, was trying to integrate Newlib. `arm-none-eabi-gcc`
includes a small `libc` implementation called
[Newlib](https://en.wikipedia.org/wiki/Newlib)
You can specify GCC to include Newlib in an embedded project by passing the
args `--specs nano.specs`.
[This is a nice article about including Newlib in an embedded
project](https://hackaday.com/2021/07/19/the-newlib-embedded-c-standard-library-and-how-to-use-it/)

However, while zig includes cross compilers for arm, with the ability to
configure the target, it doesn't seem to include an implementation of Newlib.
It seems to include other `libc` implementations that can be linked in, such as
[musl](https://musl.libc.org/). Hopefully someone can get Newlib included too!

Newlib allows you to use lots of libc standard library functions by providing
implementations for syscalls that the library functions use. In my project, the
only standard library functions I used was `memcpy`, which to be honest
wouldn't require Newlib (I could implement it myself), but I wanted to see how
to get Newlib linking anyway.

`arm-none-eabi-gcc` comes with the Newlib library, and luckily
[some other (smarter) people](https://ziggit.dev/t/stm32-porting-guide-first-pass/4414)
had already come across this problem, and I was able to use their
solution of writing a zig function in `build.zig` that would search for the
`arm-none-eabi-gcc` toolchain and find Newlib to link against.

### Map Files
Something I still haven't figured out is how to get `build.zig` to output map
files. Using the gcc toolchain, you can pass `-Wl,-Map=$@.map` as an argument
to the linker, which will create a map file, which gives you information about
all the symbols in your firmware, and where they are located. This can be super
useful for debugging any crashes on target where you have a Link Register and
Program Counter value, as they can help you determine where the crash might
have occurred. See
[this blog post](https://interrupt.memfault.com/blog/get-the-most-out-of-the-linker-map-file)
for more on map files.

It seems like zig [might never support generating map
files](https://github.com/ziglang/zig/issues/18356#issuecomment-1877618990),
but hopefully they come up with an equivalent diagnostic tool. You can get a
similar output by running  `arm-none-eabi-objdump -dh target.elf`, so its not
too big of an issue.

## Evaluation?
If the level of C interop doesn't change, and the build system gets some
additional features for more fine grained control of compiling and linking,
then I can absolutely imagine a future where Zig is used in tandem with C for
embedded software professionally.

At this point I haven't even really written any zig, just spent all my time
fiddling with the build system. Hopefully moving forward I can focus more on
the language itself.

