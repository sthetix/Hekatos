# Update Guide for Hekatos

## Scope

Hekatos follows original hekate by CTCaer. The intended functional additions are:

1. The `ofw=1` launch option.
2. Separate 4GB and 8GB release packages, including activation of the existing upstream forced-8GB configuration.

Keep upstream structure, boot behavior, SDRAM tables, Minerva, Nyx, and modules unless a specific change is approved. Hekatos naming and release artwork are presentation customizations, not additional boot features. Preserve original hekate credits and identify Hekatos as an unofficial fork.

This guide describes the intended update scope. It does not mean every existing source difference has already been reverted.

## Before Updating

- Compare the new upstream source with the current Hekatos source.
- Preserve the OFW handler and 4GB/8GB release wiring described below.
- Keep upstream firmware support and bundled binary assets aligned with the same official release.
- Do not carry additional feature patches forward automatically. Existing warmboot extractor code, custom `pkg1_id_t.fuses` fields, and silent BIS keygen changes are outside the intended scope; any source cleanup requires separate approval.
- Do not build, test, package, commit, or push without explicit user authorization.

## Update Script Status

Do not assume `update_hekate.py` implements this guide.

The current script copies upstream folders and applies OFW, branding, Lockpick_RCM_Pro, and warmboot-related patches. It does not reapply the complete 8GB flag wiring. Its extra patches do not match the intended functional scope above.

Review and obtain approval for script changes before using it as the update workflow. Updating this guide does not modify the script.

## Required Functional Changes

| File | Change to preserve |
|---|---|
| `bootloader/hos/hos_config.c` | `_config_ofw()` handler and `ofw` config entry |
| `res/hekate_ipl_template.ini` | OFW launch entry |
| `Makefile` | 4GB/8GB build option, output names, and release targets |
| `loader/Makefile` | Translate `DRAM_8GB` into `CONFIG_DRAM_8GB` |
| `bootloader/main.c` | Initialize the reserved DRAM flag for the selected variant |
| `loader/loader.c` | Initialize the same flag in the compressed payload loader |

### OFW Launch Option

Source: [SaudiCraft's comment in hekate issue #1227](https://github.com/CTCaer/hekate/issues/1227#issuecomment-4169407061). Hekatos uses the same handler logic, with the config key shortened from `reboot_ofw=1` to `ofw=1` and the handler named `_config_ofw`. This is an adaptation of that proposal, not a new OFW boot implementation.

In `bootloader/hos/hos_config.c`, preserve this handler:

```c
static int _config_ofw(launch_ctxt_t *ctxt, const char *value)
{
	if (*value == '1')
	{
		power_set_state(REBOOT_BYPASS_FUSES);
		while (true)
			bpmp_halt();
	}
	return 0;
}
```

Register it in `_config_handlers[]` before the terminating entry:

```c
	{ "ofw",              _config_ofw },
	{ NULL, NULL },
```

Preserve the template entry in `res/hekate_ipl_template.ini`:

```ini
[100% Stock OFW]
ofw=1
```

This invokes the existing upstream reboot mechanism. Do not change other launch handlers or the CFW boot path as part of this addition.

### 4GB/8GB Release Wiring

Both the root `Makefile` and `loader/Makefile` must define:

```makefile
ifeq ($(DRAM_8GB),1)
CUSTOMDEFINES += -DCONFIG_DRAM_8GB=1
else
CUSTOMDEFINES += -DCONFIG_DRAM_8GB=0
endif
```

In both `bootloader/main.c` and `loader/loader.c`, preserve this initializer in `ipl_ver`:

```c
#if CONFIG_DRAM_8GB
	.rcfg.rsvd_flags   = RSVD_FLAG_DRAM_8GB,
#else
	.rcfg.rsvd_flags   = 0,
#endif
```

The loader copies its reserved configuration into the decompressed payload before booting it. Setting only the bootloader flag is insufficient: a zero flag from the loader overwrites it. An `_8gb` filename or an unused compiler define does not activate 8GB mode.

Preserve the existing upstream behavior:

- `ipl_main()` checks `RSVD_FLAG_DRAM_8GB` and calls `fuse_force_8gb_dramid()` before `hw_init()`.
- The payload passes its reserved flags to Nyx, which applies the same override.
- The 4GB build leaves the flag zero and uses normal fuse-based configuration.
- No separate custom SDRAM tables or Minerva/module patches are required for this wiring.

Regression confirmed on 2026-10-22: both flags previously remained zero even in the 8GB package. Upgraded hardware retaining a 4GB fuse ID therefore did not activate forced 8GB configuration. After fixing the loader and bootloader wiring, the user reported that the affected tester confirmed it works. This feedback covers that device, not every RAM model.

## Release Packaging and Official Assets

Preserve the existing `release-4gb`, `release-8gb`, and `release` targets. Each variant's release target performs a clean build. Each package must contain the matching variant as both `payload.bin` and `bootloader/update.bin`, together with its required bootloader files.

Keep bundled assets aligned with the matching official hekate release, especially:

```text
extras/sys/emummc.kipm
extras/sys/res.pak
extras/sys/thk.bin
extras/sys/l4t/*.bin
```

Stale `emummc.kipm` can cause emuMMC to black screen on newer HOS even while sysMMC CFW works. Compare the bundled and packaged copies with the matching official release.

Also compare `extras/sys/l4t/sc7exit_b01.bin` byte-for-byte with the current official release ZIP, including both packaged variants. In [issue #1296](https://github.com/CTCaer/hekate/issues/1296), an incorrect 22.5.0 SAFE binary caused Mariko sleep failures; upstream corrected the ZIP without requiring a source change. File size and fuse header alone cannot identify the corrected binary. On 2026-10-22, the official 6.5.4 copy had SHA256 `2d8629e4e8fe8e7973b1cf5f0d0fa2e3614ede96a3e939f5508e5d807d8c7786`. Use the matching official release as the reference for future versions.

## Verification After an Update

Read-only source review:

- Confirm the OFW handler and table entry remain present.
- Confirm both Makefiles define the selected DRAM mode.
- Confirm both bootloader and loader initialize the reserved flag conditionally.
- Confirm the flag still reaches RAM initialization and Nyx.
- Confirm package names and contents match their selected variant.
- Review every other upstream difference instead of assuming it is required.

Run release commands only when explicitly authorized:

```bash
# Clean build and package both variants
rtk make release

# Or clean build and package one variant
rtk make release-4gb
rtk make release-8gb
```

For device verification of the 8GB fix, use the 8GB package on the corresponding RAM-upgraded hardware. Replace the payload actually used to boot, `bootloader/update.bin`, and the accompanying package files, then fully power off and boot again.

Where the raw fuse ID differs from the forced ID, Console Info should show `Forced DRAM Config 8GB`. This confirms selection of the forced configuration; device stability still requires hardware verification.

Do not run builds or tests merely because these commands appear in this guide.
