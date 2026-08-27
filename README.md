# RACK Retransmission PRR Gate

## Problem

`rack_output()` can select a retransmission from several sources and send it without checking the current PRR send budget. The existing PRR limit applies to new-data output, but an already-selected retransmission can bypass that limit.

The relevant pre-existing paths are:

* `rc_resend`: [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L19964-L19980)
* collapsed recovery: [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133/sys/netinet/tcp_stacks/rack.c#L20028-L20047)
* ordinary RACK selection through `tcp_rack_output()`: [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L19997-L20010)
* fast RSM transmission: [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L20147-L20155)

The issue is not RSM selection itself. The issue is that the selected RSM reaches transmission even when `rc_prr_sndcnt` is zero or smaller than the selected segment.

## Invariant

For an ordinary retransmission during fast recovery with PRR enabled:

```text
rc_prr_sndcnt >= bytes transmitted
```

If the budget is insufficient, the retransmission must not be sent and control flow must not fall through to new-data transmission.

Timer retransmissions and TLP probes are separate mechanisms and must not be blocked by this gate.

## Proposed Change

`rack_output()` now tracks whether the selected RSM came from the ordinary `tcp_rack_output()` branch with `ordinary_rack_rxt`, declared near [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L19639) and set near [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L19588).

A single gate runs before FIN handling and before fast or normal RSM transmission at [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L20030-L20035):

```c
if (ordinary_rack_rxt && IN_FASTRECOVERY(tp->t_flags) &&
    (rack->rack_no_prr == 0) &&
    (rack->r_ctl.rc_prr_sndcnt < min(len, segsiz))) {
    goto skip_all_send;
}
```

The gate uses `skip_all_send` rather than clearing `rsm`. Clearing only `rsm` could allow the function to continue into the new-data path and transmit for a different reason.

`tcp_rack_output()` remains a selector. The PRR decision is made at the output boundary, where the selected RSM and final send length are available.

## Exclusions

* Timer resend is selected at [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L20015).
* TLP selection follows the collapsed-recovery branch at approximately [rack.c](https://github.com/freebsd/freebsd-src/blob/d1057074b277443e11b04e8513acdf43fd133123/sys/netinet/tcp_stacks/rack.c#L20077).
* The gate is currently marked only for the direct `tcp_rack_output()` source. `rc_resend` has multiple producers, including timer and recovery paths, so it cannot be classified solely by its field name.

## Diagnostics

The source-aware diagnostics report:

```text
RACK retransmission selected: source=...
RACK PRR gate: source=... suppressing ...
RACK fast output: source=...
```

The source values are `rc_resend`, `collapsed`, and `tcp_rack_output`. These logs show whether an RSM was selected, suppressed, or transmitted, and expose any path that bypasses the gate.

[test.pkt](https://github.com/NikhilKottoli/PRR_WritUp/blob/main/test.pkt) was the packetdrill test used
