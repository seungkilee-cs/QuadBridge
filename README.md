# QuadBridge

**Turn your LG Quad DAC phone into a high-fidelity USB Audio Interface**

USB Audio Class 2.0 implementation for LG phones with ESS Sabre DACs (G7, G8, V60). This project enables peripheral/gadget mode to use your phone as an external DAC for computers and other audio sources.

## The Problem I'm Trying to Solve

I have an LG G7 sitting in a drawer. The battery is shot, Android updates stopped years ago, but the audio hardware is exceptional. The ESS ES9218P Quad DAC inside can handle 32-bit/384kHz audio with 122dB SNR and THD+N of 0.0002%. That is legitimately better than most dedicated USB DACs under \$200.

Throwing this away because the battery died feels wrong. These phones were specifically marketed for their audio capabilities. The DAC hardware is still perfectly functional. Why can't I just plug it into my computer and use it as a DAC?

Turns out, I can. But not without significant work.

## Why This Matters Beyond My Drawer

E-waste is a real problem. Lithium batteries degrade, Android support ends, but specialized audio hardware remains functional for decades. The ES9218P in the G7 was considered premium in 2018 and is still competitive today. Rather than mining more rare earth materials for new DACs, what if we could repurpose millions of these phones?

The secondary benefit is even better: if this works, these phones become hybrid devices. They can still function as digital audio players with internet connectivity, offline storage, and now also as desktop DACs. One device, multiple use cases, longer lifespan.

## My Background (Or Lack Thereof)

I need to be honest upfront. I am not a kernel developer. I have never written C professionally. My Android experience is limited to modifying build.prop files and flashing ROMs I didn't build myself.

This project is documentation of me learning as I go. If I can figure this out, others can too.

## The Research Journey

### Phase 1: The DAP Shortcut (That Didn't Exist)

When I started researching this, I looked at how modern digital audio players handle USB DAC mode. FiiO, HiBy, and other Android-based DAPs have a simple toggle in settings. You flip a switch, plug in USB, and the computer sees a UAC2 audio device. I thought maybe LG just didn't expose this feature in software.

I spent several days looking for documentation on Android's built-in USB audio capabilities. I found references to USB audio, but they were all about the phone acting as a host to external DACs. The phone sends audio out, not receives it.

Then I found Android Open Accessory (AOA) protocol documentation from 2012. AOA 2.0 included audio support. This looked promising until I read the forums. AOA audio was deprecated in Android 8.0. Even when it worked, it only supported 16-bit/44.1kHz output from Android to the host. Again, the wrong direction.

The standard Android framework has no concept of the phone acting as a USB audio receiver. The architecture assumes phones are hosts, not peripherals.

### Phase 2: Understanding the Direction Problem

This was the key realization that changed my approach. USB audio has two directions:

1. Phone as host, external DAC as peripheral - This is what USB Audio Player Pro does. The phone sends audio to a USB DAC dongle.
2. Phone as peripheral, computer as host - This is what I need. The computer sends audio to the phone's internal DAC.

Standard Android only supports direction 1. Modern DAPs that offer USB DAC mode had to implement direction 2 themselves during manufacturing. They wrote custom firmware, modified the USB stack, and pre-configured the audio routing. When you toggle USB DAC mode on a FiiO device, you are using functionality the manufacturer built before the device shipped.

LG did not build this functionality into the G7. I would have to add it myself.

### Phase 3: Discovering USB Gadget Framework

Once I understood that Android framework would not help me, I started looking at the Linux kernel underneath. Android runs on Linux, and Linux has something called the USB gadget API. This is specifically designed for devices that act as USB peripherals.

The kernel has a gadget function called f_uac2.c (USB Audio Class 2.0 function). This is what I need. If I can enable this in the kernel and configure it properly, the phone can present itself as a UAC2 audio device to any host computer.

The architecture looks like this:

```
Computer → USB Cable → Phone's USB Controller (gadget mode)
  → UAC2 gadget driver → ALSA virtual device
  → Audio routing → ES9218P DAC → 3.5mm output
```

The missing pieces:

1. LG's stock kernel probably doesn't have CONFIG_USB_CONFIGFS_F_UAC2 enabled
2. Even if enabled, there's no default configuration to activate it
3. The audio routing from USB input to the physical DAC needs to be set up
4. Android's audio framework will fight me for control of the DAC

