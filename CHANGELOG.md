# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## Unreleased

`host`: the Host as one value.  `Host` holds the 29 fields of the Bluetooth Test Protocol session of `orbit/ble` that are Host state and not the state of a test service (the link registry and active `LeLink`, the parked per-connection state, the Security Manager halves, the bonds, the pairing settings, the GATT client table, the queue of commands owed to the Controller and the rest), `host_blank(host_config(slots, name))` makes one, and `host_activate` switches the connection the per-connection fields describe.  Two Hosts in one program share no table.  `SecuritySettings`, `LinkSession` and the Appearance, Peripheral Preferred Connection Parameters and bond-capacity constants moved here from `orbit/ble/src/btp/btp_session.nv` with no change.

`cs_gap`: the Channel Sounding GAP procedures of Core Vol 3 Part C, Sections 9.7 and 10.11.1, as an engine that a program drives with `cs_gap_next`, `cs_gap_event`, `cs_gap_tick` and `cs_gap_take_result`: the capabilities of both Controllers, the default settings, the security start, the remote FAE table, the configuration, the procedure parameters and Procedure Enable, then the result events of each procedure joined into one `HciCsProcResult`.  Either ACL role and either CS role; the security levels 1 to 4 choose the configuration and the requirements.

## 0.1.0

First release.  The Host core of the noble Bluetooth stack, extracted from
`orbit/ble`: the HCI session and transports, the link table, L2CAP and the
credit based channels, ATT, the GATT server and client, GAP and the
advertisers, the Security Manager and bonding, privacy, and the isochronous
transport.  The HCI wire is read through `noble-hci`.

The 38 keys of the `orbit/ble` configuration that size the Host core are the
schema of this package, under the section `[noble]`; the constants are read as
`noble_config.MAX_*`.  Twelve keys of the audio roles and the services stay in
this schema because their cost formulas multiply a key of the core.

The public surface is what an extension module, a chip image and a PC Host call:
the ATT and GATT server and client, the links and the HCI session, the credit
based channels, the Security Manager, the advertisers, the isochronous tables,
and the encodings and builders the services share.  The unit tests are the
suites `tests/<module>_tests.nv`, which run under the smallest and the largest
configuration and under each file of `configs/`.
