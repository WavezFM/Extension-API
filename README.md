# WavezFM Room Extension API

The **WavezFM Room Extension API** is the official client-side bridge for browser extensions and user scripts running on WavezFM room pages.

It exposes room state, live events, and a small set of user actions without requiring DOM queries, simulated clicks, or private application internals.

## Overview

- Global bridge: `window.WavezFM`
- Current compatibility version: `"1"`
- Intended for browser extensions, user scripts, and page-level integrations
- Available while a WavezFM room client is active
- Supports authenticated room sessions and read-only guest previews
- Does not expose authentication credentials or private API tokens

This bridge is separate from the [`@wavezfm/api`](https://www.npmjs.com/package/@wavezfm/api) package. Use the Room Extension API for code running inside a WavezFM room page. Use the npm package or Room Bot API for external applications and unattended automation.

## Availability

The bridge is installed when the room client starts. Extensions that run at `document_start` may execute before it is ready.

```js
const api = window.WavezFM;

if (!api || api.version !== "1") {
  console.warn("WavezFM room bridge is unavailable");
}
```

For early-running scripts, wait for the bridge with a bounded timeout:

```js
async function waitForWavezFM(timeoutMs = 10_000) {
  const startedAt = Date.now();

  while (Date.now() - startedAt < timeoutMs) {
    if (window.WavezFM?.version === "1") {
      return window.WavezFM;
    }

    await new Promise((resolve) => setTimeout(resolve, 50));
  }

  return null;
}

const api = await waitForWavezFM();

if (!api) {
  console.warn("WavezFM room bridge did not become available");
}
```

### Lifecycle notes

- A direct page load outside a room may not define `window.WavezFM`.
- `api.room.getState()` returns `null` until room state is ready.
- After leaving a room through client-side navigation, the bridge object may remain installed while `getState()` returns `null` and actions return `unavailable` or `missing_room`.
- Do not retain a room-state snapshot indefinitely. Call `getState()` again or subscribe to events when fresh data is required.

### Browser extension execution context

The bridge belongs to the page's JavaScript context. Browser-extension content scripts that run in an isolated world may not be able to read page globals directly. Run the integration in the page's main world, or inject a page script and communicate with the content script through DOM events or `window.postMessage`.

## Reading room state

`room.getState()` returns the latest complete room snapshot, or `null` when no room controller is active.

```js
const state = window.WavezFM?.room.getState();

if (state) {
  console.log("Room:", state.room.name);
  console.log("Track:", state.playback?.title ?? "Nothing playing");
  console.log("Woots:", state.votes.woots);
  console.log("Users:", state.users.length);
}
```

Treat returned objects and arrays as read-only snapshots. Mutating them does not update WavezFM.

## State reference

```ts
type WavezRoomState = {
  room: {
    id: string;
    slug: string;
    name: string;
    description: string;
    isVerified: boolean;
    isPartner: boolean;
    viewerRole: string;
    queueLocked: boolean;
    activeUsersCount: number;
    queueCount: number;
  };

  currentUser: {
    id: string;
    username: string;
    displayUsername: string;
    globalRole: string;
    roomRole: string | null;
    level: number | null;
    currentLevelXp: number | null;
    xpRequired: number | null;
    totalXp: number | null;
    fanCount: number | null;
    infiniteLevel: boolean;
  } | null;

  playback: {
    playbackKey: string;
    trackId: string;
    source: "youtube" | "soundcloud";
    sourceId: string;
    title: string;
    artist: string;
    thumbnailUrl: string | null;
    durationMs: number;
    isLive: boolean;
    startedAtServerMs: number;
    djId: string;
    djUsername: string;
  } | null;

  votes: {
    trackId: string | null;
    woots: number;
    mehs: number;
    grabs: number;
    wootUserIds: string[];
    mehUserIds: string[];
    grabUserIds: string[];
    clientVote: "woot" | "meh" | null;
    clientGrabbed: boolean;
    clientGrabPlaylistId: string | null;
    canVote: boolean;
  };

  queue: {
    userIds: string[];
    count: number;
    isJoined: boolean;
    isCurrentDj: boolean;
    isLocked: boolean;
    isFull: boolean;
    currentDjId: string | null;
    currentDjUsername: string | null;
    playbackTrackId: string | null;
    entries: Array<{
      userId: string;
      username: string;
      displayUsername: string | null;
      rawUsername: string | null;
      handle: string | null;
      avatar: string | null;
      role: string;
      platformRole: string;
      level: number | null;
      xp: number | null;
      fanCount: number | null;
      infiniteLevel: boolean;
      isSuperfan: boolean;
      isFollowing: boolean;
      position: number;
      queuedTrackDurationMs: number | null;
      estimatedWaitMs: number | null;
      estimatedWaitKind: "ready" | "live" | "unknown";
    }>;
  };

  users: Array<{
    id: string;
    username: string;
    displayUsername: string | null;
    rawUsername: string | null;
    handle: string | null;
    role: string;
    platformRole: string;
    avatar: string | null;
    level: number | null;
    xp: number | null;
    fanCount: number | null;
    infiniteLevel: boolean;
    isSuperfan: boolean;
    isFollowing: boolean;
  }>;

  social: {
    superfanIds: string[];
    followingIds: string[];
    superfansCount: number;
    followingCount: number;
    isFollowingCurrentDj: boolean;
    isSuperfanWithCurrentDj: boolean;
  };

  progress: {
    currentUser: {
      id: string;
      username: string;
      level: number | null;
      currentLevelXp: number | null;
      xpRequired: number | null;
      totalXp: number | null;
      fanCount: number | null;
      infiniteLevel: boolean;
    } | null;
    users: Array<{
      userId: string;
      username: string;
      level: number | null;
      xp: number | null;
      fanCount: number | null;
      infiniteLevel: boolean;
    }>;
  };

  volume: number;

  permissions: {
    vote: boolean;
    joinQueue: boolean;
    sendChat: boolean;
  };
};
```

### Identity fields

- `username`, `rawUsername`, and `handle` contain the account handle used for mentions and lookups.
- `displayUsername` contains the visible display name when one is available.
- `currentUser` is `null` for guest previews and whenever no authenticated room user is available.

### Playback identity

Use `playback.playbackKey` to identify one playback session. It combines the track identity and server start time, so replaying the same track later produces a different key.

The playback snapshot does not expose a `paused` field. Room playback timing is server-based through `startedAtServerMs` and `durationMs`.

## Events

Subscribe to real-time bridge updates with `room.subscribe()`:

```js
const api = window.WavezFM;

const unsubscribe = api.room.subscribe(
  "playback_changed",
  (playback) => {
    if (playback) {
      console.log("Now playing:", playback.title);
    } else {
      console.log("Playback ended");
    }
  },
);

// Remove the listener when the integration is disabled.
unsubscribe();
```

### Supported events

| Event | Callback detail | Description |
| --- | --- | --- |
| `room_changed` | `WavezRoomState["room"] \| null` | Room metadata changed or the room was left |
| `playback_changed` | `WavezRoomState["playback"]` | Playback changed, ended, or was cleared |
| `votes_changed` | `WavezRoomState["votes"]` | Vote counts or the current user's vote state changed |
| `queue_changed` | `WavezRoomState["queue"]` | Queue members, positions, ETA, or queue state changed |
| `users_changed` | `WavezRoomState["users"]` | The visible room-user list changed |
| `chat_message` | `WavezChatMessage` | A new room chat message was received |
| `social_changed` | `WavezRoomState["social"]` | Following or superfan state changed |
| `progress_changed` | `WavezRoomState["progress"]` | XP, level, or fan data changed |

```ts
type WavezChatMessage = {
  id: string;
  roomId: string;
  userId: string;
  username: string;
  content: string;
  timestamp: string;
  role: string;
  platformRole: string;
  replyToId: string | null;
  system: boolean;
};
```

The subscription callback receives the event detail directly, not the browser `Event` object.

### Raw DOM events

Native DOM listeners are also supported. Event names are available through `api.events` to avoid hardcoded strings.

```js
const api = window.WavezFM;

function handlePlaybackChanged(event) {
  console.log(event.detail);
}

window.addEventListener(
  api.events.playbackChanged,
  handlePlaybackChanged,
);

window.removeEventListener(
  api.events.playbackChanged,
  handlePlaybackChanged,
);
```

Available constants:

| Property | DOM event name |
| --- | --- |
| `api.events.roomChanged` | `WavezFM:room_changed` |
| `api.events.playbackChanged` | `WavezFM:playback_changed` |
| `api.events.votesChanged` | `WavezFM:votes_changed` |
| `api.events.queueChanged` | `WavezFM:queue_changed` |
| `api.events.usersChanged` | `WavezFM:users_changed` |
| `api.events.chatMessage` | `WavezFM:chat_message` |
| `api.events.socialChanged` | `WavezFM:social_changed` |
| `api.events.progressChanged` | `WavezFM:progress_changed` |

## Actions

Actions are synchronous. They return immediately with a local acceptance result; they do not return a `Promise`.

```js
const result = window.WavezFM.actions.vote("woot");

if (!result.ok) {
  console.warn("Vote was not dispatched:", result.code);
}
```

An `ok` result means the room client accepted or dispatched the action. Server-side validation can still reject a WebSocket action afterward. Use live room state and normal WavezFM feedback as the final source of truth.

### Action result

```ts
type WavezActionResultCode =
  | "ok"
  | "unavailable"
  | "missing_room"
  | "missing_playback"
  | "self_vote_not_allowed"
  | "invalid_content"
  | "rejected"
  | "queue_locked"
  | "queue_full"
  | "already_in_queue"
  | "not_in_queue"
  | "current_dj";

type WavezActionResult = {
  ok: boolean;
  code: WavezActionResultCode;
  requestId?: string | null;
  value?: number;
};
```

- `requestId` may be returned for WebSocket actions such as voting and queue changes.
- `value` is returned by `setVolume()` with the final normalized volume.

### Vote

```js
const state = window.WavezFM.room.getState();

if (state?.permissions.vote) {
  const result = window.WavezFM.actions.vote("woot");
  console.log(result);
}
```

Supported values are `"woot"` and `"meh"`. Grab is not exposed as a bridge action because it requires the playlist-selection flow in the WavezFM UI.

Possible action-specific failures:

- `missing_playback`
- `self_vote_not_allowed`

### Join the DJ queue

```js
const result = window.WavezFM.actions.joinQueue();
```

Possible action-specific failures:

- `current_dj`
- `already_in_queue`
- `queue_locked`
- `queue_full`

### Leave the DJ queue

```js
const result = window.WavezFM.actions.leaveQueue();
```

Possible action-specific failure:

- `not_in_queue`

The current DJ may also use this action to leave the booth.

### Send a chat message

```js
const state = window.WavezFM.room.getState();

if (state?.permissions.sendChat) {
  const result = window.WavezFM.actions.sendChat("Hello, room!");
  console.log(result);
}
```

Possible action-specific failures:

- `invalid_content`
- `rejected`

The action uses the current public room-chat context. Guest previews and users without chat permission cannot use it successfully.

### Set room volume

```js
const result = window.WavezFM.actions.setVolume(25);

if (result.ok) {
  console.log("Volume set to", result.value);
}
```

- Input is rounded and clamped to the `0` through `100` range.
- The result's `value` contains the normalized volume.

### Common failures

- Any action can return `unavailable` when the room controller or a required live connection is unavailable.
- Room-dependent actions can return `missing_room` when no active room exists. `setVolume()` only depends on an active room controller and therefore returns `unavailable` when that controller is absent.

Check the current permission flags before showing integration controls:

```js
const { permissions } = window.WavezFM.room.getState() ?? {};

console.log({
  canVote: permissions?.vote === true,
  canJoinQueue: permissions?.joinQueue === true,
  canSendChat: permissions?.sendChat === true,
});
```

## Example: AutoWoot

The following example votes once for each playback session:

```js
(() => {
  const api = window.WavezFM;

  if (!api || api.version !== "1") {
    console.warn("WavezFM room bridge is unavailable");
    return;
  }

  let lastPlaybackKey = null;

  function voteForPlayback(playback = api.room.getState()?.playback ?? null) {
    if (!playback || playback.playbackKey === lastPlaybackKey) {
      return;
    }

    lastPlaybackKey = playback.playbackKey;

    const state = api.room.getState();
    if (!state?.votes.canVote || state.votes.clientVote === "woot") {
      return;
    }

    const result = api.actions.vote("woot");

    if (!result.ok) {
      console.warn("AutoWoot was not dispatched:", result.code);
    }
  }

  voteForPlayback();

  const unsubscribe = api.room.subscribe(
    "playback_changed",
    voteForPlayback,
  );

  // Call unsubscribe() when the integration is disabled.
})();
```

## Best practices

- Prefer `window.WavezFM` over DOM queries and simulated UI interaction.
- Check `api.version` before using the bridge.
- Check `room.getState()` for `null` during startup and after room navigation.
- Treat state and event payloads as read-only snapshots.
- Use `playback.playbackKey` instead of title, artist, or `trackId` alone to detect a new playback session.
- Check `state.permissions` before presenting vote, queue, or chat controls.
- Handle every `ok: false` result and branch on the stable `code` value.
- Do not treat a synchronous `ok` result as confirmation that the server persisted the action.
- Unsubscribe from bridge events when the extension or script is disabled.
- Do not automate abusive behavior, spam chat, or bypass room permissions and rate limits.

## Compatibility

`version: "1"` is the compatibility key for this bridge. WavezFM may add optional fields, events, or actions without changing the meaning of existing v1 fields. Integrations should ignore unknown fields and avoid rejecting snapshots that contain additional data.
