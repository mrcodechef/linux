# Detailed Code Changes: KVM PowerPC v4.12 to v4.13

## File: arch/powerpc/kernel/kvm.c

### Change: Added hardlockup detector disable call

**Location:** `kvm_guest_init()` function

**v4.12 version:**
```c
static int __init kvm_guest_init(void)
{
	if (!kvm_para_available())
		goto free_tmp;

	if (!epapr_paravirt_enabled)
		goto free_tmp;

	if (kvm_para_has_feature(KVM_FEATURE_MAGIC_PAGE))
		kvm_use_magic_page();

#ifdef CONFIG_PPC_BOOK3S_64
	/* Enable napping */
	powersave_nap = 1;
#endif

free_tmp:
	kvm_free_tmp();

	return 0;
}
```

**v4.13 version:**
```c
static int __init kvm_guest_init(void)
{
	/*
	 * The hardlockup detector is likely to get false positives in
	 * KVM guests, so disable it by default.
	 */
	hardlockup_detector_disable();

	if (!kvm_para_available())
		goto free_tmp;

	if (!epapr_paravirt_enabled)
		goto free_tmp;

	if (kvm_para_has_feature(KVM_FEATURE_MAGIC_PAGE))
		kvm_use_magic_page();

#ifdef CONFIG_PPC_BOOK3S_64
	/* Enable napping */
	powersave_nap = 1;
#endif

free_tmp:
	kvm_free_tmp();

	return 0;
}
```

**Also added at the top of the file:**
```c
#include <linux/nmi.h> /* hardlockup_detector_disable() */
```

---

## File: arch/powerpc/Kconfig

### Changes in architecture capabilities

**v4.12 version:**
```kconfig
select HAVE_NMI				if PERF_EVENTS
```

**v4.13 version:**
```kconfig
select HAVE_NMI				if PERF_EVENTS || (PPC64 && PPC_BOOK3S)
select HAVE_HARDLOCKUP_DETECTOR_ARCH	if (PPC64 && PPC_BOOK3S)
select HAVE_HARDLOCKUP_DETECTOR_PERF	if PERF_EVENTS && HAVE_PERF_EVENTS_NMI && !HAVE_HARDLOCKUP_DETECTOR_ARCH
```

**Also changed NMI_IPI config:**

**v4.12:**
```kconfig
config NMI_IPI
	bool
	depends on SMP && (DEBUGGER || KEXEC_CORE)
	default y
```

**v4.13:**
```kconfig
config NMI_IPI
	bool
	depends on SMP && (DEBUGGER || KEXEC_CORE || HARDLOCKUP_DETECTOR)
	default y
```

---

## File: lib/Kconfig.debug

### Major restructuring of hardlockup detector configuration

**v4.12 version:**
```kconfig
config HARDLOCKUP_DETECTOR
	def_bool y
	depends on LOCKUP_DETECTOR && !HAVE_NMI_WATCHDOG
	depends on PERF_EVENTS && HAVE_PERF_EVENTS_NMI
```

**v4.13 version:**
```kconfig
config HARDLOCKUP_DETECTOR_PERF
	bool
	select SOFTLOCKUP_DETECTOR

#
# Enables a timestamp based low pass filter to compensate for perf based
# hard lockup detection which runs too fast due to turbo modes.
#
config HARDLOCKUP_CHECK_TIMESTAMP
	bool

#
# arch/ can define HAVE_HARDLOCKUP_DETECTOR_ARCH to provide their own hard
# lockup detector rather than the perf based detector.
#
config HARDLOCKUP_DETECTOR
	bool "Detect Hard Lockups"
	depends on DEBUG_KERNEL && !S390
	depends on HAVE_HARDLOCKUP_DETECTOR_PERF || HAVE_HARDLOCKUP_DETECTOR_ARCH
	select LOCKUP_DETECTOR
	select HARDLOCKUP_DETECTOR_PERF if HAVE_HARDLOCKUP_DETECTOR_PERF
	select HARDLOCKUP_DETECTOR_ARCH if HAVE_HARDLOCKUP_DETECTOR_ARCH
	help
	  Say Y here to enable the kernel to act as a watchdog to detect
	  hard lockups.

	  Hardlockups are bugs that cause the CPU to loop in kernel mode
	  for more than 10 seconds, without letting other interrupts have a
	  chance to run.  The current stack trace is displayed upon detection
	  and the system will stay locked up.
```

