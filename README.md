# QEMU Virtual Atheros AR9285 (ath9k) – Phase 1

A QEMU PCI-Express device emulation of the Atheros AR9285 wireless NIC,
targeting compatibility with the unmodified Linux `ath9k` kernel driver.

## Project Goals

**Phase 1 (this release):** Get the Linux `ath9k` driver to successfully
probe the virtual device.  The driver must bind to the PCI device, read the
silicon revision, parse the EEPROM, and progress through `ath9k_hw_init()`
without error.

Future phases (not yet implemented) will add TX/RX descriptor DMA, a
virtual-air medium for frame exchange between multiple VMs, and eventually
a physics-inspired channel model.

## Architecture

```
┌──────────────────────────────────────────────┐
│  Guest VM (Linux kernel + ath9k driver)      │
│                                              │
│  ath9k_pci_probe()                           │
│    └─ ath9k_hw_init()                        │
│         ├─ REG_READ(AR_SREV)     ──┐        │
│         ├─ ath9k_hw_set_reset()    │        │
│         ├─ ath9k_hw_4k_fill_eep() │        │
│         └─ ath9k_hw_fill_cap()    │        │
│                                    │        │
│  PCI BAR0 MMIO (64 KiB)          │        │
└────────────────────────────────────┼────────┘
                                     │ MMIO R/W
┌────────────────────────────────────▼────────┐
│  QEMU – ath9k_pci.c device model           │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │  Register File (uint32_t[16384])        │ │
│  │  • AR_SREV → returns AR9285 v1.2 ID    │ │
│  │  • AR_RTC_* → power/reset sequencing    │ │
│  │  • AR_EEPROM → EEPROM present + valid   │ │
│  │  • All others → shadow + log warning    │ │
│  └─────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────┐ │
│  │  EEPROM Image (512 × 16-bit words)     │ │
│  │  • 4K format with valid magic + CRC     │ │
│  │  • MAC: 00:03:7F:AA:BB:CC              │ │
│  │  • 2.4 GHz only, 1×1 MIMO              │ │
│  │  • World regulatory domain              │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  PCI ID: 168C:002B (Atheros AR9285 PCIe)    │
└──────────────────────────────────────────────┘
```

## Source Files

| File | Purpose |
|------|---------|
| `src/ath9k_pci.c` | Main QEMU device model – MMIO handlers, PCI setup, lifecycle |
| `src/ath9k_regs.h` | Register addresses and bit-field definitions from the kernel's `reg.h` |
| `src/ath9k_eeprom.h` | EEPROM image generator – builds a valid 4K-format EEPROM at init |
| `src/meson.build` | Meson build integration for QEMU's build system |
| `src/Kconfig` | Kconfig entry for enabling the device |
| `scripts/integrate.sh` | Shell script to copy files into a QEMU tree and patch build files |
| `tests/test_probe.sh` | Three-level test script (registration → instantiation → guest boot) |
| `Makefile` | Top-level convenience targets |

## Prerequisites

- **QEMU source tree**: Git clone of `https://github.com/qemu/qemu.git`.
  Tested with QEMU 8.x and 9.x.  Any version with Meson build should work.

- **Build dependencies** (for QEMU itself):
  ```
  # Debian/Ubuntu
  sudo apt-get install -y \
      git build-essential meson ninja-build pkg-config \
      libglib2.0-dev libpixman-1-dev libslirp-dev \
      python3 python3-venv flex bison

  # Fedora/RHEL
  sudo dnf install -y \
      git gcc meson ninja-build pkg-config \
      glib2-devel pixman-devel libslirp-devel \
      python3 flex bison
  ```

- **For Level 3 tests**: A Linux guest disk image (qcow2) with the `ath9k`
  module built into the kernel or available as a loadable module.

## Quick Start

