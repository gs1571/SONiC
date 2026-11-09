# IS-IS integration test plan

## Scope and prerequisites

Validate the ConfigDB → frrcfgd → FRR path, including interface VRF changes,
startup rendering and the northbound CLI. The unit tests complement these cases;
they do not replace running this plan on a SONiC image with both implementation PRs.

- Install the matching buildimage and utilities changes. Enable
  `DEVICE_METADATA|localhost.frr_mgmt_framework_config=true`, restart the affected
  bgp container, and confirm isisd and frrcfgd are running.
- Prepare `default`, `VrfA` and `VrfB`, unique valid NETs and existing routed
  interfaces. Use the existing SONiC interface VRF workflow to change ownership;
  changing only the IS-IS table does not exercise the routed-interface event path.
- Exercise Ethernet, PortChannel, VLAN, loopback and VLAN subinterfaces where
  supported. Member ports must not be used in place of routed aggregates.
- For forwarding validation, use the Force10 seven-node topology with passive
  loopbacks and record the relevant peer, interface, VRF and IPv4/IPv6 prefixes.
- For multi-ASIC devices, use the same namespace for global/interface commands
  and observations. Confirm the other ASIC's configuration is unchanged.
- Capture ConfigDB, FRR running configuration, `show isis interface`, neighbors,
  route tables and frrcfgd logs before and after each transition. Inspect the
  affected VRF explicitly through the available FRR operational commands.

## Test setup and limits

The relevant configuration is CONFIG_DB (`/etc/sonic/config_db.json` when saved):
`DEVICE_METADATA`, `VRF`, the routed-interface tables and the three `ISIS_*` tables
specified in the [HLD](IS-IS-CLI-Configuration.md). Use `config isis`,
`config interface isis` and `show isis` as documented there, plus the existing
interface/VRF commands. Retain the command transcript and before/after snapshots.
No new SAI API is introduced; forwarding uses existing route/next-hop programming.

This plan validates functionality and convergence observations on the stated
seven-node topology. It does not establish scale limits or performance targets;
large-scale route/adjacency and stress qualification require a separate plan.
Failure injection in B9/B10 must be reproducible and logged, including which FRR
commands succeeded before the injected failure.

## Binding transition matrix

| Case | Action | Required result |
| --- | --- | --- |
| B1: enable-only | Enable an existing process, then an interface with only `enabled=true` | Binding reaches FRR immediately, without subsequently setting metric or circuit type. |
| B2: A → B | Configure metric 30, passive, p2p and IPv6 on an interface in VrfA; move it to VrfB | Both old IPv4 and IPv6 bindings are removed before new bindings are applied. ConfigDB IS-IS attributes remain intact and are applied to VrfB. No VrfA advertisement remains after convergence. |
| B3: default ↔ VRF | Move default → VrfA → default | Each direction removes the actually applied instance; default is not treated as an absent VRF. |
| B4: missing target process | Move from VrfA to VrfB with no enabled IS-IS process in VrfB | Old bindings are removed and no target binding is created. Enabling the VrfB process later reconciles the retained interface row without requiring another interface edit. |
| B5: delete ordering | After A → B, remove the routed-interface row and then the IS-IS row; repeat in the opposite order | The applied B binding is removed, and neither order recreates a default binding. Recreate the routed-interface row and enable IS-IS to verify recovery. |
| B6: address-only events | Add/remove IPv4 and IPv6 prefix rows beneath an enabled interface | Prefix-key events do not change the cached base-interface VRF or detach the IS-IS binding. |
| B7: IPv6 disabled | Repeat A → B with `ipv6_enabled=false` | Old IPv4/IPv6 bindings are cleaned up; only the new IPv4 binding is enabled. |
| B8: parent disable/enable | With retained interface rows, set the parent `enabled=false`, then true | Disable removes bindings; re-enable reconciles them and restores retained timer values (also test refresh 100/lifetime 500 against the recreated process defaults). Separately verify that CLI process deletion refuses until interface rows are removed. |
| B9: failed batch | Inject a failed FRR command batch during A → B, including failure after partial application | The last successful binding is not falsely advanced. On the next relevant event, cleanup considers both possible A/B bindings before applying the target. After removing the fault, trigger a retry and verify convergence. No periodic retry is assumed. |
| B10: failure then delete | After the B9 partial failure, delete the interface configuration instead of retrying the move | Cleanup removes both possible bindings, including partially applied B; no binding is recreated. |
| B11: startup | Save a configuration in VrfA, restart bgp/frrcfgd, then move to B and delete | Startup seeds applied binding state from the restored configuration; the first post-startup move/delete still removes A correctly. Repeat template restore and supported command-replay restore modes. |
| B12: namespace isolation | Repeat B2/B4 on one ASIC namespace | Commands and routes affect only the selected ASIC; the other ASIC's binding and neighbors remain unchanged. |