This meant I needed to:

- Build a custom kernel with the right configuration
- Write scripts to configure USB gadget mode at runtime
- Set up ALSA (Linux audio system) to route the audio correctly
- Either bypass or modify Android's audio HAL to leave the DAC alone

### Phase 4: Breaking Down an Impossible Problem

I have never built an Android kernel before. I have never worked with ALSA. I have basic familiarity with C but have never contributed to kernel code. USB specifications are hundreds of pages of technical documentation that assumes prior knowledge.

So I broke it into smaller pieces I could tackle one at a time:

1. Can I build any kernel for this phone and have it boot? (Answer: yes, via Docker on macOS)
2. Can I enable USB gadget support and have the host computer recognize something? (Test with simplest gadget first)
3. Can I get the UAC2 gadget to create an ALSA device? (Check /proc/asound/cards)
4. Can I manually route audio from that ALSA device to the DAC? (Use command-line tools)
5. Can I automate the routing? (Write a daemon)
6. Can I make it survive reboots? (Init scripts and systemd-style services)

Each step has clear success criteria. Each step can be tested independently. If something breaks, I know which piece failed.

## The Technical Approach

### High-Level Architecture

The implementation has four layers:

**Layer 1: Kernel (USB Gadget Support)**

- Enable CONFIG_USB_CONFIGFS and CONFIG_USB_CONFIGFS_F_UAC2
- Enable CONFIG_SND_ALOOP for virtual audio routing
- Build and flash custom kernel

**Layer 2: USB Configuration (Runtime Gadget Setup)**

- Use ConfigFS to create UAC2 gadget configuration
- Set audio parameters (sample rate, bit depth, channels)
- Bind to USB Device Controller (UDC)

**Layer 3: Audio Routing (ALSA Bridge)**

- Route USB gadget ALSA input to ES9218P ALSA output
- Handle sample rate conversion if needed
- Minimize latency

**Layer 4: Android Integration (System Services)**

- Prevent Android audio framework from interfering
- Provide user interface for toggling DAC mode
- Handle edge cases (USB disconnect, sample rate changes)

### Development Phases

**Phase 1: Environment Setup** (Status: Complete for macOS M1)

- Docker build environment for kernel compilation
- Case-sensitive filesystem for kernel source
- ADB and fastboot tools configured
- Device bootloader unlocked and backed up

Milestone: Can run docker container and cross-compile basic ARM64 code

**Phase 2: Kernel Compilation** (Status: In Progress)

- Download LG G7 kernel source (LineageOS fork)
- Configure with USB gadget support
- Build kernel image
- Package into flashable boot.img

Milestone: Phone boots with custom kernel, no functionality lost

**Phase 3: USB Gadget Proof of Concept** (Status: Not Started)

- Create ConfigFS script to enable UAC2 gadget
- Test if host computer recognizes USB audio device
- Verify ALSA device creation on phone

Milestone: lsusb on computer shows UAC2 device, /proc/asound/cards on phone shows UAC2Gadget

**Phase 4: Audio Routing** (Status: Not Started)

- Load snd-aloop kernel module
- Configure ALSA to route USB input to ES9218P output
- Write routing daemon for continuous audio bridge
- Test latency and audio quality

Milestone: Audio plays through 3.5mm jack when sent from computer

**Phase 5: System Integration** (Status: Not Started)

- Create init service to configure gadget on boot
- Write Android app for user control
- Handle SELinux policies
- Add sample rate switching support

Milestone: Toggle works from app, survives reboot

**Phase 6: Optimization** (Status: Not Started)

- Reduce latency
- Add support for 96kHz/192kHz/384kHz
- Implement proper power management
- Handle edge cases and error recovery

Milestone: Stable for daily use

**Phase 7: Portability** (Status: Not Started)

- Port to LG G8 (Snapdragon 855, ES9218P)
- Port to LG V60 (Snapdragon 865, ES9219)
- Create unified installer
- Document device-specific configurations

Milestone: Working on all three devices with single codebase

## Hardware Specifications

### LG G7 ThinQ Audio System

**DAC Chip:** ESS ES9218P Sabre 32-bit Quad DAC

**Specifications:**

- Dynamic Range: 122 dB
- THD+N: 0.0002%
- Maximum Sample Rate: 384 kHz
- Bit Depth: 32-bit
- Output Power: 2.0 Vrms
- Filters: 4x digital filters (Sharp/Slow/Short/Slow)

