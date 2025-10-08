# Executive Summary: KVM PowerPC Issue Between v4.12 and v4.13

## What Changed?

Between Linux kernel v4.12 and v4.13, KVM for PowerPC broke due to changes in the hardlockup detector subsystem configuration.

## The Key Changes

1. **New code in arch/powerpc/kernel/kvm.c:**
   - Added a call to `hardlockup_detector_disable()` in the KVM guest initialization function
   - Purpose: Prevent false positive hard lockup warnings when running as a KVM guest

2. **Hardlockup detector configuration restructured:**
   - Changed from automatic configuration (`def_bool y`) to user-configurable (`bool`)
   - Split into architecture-specific and perf-based implementations
   - Now requires `DEBUG_KERNEL=y` to be enabled

3. **PowerPC architecture support:**
   - PPC64 Book3S systems got new `HAVE_HARDLOCKUP_DETECTOR_ARCH` support
   - Other PowerPC variants rely on `HAVE_HARDLOCKUP_DETECTOR_PERF`

## Why It Breaks

The problem occurs when:

1. **CONFIG_KVM_GUEST is enabled** (for running Linux as a KVM guest on PowerPC)
2. **CONFIG_HARDLOCKUP_DETECTOR is NOT enabled** (because of missing dependencies or DEBUG_KERNEL=n)
3. The code calls `hardlockup_detector_disable()` but the function becomes a no-op stub

This creates a mismatch:
- The code *intends* to disable the hardlockup detector to prevent false positives
- But on systems without `CONFIG_HARDLOCKUP_DETECTOR`, there's no detector to disable
- However, if someone enables `CONFIG_HARDLOCKUP_DETECTOR` manually, the KVM guest code won't disable it properly on all PowerPC variants

## Affected Systems

- **32-bit PowerPC systems** - Don't have architecture-specific hardlockup detector support
- **64-bit non-Book3S PowerPC systems** - Same issue as 32-bit
- **Systems without DEBUG_KERNEL=y** - Can't enable CONFIG_HARDLOCKUP_DETECTOR at all
- **Systems without proper PERF_EVENTS configuration** - May not meet dependencies

## What Actually "Broke"

The nature of the breakage is subtle:

1. **Not a compilation error** - The code compiles fine (stub function is provided)
2. **Not a runtime crash** - No immediate failure occurs
3. **Configuration dependency issue** - The intended behavior (disabling hardlockup detector in KVM guests) may not work on all PowerPC configurations
4. **Potential for false positives** - On systems that DO have hardlockup detection but the disable function doesn't work, KVM guests might get spurious lockup warnings

## Files Modified

1. `arch/powerpc/kernel/kvm.c` - Added hardlockup_detector_disable() call
2. `arch/powerpc/Kconfig` - Added architecture capability flags
3. `lib/Kconfig.debug` - Restructured hardlockup detector configuration
4. `kernel/watchdog.c` - Moved hardlockup_detector_disable() implementation here
5. `kernel/Makefile` - Changed build conditions for watchdog_hld.o

## Git References

- Tag v4.12: Last working version
- Tag v4.13: First version with the issue
- File: `arch/powerpc/kernel/kvm.c` - Primary location of the new code
- Commit: The changes were part of the v4.13 merge window

## Recommendations for Investigation

To understand the full impact, one would need to:

1. Identify which PowerPC configurations were commonly used between v4.12 and v4.13
2. Determine if those configurations had `CONFIG_HARDLOCKUP_DETECTOR` enabled
3. Check if there were user reports of:
   - Spurious hard lockup warnings in KVM guests
   - Configuration issues when building kernels with `CONFIG_KVM_GUEST=y`
4. Review the git log for any follow-up fixes to this code after v4.13

## Conclusion

The "breakage" is primarily a functional regression where the intended purpose (disabling hardlockup detection in KVM guests) may not be achieved on all PowerPC configurations, particularly 32-bit systems and systems without the necessary debug and perf infrastructure. The code itself compiles and runs, but the behavior may not match expectations depending on kernel configuration.