Repeat B2–B7 for each supported routed-interface type. For dual-stack forwarding,
verify the passive loopback routes and traffic in the new VRF, and verify their
withdrawal from the old VRF. Record measured convergence time; do not treat a
successful Redis write or command exit status as sufficient evidence.

## Other CR regression cases

| Area | Cases and acceptance criteria |
| --- | --- |
| Global commands | Valid/nonexistent VRFs; duplicate enable preserves NET/level/hostname; hostname enable/disable never changes FRR's global hostname. |
| NET | Reject missing system-id and nonzero NSEL; replace a valid NET and confirm only the new NET remains. |
| Interface validation | Alias naming mode; nonexistent interface; PortChannel/VLAN member rejection; attributes before enable; global process prerequisite; removal of disabled/stale IS-IS rows after an interface is removed or becomes a member. |
| Timers | Accept MTU 128–4352, refresh 1–65235, lifetime 350–65535 subject to lifetime ≥ refresh + 300. Reject invalid ranges/combinations. Exercise increasing/decreasing pairs, field resets and whole-row deletion. |
| Defaults | Remove circuit type, metric and passive; compare running state with a cold render of the same ConfigDB. LAN, metric 10 and passive off must agree. |
| IPv6 mode | Test IPv6 MT and single-topology peers, and interface IPv6 disabled. Confirm actual adjacency and route installation rather than only configuration text. |
| Show commands | Summary, neighbor, database/detail and interface; text/JSON; namespace routing; unavailable daemon and nonzero command exits. |
| Model validation | Run the new IS-IS YANG positive/negative fixtures, full ConfigDB validation and supported GCU operations, including leafrefs and interface enabled. |
| Packaging/lifecycle | Confirm one packaged IS-IS template is included, and isisd follows the shared supervisord startup/stop policy. |

## Default-instance IS-IS/SRv6 persistence