```bash
# 1. Clone QEMU (if you haven't already)
git clone https://github.com/qemu/qemu.git /path/to/qemu
cd /path/to/qemu
git submodule update --init --recursive

# 2. Integrate the virtual ath9k device
cd /path/to/this-project
make integrate QEMU_SRC=/path/to/qemu

# 3. Configure QEMU (x86_64 only, with debug symbols)
make configure QEMU_SRC=/path/to/qemu

# 4. Build QEMU
make build QEMU_SRC=/path/to/qemu

# 5. Run the test suite
make test QEMU_SRC=/path/to/qemu

# 6. (Optional) Run with a Linux guest
make test QEMU_SRC=/path/to/qemu GUEST_IMAGE=/path/to/guest.qcow2
```

Or do it all in one command:

```bash
make all QEMU_SRC=/path/to/qemu
```

## Manual Build (Step by Step)

If you prefer not to use the Makefile:

```bash
# Copy files into QEMU tree
./scripts/integrate.sh /path/to/qemu

# Configure and build
cd /path/to/qemu
mkdir -p build && cd build
../configure --target-list=x86_64-softmmu --enable-debug
make -j$(nproc)

# Verify the device is registered
./qemu-system-x86_64 -device help 2>&1 | grep ath9k

# Run with the device attached
./qemu-system-x86_64 \
    -machine q35 \
    -m 512 \
    -device ath9k-virt \
    -d guest_errors,unimp \
    -D /tmp/ath9k.log \
    -drive file=guest.qcow2,format=qcow2,if=virtio \
    -nographic
```

## Debugging

### Viewing Register Accesses

The device logs every register access through QEMU's logging system.
Enable it with:

```bash
-d guest_errors,unimp -D /tmp/ath9k.log
```

This produces output like:

```
ath9k-virt: READ  AR_SREV                  = 0x030020ff (AR9285 v1.2)
ath9k-virt: WRITE AR_RTC_FORCE_WAKE        = 0x00000001
ath9k-virt: READ  AR_RTC_STATUS            = 0x00000002
ath9k-virt: READ  EEPROM [0] = 0xa55a
ath9k-virt: WARNING: UNHANDLED read  REG(0x00a4)       = 0x00000000
```

**Every unhandled register** logs a warning with the exact address and
current value.  This is deliberate: it makes it trivial to identify which
registers the driver touches next, so you can add proper handling.

After 200 warnings of each type, further messages are suppressed to avoid
flooding the log.

### Guest Kernel Messages

Inside the guest VM, check `dmesg` for ath9k messages:

```bash
dmesg | grep -i 'ath\|wifi\|802\.11'
```

Expected output on success:

```
ath: EEPROM regdomain: 0x0
ath: Country alpha2 being used: 00
ath: Regpair used: 0x1f
ieee80211 phy0: Atheros AR9285 Rev:2
```

Expected output if EEPROM parsing fails:

```
ath: phy0: Unable to initialize hardware; initialization status: -22
ath9k 0000:XX:XX.0: Failed to initialize device
```

If you see the failure message, check the QEMU log for which EEPROM
offset or register read caused the issue, then adjust the EEPROM image
or register handler accordingly.

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `ath9k-virt` not in `-device help` | Device not compiled in | Re-run `scripts/integrate.sh` and rebuild |
| `Mac Chip Rev 0x00.0 is not supported` | AR_SREV returning wrong value | Check that `ath9k_mmio_read()` handles `0x4020` |
| `Unable to initialize hardware; status: -22` | EEPROM validation failed | Check EEPROM magic, version, checksum, opCapFlags |
| `Unable to initialize hardware; status: -5` | RTC reset timeout | AR_RTC_STATUS must return `AR_RTC_STATUS_ON` |
| `no band has been marked as supported` | opCapFlags missing 11G bit | Verify `AR5416_OPFLAGS_11G` is set in EEPROM |
| `0xdeadbeef` in register reads | Read from unmapped offset | Ensure all accessed offsets return from the shadow array |

## Design Decisions

1. **Conservative defaults**: Unknown registers return zero from the shadow
   array, not `0xdeadbeef`.  The driver interprets `0xdeadbeef` as a hardware
   hang and forces a reset.  Zero is a safe no-op for most control registers.

2. **Logging over silence**: Every access is traced.  In Phase 1, visibility
   into the driver's behaviour is more valuable than performance.

