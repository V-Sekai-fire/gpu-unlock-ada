# bhyve passthrough

This note covers an RTX 4090 (`10de:2684`) passed through to a Linux guest by FreeBSD bhyve.
It was tested on one card, on FreeBSD 15.1 with an Ubuntu 26.04 guest (kernel 7.0.0-38),
NVIDIA 595.99.02 open modules and a 275 W power limit, on 2026-10-26.

## The problem

The module ends its transaction with a PCIe Function Level Reset from inside the guest. Under
bhyve the card's BARs read `0xffffffff` after that reset. `wait_post_flr_bar` times out, the
unlock fails with `-110`, and the module keeps BAR0, so NVIDIA cannot bind the card.

## The workaround

`gpu_unlock.defer_flr=1` stops the transaction just before the in-guest FLR. The selectors are
already written at that point. The hypervisor then performs the reset when the VM shuts down,
and the selectors survive it.

1. Boot the guest with `gpu_unlock.targets=<address> gpu_unlock.defer_flr=1` on a card that
   has been through a power cycle. The log shows `FLR deferred to the hypervisor` with
   `ss0=0x88888888 ss1=0x8`. NVIDIA cannot bind in this boot.
2. Shut the VM down. bhyve exits and resets the passthrough device.
3. Start the VM without the `gpu_unlock.*` options. NVIDIA binds normally, and the selectors
   still read `0x88888888` and `0x8`.

A warm reboot of the host clears the selectors, so the sequence has to run after every host boot.

## Measured

Same card, cubin, power limit, driver and launch settings in both states:

| Mode | Stock | Unlocked |
|---|---|---|
| FP16/FP32 `mma.sync` | 154.0 Top/s | 306.5 Top/s |
| TF32 | 80.2 Top/s | 153.4 Top/s |
| INT4 (control) | 1275.1 Top/s | 1270.8 Top/s |

All 270,336 checked cells matched in every mode. Decode and prefill of a ternary 27B LLM
(ninfer) did not change between the two states.
