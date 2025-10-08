# KVM PowerPC Breakage Analysis: v4.12 to v4.13

## Summary

KVM for PowerPC broke between kernel versions v4.12 and v4.13 due to a change in the hardlockup detector configuration system combined with a new call to `hardlockup_detector_disable()` in the PowerPC KVM guest initialization code.

## The Change

In v4.13, the following code was added to `arch/powerpc/kernel/kvm.c` in the `kvm_guest_init()` function:

```c
/*
 * The hardlockup detector is likely to get false positives in
 * KVM guests, so disable it by default.
 */
hardlockup_detector_disable();
```

This change also included:
```c
#include <linux/nmi.h> /* hardlockup_detector_disable() */
```

## The Root Cause

### v4.12 Configuration

At v4.12, the hardlockup detector configuration was:

**In `lib/Kconfig.debug`:**
```kconfig
config HARDLOCKUP_DETECTOR
	def_bool y
	depends on LOCKUP_DETECTOR && !HAVE_NMI_WATCHDOG
	depends on PERF_EVENTS && HAVE_PERF_EVENTS_NMI
```

**In `arch/powerpc/Kconfig`:**
- PowerPC did NOT select `HAVE_HARDLOCKUP_DETECTOR_ARCH` or `HAVE_HARDLOCKUP_DETECTOR_PERF`
- PowerPC only had: `select HAVE_NMI if PERF_EVENTS`

**In `kernel/Makefile`:**
```makefile
obj-$(CONFIG_HARDLOCKUP_DETECTOR) += watchdog_hld.o
```

### v4.13 Configuration

At v4.13, the hardlockup detector configuration was changed significantly:

**In `lib/Kconfig.debug`:**
```kconfig
config HARDLOCKUP_DETECTOR
	bool "Detect Hard Lockups"
	depends on DEBUG_KERNEL && !S390
	depends on HAVE_HARDLOCKUP_DETECTOR_PERF || HAVE_HARDLOCKUP_DETECTOR_ARCH
	select LOCKUP_DETECTOR
	select HARDLOCKUP_DETECTOR_PERF if HAVE_HARDLOCKUP_DETECTOR_PERF
	select HARDLOCKUP_DETECTOR_ARCH if HAVE_HARDLOCKUP_DETECTOR_ARCH
```

**In `arch/powerpc/Kconfig`:**
PowerPC v4.13 now includes:
```kconfig
select HAVE_NMI				if PERF_EVENTS || (PPC64 && PPC_BOOK3S)
select HAVE_HARDLOCKUP_DETECTOR_ARCH	if (PPC64 && PPC_BOOK3S)
select HAVE_HARDLOCKUP_DETECTOR_PERF	if PERF_EVENTS && HAVE_PERF_EVENTS_NMI && !HAVE_HARDLOCKUP_DETECTOR_ARCH
```

**In `kernel/Makefile`:**
```makefile
obj-$(CONFIG_HARDLOCKUP_DETECTOR_PERF) += watchdog_hld.o
```

### The Problem

The function `hardlockup_detector_disable()` is defined in different places depending on configuration:

**In `include/linux/nmi.h`:**
```c
#if defined(CONFIG_HARDLOCKUP_DETECTOR)
extern void hardlockup_detector_disable(void);
#else
static inline void hardlockup_detector_disable(void) {}
#endif
```

**Implementation:**
- At v4.12: in `kernel/watchdog_hld.c` (built when `CONFIG_HARDLOCKUP_DETECTOR` is set)
- At v4.13: in `kernel/watchdog.c` (inside `#ifdef CONFIG_HARDLOCKUP_DETECTOR` block)

## Why It Breaks

1. **Configuration dependency changed:** In v4.13, `CONFIG_HARDLOCKUP_DETECTOR` now depends on either `HAVE_HARDLOCKUP_DETECTOR_PERF` OR `HAVE_HARDLOCKUP_DETECTOR_ARCH`

2. **PowerPC architecture support:** PowerPC Book3S (64-bit) systems select `HAVE_HARDLOCKUP_DETECTOR_ARCH`, but this depends on the specific PowerPC configuration being used.

3. **Build configuration mismatch:** If a PowerPC system doesn't meet the requirements for `HAVE_HARDLOCKUP_DETECTOR_PERF` or `HAVE_HARDLOCKUP_DETECTOR_ARCH`, then `CONFIG_HARDLOCKUP_DETECTOR` won't be enabled.

4. **Stub vs Real Function:** When `CONFIG_HARDLOCKUP_DETECTOR` is not enabled:
   - The header provides a static inline stub: `static inline void hardlockup_detector_disable(void) {}`
   - This should work fine for compilation
   