3. **EEPROM in a header**: The EEPROM image is generated at device
   realization time by `ath9k_eeprom_init_4k()` in a header file.  This
   avoids external data file dependencies and keeps the build self-contained.

4. **Immediate completion**: Operations that the driver polls for (RTC
   wakeup, EEPROM reads) complete instantly.  The driver sees the done
   bit on its first poll.  This is acceptable for Phase 1 where we don't
   emulate timing.

5. **Single BAR**: The AR9285 uses a single 64 KiB memory-mapped BAR.
   We register it as BAR0 with `PCI_BASE_ADDRESS_SPACE_MEMORY`.

## Register Coverage

The following registers are explicitly handled in Phase 1:

| Register | Address | Read | Write | Notes |
|----------|---------|------|-------|-------|
| AR_SREV | 0x4020 | ✓ | — | Returns AR9285 v1.2 signature |
| AR_CFG | 0x0014 | ✓ | ✓ | Basic config, shadow register |
| AR_RTC_RC | 0x7000 | ✓ | ✓ | Reset control, sets status ON |
| AR_RTC_RESET | 0x7040 | ✓ | ✓ | Reset enable, sets status ON |
| AR_RTC_STATUS | 0x7044 | ✓ | — | Always returns ON |
| AR_RTC_FORCE_WAKE | 0x704C | ✓ | ✓ | Transitions to AWAKE |
| AR_RTC_PLL_CONTROL | 0x7014 | ✓ | ✓ | Shadow register |
| AR_EEPROM | 0x401C | ✓ | ✓ | Reports present + valid |
| AR_EEPROM_STATUS_DATA | 0x407C | ✓ | — | Reports not busy |
| EEPROM window | 0xC000+ | ✓ | — | 512 × 16-bit words, 4K format |
| AR_ISR/IMR/IER | various | ✓ | ✓ | Interrupt stubs (no IRQs raised) |
| AR_GPIO_* | 0x4048+ | ✓ | ✓ | Shadow registers |
| AR_WA | 0x4004 | ✓ | ✓ | Workaround register, AR9285 default |
| AR_STA_ID0/1 | 0x8000/4 | ✓ | ✓ | Station MAC, shadow |
| AR_PM_STATE | 0x4008 | ✓ | — | Reports awake |
| All others | * | ✓ | ✓ | Shadow register + UNHANDLED warning |

## EEPROM Image

The generated EEPROM uses the AR9285 "4K" format:

- **Magic**: `0xa55a` (AR5416_EEPROM_MAGIC)
- **Version**: 14.14 (`0x0E0E`)
- **opCapFlags**: 2.4 GHz, HT20, HT40
- **Regulatory domain**: World (0x0000 / 0x001f)
- **MAC address**: `00:03:7F:AA:BB:CC` (Atheros OUI)
- **TX/RX chain mask**: 1×1 (single stream)
- **Checksum**: Valid XOR checksum

## Contributing

When adding support for new registers:

1. Add the register address to `ath9k_regs.h`
2. Add a `case` in `ath9k_mmio_read()` and/or `ath9k_mmio_write()`
3. Add the register name to the `ath9k_reg_names[]` lookup table
4. Test with the guest and verify the UNHANDLED warning disappears

The coding style follows the QEMU project conventions:
- 4-space indentation (no tabs)
- `lower_case_with_underscores` for functions and variables
- `UPPER_CASE` for macros and constants
- Every function and data structure has a comment explaining its purpose

## License

This project is licensed under the GNU General Public License v2.0 or later
(GPL-2.0-or-later), matching the Linux ath9k driver and the QEMU project.

## References

- [Linux ath9k driver source](https://github.com/torvalds/linux/tree/master/drivers/net/wireless/ath/ath9k)
- [QEMU device model documentation](https://www.qemu.org/docs/master/devel/device-emulation.html)
- [open-ath9k-htc firmware](https://github.com/qca/open-ath9k-htc-firmware)
- [Linux Wireless ath9k documentation](https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath9k.html)
- [mac80211_hwsim](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/virtual/mac80211_hwsim.c) – reference virtual Wi-Fi driver
- [wmediumd](https://github.com/bcopeland/wmediumd) – virtual air medium daemon
