# Quick Reference: KVM PowerPC v4.12 to v4.13 Changes

## One-Line Summary
**KVM for PowerPC broke because v4.13 added a call to `hardlockup_detector_disable()` while simultaneously restructuring the hardlockup detector configuration, making it unavailable on some PowerPC configurations.**

## The Smoking Gun

**File:** `arch/powerpc/kernel/kvm.c`

**Function:** `kvm_guest_init()`

**What was added in v4.13:**
```c
hardlockup_detector_disable();
```

## The Configuration Problem

| Aspect | v4.12 | v4.13 |
|--------|-------|-------|
| **CONFIG_HARDLOCKUP_DETECTOR** | `def_bool y` (automatic) | `bool` (user-selectable) |
| **Depends on** | `LOCKUP_DETECTOR && !HAVE_NMI_WATCHDOG && PERF_EVENTS && HAVE_PERF_EVENTS_NMI` | `DEBUG_KERNEL && !S390 && (HAVE_HARDLOCKUP_DETECTOR_PERF \|\| HAVE_HARDLOCKUP_DETECTOR_ARCH)` |
| **PowerPC 32-bit support** | Via PERF_EVENTS | None (no HAVE_HARDLOCKUP_DETECTOR_ARCH) |
| **PowerPC 64-bit Book3S** | Via PERF_EVENTS | Via HAVE_HARDLOCKUP_DETECTOR_ARCH |

## Function Behavior

When `CONFIG_HARDLOCKUP_DETECTOR` is:

- **Enabled:** Function exists in `kernel/watchdog.c`, clears `NMI_WATCHDOG_ENABLED` flag
- **Disabled:** Function is a no-op inline stub in `include/linux/nmi.h`

## Impact Matrix

| Configuration | CONFIG_HARDLOCKUP_DETECTOR | Function Behavior | Impact |
|---------------|---------------------------|-------------------|---------|
| PPC32 + DEBUG_KERNEL=n | Disabled | No-op stub | Intended behavior not achieved |
| PPC32 + DEBUG_KERNEL=y + no PERF | Disabled | No-op stub | Intended behavior not achieved |
| PPC64 Book3S + DEBUG_KERNEL=y | Can be enabled | Works correctly | Intended behavior achieved |
| PPC64 non-Book3S | Disabled | No-op stub | Intended behavior not achieved |

## Diff Summary

```diff
diff --git a/arch/powerpc/kernel/kvm.c b/arch/powerpc/kernel/kvm.c
+#include <linux/nmi.h> /* hardlockup_detector_disable() */
 
 static int __init kvm_guest_init(void)
 {
+       /*
+        * The hardlockup detector is likely to get false positives in
+        * KVM guests, so disable it by default.
+        */
+       hardlockup_detector_disable();
+
        if (!kvm_para_available())
```

```diff
diff --git a/arch/powerpc/Kconfig b/arch/powerpc/Kconfig
-       select HAVE_NMI                         if PERF_EVENTS
+       select HAVE_NMI                         if PERF_EVENTS || (PPC64 && PPC_BOOK3S)
+       select HAVE_HARDLOCKUP_DETECTOR_ARCH    if (PPC64 && PPC_BOOK3S)
+       select HAVE_HARDLOCKUP_DETECTOR_PERF    if PERF_EVENTS && HAVE_PERF_EVENTS_NMI && !HAVE_HARDLOCKUP_DETECTOR_ARCH
```

```diff
diff --git a/lib/Kconfig.debug b/lib/Kconfig.debug
 config HARDLOCKUP_DETECTOR
-       def_bool y
-       depends on LOCKUP_DETECTOR && !HAVE_NMI_WATCHDOG
-       depends on PERF_EVENTS && HAVE_PERF_EVENTS_NMI
+       bool "Detect Hard Lockups"
+       depends on DEBUG_KERNEL && !S390
+       depends on HAVE_HARDLOCKUP_DETECTOR_PERF || HAVE_HARDLOCKUP_DETECTOR_ARCH
+       select LOCKUP_DETECTOR
+       select HARDLOCKUP_DETECTOR_PERF if HAVE_HARDLOCKUP_DETECTOR_PERF
+       select HARDLOCKUP_DETECTOR_ARCH if HAVE_HARDLOCKUP_DETECTOR_ARCH
```

## Key Insight

The issue is **not a compilation error**, but a **configuration-dependent functional regression**. The code compiles fine because the header provides a stub function when `CONFIG_HARDLOCKUP_DETECTOR` is disabled. However, this means the intended purpose (preventing false positive hard lockup warnings in KVM guests) is not achieved on PowerPC configurations that don't enable the hardlockup detector.

## Related Subsystems

- **Watchdog/NMI subsystem** - Core subsystem that was restructured
- **KVM paravirtualization** - Consumer of the hardlockup_detector_disable() function
- **PowerPC architecture** - Platform with varying levels of hardlockup detector support

## To Reproduce

1. Configure kernel with:
   - `CONFIG_KVM_GUEST=y`
   - `CONFIG_DEBUG_KERNEL=n` (or on PPC32 system)
2. Build kernel
3. Boot as KVM guest
4. Observe that hardlockup detector is not disabled (if it was somehow enabled)

## Historical Context

This change was part of a larger effort to:
1. Allow architectures to provide their own hardlockup detector implementations
2. Separate PERF-based detection from architecture-specific implementations
3. Make hardlockup detection more configurable and less automatic

The KVM guest code change was likely done to address reports of false positive hard lockup warnings when running Linux as a KVM guest on PowerPC, but the configuration system changes broke the assumptions about when the function would be available.
