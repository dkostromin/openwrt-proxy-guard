# openwrt-proxy-guard

`safe-proxy-reload` — a small POSIX-sh wrapper for OpenWrt routers that run **forkop** or **podkop**.
It wraps a planned restart in a fail-closed `nft` guard, so the seconds when the proxy engine is
down become a short outage instead of a window where captured clients leave through the home WAN
with the real IP.

## What actually leaks (measured, not assumed)

Measured on a Netis NX32U running forkop + sing-box on OpenWrt, sampling `nft list ruleset` every
200 ms around each operation and probing the exit IP from a captured client:

| Operation | tproxy rules (port 1602) | Client on the capture list sees |
|---|---|---|
| `forkop reload` | stay installed the whole time | short stalls, never the home IP |
| `kill -9 sing-box` | stay installed; procd restarts it | a few timeouts, never the home IP |
| `service stop`, reboot, failed start | **removed** | direct routing — the real leak |
| `podkop` stop | **removed** (its `podkop-guard` restores normal routing by design) | direct routing |

The nft rules are installed by the *service*, not by the sing-box process, so a crash fails closed:
packets keep being sent to the tproxy socket and are dropped. The leak window exists only when the
rules themselves are gone.

## What the wrapper does

1. Collects the captured sources:
   - forkop — the union of `fully_routed_ips` over **enabled** sections;
   - podkop — `podkop.settings.source_network_interfaces`;
   - or explicit interfaces via `--guard-iif` (for boxes that route by community/domain lists).
2. Inserts one `drop` rule into `inet fw4 forward` for those sources, with counters and the
   comment `hermes-guard`.
3. Runs `forkop reload` or `podkop restart`.
4. Waits until sing-box is running and tproxy rules are present again (up to 30 s).
5. Removes the guard and prints its counters.

`packets=0` on a plain reload is the expected result — it proves nothing fell through. A non-zero
counter means traffic really did try to go direct during the operation.

A detached failsafe removes the guard after `--window` seconds (default 90) even if the script is
killed or the SSH session drops, so a failed run cannot leave the LAN offline.

## Install

```sh
scp safe-proxy-reload root@192.168.1.1:/usr/bin/
ssh root@192.168.1.1 chmod +x /usr/bin/safe-proxy-reload
```

Requirements: OpenWrt with `nft` (fw4), `uci`, BusyBox `sh`. No dependencies beyond that.

## Usage

```sh
safe-proxy-reload                     # auto-detect the running service
safe-proxy-reload forkop              # explicit
safe-proxy-reload podkop
safe-proxy-reload --dry-run           # show the guard match, change nothing
safe-proxy-reload forkop --window 120 # keep the guard longer on a slow start
safe-proxy-reload forkop --guard-iif br-lan
```

Exit codes: `0` ok, `1` usage/detection, `2` guard failed (nothing was reloaded),
`3` reload failed (guard kept), `4` engine did not come back in time (guard kept).

## Verify by hand

```sh
# rules must never disappear during the operation
while :; do echo "$(date +%H:%M:%S) rules=$(nft list ruleset | grep -c 1602) pid=$(pidof sing-box)"; sleep 0.2; done

# a captured client must never report the home WAN IP
curl -s --max-time 1 https://api.ipify.org; echo

# no leftover guard
nft list chain inet fw4 forward | grep -c hermes-guard    # 0
```

Removing a stuck guard:

```sh
nft -a list chain inet fw4 forward | awk '/hermes-guard/{print $NF}' \
  | while read h; do nft delete rule inet fw4 forward handle $h; done
```

## Caveats

- The guard drops **all** forwarded traffic from the captured sources for those seconds, including
  LAN-to-LAN between them. That is the point: fail closed.
- DNS is not covered by the guard. With dnsmasq pointing at sing-box (`127.0.0.42`), name
  resolution fails while the engine is down; browsers with their own DoH resolver will bypass that
  and resolve directly. If DNS-level leakage matters, pin a resolver you trust.
- On boxes where forkop routes by domain lists and no `fully_routed_ips` exist, the script refuses
  to run unless you pass `--guard-iif`.

Tested on OpenWrt 24.x with forkop 1.0.x, sing-box 1.13.13, BusyBox ash (Netis NX32U, Cudy WR3000S).

## License

MIT
