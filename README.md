# One console, two browsers: building a shared PlayStation room

**By Ahmad Zushan — Published 10 September 2026 · Updated 11 September 2026 · Version 1.2**

Our goal at [ps.zooshan.com](https://ps.zooshan.com/) is to bring local two-player PlayStation games to people who are in different places, without asking either player to install and configure an emulator. One console runs on our server. Two browsers become its screen and its two controllers.

This article documents our implementation, its approach to responsiveness, and its spectator experience. Version 1.1 introduced spectators, operator controls and a tournament dashboard. Version 1.2 records controller handoff and queues, four independent room slots, email-associated hosting keys, silent gameplay recordings and optional room voice. It also maintains an important distinction between a shared simulation and equal network latency.

> One emulated console, two remote controller seats. Neither player runs the match locally—but their network and display delays can still differ.

## What runs where

The emulator runs once on the server, not independently inside each browser. Its native controller interface receives Player 1 and Player 2 inputs. The same emulated game state produces the source pictures and sound for both participants.

Each active controller browser has its own game WebRTC connection to the service. Video comes from that shared source, and controller messages travel back through a WebRTC data channel. When direct WebRTC is unavailable, an authenticated WebSocket over HTTPS provides a fallback. The fallback can behave differently from the fast connection, including having higher latency. Waiting spectators use shared silent JPEG pictures over HTTPS, without a dedicated game WebRTC encoder. Optional room voice is a separate connection described below.

This is remote local multiplayer: the game must already support two local controllers. Sharing a room does not turn a single-player title into a multiplayer game. It also does not require synchronizing two independently running emulator copies.

In an ordinary room, the creator initially controls Player 1, and the first invited browser normally controls Player 2. Players can release a controller to the next eligible queued participant without ending the room. In a tournament, the selected registrations receive those two controller seats instead. Controller identity is assigned by the service from the authenticated session rather than accepted from a client-supplied player number. Organizers retain room-management privileges; receiving Player 1 does not itself confer those privileges.

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
- **Game sound is genuinely optional.** Game sound starts off. Each player can enable it independently; a player with game sound disabled receives video-only game WebRTC rather than a hidden game-audio track. When nobody is listening to game sound, the worker stops preparing outgoing audio packets while preserving the emulator's internal sound state. Room voice is independently opt-in.
- **Visible diagnostics.** The player separates emulator speed, server capture delay, receiver buffering and network RTT where the browser exposes them. This helps identify which part of the path needs attention.

No streaming profile can remove the distance to a server or guarantee the behavior of a player's mobile connection. An honest status display is more useful than labeling every problem “emulator lag.”

## What we have checked so far

On 10 September 2026, we ran an engineering validation using two automated receiving peers on a second server, connecting over the network to the emulation server. This was not a controlled fairness study involving two human players on different home networks.

The run covered approximately seven and a half minutes of measured phases: both players muted, one listening, both listening, then both muted again. Both peers received video. Controller-channel round trips, independent sound choices, fallback audio toggles and save download passed. Sampled emulation remained at 60 FPS, and sampled server capture delay was approximately 2.6–3.6 ms. The app's CPU limit did not report throttling during the inspected snapshots.

Those capture-delay figures are only one internal portion of the pipeline. They are not 3 ms internet gameplay latency, not a measurement of physical button-to-screen response, and not proof that both players experience the same delay. The public article records the scope and limitations of this internal check; it is not an independent benchmark.

For the 11 September voice release, automated checks covered room-cookie authorization, controller handoff, host microphone permissions and turning voice off during a pending microphone request or connection. Native clients sending generated audio verified same-room delivery and isolation from a second room. An off-server Windows client also received generated audio through production HTTPS signaling and internet media, without using a microphone. These checks do not establish human conversational quality, physical input latency, mobile-browser compatibility or unchanged gameplay performance at maximum room occupancy.

## Spectators and organized matches

**Spectators cannot control the game while waiting.**

The original version 1.0 article described spectators as a proposal; version 1.1 added bounded watching. The current ordinary-room flow gives an available controller to the first eligible visitor and queues later visitors, subject to host settings. Releasing a controller promotes the next eligible queued member; someone who releases can watch or rejoin at the end of the queue. Invitations expire or can be revoked. An existing member reopening a link retains their membership; opening another tab can replace that membership's connection.

Spectators receive pictures but no accepted controller-input channel while waiting. Joining the queue requests a future server-assigned seat; it does not let a visitor choose or seize an occupied controller. Ordinary invited viewers cannot upload a disc, restore a state or manage the room. The original room manager retains management even after releasing their controller. The organizer can revoke an invitation independently of removing already-connected viewers. The default limit is two spectators, configurable from zero to four. “Anyone can watch” means people admitted through the room link or the organizer-enabled tournament watch flow—not unrestricted access to every private game.

The spectator picture is encoded once and shared at up to 15 frames per second. Each additional viewer still consumes bandwidth and connection-handling resources; joining optional voice adds separate audio-relay work. These are deliberately small limits, not a guarantee that adding viewers cannot affect player responsiveness. Larger audiences would require further load testing and potentially a separate distribution layer. Viewers can also see the action at different times.

The operator desk exposes validated output-resolution and stream-frame-cap settings, pre-start emulation choices, spectator controls and bounded diagnostics. It also selects among active rooms and retains session history. The tournament dashboard supports registration-order single elimination or round robin, a waiting queue and organizer-confirmed series results. Each tournament and each console can have only one live match at a time; separate tournaments can use separate available room slots. Old controller assignments and game connections are invalidated when assignments change, but same-room voice can continue. Registration and results persist separately from temporary game uploads. Results are confirmed by an organizer, not automatically recognized from gameplay.

## Four rooms and browser-bound hosting keys

The service admits up to four independent rooms, with a separate emulator worker and private uploads for each. Two players in one room share that room's simulation; they do not share a game state with another room. Four slots are a capacity ceiling, not a promise of full-speed emulation for every combination of games and streams. Server CPU, memory, disk and network capacity remain shared.

The home page places **Generate a 24-hour key** prominently beside **Unlock hosting**. A visitor can [generate a day pass](https://ps.zooshan.com/keys/) using a name and email, then enter that same email and the key to activate hosting. The current day pass is a free preview; payments and credits are not enabled. Validity starts at generation, and a key does not reserve an available room slot.

An ordinary key can host one active room and is bound to one browser through a private cookie. End the room and release its activation before moving to another browser; closing a tab does not instantly release an active room. A room ends after at most two hours or when its key expires, whichever comes first. The operator's separate master key has no expiry and supports multiple rooms and browsers within the same four-room ceiling.

Email is associated with a key, not verified ownership or a recovery factor. A raw key is shown once and stored as a hash. Browser binding is possession of a cookie, not a hardware identity check, so users must keep both keys and cookies private. The [privacy page](https://ps.zooshan.com/about.html) explains contact records, session history and retention.

## Room voice without coupling it to game sound

Players and queued viewers can choose **Join voice chat** to talk inside their room. Voice is off by default. Until a visitor joins, the voice SDK is not loaded and no microphone capture, voice polling or voice WebRTC connection is allocated. Enabling voice does not enable game sound, and enabling game sound does not enable a microphone.

**Mute microphone** stops capture while keeping listening connected. **Voice off** stops the microphone, disconnects voice and cancels an in-progress join—even if microphone permission arrives after Off was selected. Returning to voice requires another explicit Join. Hosts can prevent a participant from publishing microphone audio; allowing it again does not remotely turn their microphone on. Controller promotion or release within the same room does not interrupt voice membership.

Voice uses a separate audio-only [LiveKit](https://github.com/livekit/livekit) forwarding service and mono Opus audio, rather than adding a microphone track to the emulator's game connection. The application checks room membership and participant identity for signaling, limits room voice occupancy and revokes voice access when membership ends. Voice does not grant controller or room-management permissions.

This separation avoids adding voice encoding or mixing to the emulator process, but it cannot guarantee zero performance impact. Enabled voice still uses the device's microphone processing, network bandwidth and shared server resources. The idle voice service also has a small baseline cost. We place resource limits on that service and keep the game transport independent; saturated connections or a busy server can still increase delay. Restrictive networks may block voice even while another game connection method works.

Voice is encrypted in transit to and from the relay, not end-to-end encrypted between participants. The application stores no voice recordings or transcripts, and voice is excluded from gameplay recordings. Other participants can still independently record what they hear. Microphone permission and a clear Off control are important privacy controls, not a promise that spoken information cannot leave the room.

## Optional silent recordings

The room owner can explicitly record gameplay using a separate low-priority encoder. Recordings are silent: no game sound, voice chat, microphone, camera or personal desktop is captured. A recording indicator is shown to participants. One recording can run across the server at a time, at up to 320 × 240 and 15 frames per second, with duration, storage and performance limits.

Clips are private until shared, normally expire after 24 hours, and can be removed sooner. A temporary sharing link allows its holder to view or download the clip until expiry or revocation. Revoking a link cannot retrieve an already downloaded copy. Recording work remains additional shared-server load; lower priority and automatic stopping are safeguards, not a zero-impact guarantee. Only record and share footage you are entitled to use.

## Attribution and earlier work

This project and this implementation account are presented by Ahmad Zushan. We are publishing a dated description of what we built and what we plan to explore—not claiming to have invented browser game streaming, remote local multiplayer or the emulator itself.

CloudRetro is an important earlier example. In his [15 April 2020 architectural article](https://webrtchacks.com/open-source-cloud-gaming-with-webrtc/), its creator described server-side emulation, browser streaming, shared game links and two-player retro gameplay across devices. Our implementation belongs in that established technical context. Whether any narrowly defined improvement qualifies for patent protection requires a separate, claim-specific professional assessment.

Emulation in our service uses [PCSX-ReARMed](https://github.com/libretro/pcsx_rearmed) through the libretro interface. Game streaming uses [aiortc](https://github.com/aiortc/aiortc) and established WebRTC protocols. Room voice uses LiveKit server and its client SDK under Apache-2.0. Credit and the applicable licenses remain with their respective contributors. The project is independent and is not affiliated with or endorsed by Sony or PlayStation. No games or BIOS files are provided with the article; users must have the rights needed for the files they use.

## A public record, not exclusive ownership

The byline, publication date and public revision history make this account attributable and citable. They do not prove that an idea was first conceived here, grant a patent, or prevent others from independently building similar systems. Public disclosure can affect future patent options; [WIPO's patent guidance](https://www.wipo.int/en/web/patents/faq_patents) explains the importance of novelty, earlier disclosures and filing decisions. This note is general information, not legal advice.

Suggested citation: Ahmad Zushan. “One console, two browsers: building a shared PlayStation room.” Version 1.2, updated 11 September 2026; first published 10 September 2026. https://ps.zooshan.com/article

The [public article repository](https://github.com/Zushan/ps-streaming-architecture) contains this publication and its revision history, not the private application repository, deployment credentials, user uploads or game assets.
