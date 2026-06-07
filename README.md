 > ## ⚠️ This is a fork of [`OpenActionAPI/rust`](https://github.com/OpenActionAPI/rust)
>
> **Why this fork exists**
> The upstream crate has no inbound event for `touchTap`, so when an OpenDeck host sends a
> `touchTap` (a tap or long-press on a Stream Deck + / + XL encoder LCD touchscreen) the
> event fails to deserialize and is silently dropped — a plugin can never react to it.
> This fork adds that event end-to-end on the **`add-touch-tap-event`** branch:
> a `TouchTap` inbound variant, a `handle_touch_tap` runtime handler, and a default-no-op
> `Action::touch_tap(instance, settings, hold, tap_pos)` trait method. The change is purely
> additive and backward-compatible.
>
> **What depends on this fork**
> - [`koconnorgit/opendeck-volume-controller`](https://github.com/koconnorgit/opendeck-volume-controller)
>   — its `Cargo.toml` has `[patch.crates-io] openaction = { git = ".../koconnorgit/rust", branch = "add-touch-tap-event" }`,
>   and its `Cargo.lock` pins a commit on that branch. This is how its "tap the LCD to mute"
>   feature receives touch events.
>
> The matching **host side** (emitting `touchTap`) lives in
> [`koconnorgit/OpenDeck`](https://github.com/koconnorgit/OpenDeck) as part of that fork's
> Stream Deck + XL support; stock `nekename/OpenDeck` does not emit `touchTap`.
>
> **⚠️ Do not delete the `add-touch-tap-event` branch.** A downstream `Cargo.lock` pins it;
> removing it (or this repo) breaks `cargo build` for the volume-controller plugin.
>
> **Upstreaming:** considered and intentionally **not** pursued — this stays a permanent
> personal fork rather than an upstream PR.

---

# openaction-rs

A Rust crate for creating plugins for the [OpenAction API](https://openaction.amankhanna.me) (backwards-compatible with the Stream Deck SDK)

```rust
use openaction::*;

use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Default, Clone)]
#[serde(default)]
struct CounterSettings {
	value: u32,
}

struct CounterAction;
#[async_trait]
impl Action for CounterAction {
	const UUID: ActionUuid = "com.example.counter.counter";
	type Settings = CounterSettings;

	async fn key_up(
		&self,
		instance: &Instance,
		settings: &Self::Settings,
	) -> OpenActionResult<()> {
		let mut clone = settings.clone();
		clone.value = settings.value + 1;
		instance.set_settings(&clone).await?;
		instance.set_title(Some(clone.value.to_string()), None).await
	}
}

#[tokio::main]
async fn main() -> OpenActionResult<()> {
	{
		use simplelog::*;
		if let Err(error) = TermLogger::init(
			LevelFilter::Debug,
			Config::default(),
			TerminalMode::Stdout,
			ColorChoice::Never,
		) {
			eprintln!("Logger initialization failed: {}", error);
		}
	}

	register_action(CounterAction).await;

	run(std::env::args().collect()).await
}
```
