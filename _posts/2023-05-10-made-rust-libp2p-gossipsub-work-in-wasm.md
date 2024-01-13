---
layout: post
title: "Made rust-libp2p gossipsub work in WASM"
date:   2023-05-10 23:45:00 +09:00
---

## Background

In [rust-libp2p](https://github.com/libp2p/rust-libp2p) v0.51.3, the [`gossipsub`](https://github.com/libp2p/rust-libp2p/tree/libp2p-v0.51.3/protocols/gossipsub) module cannot be compiled with the target `wasm32-unknown-unknown`
because the [`wasm-timer`](https://github.com/tomaka/wasm-timer) crate required by `gossipsub` throws the following error, as mentioned in [my previous post](https://oudwud.dev/posts/2305042345-exploring-rust-libp2p/#wasm-limitations).
```bash
$ wasm-pack build

[INFO]: 🎯  Checking for the Wasm target...
[INFO]: 🌀  Compiling to Wasm...
   Compiling libp2p-gossipsub v0.45.0 (https://github.com/libp2p/rust-libp2p.git?branch=master#14938043)
error[E0599]: no method named `checked_add` found for struct `wasm_timer::Instant` in the current scope
   --> /Users/yjlee/.cargo/git/checkouts/rust-libp2p-98135dbcf5b63918/1493804/protocols/gossipsub/src/peer_score.rs:872:34
    |
871 |   ...                   let window_time = validated_time
    |  _________________________________________-
872 | | ...                       .checked_add(topic_params.mesh_message_deliveries_window)
    | |                           -^^^^^^^^^^^ method not found in `Instant`
    | |___________________________|
    |
```
There might be several options to resolve the issue, as [discussed](https://github.com/libp2p/rust-libp2p/issues/2497#issuecomment-1038923700) by the libp2p team.
Among them, I chose to fork the `wasm-timer` crate with minimum changes
because rust-libp2p gossipsub deeply depends on the `wasm-timer::Interval` which is very [critical for the performance of gossipsub](https://github.com/libp2p/rust-libp2p/pull/2506).


## What I've changed

In my [fork](https://github.com/youngjoon-lee/wasm-timer), I've made only single [commit](https://github.com/youngjoon-lee/wasm-timer/commit/15e8835322f2912266a4ce9df46b42779e4eb718).
that replaces `std::time::Instant` with [`instant::Instant`](https://crates.io/crates/instant) that works in both non-WASM and WASM.
Thanks to `instant::Instant`, I was able to remove all `Instant` definition and implementations from my fork.
On the other hand, I left all `wasm-timer::Interval`-related codes as they are, because I didn't want to violate the current gossipsub performance that the libp2p team has measured.

After pushing my `wasm-timer` fork to the repository, I forked `rust-libp2p` as well in order to update the `wasm-timer` dependence to my fork: [d81554](https://github.com/youngjoon-lee/rust-libp2p/commit/d81554d5fda38c3cbb5677e3f5f96eb21101dbb7).
In that commit, I also had to enable the `gossipsub` module for the `wasm32-unknown-unknown` target, which has been disabled so far becauase of the reason that I mentioned above.


## Does it work?

Yes! I tested it in my personal project.
I made a simple Rust WASM library that requires gossipsub, and ran `wasm-pack build` ([youngjoon-lee/jiri@7a9aa17](https://github.com/youngjoon-lee/jiri/commit/7a9aa1712f349c2369f37c673b1048b8c448f549#diff-e713d3a76800ed4c40a59bbd9acce179c52889a54293e4656d1ea38c5c47eedfR15)).
Everything was complied successfully and the gossipsub worked in browser.
![](/assets/rust-libp2p-gossipsub-wasm.png)
