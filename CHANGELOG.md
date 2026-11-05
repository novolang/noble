# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
