# fix(client): don't panic when layout debounce outlives watch loop

## Problem

`watch_menu` can panic the whole task with `should send: SendError { .. }`.

## Root cause

For each `LayoutUpdated` signal, `watch_menu` spawns a debounce task that sleeps
`LAYOUT_UPDATE_INTERVAL_MS`, then sends the parent id back to the loop over
`layout_tx`. That task holds a **clone** of the sender, so it can outlive the
loop.

If the loop exits first — the layout fetch arms `break` on error or timeout:

```rust
Ok(Err(err)) => { error!("error fetching layout: {err:?}"); break; }
Err(_)       => { error!("Timeout getting layout");          break; }
```

— then `layout_rx` is dropped. When the in-flight debounce task wakes, the
receiver is gone, so the send fails and the old code unwraps it:

```rust
layout_tx.send(args.parent).await.expect("should send");   // panics
```

## Fix

Ignore the send error; a dropped receiver just means the loop already exited and
there is nothing to dispatch. (+2/−1 in `src/client.rs`.)

```rust
// receiver gone => watch loop exited; nothing to dispatch
let _ = layout_tx.send(args.parent).await;
```

## Reproduction

The failure is the `Sender::send` / `.expect()` contract, reduced from
`watch_menu`. Drop it into a scratch binary (the crate already depends on
`tokio`) and run it:

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (layout_tx, layout_rx) = mpsc::channel::<i32>(4);

    // a debounce task spawned while the watch loop is alive
    let task = tokio::spawn(async move {
        sleep(Duration::from_millis(50)).await;          // LAYOUT_UPDATE_INTERVAL_MS
        layout_tx.send(0).await.expect("should send");   // master: panics
        // let _ = layout_tx.send(0).await;              // with PR: no-op
    });

    // meanwhile the watch loop exits (layout fetch error/timeout) -> rx dropped
    drop(layout_rx);

    task.await.unwrap();
}
```

### Result

```
master:  thread '...' panicked at ...: should send: SendError { .. }
with PR: exits cleanly
```
