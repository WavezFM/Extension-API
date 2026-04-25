---

# 🎧 WavezFM Room Extension API

The **WavezFM Room Extension API** provides a stable, official client-side bridge for browser extensions and user scripts running on WavezFM room pages.

It allows integrations **without relying on DOM queries or UI interaction hacks**.

---

## 🚀 Overview

* Global access via `window.WavezFM`
* Current version: `v1`
* Designed for **extensions, scripts, and integrations**
* Works only inside room pages

---

## 📦 Availability

```js
const api = window.WavezFM;

if (!api || api.version !== '1') {
  console.warn('WavezFM bridge unavailable');
}
```

### Notes

* `window.WavezFM` is only available in room pages
* `api.room.getState()` returns `null` outside a room
* Always check `api.version` for compatibility

---

## 🧠 Room State

Retrieve a **snapshot of the current room state**:

```js
const state = window.WavezFM.room.getState();

if (state) {
  console.log(state.room.name);
  console.log(state.playback?.title);
  console.log(state.votes.woots);
}
```

---

### 🧾 State Structure

```ts
type WavezRoomState = {
  room: {
    id: string;
    slug: string;
    name: string;
    description: string;
    isVerified: boolean;
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
    source: 'youtube' | 'soundcloud';
    sourceId: string;
    title: string;
    artist: string;
    thumbnailUrl: string | null;
    durationMs: number;
    isLive: boolean;
    startedAtServerMs: number;
    paused: boolean;
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
    clientVote: 'woot' | 'meh' | null;
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
      avatar: string | null;
      role: string;
      platformRole: string;
      level: number | null;
      xp: number | null;
      fanCount: number | null;
      infiniteLevel: boolean;
      isFriend: boolean;
      isFollowing: boolean;
      position: number;
      queuedTrackDurationMs: number | null;
      estimatedWaitMs: number | null;
      estimatedWaitKind: 'ready' | 'live' | 'unknown';
    }>;
  };

  users: Array<{
    id: string;
    username: string;
    displayUsername: string | null;
    role: string;
    platformRole: string;
    avatar: string | null;
    level: number | null;
    xp: number | null;
    fanCount: number | null;
    infiniteLevel: boolean;
    isFriend: boolean;
    isFollowing: boolean;
  }>;

  social: {
    friendIds: string[];
    followingIds: string[];
    friendsCount: number;
    followingCount: number;
    isFollowingCurrentDj: boolean;
    isFriendWithCurrentDj: boolean;
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

---

## 🔔 Events

Subscribe to real-time updates:

```js
const unsubscribe = window.WavezFM.room.subscribe(
  'playback_changed',
  (playback) => {
    console.log('Track changed:', playback);
  }
);

// Later
unsubscribe();
```

---

### 📡 Supported Events

| Event Name         | Description                 |
| ------------------ | --------------------------- |
| `room_changed`     | Room metadata updated       |
| `playback_changed` | New track / playback change |
| `votes_changed`    | Vote counts updated         |
| `queue_changed`    | Queue state updated         |
| `users_changed`    | Users list updated          |
| `chat_message`     | New chat message received   |
| `social_changed`   | Social state updated        |
| `progress_changed` | Progress data updated       |

---

### 🧩 Raw DOM Events (Optional)

If you prefer native listeners:

```js
window.addEventListener('WavezFM:playback_changed', (e) => {
  console.log(e.detail);
});
```

Available events:

* `WavezFM:room_changed`
* `WavezFM:playback_changed`
* `WavezFM:votes_changed`
* `WavezFM:queue_changed`
* `WavezFM:users_changed`
* `WavezFM:chat_message`
* `WavezFM:social_changed`
* `WavezFM:progress_changed`

---

## ⚡ Actions

All user-triggered interactions are exposed via:

```js
window.WavezFM.actions
```

---

### 👍 Vote

```js
const result = window.WavezFM.actions.vote('woot');
```

#### Supported values

* `'woot'`
* `'meh'`

#### Result

```ts
type WavezActionResult = {
  ok: boolean;
  code:
    | 'ok'
    | 'unavailable'
    | 'missing_room'
    | 'missing_playback'
    | 'self_vote_not_allowed';
  requestId?: string | null;
};
```

---

### 🎧 Join Queue

```js
window.WavezFM.actions.joinQueue();
```

Possible error codes:

* `current_dj`
* `already_in_queue`
* `queue_locked`
* `queue_full`

---

### 🚪 Leave Queue

```js
window.WavezFM.actions.leaveQueue();
```

Possible error codes:

* `not_in_queue`

---

### 💬 Send Chat Message

```js
window.WavezFM.actions.sendChat('hello room');
```

Possible error codes:

* `invalid_content`
* `rejected`

---

### 🔊 Set Volume

```js
window.WavezFM.actions.setVolume(25);
```

* Range: `0 → 100`
* Values are automatically clamped

---

## 🤖 Example: AutoWoot

Automatically votes once per track:

```js
(() => {
  const api = window.WavezFM;

  if (!api || api.version !== '1') {
    console.warn('WavezFM bridge unavailable');
    return;
  }

  let lastPlaybackKey = null;

  const voteForCurrentTrack = () => {
    const state = api.room.getState();
    const playback = state?.playback;

    if (!playback) return;

    if (playback.playbackKey === lastPlaybackKey) return;

    lastPlaybackKey = playback.playbackKey;

    if (!state.votes.canVote) return;

    const result = api.actions.vote('woot');
    console.log('AutoWoot result:', result);
  };

  voteForCurrentTrack();

  api.room.subscribe('playback_changed', voteForCurrentTrack);
})();
```

---

## 🧠 Best Practices

* Prefer the official bridge API over DOM queries or simulated UI interactions for core actions.
* Use `window.WavezFM.actions` for voting, queue actions, chat, and volume control.
* Use `playback.playbackKey` to detect track changes instead of comparing title or artist text.
* Always handle `ok: false` action results and branch on the returned `code`.
* Use `requestId` only for client-side logging and debugging; it is not a server-side identifier.
* `votes_changed` includes `wootUserIds`, `mehUserIds`, and `grabUserIds` for per-user reaction state.
* `queue_changed` includes detailed queue entries, ETA data, and social/progression metadata.
* `social_changed` exposes friendship and following relationships resolved for the current room session.
* `progress_changed` exposes XP, level, and fan data for the current user and visible room users.
* Treat `version: "1"` as the compatibility key; future bridge versions may add more actions without changing the meaning of v1.

---
