---
layout: post
title: "Exploring rust-libp2p"
date:   2023-05-04 23:45:00 +09:00
---

## Connectivity Tests

In [libp2p.io](https://libp2p.io), [the current state of Transport implementations](https://libp2p.io/implementations/) in each programming language is already written. But, I’ve seen there have been so many changes in rust-libp2p, as described in the [rust-libp2p in 2022](https://libp2p.io/2023-01-12-rust-libp2p-in-2022/) blog post. 

After reading the [libp2p Connectivity](https://connectivity.libp2p.io/) document, I've tested if 'dialing' works for each of the following scenarios using [rust-libp2p](https://github.com/libp2p/rust-libp2p) v0.51.3.

|Scenario|Dialing|Transports tested|
|--------|-------|----------|
|Standalone -> Standalone|Successful|TCP, WebSocket, <cite>WebRTC[^1]</cite>|
|WASM browser -> Standalone|Successful|<cite>WebSocket[^2]</cite>|
|WASM browser <- Standalone|<cite>Failed[^3]</cite>|WebSocket|
|WASM browser -> WASM browser|<cite>Failed[^3]</cite>|WebSocket|
|<cite>JS browser[^4]</cite> -> Standalone|Successful|WebRTC|
|Private -> (<cite>Relay[^5]</cite>) -> Private|Successful|TCP|
|<cite>Private[^6]</cite> -> (<cite>Relay[^5]</cite>) -> Private with [Hole-punching](https://docs.rs/libp2p/latest/libp2p/tutorials/hole_punching/index.html)|<cite>Failed[^7]</cite>|TCP|

[^1]: [WebRTC example](https://github.com/libp2p/rust-libp2p/blob/libp2p-v0.51.3/transports/webrtc/examples/listen_ping.rs)
[^2]: [vincev/wasm-p2p-chat](https://github.com/vincev/wasm-p2p-chat) based on [vincev/libp2p-websys-transport](https://github.com/vincev/libp2p-websys-transport) that [will be included in the rust-libp2p offically](https://github.com/libp2p/rust-libp2p/issues/3611)
[^3]: [Browser nodes cannot listen for incoming conns.](https://docs.libp2p.io/concepts/transports/webrtc/#webrtc-private-to-private)
[^4]: [js-libp2p-webrtc browser-to-server example](https://github.com/libp2p/js-libp2p-webrtc/tree/main/examples/browser-to-server)
[^5]: [Relay-server example](https://github.com/libp2p/rust-libp2p/blob/libp2p-v0.51.3/examples/relay-server)
[^6]: [dCUtR example](https://github.com/libp2p/rust-libp2p/blob/libp2p-v0.51.3/examples/dcutr)
[^7]: It seems that the hole-punching is [not always successful](https://github.com/libp2p/rust-libp2p/discussions/3458#discussioncomment-5015353). Used AWS for relayer and listener (in private subnet with NAT), and my laptop for dialer.

## WASM Limitations

I've found that the following features of rust-libp2p cannot be enabled for WASM.
```
gossipsub, mdns, dns, tokio
```
If WASM codes import those features, the following error occurs:
```bash
error[E0432]: unresolved import `libp2p::gossipsub`
  --> core/src/p2p.rs:20:5
   |
20 |     gossipsub, identity,
   |     ^^^^^^^^^ no `gossipsub` in the root
```
In regard to gossipsub especially, the gossipsub has been disabled for the `wasm32-unknown-unknown` target by [rust-libp2p/pull/2506#issuecomment-1036448620](https://github.com/libp2p/rust-libp2p/pull/2506#issuecomment-1036448620)
because of the following reasons:
- The custom `Interval` implementation in rust-libp2p wasn't performant: [rust-libp2p/issues/2497](https://github.com/libp2p/rust-libp2p/issues/2497). So, they decided to revert back to [wasm-timer](https://github.com/tomaka/wasm-timer) by [rust-libp2p/pull/2506](https://github.com/libp2p/rust-libp2p/pull/2506).
- However, the `rust-libp2p` with `wasm-timer` cannot be compiled for `wasm32-unknown-unknown` with the following error:
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
- Fortunately, it seems that we can use the [instant](https://github.com/sebcrozet/instant) crate instead of `wasm_timer::Instant`. This try had been done by [rust-libp2p/pull/2320](https://github.com/libp2p/rust-libp2p/pull/2320), but has been reverted by [rust-libp2p/pull/2506](https://github.com/libp2p/rust-libp2p/pull/2506) as above. 
- But the one remaining problem was that the gossipsub needs the `Interval` struct which is also dependant on the `Instant`. To resolve this, the libp2p team is thinking of contributing an `Interval` implementation to the `futures-timer` crate: [rust-libp2p/issues/2497#issuecomment-1038923700](https://github.com/libp2p/rust-libp2p/issues/2497#issuecomment-1038923700)
- But, I think the easier solution would be just implementing the missing `wasm_imter::Instant.checked_add()` function above into the `wasm_timer` because the `wasm_timer` already provides the `Interval` struct which is being used by `rust-libp2p` in production. However, it seems that the libp2p the doesn't want to keep maintaining the `wasm-timer` crate.