**Key differences:**
1. Changed from `def_bool y` (automatic) to `bool` (user-selectable)
2. Now depends on `DEBUG_KERNEL`
3. Requires architecture support via `HAVE_HARDLOCKUP_DETECTOR_PERF` or `HAVE_HARDLOCKUP_DETECTOR_ARCH`
4. Added new sub-configurations: `HARDLOCKUP_DETECTOR_PERF` and `HARDLOCKUP_CHECK_TIMESTAMP`

---

## File: include/linux/nmi.h

### Function declaration (unchanged between versions)

**Both v4.12 and v4.13:**
```c
#if defined(CONFIG_HARDLOCKUP_DETECTOR)
extern void hardlockup_detector_disable(void);
extern unsigned int hardlockup_panic;
#else
static inline void hardlockup_detector_disable(void) {}
#endif
```

**Behavior:**
- When `CONFIG_HARDLOCKUP_DETECTOR` is enabled: function is declared as extern
- When `CONFIG_HARDLOCKUP_DETECTOR` is disabled: function becomes a no-op inline stub

---

## File: kernel/watchdog.c (v4.13) / kernel/watchdog_hld.c (v4.12)

### Implementation location changed

**v4.12: In kernel/watchdog_hld.c:**
```c
void hardlockup_detector_disable(void)
{
	watchdog_enabled &= ~NMI_WATCHDOG_ENABLED;
}
```

**v4.13: In kernel/watchdog.c (inside #ifdef CONFIG_HARDLOCKUP_DETECTOR):**
```c
/*
 * We may not want to enable hard lockup detection by default in all cases,
 * for example when running the kernel as a guest on a hypervisor. In these
 * cases this function can be called to disable hard lockup detection. This
 * function should only be executed once by the boot processor before the
 * kernel command line parameters are parsed, because otherwise it is not
 * possible to override this in hardlockup_panic_setup().
 */
void hardlockup_detector_disable(void)
{
	watchdog_enabled &= ~NMI_WATCHDOG_ENABLED;
}
```

**Implementation is the same, but:**
1. Moved from `watchdog_hld.c` to `watchdog.c`
2. Added detailed comment explaining the purpose
3. Still guarded by `#ifdef CONFIG_HARDLOCKUP_DETECTOR`

---

## File: kernel/Makefile

### Build rules changed

**v4.12:**
```makefile
obj-$(CONFIG_LOCKUP_DETECTOR) += watchdog.o
obj-$(CONFIG_HARDLOCKUP_DETECTOR) += watchdog_hld.o
```

**v4.13:**
```makefile
obj-$(CONFIG_LOCKUP_DETECTOR) += watchdog.o
obj-$(CONFIG_HARDLOCKUP_DETECTOR_PERF) += watchdog_hld.o
```

**Impact:**
- `watchdog_hld.o` is now built only when `CONFIG_HARDLOCKUP_DETECTOR_PERF` is enabled
- Previously it was built when `CONFIG_HARDLOCKUP_DETECTOR` was enabled
- This means the file organization changed to support both PERF-based and ARCH-specific implementations

---

## Summary of Impact

The changes create a more flexible hardlockup detector configuration system that allows architectures to provide their own implementation. However, this also means:

1. **For PPC64 Book3S systems:** Can use ARCH-specific hardlockup detector, function works as intended
2. **For PPC32 or other PPC64 systems:** May not have `CONFIG_HARDLOCKUP_DETECTOR` enabled, function becomes a no-op stub
3. **Configuration dependency:** Now requires `DEBUG_KERNEL=y` to even enable hardlockup detection
4. **Behavioral change:** What was automatic in v4.12 is now user-configurable in v4.13

The KVM guest code now calls `hardlockup_detector_disable()` unconditionally, but whether this actually does anything depends on the kernel configuration and architecture support.
