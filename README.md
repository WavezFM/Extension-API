# WavezFM Room Extension API

`window.WavezFM` is the official client-side bridge for browser extensions and user scripts on WavezFM room pages.

This API exists so extensions can integrate with the room without querying or clicking the DOM.

## Availability

- The bridge is available as `window.WavezFM`
- `window.WavezFM.version` is currently `"1"`
- Outside a room page, `window.WavezFM.room.getState()` returns `null`

## Quick Start

```js
const api = window.WavezFM;

if (!api || api.version !== '1') {
  console.warn('WavezFM bridge unavailable');
}
```

## Room State

Use `window.WavezFM.room.getState()` to read the current room snapshot.

```js
const state = window.WavezFM.room.getState();

if (state) {
  console.log(state.room.name);
  console.log(state.playback?.title);
  console.log(state.votes.woots);
}
```

### State shape

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
      estimatedWaitKind: 'ready' | 'live' | 'unknown';
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

## Events

Subscribe with `window.WavezFM.room.subscribe(eventName, handler)`.

```js
const unsubscribe = window.WavezFM.room.subscribe('playback_changed', (playback) => {
  console.log('track changed', playback);
});

// Later:
unsubscribe();
```

### Supported events

- `room_changed`
- `playback_changed`
- `votes_changed`
- `queue_changed`
- `users_changed`
- `chat_message`
- `social_changed`
- `progress_changed`

### Raw DOM events

Extensions that prefer raw `CustomEvent` listeners can also listen to:

- `WavezFM:room_changed`
- `WavezFM:playback_changed`
- `WavezFM:votes_changed`
- `WavezFM:queue_changed`
- `WavezFM:users_changed`
- `WavezFM:chat_message`
- `WavezFM:social_changed`
- `WavezFM:progress_changed`

## Actions

All official extension actions live under `window.WavezFM.actions`.

### Vote

```js
const result = window.WavezFM.actions.vote('woot');
```

Supported vote types:

- `'woot'`
- `'meh'`

Result shape:

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

### Join queue

```js
window.WavezFM.actions.joinQueue();
```

Possible non-success codes:

- `current_dj`
- `already_in_queue`
- `queue_locked`
- `queue_full`

### Leave queue

```js
window.WavezFM.actions.leaveQueue();
```

Possible non-success codes:

- `not_in_queue`

### Send chat

```js
window.WavezFM.actions.sendChat('hello room');
```

Possible non-success codes:

- `invalid_content`
- `rejected`

### Set volume

```js
window.WavezFM.actions.setVolume(25);
```

The value is clamped to `0-100`.

## Official AutoWoot Example

This example votes once per new playback item, without reading or clicking the DOM.

```js
void (async () => {
  const startedAt = Date.now();
  let api = window.WavezFM;

  while (api?.version !== '1' && Date.now() - startedAt < 10_000) {
    await new Promise((resolve) => window.setTimeout(resolve, 50));
    api = window.WavezFM;
  }

  if (!api || api.version !== '1') {
    console.warn('WavezFM bridge unavailable');
    return;
  }

  let handledPlaybackKey = null;
  let retryTimeoutId = null;

  const scheduleRetry = () => {
    if (retryTimeoutId !== null) {
      return;
    }

    retryTimeoutId = window.setTimeout(() => {
      retryTimeoutId = null;
      voteForCurrentTrack();
    }, 500);
  };

  const voteForCurrentTrack = () => {
    const state = api.room.getState();
    const playback = state?.playback;

    if (!state || !playback || playback.playbackKey === handledPlaybackKey) {
      return;
    }

    if (!state.votes.canVote) {
      if (!state.queue.isCurrentDj) {
        scheduleRetry();
      }
      return;
    }

    const result = api.actions.vote('woot');

    if (result.ok && result.requestId) {
      handledPlaybackKey = playback.playbackKey;
      if (retryTimeoutId !== null) {
        window.clearTimeout(retryTimeoutId);
        retryTimeoutId = null;
      }
      return;
    }

    console.warn('AutoWoot was not dispatched:', result.code);
    scheduleRetry();
  };

  voteForCurrentTrack();

  const unsubscribePlayback = api.room.subscribe(
    'playback_changed',
    voteForCurrentTrack,
  );
  const unsubscribeVotes = api.room.subscribe(
    'votes_changed',
    voteForCurrentTrack,
  );

  // Call both functions and clear retryTimeoutId when the integration is disabled.
  void unsubscribePlayback;
  void unsubscribeVotes;
})();
```

## Notes For Extension Authors

- Do not query room controls by selector when the bridge already exposes the needed action
- Use `playback.playbackKey` to detect track changes instead of only the title
- `votes_changed` now includes the user id arrays for woot, meh and grab
- `queue_changed` now includes detailed queue entries with ETA and social/progression metadata
- `social_changed` exposes superfan/following info already resolved for the current room session
- `progress_changed` exposes XP/level/fan information for the current user and visible room users
- Always handle `ok: false` results
- `requestId` is client-side only and is useful for logging/debugging
- Future bridge versions may add more actions, but `version: "1"` should be treated as the compatibility key
