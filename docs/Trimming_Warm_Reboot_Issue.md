# Packet Trimming over Warm Reboot: Test Failure after the FW Fence Fix (RM 5301723)

---

## 1. Background

When an egress queue is full, a switch normally **drops** new packets. The sender only notices the loss after a timeout, so recovery is slow.

With **packet trimming**, the switch does not drop the packet. It cuts the packet down to a configured size, keeping the headers.
The receiver gets the trimmed packet, knows exactly which packet was lost, and can ask for a retransmit right away.


## 2. How verification tests trimming

**The problem:** In order to generate trimming, a queue must be full. In a real network that comes from many devices at line rate. Our sonic-mgmt tests send traffic from the PTF host, a software traffic generator. It is hard to fill a queue.

**The solution, used by all trimming tests:** create the congestion artificially.
1. **Block the queue.** Attach a scheduler with `pir=1, cir=1` to queue 1. SAI programs this as shaper rate 0, so queue 1 sends nothing.
2. **Disable packet aging (NVIDIA only).** Normally the ASIC drops packets that sit in the buffer longer than a lifetime limit. The aging would keep freeing space, so some new packets would be queued instead of trimmed. The test sets the lifetime limits to "infinite" through an SDK script, so the queue stays full.
3. **Fill the buffer** with packets sent to queue 1.
4. **Every new packet** sent to queue 1 now gets trimmed. The trimmed copy leaves through the trim queue, which is not blocked. The PTF host captures it, and the test checks its size, DSCP and counters.
5. **Release the queue** at the end.


## 3. How we test trimming over warm reboot, and the problem

**Requirement:** while the switch warm reboots (ISSU), the data plane keeps forwarding, and trimming must keep working with no lost trimmed packets.

**Test steps:**
1. Configure trimming and enable it on the buffer profile. Packet aging is disabled.
2. Block queue 1 on the egress port with the zero-rate shaper.
3. PTF sends a continuous UDP stream at 10,000 pps with 1,000-byte packets mapped to queue 1. Each packet carries a sequence counter at the **start** of the payload, so the counter survives trimming.
4. Wait 10s for the queue to fill, so that every packet is trimmed.
5. Start capturing the trimmed packets on the PTF host.
6. Run `warm-reboot` and wait until it finishes, including the warmboot finalizer.
7. Verify no trim packet lose during the warm reboot.
8. Release the queue and check that trimming still works after the reboot.

**What happens with FW xx.2020.2918 and later:**
1. Warm reboot starts. Before saving its state, the SDK runs a **resource-release fence**.
2. The fence waits for queue 1 to drain. Queue 1 is blocked and aging is disabled, so it **never drains**. A hardware register sample "in progress" for minutes while queue occupancy stays the same.
3. The syncd's counter-polling threads time out. The syncd PRE-SHUTDOWN watchdog fires after about 30 s.
4. About 60s after the pre-shutdown request, the warm-reboot script logs `Syncd pre-shutdown failed`. **It still continues and kexecs into the new image**, with incomplete ISSU state.
5. In the new image `issu_init` fails, syncd cannot create the switch, and orchagent exits. Critical services never come up, and the test fails.

**Reproduced on:**
- SN5640 (SPC5, ISSUv1): r-bison-04, r-bison-16, r-bison-22
- SN6600_LD (SPC6, ISSUv2): slm-180

## 4. Why warm reboot pass in previous and fails now
Before a warm reboot, the SDK runs a **fence**: it waits until the packets already inside the switch have moved on, so that it can safely release hardware resources.

- **Before (FW 37.2020.1136 and earlier):** part of the fence (the DQA/DQL stages) had been missing by mistake since Nov 2024. These stages sit in front of the queue shaper, so the fence did not wait for the blocked queue. The fence completed, and the warm reboot passed.
- **Now (FW 37.2020.2918 and later):** FW found the gap and restored these stages for SPC4–6. The fence now waits for the blocked queue. That queue never drains (shaper rate 0, aging disabled), so the fence never completes and the warm reboot fails.


## 5. Impact on us
We cannot verify "no trimmed-packet loss during warm reboot" with PTF. With PTF the only way to get trimming is a queue that cannot drain, and the corrected fence does not allow that during warm reboot.


## 6. References
- RM 5301723 (https://redmine.nvidia.com/issues/5301723)
- sonic-net/sonic-mgmt#27268
- pelican CR 1499267
- SDK commit 7bb8a06d12
- Email thread: "[RM 5301723] FW 2918 fence change breaks warm reboot when an egress queue cannot drain: product bug or unsupported config?"