# One console, two browsers: building a shared PlayStation room

**By Ahmad Zushan — Published 10 September 2026 · Version 1.0**

Our goal at [ps.zooshan.com](https://ps.zooshan.com/) is to bring local two-player PlayStation games to people who are in different places, without asking either player to install and configure an emulator. One console runs on our server. Two browsers become its screen and its two controllers.

This article documents our implementation, its approach to responsiveness, and our proposed spectator experience. It also draws an important distinction between a shared simulation and equal network latency.

> One emulated console, two remote controller seats. Neither player runs the match locally—but their network and display delays can still differ.

## What runs where

The emulator runs once on the server, not independently inside each browser. Its native controller interface receives Player 1 and Player 2 inputs. The same emulated game state produces the source pictures and sound for both participants.

Each browser has its own WebRTC connection to the service. Video comes from that shared source, and controller messages travel back through a WebRTC data channel. When direct WebRTC is unavailable, an authenticated WebSocket over HTTPS provides a fallback. The fallback can behave differently from the fast connection, including having higher latency.

This is remote local multiplayer: the game must already support two local controllers. Sharing a room does not turn a single-player title into a multiplayer game. It also does not require synchronizing two independently running emulator copies.

The room creator controls Player 1, and an invited browser controls Player 2. Controller identity is assigned by the service from the authenticated session rather than accepted from a client-supplied player number. The creator still has room-management privileges, such as ending the session; the two roles do not have identical administrative permissions.

## A shared host is not equal ping

When one participant plays directly on a machine and another connects remotely, only the remote participant must make the full network trip to that machine. Our design removes that particular local-host advantage: both participants are remote clients of the server, including the person who created the room.

That is a useful architectural property, but it is not a guarantee of equal response times. The route from Player 1 to the server can differ from Player 2's route. Geography, routing, congestion, Wi-Fi, mobile data, packet loss, browser decoding, controller polling and display refresh all affect what each person experiences.

As an illustration—not a measurement of our users—a player with a 20 ms round-trip network path and a player with an 80 ms path do not acquire equal latency simply by joining the same server. Faster-arriving input can reach an earlier emulation frame. Even on identical network paths, the two screens need not display a frame at exactly the same moment.

The live implementation does not deliberately equalize per-player delay. It does not include cross-player input scheduling, RTT-based compensation or rollback netcode. A player with a better connection may still have a responsiveness advantage. We therefore describe this as a shared remote-host design, not “zero lag,” “identical ping,” or a guarantee of competitively equal conditions.

[WebRTC's statistics specification](https://www.w3.org/TR/webrtc-stats/) distinguishes network round-trip time, receiver buffering and decoding statistics. None of those measurements alone is the complete time from pressing a button to seeing its result on screen.

## What we chose to optimize

Our implementation prioritizes fresh input and fresh pictures over a queue of old events. These are engineering choices in this project, not claims that the underlying techniques are new inventions.

- **One shared simulation.** Both controller ports feed the same emulator instance and emulation loop.
- **Authenticated controller seats.** The server separates Player 1 from Player 2, rejects stale input sequences and bounds message rates.
- **Bounded input and video queues.** Latest-state handling, brief button-press preservation and stale-input release help avoid delayed commands being replayed after a backlog.
- **A responsive streaming profile.** The current profile uses 320 × 240 pictures, up to 60 streamed frames per second when the game produces them, and low-delay H.264 encoding. A changing source frame rate is not automatically an emulator slowdown.
- **Sound is genuinely optional.** Sound starts off. Each player can enable it independently; a player with sound disabled receives video-only WebRTC rather than a hidden audio track. When nobody is listening, the worker stops preparing outgoing audio packets while preserving the emulator's internal sound state.
- **Visible diagnostics.** The player separates emulator speed, server capture delay, receiver buffering and network RTT where the browser exposes them. This helps identify which part of the path needs attention.

No streaming profile can remove the distance to a server or guarantee the behavior of a player's mobile connection. An honest status display is more useful than labeling every problem “emulator lag.”

## What we have checked so far

On 10 September 2026, we ran an engineering validation using two automated receiving peers on a second server, connecting over the network to the emulation server. This was not a controlled fairness study involving two human players on different home networks.

The run covered approximately seven and a half minutes of measured phases: both players muted, one listening, both listening, then both muted again. Both peers received video. Controller-channel round trips, independent sound choices, fallback audio toggles and save download passed. Sampled emulation remained at 60 FPS, and sampled server capture delay was approximately 2.6–3.6 ms. The app's CPU limit did not report throttling during the inspected snapshots.

Those capture-delay figures are only one internal portion of the pipeline. They are not 3 ms internet gameplay latency, not a measurement of physical button-to-screen response, and not proof that both players experience the same delay. The public article records the scope and limitations of this internal check; it is not an independent benchmark.

## Spectators: the next design step

**Proposed feature — not available in the current release.**

We want a person who is not playing to open an authorized watch link and follow the match without taking either controller seat. The current application has two authenticated player seats only. Sharing a player invitation is not a spectator mechanism, and opening another tab for an existing seat can replace that seat's connection.

The proposed design adds a separate read-only viewer role. A spectator would receive the game's media but have no controller-input channel or permission to upload a disc, restore a state, claim a player seat or manage the room. Watch access should be independently revocable and subject to the host's consent and a viewer limit. “Anyone can watch” should mean anyone the host chooses to admit, not that every private game is automatically public.

Spectators must not slow the people playing. Each extra viewer consumes bandwidth and potentially encoding resources, so a production version needs capacity limits, load tests and isolation of the player path. Larger audiences may require a separate distribution layer, such as a selective forwarding unit. This is a design direction, not a deployed scaling guarantee; viewers may also see the action at different times.

## Attribution and earlier work

This project and this implementation account are presented by Ahmad Zushan. We are publishing a dated description of what we built and what we plan to explore—not claiming to have invented browser game streaming, remote local multiplayer or the emulator itself.

CloudRetro is an important earlier example. In his [15 April 2020 architectural article](https://webrtchacks.com/open-source-cloud-gaming-with-webrtc/), its creator described server-side emulation, browser streaming, shared game links and two-player retro gameplay across devices. Our implementation belongs in that established technical context. Whether any narrowly defined improvement qualifies for patent protection requires a separate, claim-specific professional assessment.

Emulation in our service uses [PCSX-ReARMed](https://github.com/libretro/pcsx_rearmed) through the libretro interface. Streaming uses [aiortc](https://github.com/aiortc/aiortc) and established WebRTC protocols. Credit and the applicable licenses remain with their respective contributors. The project is independent and is not affiliated with or endorsed by Sony or PlayStation. No games or BIOS files are provided with the article; users must have the rights needed for the files they use.

## A public record, not exclusive ownership

The byline, publication date and public revision history make this account attributable and citable. They do not prove that an idea was first conceived here, grant a patent, or prevent others from independently building similar systems. Public disclosure can affect future patent options; [WIPO's patent guidance](https://www.wipo.int/en/web/patents/faq_patents) explains the importance of novelty, earlier disclosures and filing decisions. This note is general information, not legal advice.

Suggested citation: Ahmad Zushan. “One console, two browsers: building a shared PlayStation room.” Version 1.0, 10 September 2026. https://ps.zooshan.com/article

The [public article repository](https://github.com/Zushan/ps-streaming-architecture) contains this publication and its revision history, not the private application repository, deployment credentials, user uploads or game assets.
