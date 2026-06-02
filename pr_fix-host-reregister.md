# fix(client): re-register host after watcher takeover

## Problem

When a tray client takes over the `org.kde.StatusNotifierWatcher` name from a
previous owner, its tray ends up empty and **stays** empty — icons never come
back. This happens whenever two host instances overlap, e.g. restarting or
reloading a bar while the old process is still briefly alive.

## Root cause

`Client::new` registers the host once, against *whoever currently owns* the
watcher name. If this client was queued behind an existing owner, that
`RegisterStatusNotifierHost` lands on the **other** watcher.

When the prior owner exits and this client receives `NameAcquired` for the
watcher name, it becomes the watcher — but its host registration was on the
now-dead one, so its own watcher reports `IsStatusNotifierHostRegistered =
false`. Items gate (re-)registration on that property, so they never register
with the new watcher → empty tray. The existing `NameAcquired` handler clears
items but never re-registers the host.

## Fix

Re-register the host inside the `NameAcquired` handler, so the property becomes
`true` again and items re-register. (+19/−2 in `src/client.rs`.)

## Reproduction

Runs on a private bus (never touches your real session bus) and uses the
shipped `examples/basic.rs` as the tray client. Save as `repro.sh` in the repo
root and run `./repro.sh`:

```bash
#!/usr/bin/env bash
# After a StatusNotifierWatcher ownership takeover, the new watcher's
# IsStatusNotifierHostRegistered is left false -> tray items never re-register.
#   master: false    with PR: true
set -euo pipefail
cd "$(dirname "$0")"

if [[ -z "${IN_BUS:-}" ]]; then
  cargo build --example basic >/dev/null 2>&1
  exec env IN_BUS=1 dbus-run-session -- "$0"   # isolated bus
fi

W=org.kde.StatusNotifierWatcher
bc() { busctl --address="$DBUS_SESSION_BUS_ADDRESS" "$@" 2>/dev/null; }
prop()  { bc get-property $W /StatusNotifierWatcher $W IsStatusNotifierHostRegistered | awk '{print $2}'; }
owner() { bc call org.freedesktop.DBus /org/freedesktop/DBus org.freedesktop.DBus GetNameOwner s $W | awk '{print $2}'; }

BIN=target/debug/examples/basic
$BIN >/dev/null 2>&1 & A=$!                                   # A: becomes watcher + registers host
until [[ "$(prop)" == true ]]; do sleep 0.1; done
a=$(owner)
$BIN >/dev/null 2>&1 & B=$!                                   # B: queues for name, host lands on A's watcher
sleep 1
kill "$A"                                                     # A exits -> B takes over the watcher name
until [[ -n "$(owner)" && "$(owner)" != "$a" ]]; do sleep 0.1; done
sleep 1                                                       # let B handle NameAcquired
echo "IsStatusNotifierHostRegistered after takeover: $(prop)"
kill "$B" 2>/dev/null || true
```

### Result

```
master:  IsStatusNotifierHostRegistered after takeover: false
with PR: IsStatusNotifierHostRegistered after takeover: true
```
