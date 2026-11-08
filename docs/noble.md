# Modules, configuration and the HCI seam

`noble` is the Host of the noble stack: the layers between an HCI transport and
an application, with no register and no Controller code.

The application API, a first Host program against a Controller, and the porting of the Controller to
another chip are described in the noble documentation set on novo-lang.org:
[noble](https://novo-lang.org/docs/noble/README.html).

## What it reads

| Package | What the Host takes from it |
|---|---|
| `noble-hci` | the opcode and event constants, the frame types, the H4 codec, `BdAddr` and `Uuid` |
| `crypto-ble-ecc` | the software AES-128, AES-CMAC and P-256 the Security Manager and the Database Hash use, and the resolvable private address functions |

The Host imports nothing else of the stack.  It does not import `noble-controller`
or `noble-radio`, and `tests/orbit/test_ble_core_imports_no_extension.sh` and
`orbit/ble/tests/seam_imports.sh` in the repository hold the rule: no module of
this package imports a module of the Controller or of `orbit/ble`.

## The layers

1. **Transport and session.**  `hci_transport` declares the `HciTransport`
   trait: one HCI packet layer over a per-transport framing.  `hci_uart` and
   `hci_usb_cdc` implement it over a UART and a USB CDC-ACM pipe, and
   `hci_deframer` cuts an H4 byte stream into packets.  `hci_session` pairs a
   command with the Command Complete or Command Status event that answers it,
   and `hci_init` and `advertiser` build and send the start-up commands.
2. **Links.**  `links` holds the registry of connection slots, each named by a
   `LinkId` (a slot and its generation), and the credit pool of the Controller's
   ACL buffers.  `le_link` is one connection: it takes HCI events and ACL data
   packets, reassembles L2CAP PDUs and hands each one to its channel.
3. **Channels.**  `l2cap` codes the PDUs of an LE-U link, and `l2cap_coc` holds
   the LE and enhanced credit based channels.  ATT rides the fixed channel 4
   and the Security Manager the fixed channel 6.
4. **ATT and GATT.**  `att` codes the ATT PDUs.  `gatt_db` and `gatt_server`
   hold the attribute database and the server that answers from it (permissions,
   Robust Caching, the prepare queue, one `AttLink` for each connection),
   `att_server` is the ATT server over a flat handle table and
   `att_server_scratch` writes its responses into the fixed-RAM scratch of
   `att_scratch`, `gatt` holds the client procedures, and `gatt_client` is the
   client engine for several links in fixed storage.
5. **GAP.**  `gap`, `advertiser`, `extended_advertiser`, `periodic_sync` and
   `private_address` hold the roles, the advertising of every kind, the
   synchronization to a periodic train and the private addresses.
6. **Security.**  `smp_initiator` and `smp_responder` run the two sides of a
   pairing over `smp_core`, `smp` and `smp_keys`, `smp_bond` keeps the bonds, and
   `encrypted_data` is Encrypted Advertising Data.  `smp_fsm`, `smp_handler` and
   `smp_ctx` are the Just Works state machine of the fixed-RAM image.
7. **Channel Sounding.**  `cs_gap` is the Host's side of the GAP procedures of
   Core Vol 3 Part C, Sections 9.7 and 10.11: a state engine in the shape of the
   plans of `orbit/ble/src/btp/btp_hci.nv`.  The program asks `cs_gap_next` for the
   next command, sends it, feeds each HCI event to `cs_gap_event` and calls
   `cs_gap_tick` with its clock.  It reads and writes the records of `noble-hci`
   (`hci_cs`) and performs no I/O.
8. **Isochronous transport.**  `iso_cis` and `iso_big` hold the connected and
   broadcast isochronous tables, `iso_host` routes an HCI event to them, `iso_flow`
   counts the Controller's ISO buffers, `iso_setup` brings the feature up, and
   `isoal` and `isoal_timing` are the Isochronous Adaptation Layer.
9. **The Host.**  `host` holds the state of one Host in one value: `Host`, made
   by `host_blank(host_config(slots, name))`.  It owns the link registry and
   the `LeLink` of the active connection with the state parked for the others,
   the GATT database each link serves, the client table, both Security Manager
   halves, the bonds, the pairing settings and the queue of HCI commands the
   Host owes the Controller.  A front end holds a `Host` beside its own state:
   the BTP session of `orbit/ble` does.  The four fixed-RAM buffers of
   `att_scratch` and `smp_ctx` (module-level statics) belong to the co-resident
   image, which has no `Host` value; no `Host` function reaches them.
10. **The Host API, Layer A.**  `noble` holds the values an application sees
   (`Status` with the named codes of ATT, HCI and SMP and the Host's own, `ConnId`,
   `Addr`, `Security`, `ConnParams`, `ConnInfo`, `AdvReport`, `Event`, `Request`), the
   trait `HostPolicy` with a default for each decision, the trait `BondStore` with
   `Bond` and `RamBonds`, and `Api`, the fixed storage of the layer (the event queue,
   the connection table, the call in flight) that a `Host` carries.  `host_api` holds
   the engine: `begin` takes a request, `poll` reads the transport, the Security
   Manager, the links and the channels and calls the hooks, `result` and
   `next_event` hand the outcome and the events to the application.  The
   adapters `from_client_status`, `from_client_error`, `from_advertiser_error`,
   `from_smp_reason` and `from_profile_error` read the older error forms of the
   package as a `Status`.  `hci_transport` gives `HciLink` the clock (`now_ms`) and
   the wait (`park`), both with a default; the layer itself calls neither.  The Host
   API serves the requests of connecting, disconnecting, updating the parameters,
   securing a link, the credit based channels, advertising, scanning, synchronisation
   to a periodic train, the accept list and the GATT client, and declares the GATT
   server (`Host.add_service`, `add_char`, `bind_function`, `bind_event`).
   **Layer B** is in `noble` too: the `Adapter` (`adapter(link, policy, bonds)`) that
   holds a `Host`, a transport, a policy and a bond store and whose methods wait
   (`step`, `run`, `start`, `wait_event`, `connect`, `adv_start`, `scan_start`, `find`,
   `discover`, `read`, `write`, `subscribe`, `notify`, `respond`), the trait `Service`
   with `plug` (the product's list of services), `Out`, `Outbox`, `pump` and `serve`, the payload builders (`ad_flags`,
   `ad_uuid16`, `ad_name`) and `UartHci`.  `gatt_server` holds the requests for the
   attributes bound to the application (`GattDb.bound`, `AttLink.held`).  NOT
   IMPLEMENTED in it: `HostPolicy.authorize_write`, bound attributes on an Enhanced
   ATT bearer, extended scan reports and periodic advertising reports.

`profile_builder`, `notify`, `wire` and `control_common` are what the modules
that add a service to a `GattServer` share.  The audio roles are the package
`noble-audio` and the generic services are the package `noble-services`; both
import these modules by name, and `fixtures`, the helpers their tests share, is a
module of this package.

## Configuration

The schema is the `[config.<key>]` tables of `novo.toml`: 41 keys (`max_connections`,
`max_att_mtu`, `acl_octets`, `acl_buffers`, `max_attributes`, `max_value_bytes`,
`max_uuid128`, `max_cccd`, `max_bearers`, `max_coc_channels`, `max_bonds`,
`max_pairing`, `max_cis`, `max_big`, `max_bis`, `iso_sdu_octets`, `iso_buffers`,
`sequenced_slots`, `hci_buffer_octets`, `hci_rx_flow_control` and the GATT client
engine's `max_client_links`, `client_bearers`, `client_cache_records`,
`client_queue` and `client_value_bytes`) and the six of the Host API (`event_queue`, the events the queue holds, 0 in the
default so that a Host that does not use the API pays for none; `att_value_bytes`, the octets of one value or SDU a call or an
event carries; `max_adv_data`, the octets of the advertising data of a report; `max_name`, the octets of a device name `ad_name` writes, and `outbox`, the commands a service queues in one turn of `pump`, both values on the caller's stack and no storage of the Host; and `max_services`, the services a product's list holds, which sizes the owner table of the Adapter, 48 B at the default of 4).  The ranges are the bounds of the
Core Specification where it states one, and a relation between keys is a field
of the schema, which `novo config check` refuses a file to break.

Twelve more keys size the storage of the audio roles and the services
(`max_ases`, `max_aics`, `max_vocs`, `max_presets`, `max_set_size`,
`max_unicast_streams`, `max_bcast_sources`, `max_bcast_sinks`, `max_gaf_clients`,
`max_ctl_instances`, `max_has_cache` and `max_bas`).  They are keys of this schema
because a cost formula reads only the keys of its own schema, and each of them
multiplies a key of the core: the ASE rows of every connection, the Broadcast
Source rows over the BIS count, the Unicast streams over the SDU octets.  They
belong to `noble-audio` and `noble-services` by what they size, and move there when
a formula can read the keys of another package
(`features/tooling/config-keys-of-a-dependency-in-a-cost-formula.md`).

A key that is `0` switches its feature off, and the table of the feature keeps
one slot.  `limits` derives the values the keys do not say by themselves, and
`budget` adds the storage of the tables up and holds the sum to the total
`novo config explain` prints.  The RAM the board leaves the Host
(`host_budget_bytes`) is a key of `orbit/ble`, which adds the storage of its
own modules and the sums of `noble-audio` and `noble-services` to the sum of
this package.

`configs/<profile>.toml` are the profiles the package's tests run under.  The
same names exist in `orbit/ble/configs/`, whose `[noble]` sections equal these;
the script `tests/orbit/test_ble_config_equals_limits.sh` checks the two.  `layer_a.toml` has no
counterpart there: it gives the tests of the Host API their event queue (`event_queue = 4`).

## The public surface

An extension module, a chip image and a PC Host name 600 items of the package;
they are `pub`, each with a doc comment, and every other item is private.
`novo doc` prints them.

| Layer | Modules and public items |
|---|---|
| ATT and GATT | `gatt_server` (89), `gatt_client` (78), `att` (36), `att_scratch` (11), `gatt_db` (8), `att_server` (3), `att_server_scratch` (2) |
| Host | `host` (10) |
| Host API | `noble` (about 120 values, constants and functions), `host_api` (26) |
| Links and HCI | `le_link` (38), `links` (16), `hci_session` (4), `hci_init` (3), `hci_deframer` (1) |
| Channels | `l2cap_coc` (17), `l2cap` (7) |
| Security | `smp` (26), `smp_ctx` (23), `smp_keys` (18), `smp_bond` (7), `smp_initiator` (7), `smp_responder` (6), `smp_fsm` (4), `smp_core` (2), `smp_handler` (1) |
| GAP | `advertiser` (16), `extended_advertiser` (15), `private_address` (7), `periodic_sync` (4), `gap` (3) |
| Isochronous | `iso_cis` (17), `iso_host` (14), `iso_big` (13), `iso_setup` (10), `iso_flow` (8), `iso_event` (6), `isoal` (4) |
| Shared with the services | `wire` (39), `profile_builder` (9), `control_common` (7), `encrypted_data` (5), `notify` (1), `limits` (1), `fixtures` (5) |

The configuration module `noble_config` is generated from the keys of
`novo.toml`; a consumer reads its constants and sets them in its own config file.

## Tests

The unit tests of a module are the suite `tests/<module>_tests.nv`, which
`novo test tests/<module>_tests.nv` runs; `src/` holds none.  A suite sees the
private names of the package, so it names a module's helpers directly.  A
fixture that the suites of two modules share stays in `src/` under a heading
that says so, because a suite cannot import another suite.  Three suites drive
several modules together: `gatt_client_engine_tests` runs each client procedure
against the GATT server, `integration_gatt_tests` walks a discovery and a read
from the wire up, and `two_centrals_tests` serves two connections from one
database.  `att_server_scratch_tests` holds the fixed-RAM ATT paths to the
answers of `att_server.handle_request`.

A test that reads a limit computes the figure it expects from the configuration
key (`limits.frame_octets()`, `noble_config.MAX_ATT_MTU`) and not from the
default.  The publish runs every suite under the smallest and the largest
configuration the schema allows and under each file of `configs/` (`novo config
ci`), so a test whose subject is absent under a configuration carries `@when` on
the key it needs: the client engine suite on the client limits, the isochronous
suites on `MAX_CIS`, `MAX_BIG`, `MAX_BIS` and `ISO_SDU_OCTETS`.  Every suite
keeps at least one test that runs under every configuration.  The tooling takes
a `@when` expression of the keys themselves; a `const fn` in the test file that
reads the configuration fails with E2922, so the expression is written out at
each test.