Run the proposed [sonic-mgmt recovery test](https://github.com/sonic-net/sonic-mgmt/pull/29119)
on a single-ASIC T0 lab device with the persistence implementation, unified
FRR and template-based restore. The device must have `Loopback0` in the default
VRF, no existing IS-IS/SRv6 profile in CONFIG_DB or FRR, and an Established BGP
peer that accepts at least one prefix. This scenario is independent of the
seven-node binding tests. The test uses a passive interface and a static uN;
it validates control-plane configuration and BGP recovery, without asserting
IS-IS adjacency formation, ASIC programming or SRv6 packet forwarding.

Before applying the [CONFIG_DB example](IS-IS-CLI-Configuration.md#default-instance-is-issrv6-persistence):

1. Confirm that `frrcfgd` is running with
   `sudo docker exec bgp supervisorctl status frrcfgd`. Record the image version,
   the peer address, its accepted-prefix count and its BGP connection counters.
2. Back up the existing `/etc/sonic/config_db.json` and export live CONFIG_DB to a
   separate file on persistent storage. The live database can include unsaved changes.
3. Apply the example, confirm the FRR baseline below, then run
   `sudo config save -y`. Compare live and saved rows before triggering recovery.

For every case, inspect the live tables with
`sonic-cfggen -d --var-json TABLE`, where `TABLE` is each of the four table names
in the example. For the saved baseline, use
`sonic-cfggen -j /etc/sonic/config_db.json --var-json TABLE`. Check
`sudo docker exec bgp vtysh -c 'show running-config'` for:

- `router isis default` with the configured NET, level 2 and the SRv6
  locator/interface association.
- A passive `Loopback0` with the IPv6 IS-IS binding, no IPv4 IS-IS binding
  and metric 20 (30 after the offline update).
- The locator prefix `2001:db8:100::/48` in format `usid-f3216`, and static SID
  `2001:db8:100:a::/128` with behavior `uN` (absent after the offline delete).

| Case | Action | Required result |
| --- | --- | --- |
| P1: routing restart | Record the bgp container start time and run `sudo systemctl restart bgp`. | Container start time changes. CONFIG_DB and FRR match the baseline. The BGP peer is Established and accepts at least the baseline prefix count. |
| P2: configuration reload | From the recovered baseline, run `sudo config reload -y` once. | The saved profile returns to live CONFIG_DB and FRR. The peer is Established and accepts at least the baseline prefix count without manually restarting neighbors. |
| P3: cold boot | Record `/proc/sys/kernel/random/boot_id`, perform a cold reboot, then read it again. | The boot ID changes and the saved profile returns to live CONFIG_DB and FRR. Poll BGP every 5 seconds: require at least 30 seconds with the peer Established, at or above the baseline accepted-prefix count and unchanged connection counters. Reset the window on a failed sample or a counter change. |
| P4: daemon restart after offline changes | With the P3 baseline restored, record the bgpd PID and peer counters. Stop only frrcfgd, change the interface metric from 20 to 30, and delete the static SID row. Start frrcfgd again. | While frrcfgd is stopped, CONFIG_DB shows metric 30 and no SID row, while FRR still shows metric 20 and the SID. After startup, FRR shows metric 30 and no static SID; the locator and the remaining profile stay configured. The bgpd PID and both connection counters are unchanged, and the peer meets the baseline prefix count. |

For P4, use the following commands on the DUT. Confirm the stopped-state
observations before starting frrcfgd, and start it even if an observation fails.

```bash
sudo docker exec bgp supervisorctl pid bgpd
sudo docker exec bgp supervisorctl stop frrcfgd
sudo docker exec bgp supervisorctl status frrcfgd  # Expect STOPPED (exit 3)
sudo sonic-db-cli CONFIG_DB HSET 'ISIS_INTERFACE|Loopback0' metric 30
sudo sonic-db-cli CONFIG_DB DEL 'SRV6_MY_SIDS|ISIS_PERSIST_TEST|2001:db8:100:a::/128'
# Compare live CONFIG_DB with the FRR running configuration here.
sudo docker exec bgp supervisorctl start frrcfgd
```

The automated test uses a 300-second timeout for each convergence or stability
check. Record the FRR and BGP results separately; an Established session with no
accepted prefixes does not pass. On failure, collect frrcfgd and bgp container
logs, including any startup fallback message. Schema tests should also reject an
IS-IS SRv6 association with only one of `srv6_locator` and `srv6_interface` set.

After the run, reload the original live CONFIG_DB snapshot, verify the test
objects are absent from CONFIG_DB and FRR and BGP is back to baseline, then put
the original saved file back on disk. Verify it matches the original saved copy.

## Cleanup and reporting

Restore original interface VRFs and addressing, remove test IS-IS interface rows,
then remove test timers/processes/VRFs. Restore saved configuration as appropriate.
Do not leave failure injection active. Verify there are no leftover bindings,
adjacencies or advertised test loopbacks in any affected VRF or namespace.

Attach image/commit identifiers, topology, namespace, each case result, ConfigDB
and FRR snapshots, relevant logs and forwarding observations. Report unexecuted
cases as not run. Warm reboot/fastboot route preservation is not established by
cold-startup success and requires a separate supported-feature validation plan.
