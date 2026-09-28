# BMC CPLD Firmware Update And Recovery Test Plan

## Definitions/Abbreviation

| **Term** | **Description** |
|----------|-----------------|
| BMC | Baseboard Management Controller |
| CPLD | Complex Programmable Logic Device |
| fwutil | SONiC firmware CLI utility used to show/install/update platform component firmware |
| JTAG | IEEE 1149.1 serial interface used for in-system device programming |
| Firmware package | Vendor container that carries the CPLD programming file(s) together with version metadata |

## Overview

A switch has CPLDs that control power sequencing, thermal management and fan control. Their firmware is normally programmed from the host SONiC OS, which only works while the host is running. If the CPLD firmware is bad, the switch may not power on at all, because the CPLD is what starts it. There is then no host to program from, and without a BMC the switch is stuck.

The BMC solves this. It runs in its own power domain, so it stays up when the switch does not, and it has its own JTAG connection to the same CPLD. From the BMC, `fwutil` can program the switch CPLD, both for a routine update and to recover it after a bad image. The CPLD is listed as a component in the BMC's `platform_components.json`, so the same commands do both.

This test plan verifies that `fwutil` on the BMC updates the CPLD and reports its version correctly, rejects invalid firmware without changing the installed image, recovers a switch that cannot start because of a bad CPLD, and logs the operation. It reuses the fwutil test framework under `tests/platform_tests/fwutil/`, adding the CPLD as a component on BMC topologies.

## Test Cases

### Test Case 1: CPLD Firmware Update via `fwutil install`

**Objective**: Verify `fwutil install` uses the CPLD from a firmware package named on the command line, that the user is told a power cycle is required, and that the new version is reported once the power cycle has been done.

**Test Steps**
1. Record the CPLD version reported by `fwutil show status` on the BMC and on the host. Both sides read the same physical CPLD over separate paths.
2. Run `fwutil install chassis component <CPLD> fw <path-to-package> -y`, and verify it succeeds and reports that a power cycle is required to activate the firmware.
3. Verify both sides still report the version from step 1, because the new image is not active yet.
4. Power cycle the system, then verify both bmc and switch sides report the same target version.
5. Restore the original firmware version.

---

### Test Case 2: CPLD Firmware Update via `fwutil update`

**Objective**: Verify that `fwutil update` uses the firmware metadata (`platform_components.json`) staged in the current image to flash the CPLD.

**Test Steps**
1. Record the CPLD version reported by `fwutil show status` on the BMC and on the host.
2. Stage a `platform_components.json` describing the target CPLD firmware, together with the firmware package, into the current image on the BMC.
3. Run `fwutil update chassis component <CPLD> fw -y` and verify it succeeds with no errors.
4. Power cycle the system, then verify both sides report the target version.
5. Restore the original firmware version.

---

### Test Case 3: CPLD Firmware Version Policy (same / older / forced)

**Objective**: Verify `fwutil update` enforces the version policy for the CPLD: it skips the update when the package carries the version that is already installed, it skips a downgrade, and it programs an older version only when the force option is given.

**Note**: The version policy lives in `fwutil update`, which resolves the firmware from `platform_components.json` and skips the update when the installed version already matches, unless the force option `-f` is given. `fwutil install` takes the package named on the command line and performs no version comparison at all, and `-f` exists only on `fwutil update`, so this test case is written around `fwutil update`. `--force-update`, which both commands accept, is a different flag: it tells the component backend to reinstall, and does not bypass the version check. The comparison is also only possible when the platform API can read a version out of the firmware package, so this test case does not apply on platforms whose CPLD package carries no version metadata.

**Test Steps**
1. Record the installed CPLD version.
2. Stage the package whose version equals the installed one, run `fwutil update chassis component <CPLD> fw -y`, and verify the update is skipped, no programming is performed, and the version is unchanged.
3. Stage an older package, run the same command without `-f`, and verify the update is skipped and the version is unchanged.
4. Repeat step 3 with `-f`, power cycle the system, and verify both sides report the older version.
5. Restore the newest firmware version.

---

### Test Case 4: Reject Invalid CPLD Firmware Input

**Objective**: Verify `fwutil` refuses to program the CPLD when given invalid firmware, and keep the installed images unchanged. Three inputs are covered, each failing at a different stage of validation:
- A truncated package, which cannot be opened at all
- A package that opens but whose metadata is incomplete, so the programming file it names cannot be resolved
- A valid package built for a different device

**Test Steps**
1. Record the installed CPLD version.
2. Run `fwutil install chassis component <CPLD> fw <path> -y` for each invalid input in turn, and verify each returns a non-zero exit code with an error naming the reason.
3. After each attempt, verify both sides still report the version from step 1, and that the BMC is still up with its services running.

---

### Test Case 5: CPLD Recovery When The Host CPU Cannot Power On

**Objective**: Verify that the BMC can recover a CPLD whose firmware prevents the host CPU from powering on, and that the system boots normally afterwards.

**Test Steps**
1. Record the installed CPLD version.
2. Using a faulty CPLD prevents the switch from powering on.
3. From the BMC, run `fwutil install chassis component <CPLD> fw <valid-package> -y` to install a normal CPLD and verify it succeeds.
4. Power cycle the system, then verify the host SONiC boots normally and both sides report the recovered version.

---

### Test Case 6: Verify CPLD Firmware In Log And Techsupport

**Objective**: Verify that a CPLD firmware update is recorded in the system log, and that the support dump captures the CPLD firmware version both before and after the update.

**Test Steps**
1. Record the installed CPLD version and verify `show techsupport` on the BMC captures it.
2. Run a CPLD firmware update and, before power cycling the device, verify `show logging` on the BMC records the start of the update and its completion or failure.
3. Power cycle the system, then verify `show techsupport` on the BMC captures the new version.
4. Restore the original firmware version.

## Related Documents

| **Document Name** | **Link** |
|-------------------|----------|
| SONiC fwutil HLD | [fwutil.md](https://github.com/sonic-net/SONiC/blob/master/doc/fwutil/fwutil.md) |
| Support BMC HLD | [PR #2062](https://github.com/sonic-net/SONiC/pull/2062) |
| BMC High-Level Test Plan | [BMC-high-level-test-plan.md](BMC-high-level-test-plan.md) |
| BMC Firmware Upgrade Test Plan | [PR #26148](https://github.com/sonic-net/sonic-mgmt/pull/26148) |
| BMC Firmware Flavor Support Test Plan | [BMC-firmware-flavor-support-test-plan.md](BMC-firmware-flavor-support-test-plan.md) |
| FWUtil Test Plan (generic) | [FWUtil-test-plan.md](../FWUtil-test-plan.md) |