**Why This Matters:**
These specifications exceed many dedicated USB DACs in the \$100-200 range. The ES9218P was ESS Technology's flagship mobile DAC in 2018. For comparison, popular desktop DACs like the Schiit Modi (\$129) use the AKM4490 chip with similar specifications.

### What Makes LG's Implementation Special

LG didn't just add the ES9218P chip. They implemented proper analog design with separate power rails, high-quality capacitors, and signal path optimization. The G7 can drive headphones up to 600 ohms impedance, which is unusual for phones.

This level of hardware investment is why it feels wasteful to discard these devices just because the battery or OS is outdated.

## Project Status

**Current Phase:** 2 (Kernel Compilation)

**What Works:**

- Docker build environment on macOS M1
- Kernel source acquired and configured
- Build toolchain tested

**What Doesn't Work Yet:**

- Everything else (this is very early stage)

**What I'm Working On Right Now:**

- First successful kernel build and boot test
- Verifying USB gadget support in kernel

## Documentation I'm Learning From

I am not inventing anything new. I am combining existing Linux kernel features that were designed for exactly this purpose. Here are the primary sources I am reading:

**Linux Kernel Documentation:**

- USB Gadget API documentation (kernel.org)
- USB Audio Class 2.0 gadget function (f_uac2.c source)
- ConfigFS USB gadget configuration guide
- ALSA API documentation

**Android Documentation:**

- AOSP kernel building guide
- Android audio architecture overview
- SELinux policy writing guide

**Community Resources:**

- XDA Developers kernel building tutorials
- LineageOS device bring-up documentation
- Various GitHub projects for USB audio gadget implementations

**Hardware Documentation:**

- ESS ES9218P datasheet
- Qualcomm Snapdragon 845 USB controller documentation
- USB 2.0 and Audio Class 2.0 specifications

The hardest part is not finding documentation. The hardest part is understanding which pieces apply to my specific situation and which are irrelevant.

## Why Open Source This?

Several reasons:

1. **Accountability:** If I document publicly, I am more likely to finish.
2. **Learning in Public:** Someone else might be stuck on the same problem. If I document what works and what doesn't, that saves them time.
3. **Collaboration:** I don't know what I don't know. Someone with more experience might see an obvious solution I'm missing.
4. **Sustainability:** If this works, it should outlive my interest in maintaining it. Open source means others can continue if I move on.
5. **Proof It's Possible:** Several people told me this couldn't be done or wasn't worth the effort. I would like to prove otherwise.

## Roadmap

**Short-term (Proof of Concept):**

- [ ] Boot custom kernel on G7
- [ ] Host computer recognizes phone as USB audio device
- [ ] Audio plays through 3.5mm jack from computer

**Medium-term (Minimum Viable Product):**

- [ ] Stable for daily use
- [ ] Simple toggle to enable/disable DAC mode
- [ ] Support for 44.1kHz, 48kHz, 96kHz, 192kHz

**Long-term (Feature Complete):**

- [ ] Port to G8 and V60
- [ ] Android app with sample rate control
- [ ] Automatic sample rate switching
- [ ] Power management optimization
- [ ] Installer package for easy deployment

**Very Long-term (Ambitious):**

- [ ] Support other LG phones (V40, V50)
- [ ] Support other ESS DAC-equipped devices
- [ ] Upstream kernel patches if acceptable quality

## Contributing

This is very early stage. I am not ready for code contributions yet because I am still learning the codebase myself. However, documentation improvements, testing on other devices, and issue reports are welcome.

If you have experience with USB gadget drivers, ALSA audio routing, or Android audio HAL modifications, your guidance would be extremely valuable.

## License

MIT License. Use this however you want. If it helps keep a phone out of a landfill, that's success.

## Current Progress Log

**2025-10-01:** Project started. Research phase complete. Docker environment configured. Kernel source acquired. README written to document approach and motivation.

**Next Steps:** Configure kernel with UAC2 support and attempt first build.

**A Note on Expectations**

This might not work. The audio quality might be degraded by routing overhead. Android might interfere in ways I cannot bypass. There might be a fundamental hardware limitation I don't know about yet.

But I won't know unless I try. And if I document the failures as thoroughly as the successes, at least the next person won't waste time repeating my mistakes.

That's the experiment.
