this is my attempt to reverse engineer Marvell HDD SoCs and run Linux on them

Marvell 88iXXXX SoCs were used in consumer HDDs in 2005~2011. newer SoCs (apparently after 88i94XX?) are using Cortex R designs instead of in-house ARMv5 (or v6?) compatible CPUs.

there's no datasheets, product briefs or SDKs available for them. although HDDs with these often have JTAG pads visible, as well as lack of secure boot, which makes them pretty good target for learning compared to newer obscure Cortex R chips (like AVAGO).

based on my HDD collection, Samsung was a primary OEM using Marvell SoCs. WD ditched their custom ARM(?) SoCs in flavor of Marvell, but it's not clear how long they were using it. Seagate was staying at ST SoCs instead (or were they using Marvell too? i have only one old Seagate HDD).

upstream Linux has support for the Marvell Orion/Kirkwood families, but their iomap is really different compared to these.

SoCs:
- [Marvell 88i8846](#wd3200aaks-fried-rest-in-piss)
- [Marvell 88i9122](#samsung-hd204ui)

Potential SoCs for future:
- Marvell 88i6725S-TFJ1 (paired with 8MB RAM, booting Linux should be doable but painful; lot of unlabeled pads near the SoC, JTAG discovery might be painful too; HDD: Samsung HD200HJ)
- Marvell 88i6523-LFH1 (paired with 2MB RAM, i don't think it's possible to boot modern kernel on this thing; JTAG is visible; HDD: Samsung SP0842N, not even SATA)

## General info
### Boot chain
roughly looks like this:

BootROM -> map flash -> run decompressor -> read bootloader & RTOS and decompress -> run bootloader -> ??? -> RTOS

### Flash header
this was only checked for internal flash on 88i8846.

```c
struct __attribute__((packed)) Header
{
  unsigned __int8 id; /* 0x5A for bootloader (?) */
  unsigned __int8 type; /* 1 - compressed code, 3 - compressed data, 4 - bootloader */
  unsigned __int16 upper_compressed; /* decompressed size high word */
  unsigned int size_and_checksum; /* size + 1 */
  unsigned int size; /* compressed size */
  unsigned int flash_start; /* offset */
  unsigned int load_addr; /* load addr */
  unsigned int entrypoint; /* entrypoint, 0xffffffff if no need to jump */
  unsigned int unk;
  unsigned __int16 lower_compressed; /* decompressed size low word */
  unsigned __int8 pad;
  unsigned __int8 checksum; /* header checksum */
};
```
kudos to [malwaretech](https://www.malwaretech.com/2015/04/hard-disk-firmware-hacking-part-1.html) for figuring it out.

compression is very likely some custom one.

### RTOS
not really much to say about it. it looks very different in different vendors. i assume there's some kind of BSP and then vendors customize it.

### JTAG
cp15 doesn't work properly. MMU/caches status not reported properly in OpenOCD.

## WD3200AAKS (fried, rest in piss)
### SoC
![SoC](./88i8846/soc.jpg)

```
Marvell 88i8846-TFJ2
YPMS005AW
0922 A1P
TW
```

| Component | Details | Notes |
| :-- | :--: | :--: |
| CPU | 2x Feroceon ARMv6TEJ(?) (AMP, but may be reprogrammed to SMP in the RTOS) | no idea if it's v5 or v6 |

iomap:
| HW | Address | Notes |
| :-- | :--: | :--: |
| SRAM | 0x04000000 - 0x04007fff | SRAM, used for stack |
| DRAM | 0x24000000 - 0x25000000 | RTOS userspace\* |
| UART (not 8250-compatible by layout) | 0x1C00A620 | + 4 for THR, + 0x10 for LSR |
| Internal Flash | 0xfff00000 - ? | | 
| BootROM | 0xffff0000 - 0xffffffff | CPU0 uses dummy bootrom for debugging, CPU1 switches to the bootloader and RTOS |

### PMIC
![PMIC](./88i8846/pmic.jpg)

```
SMOOTH
L7251 3.1
AAAAY V5
TWN 99 921
ST (e3) 6AB
```

Specs unknown. fried.

### DRAM
![DRAM](./88i8846/dram.jpg)

```
Winbond
W9412G61H-5
0912W
69012A900
```

16 MB DDR

### Misc hardware
![oscillator](./88i8846/osc.jpg)

25 MHz external oscillator

### RTOS
vectors at 0xffff0000 (???), i can't really tell about this one, it was fried and i have no way to confirm some details from the other SoCs.

### BootROM
- [CPU0](./88i8846/core0brom.bin)
- [CPU1](./88i8846/core1brom.bin)

### UART
on the jumper pads, baud unknown. manual fixup needed to print something.

### JTAG
- Pinout: refer to [88i8846 research](https://www.malwaretech.com/2015/04/hard-disk-firmware-hacking-part-3.html).
- [OpenOCD config](./88i8846/88i8846.cfg)

may die if both cores are halted. or just randomly die.

### Boot logs
some very small amount of characters. the drive is not talkative at all.

## Samsung HD204UI
### SoC
![SoC](./88i9122/soc.jpg)

```
Marvell 88i9122-TFJ2
P4T9050.2
1102 B2E
TW
```

| Component | Details | Notes |
| :-- | :--: | :--: |
| CPU | 2x Feroceon ARMv6TEJ(?) (AMP, but may be reprogrammed to SMP in the RTOS) | no idea if it's v5 or v6, linux says v6TEJ |

seems to be newer revision of 88i8846's cores

iomap:
| HW | Address | Notes |
| :-- | :--: | :--: |
| DRAM | 0x0 ~ 0x00018000 | RTOS kernel\* |
| SRAM | 0x04000000 - 0x04007fff | SRAM, used for stack |
| DRAM | 0x10000000 - 0x12000000 | RTOS userspace\*; mirrored at 0x24000000, for some unknown reason the bootloader loads RTOS code to the mirror. it likely means the mirror is uncached, at least because zImage decompressor is insanely slow |
| UART (not 8250-compatible by layout) | 0x1C00A620 | + 4 for THR, + 0x10 for LSR |
| Timer | 0x1c00a200 - 0x1c00a250 | has 10 timers, timer_ctrl(n) = (n * 0x8), timer_val(n) = ((n * 0x8) + 0x4) |
| BootROM | 0xffff0000 - 0xffffffff | CPU0 uses dummy bootrom for debugging, CPU1 switches to the bootloader and RTOS |

### PMIC
![PMIC](./88i9122/pmic.jpg)

```
SH61258
12ADCLW
GJAMTYKC
CU 64
```

| Reference voltage | Actual voltage |
| :-- | :--: |
| 3.3V IO | 2.5V |
| 1.8V IO | 1.6V |

### DRAM
![DRAM](./88i9122/dram.jpg)

```
SAMSUNG 104
K4H561638N-LCCC
H5616
YAK02466U
```

32 MB DDR

### RTOS
vectors at 0x0, CPU0 gets reprogrammed to do (likely) motor tasks (always FIQ mode when i halt it), CPU1 runs something else (IRQ/Supervisor (???)/System/other) modes.

### BootROM
- [CPU0](./88i9122/core0brom.bin)
- [CPU1](./88i9122/core1brom.bin)

### UART
on the jumper pads, 57600 baud.

### JTAG
please forgive my soldering skills
![JTAG](./88i9122/jtag.jpg)

| Pin | Pin |
| :--: | :--: |
| ? | ? |
| reset | TCK |
| GND | TMS |
| TDO | TDI |

[OpenOCD config](./88i9122/88i9122.cfg)

both cores can be halted without issues.

### Boot logs
```
 ActiveFW : 00
             FWVer : 0001
                         SATA PLL cal done

                                          DDR size detected = 32MB
                                                                  *PA VID=0000 PN=0009 Rev=0001- 785x Found
                                                                                                           *PA VID=0000 PN=0009 Rev=0001- 785x Found
                                                                                                                                                    U
S_0
RV Sensor Circuit Enabled
Shock Sensor Circuit Enabled
SO_1
SpinStartUp: mcSpinRPM = 0
RPM at Handoff: 482
 Temp : 21 degC 
SpinOk
mS1 00000003 
SK C:134958 H:0
Loaded FIT ( 0: 0: 1)
CalibTable Loaded. Rev:0x1B
Selective MARC NX Loaded
ResoTable Loaded. Rev:0x01
RRO1xTable Loaded. Rev:0x01
Fw Active 0000
Ovly loaded to 0x00014D00
Ovly loaded to 0x1002E300
FdtTable Loaded. Rev:0x02
Reading Serial Num Pass
Up MC

PwrOn RRO1x @ H0
Table) cos = 0, sin = -1280
Coeff) cos = -545, sin = 756


PwrOn RRO1x @ H2
Table) cos = -256, sin = 768
Coeff) cos = -4079, sin = 1079


PwrOn RRO1x @ H4
Table) cos = 1792, sin = 512
Coeff) cos = -8671, sin = 5127

TgtCyl:    2032 
Hd:   0 Zn:   0 Avg.:-    125 
TgtCyl:  264688 
Hd:   0 Zn:   1 Avg.:      79 

SVCAL(0080,0000)-->PASS
RecordValid Ok : 0C07E47D 0007E41D
ReadyTime = 10299808 us


[DIPM_TMOUT:00000000]
ENG>
```

## Notes
\* - i have no idea if the RTOS has actual kernel/userspace split, but every time i halt CPU it usually runs at vectors. i assume vectors are placed near the 'core' functions. though 'kernel' calls to the 'userspace' for printf. lol.

## Linux
yes you can boot Linux on this thing (not pushed yet because i want to get it booting to the shell first :P). currently tested only on 88i9122, but may work on 88i8846 too. halt ONLY after RTOS is ready. openocd commands:
```
targets 0
halt
targets 1
halt
load_image ../zImage 0x24800000 bin
reg pc 0x24800000
reg r0 0
reg r1 0xffffffff
reg r2 0
arm core_state arm
resume
```

decompressor is SLOW. by slow i mean REALLY SLOW.

status:
- works: debug_ll
- doesn't work: printk, everything else

## Credits
it would be very hard to figure out some things myself. honorable mention to resources which helped me a lot.
- [88i8846 research](https://www.malwaretech.com/2015/04/hard-disk-firmware-hacking-part-1.html)
- [88i6545/88i6745n forum thread](https://forum.hddguru.com/viewtopic.php?f=13&t=20324)

## License
this readme and images under [CC 4.0 SA](./LICENSE.CC), openocd configs under [AGPLv3](./LICENSE.AGPL) 
