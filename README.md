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
| `host` | the Host as one value (`Host`, made by `host_blank` from a `HostConfig`): the links of both roles, the GATT database and client table, both Security Manager halves, the bonds, the pairing settings and the queue of HCI commands owed to the Controller; two Hosts in one program share no table.  The BTP session of `orbit/ble` holds a `Host` beside the states of its test services |
| `noble`, `host_api` | the Host API, Layer A: the values an application holds (`Status`, `ConnId`, `Event`, `Request`, `ConnInfo`), the policy trait `HostPolicy` and the bond store `BondStore` with `RamBonds`, and the calls that drive a `Host` without waiting (`begin`, `poll`, `result`, `next_event`) |
| `links`, `le_link` | the registry of connection slots, the credit pool of the Controller's ACL buffers, and one LE link: its HCI events, its ACL data, and the dispatch of its L2CAP PDUs to ATT, SMP and the credit based channels |
| `hci_session`, `hci_deframer`, `hci_transport`, `hci_uart`, `hci_usb_cdc`, `hci_init` | the pairing of a command with its answer, the H4 byte stream cut into packets, the `HciTransport` trait and its UART and USB CDC-ACM transports, and the builders of the start-up commands |
| `att`, `att_server`, `att_scratch`, `att_server_scratch` | the ATT PDU codec, the ATT server, and the fixed-RAM scratch buffer and response builders the device images use |
| `gatt_db`, `gatt_server`, `gatt`, `gatt_client` | the attribute database and the GAP and Generic Attribute services, the server that answers from it (permissions, Robust Caching, the prepare queue), the client procedures, and the client engine for several links in fixed storage |
| `l2cap`, `l2cap_coc` | the L2CAP PDUs of an LE-U link, and the LE and enhanced credit based channels |
| `gap`, `advertiser`, `extended_advertiser`, `periodic_sync`, `private_address` | the GAP roles and AD data, the bring-up and legacy advertising, extended and periodic advertising, periodic advertising synchronization, and the resolvable and non-resolvable private addresses |
| `smp`, `smp_core`, `smp_initiator`, `smp_responder`, `smp_fsm`, `smp_handler`, `smp_ctx`, `smp_keys`, `smp_bond`, `encrypted_data` | the Security Manager of both roles with LE Secure Connections and legacy pairing, the keys in wire byte order, the bond table, and Encrypted Advertising Data |
| `iso_host`, `iso_cis`, `iso_big`, `iso_flow`, `iso_event`, `iso_setup`, `isoal`, `isoal_timing` | the Host's connected and broadcast isochronous tables, the credits of the HCI ISO buffers, the bring-up, and the Isochronous Adaptation Layer |
| `cs_gap` | the GAP procedures of Channel Sounding ([Vol 3] Part C, Sections 9.7 and 10.11) as a plan engine for a Host on either ACL role and in either CS role: the commands from the capabilities to Procedure Enable, the events between them, the security levels 1 to 4, and the results of each procedure joined into one `HciCsProcResult` |
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

## Layer A: driving a Host without waiting

An application that does not speak the Bluetooth Test Protocol holds a `Host` and calls four functions on it.
`begin(var h, req)` takes one request (`Request.Connect`, `Disconnect`, `UpdateParams`, `Secure`, `CocOpen`, `CocClose`,
and `Start` to bring the Controller up) and answers at once whether it was accepted: `E_NOT_READY` before `Start` ended,
`E_BUSY` while another call is in flight, `E_STALE` for a connection that is gone, `E_NO_ROOM` at `max_links`, and
`E_BAD_ARGUMENT` for an argument the specification refuses or a request that this layer does not serve.
`poll(var h, var t, var p, var b, now_ms)` runs the Host once: it carries out the call, reads what the Controller sent
through the transport `t` (an `HciLink`), advances the timers to `now_ms`, calls the hooks of the policy `p` (a `HostPolicy`)
and writes what is owed.  `result(var h)` answers the outcome of the call once it has ended, and `next_event(var h)` the next
`Event` the application did not ask about.  Nothing in it waits, reads a clock or sleeps.

