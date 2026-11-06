# noble

The Host of the noble Bluetooth Low Energy stack, written to Bluetooth Core
Specification 6.3.  The package holds the layers a Host needs between an HCI
transport and an application: the link table, L2CAP and the credit based
channels, ATT, the GATT server and client, GAP and the advertisers, the
Security Manager with bonding and privacy, and the isochronous transport.  It
talks to a Controller only through the HCI seam of `noble-hci`, so any
Controller that speaks HCI works under it: the Link Layer of
`noble-controller`, a vendor Controller over UART or USB, or the in-RAM link
of a one-chip image.

The modules hold state in fixed tables whose sizes are keys of the package
configuration, and the embedded tier marks the Host's functions.  The Host
runs on a device (the nRF52 and nRF54 images of `orbit/ble`) and on a PC
(the AutoPTS harness and the PC Host of the demonstration).  The roles above
the core (the LE Audio roles and the GATT services) are not in this package;
they import its modules by name.

| Module | Holds |
|---|---|
| `links`, `le_link` | the registry of connection slots, the credit pool of the Controller's ACL buffers, and one LE link: its HCI events, its ACL data, and the dispatch of its L2CAP PDUs to ATT, SMP and the credit based channels |
| `hci_session`, `hci_deframer`, `hci_transport`, `hci_uart`, `hci_usb_cdc`, `hci_init` | the pairing of a command with its answer, the H4 byte stream cut into packets, the `HciTransport` trait and its UART and USB CDC-ACM transports, and the builders of the start-up commands |
| `att`, `att_server`, `att_scratch`, `att_server_scratch` | the ATT PDU codec, the ATT server, and the fixed-RAM scratch buffer and response builders the device images use |
| `gatt_db`, `gatt_server`, `gatt`, `gatt_client` | the attribute database and the GAP and Generic Attribute services, the server that answers from it (permissions, Robust Caching, the prepare queue), the client procedures, and the client engine for several links in fixed storage |
| `l2cap`, `l2cap_coc` | the L2CAP PDUs of an LE-U link, and the LE and enhanced credit based channels |
| `gap`, `advertiser`, `extended_advertiser`, `periodic_sync`, `private_address` | the GAP roles and AD data, the bring-up and legacy advertising, extended and periodic advertising, periodic advertising synchronization, and the resolvable and non-resolvable private addresses |
| `smp`, `smp_core`, `smp_initiator`, `smp_responder`, `smp_fsm`, `smp_handler`, `smp_ctx`, `smp_keys`, `smp_bond`, `encrypted_data` | the Security Manager of both roles with LE Secure Connections and legacy pairing, the keys in wire byte order, the bond table, and Encrypted Advertising Data |
| `iso_host`, `iso_cis`, `iso_big`, `iso_flow`, `iso_event`, `iso_setup`, `isoal`, `isoal_timing` | the Host's connected and broadcast isochronous tables, the credits of the HCI ISO buffers, the bring-up, and the Isochronous Adaptation Layer |
| `profile_builder`, `notify`, `wire`, `control_common`, `fixtures` | what a service adds to a `GattServer` (the steps that add its characteristics, the notifications a change owes the clients, the integer encodings, and the steps the control services share), and the helpers the tests of the services share (an ATT request, the first connection, a descriptor write and a read) |
| `limits`, `budget` | the values the configuration derives, and the RAM the configured tables add up to, held to the total `novo config explain` prints |

A program creates a `GattServer`, adds the services it offers, opens a link in
`links` when the Controller reports a connection, and hands each ATT request it
receives to `gatt_server.receive`, which answers the PDUs to send back:

```novo
use gatt_server

// A server with the GAP service answers an ATT Exchange MTU Request with one PDU.
fn mtu_request_is_answered() -> Bool
    var s = gatt_server.host_server()
    let out = gatt_server.receive(var s.db, var s.link, [0x02, 0xF7, 0x00], 0)
    list.len(out) == 1
```

The sizes of the tables are keys of the package configuration.  A consumer
writes a `[noble]` section in its config file (`max_connections`,
`max_att_mtu`, `max_attributes`, `max_bonds`, `max_cis`, `max_big` and the
others); the build writes them as `noble_config.MAX_CONNECTIONS` and the like,
and `novo config explain` prints the storage the keys imply.  `configs/` holds
the profiles this package's tests run under.

The audio roles and the services, `orbit/ble`, and a chip package depend on
this package.  It depends on `noble-hci` and `crypto-ble-ecc` only: no module
imports `noble-controller`, so an image that holds a Host and no Controller
carries no Link Layer code.  `docs/noble.md` describes the modules, the
configuration and the seams.

## Install, build and test

```sh
novo pkg add noble
```

adds the package to the `[dependencies]` table of the consumer's `novo.toml`.  In a checkout of the package:

```sh
novo build
novo test
```

`novo build` type-checks and effect-checks every module under `src/`; the package is a library and writes no binary.
`novo test` runs the suites under `tests/`: one suite for each module, `tests/<module>_tests.nv`, and three that drive
several modules together (`gatt_client_engine_tests.nv`, `integration_gatt_tests.nv` and `two_centrals_tests.nv`).
`novo test tests/<module>_tests.nv` runs one of them.  A suite takes a profile with `--config configs/pc.toml`;
`novo config ci` runs every suite under the smallest and the largest configuration the schema allows and under each file
of `configs/`.  The tests are not part of the published package.