5. **The actual breakage:** The issue occurs when:
   - PowerPC KVM guest code calls `hardlockup_detector_disable()`
   - The system configuration doesn't enable `CONFIG_HARDLOCKUP_DETECTOR`
   - But the stub function may not be sufficient for the actual runtime behavior expected

## Affected Configurations

This primarily affects:
- 32-bit PowerPC systems (which don't have `HAVE_HARDLOCKUP_DETECTOR_ARCH`)
- 64-bit non-Book3S PowerPC systems
- Systems where `PERF_EVENTS` is not configured
- Any configuration where `CONFIG_HARDLOCKUP_DETECTOR` ends up being disabled

## Related Files Changed Between v4.12 and v4.13

1. `arch/powerpc/kernel/kvm.c` - Added the call to `hardlockup_detector_disable()`
2. `kernel/watchdog.c` - Reorganized to include hardlockup detector functions
3. `kernel/watchdog_hld.c` - Build conditions changed
4. `lib/Kconfig.debug` - Major restructuring of hardlockup detector configuration
5. `arch/powerpc/Kconfig` - Added new hardlockup detector architecture support flags

## The Actual Breakage Scenario

Based on the configuration changes, here's what breaks:

### For 32-bit PowerPC or non-Book3S 64-bit PowerPC:

1. These systems do NOT select `HAVE_HARDLOCKUP_DETECTOR_ARCH` (only PPC64 && PPC_BOOK3S does)
2. They may or may not have `PERF_EVENTS` enabled
3. Without either flag, `CONFIG_HARDLOCKUP_DETECTOR` cannot be enabled in v4.13
4. The call to `hardlockup_detector_disable()` becomes a no-op inline stub
5. **This is not necessarily a compilation error, but a functional change:**
   - In v4.12, these systems never called `hardlockup_detector_disable()` (the code didn't exist)
   - In v4.13, they call it but it does nothing (stub function)
   - The intended purpose - to disable false positive hard lockup detection in KVM guests - is not achieved

### For PPC64 Book3S systems:

1. These systems select `HAVE_HARDLOCKUP_DETECTOR_ARCH`
2. This means `CONFIG_HARDLOCKUP_DETECTOR` can be enabled (depends on `DEBUG_KERNEL` and not `S390`)
3. The function should work as intended - disabling the hard lockup detector

### The Configuration Dependency Issue

The key change in v4.13 is that `CONFIG_HARDLOCKUP_DETECTOR` changed from:
- **v4.12**: `def_bool y` (automatically enabled when dependencies met) with `depends on LOCKUP_DETECTOR && !HAVE_NMI_WATCHDOG && PERF_EVENTS && HAVE_PERF_EVENTS_NMI`
- **v4.13**: `bool "Detect Hard Lockups"` (user-selectable) with `depends on DEBUG_KERNEL && !S390 && (HAVE_HARDLOCKUP_DETECTOR_PERF || HAVE_HARDLOCKUP_DETECTOR_ARCH)`

This means:
1. It's now user-configurable (not automatic)
2. It requires `DEBUG_KERNEL=y` to even be available
3. It requires architecture support that didn't exist for all PowerPC variants

### Potential Runtime Issues

On systems where the hardlockup detector was previously automatically disabled by the KVM guest code but now isn't (because `CONFIG_HARDLOCKUP_DETECTOR` is not enabled), there could be:
- False positive hard lockup warnings when running as a KVM guest
- However, if `CONFIG_HARDLOCKUP_DETECTOR` isn't enabled, there's no detector to generate false positives anyway
- **The real issue**: On systems that DO have `CONFIG_HARDLOCKUP_DETECTOR` enabled but are running as KVM guests, the detector should be disabled to prevent false positives

## Conclusion

The breakage is caused by the unconditional call to `hardlockup_detector_disable()` in PowerPC KVM guest initialization code, combined with a major restructuring of how the hardlockup detector is configured across different architectures. 

The specific issue is that:
1. The code now calls `hardlockup_detector_disable()` early in boot on all PowerPC KVM guests
2. On systems where `CONFIG_HARDLOCKUP_DETECTOR` is not enabled (e.g., 32-bit PowerPC, or systems without `DEBUG_KERNEL=y`), this becomes a no-op
3. This is generally not a problem unless someone has a custom configuration with hardlockup detection enabled but not properly supported
4. The real breakage likely occurs when configuration expectations don't match the actual architecture support, or when `CONFIG_KVM_GUEST=y` is set but the system configuration doesn't properly support or doesn't enable `CONFIG_HARDLOCKUP_DETECTOR`

The most likely scenario for actual breakage is on systems that:
- Have `CONFIG_KVM_GUEST=y` 
- Have `CONFIG_DEBUG_KERNEL=y`
- Have the necessary architecture support for hardlockup detection
- But the hardlockup detector configuration is somehow incomplete or misconfigured