A failure is a `Status`: the layer whose number it carries (`LAYER_HCI`, `LAYER_ATT`, `LAYER_SMP`, `LAYER_L2CAP` or
`LAYER_LOCAL`) and that layer's code, with the names the specification gives (`ATT_INSUFFICIENT_AUTHENTICATION`,
`HCI_CONNECTION_TIMEOUT`, `SMP_PASSKEY_ENTRY_FAILED`) and the Host's own (`E_BUSY`, `E_STALE`, `E_REFUSED`).  `Status.name()`
gives the text for a log line.  A connection is a `ConnId`, a slot and the generation of that slot, so a name kept after the
link ended is refused with `E_STALE` and never acts on the next device that took the slot.

The decisions the specification leaves to the product are the methods of `HostPolicy`, each with a default: whether to keep
a link (`accept_connect`), whether to accept a pairing (`accept_pairing`), the passkey and the numeric comparison
(`ask_passkey`, `confirm_numeric`, which answer `PENDING` to be asked through `Event.PasskeyRequested` and
`NumericRequested` and answered with `passkey_reply` and `numeric_reply`), whether to accept connection parameters
(`accept_params`), and what to do after a link was made or ended (`on_connected`, `on_disconnected`, which answer an
`Action`: `Stay`, `Readvertise`, `AdvertiseNonConnectable` or `Reconnect(after_ms)`).  The keys are the product's too: a
`BondStore` holds them, and `RamBonds` is one that keeps `MAX_BONDS` bonds in RAM.

The queue holds `EVENT_QUEUE` events.  An `Event.Advert` or `Event.Notification` that does not fit is dropped and counted in
`Event.Dropped`; the events of a connection's state (`Connected`, `Disconnected`, `ParamsUpdated`, `SecurityChanged` and the
two pairing requests) are never dropped, because the queue keeps a slot for each connection.  `event_queue` is 0 in the
default configuration, so a Host that does not use the API pays for no queue; a product that does writes `event_queue = 4` in
its `[noble]` section.

```novo
use host
use host_api
use noble

// Connect to `peer` and poll the Host until the call ends or 100 turns have passed.  A product that
// owns its loop does the same between its other work; the clock is the argument of `poll`.
fn connect_to<T: HciLink[e]>(var h: host.Host, var t: T, var p: noble.DefaultPolicy, var b: noble.RamBonds, peer: noble.Addr) -> noble.Status [e, mutate]
    let started = host_api.begin(var h, noble.Request.Connect(peer, noble.conn_params_default()))
    if not started.is_ok()
        return started
    for turn in 0..100
        host_api.poll(var h, var t, var p, var b, turn * 10)
        match host_api.result(var h)
            Some(done) => return done
            None => ()
    noble.failed(noble.LAYER_LOCAL, noble.E_TIMEOUT)
```

This layer serves `Start`, `Connect`, `ConnectListed`, `ConnectCancel`, `Disconnect`, `UpdateParams`, `Secure`, `CocOpen` and
`CocClose`, the replies `passkey_reply`, `numeric_reply`, `unbond` and `coc_send`, and `coc_listen`.  Advertising, scanning,
synchronization, the accept list, the GATT client and server calls, the `Adapter` that waits, and the service modules are NOT
IMPLEMENTED here; `begin` answers their requests `E_BAD_ARGUMENT`.  The `Host` still carries the Bluetooth Test Protocol
session of `orbit/ble`, which does not use these calls.

The sizes of the tables are keys of the package configuration.  A consumer
writes a `[noble]` section in its config file (`max_connections`,
`max_att_mtu`, `max_attributes`, `max_bonds`, `max_cis`, `max_big` and the
others); the build writes them as `noble_config.MAX_CONNECTIONS` and the like,
and `novo config explain` prints the storage the keys imply.  `configs/` holds
the profiles this package's tests run under; `configs/layer_a.toml` gives the Host API tests their event queue.

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
`host_api_tests.nv` drives a Host through `begin`, `poll` and `next_event` over a scripted Controller, and two Hosts through a
pairing; its cases that need an event queue run with `--config configs/layer_a.toml`.
`novo test tests/<module>_tests.nv` runs one of them.  A suite takes a profile with `--config configs/pc.toml`;
`novo config ci` runs every suite under the smallest and the largest configuration the schema allows and under each file
of `configs/`.  The tests are not part of the published package.
