# MiniChatCore Complete Developer Guide

**Add video chat to your web app in minutes**

Version 0.29 | February 4, 2026

---

## Table of Contents

1. [30-Second Quickstart](#30-second-quickstart)
2. [Core Concepts](#core-concepts)
3. [API Reference](#api-reference)
4. [UI Patterns & Design Suggestions](#ui-patterns--design-suggestions)
5. [Complete Examples](#complete-examples)
   - [Minimal (50 lines)](#minimal-example)
   - [Standard (150 lines)](#standard-example)
   - [Full-Featured (250+ lines)](#full-featured-example)
6. [Appendix A: Video Layout & Interaction Guide](#appendix-a-video-layout--interaction-guide)
7. [Appendix B: VibeLive Color System & Design Tokens](#appendix-b-vibelive-color-system--design-tokens)

---

## 30-Second Quickstart

This is all you need for a working video chat:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Quick Video Chat</title>
</head>
<body>
    <input id="name" placeholder="Your name">
    <input id="room" placeholder="Room code">
    <button onclick="joinRoom()">Join</button>
    <button onclick="chat.toggleVideo()">Toggle Camera</button>
    <div id="videos"></div>

    <script type="importmap">
    {
        "imports": {
            "minichat-core": "https://proto2.makedo.com:8883/v03/scripts/minichat-core.js",
            "config": "https://proto2.makedo.com:8883/v03/scripts/configServer.js"
        }
    }
    </script>

    <script type="module">
        import MiniChatCore from 'minichat-core';

        // For quick testing, use: contextId: 'Kw6w6w6w6w', contextAuthToken: 'Kw6w6w6w6w'
        const chat = window.chat = new MiniChatCore({
            contextId: 'YOUR_CONTEXT_ID',
            contextAuthToken: 'YOUR_CONTEXT_TOKEN'
        });

        // Show remote video when someone joins
        chat.onMemberStreamStart = (memberId, streamType) => {
            const video = document.createElement('video');
            video.autoplay = true;
            document.getElementById('videos').appendChild(video);
            chat.setMemberVideoElement(memberId, streamType, video);
        };

        window.joinRoom = async () => {
            await chat.signupAnonymous(document.getElementById('name').value);
            await chat.enterByRoomCode(document.getElementById('room').value);
            await chat.goLive();
        };
    </script>
</body>
</html>
```

**That's it.** ~40 lines gets you a working video chat. Everything below expands on this foundation.

> **âš ï¸ Module Scope Note**: Functions inside `<script type="module">` are not global. To use them with inline `onclick` handlers, assign to `window`:
> ```javascript
> window.joinRoom = () => { ... };  // âœ… Accessible from onclick
> function joinRoom() { ... }       // âŒ Module-scoped, invisible to HTML
> ```
> The same applies to `chat` â€” note `const chat = window.chat = new MiniChatCore(...)` makes it globally accessible.

---

## Core Concepts

### 1. Event-Driven Architecture

MiniChatCore uses callbacks. You assign functions, and MiniChatCore calls them when things happen:

```javascript
const chat = new MiniChatCore(config);

chat.onLogin = (user) => { /* user authenticated */ };
chat.onMemberJoined = (memberId) => { /* someone entered */ };
chat.onMemberStreamStart = (memberId, streamType) => { /* video arrived */ };
```

This keeps your code reactiveâ€”respond to events, don't poll for state.

### 2. Three Entry Paths

Users enter video chat three ways:

| Path | Use Case | Flow |
|------|----------|------|
| **Login** | Registered users | `login()` â†’ `loadChannels()` â†’ `selectChannel()` â†’ `goLive()` |
| **Create Room** | Start new room | `signupAnonymous()` â†’ `createChannel()` â†’ `enterByRoomCode()` â†’ `goLive()` |
| **Join Room** | Enter via code | `signupAnonymous()` â†’ `enterByRoomCode()` â†’ `goLive()` |

**Note:** If already logged in, skip `signupAnonymous()`â€”the other methods work with your existing session.

### 3. Member Lifecycle: LOBBY â†’ LIVE â†’ EXIT

Members progress through states:

```
LOBBY    â†’    LIVE    â†’    LOBBY or EXIT
(observing)  (streaming)   (back or gone)
```

- **LOBBY**: Channel selected, can see other members, no WebRTC yet
- **LIVE**: WebRTC connected, sending/receiving media (`goLive()`)
- **EXIT**: Fully departed, camera released (`exitChannel()`)

Use `returnToLobby()` to disconnect WebRTC while staying in the channel (quick rejoin).

### 4. Video Element Management

MiniChatCore manages `<video>` element bindings automatically:

```javascript
// Your local camera preview (with optional placeholder for auto-hide)
chat.setLocalVideoElement(videoEl, placeholderEl);

// Remote member's stream (in onMemberStreamStart)
chat.setMemberVideoElement(memberId, streamType, videoElement);
```

When you pass a placeholder element, MiniChatCore will automatically:
- Show the video and hide the placeholder when camera is enabled
- Hide the video and show the placeholder when camera is disabled

You create the `<video>` elements; MiniChatCore attaches the streams.

### 5. Mute vs Stop

Two different concepts:

| Action | What Happens | Camera Light | Others See |
|--------|--------------|--------------|------------|
| `toggleVideo()` | Start/stop capture | On/Off | Video appears/disappears |
| `toggleMuteVideo()` | Hide while capturing | Stays On | Black frame |

Same pattern for audio with `toggleAudio()` and `toggleMuteAudio()`.

---

## API Reference

### Constructor

```javascript
const chat = new MiniChatCore({
    contextId: 'YOUR_CONTEXT_ID',        // Required for anonymous signup
    contextAuthToken: 'YOUR_CONTEXT_TOKEN'
});
```

> **ðŸ§ª Test Credentials**: For quick experimentation, you can use `contextId: 'Kw6w6w6w6w'` and `contextAuthToken: 'Kw6w6w6w6w'`. These are shared test credentials â€” use your own context for production.

### Authentication Methods

| Method | Description |
|--------|-------------|
| `login(email, password)` | Login with existing account |
| `signupAnonymous(username)` | Create anonymous guest account |
| `logout()` | Full logout and cleanup |

### Channel Methods

| Method | Description |
|--------|-------------|
| `loadChannels()` | Fetch user's channels (triggers `onChannelsLoaded`) |
| `selectChannel(channelId)` | Select from loaded channels (enters LOBBY) |
| `createChannel({ title, allowsGuests })` | Create new room, returns `{ id, room_code, title }` |
| `enterByRoomCode(roomCode)` | Enter room via shareable code (enters LOBBY) |
| `enterByChannelId(channelId)` | Enter room via database ID (enters LOBBY) |
| `getChannelByRoomCode(roomCode)` | Fetch channel info without entering |
| `getChannelById(channelId)` | Fetch channel info by ID |

### Connection Methods

| Method | Description |
|--------|-------------|
| `goLive()` | Connect WebRTC, go from LOBBY to LIVE |
| `returnToLobby()` | Disconnect WebRTC, return to LOBBY |
| `exitChannel()` | Full teardown, release camera/mic |

### Media Control Methods

| Method | Description |
|--------|-------------|
| `toggleAudio()` | Start/stop microphone |
| `toggleVideo()` | Start/stop camera |
| `toggleScreencast()` | Start/stop screen share |
| `toggleMuteAudio()` | Mute/unmute mic (while capturing) |
| `toggleMuteVideo()` | Hide/show video (while capturing) |

### Member Methods

| Method | Description |
|--------|-------------|
| `getMember(memberId)` | Get member info object |
| `getMemberIds()` | Get array of all member IDs |
| `getMemberMediaStates(memberId)` | Get detailed media states |
| `getCurrentChannelMembers()` | Fetch all members from server |
| `getDisplayStatus(rawStatus)` | Convert status â†’ `'ACTIVE'` / `'LOBBY'` / `'INACTIVE'` |
| `setMemberVideoElement(memberId, streamType, videoEl)` | Bind video element |
| `clearMemberVideoElement(memberId, streamType)` | Unbind video element |

### DOM Management Methods

| Method | Description |
|--------|-------------|
| `setLocalVideoElement(videoEl, placeholderEl?)` | Set local camera preview element |

### Getters

| Property | Type | Description |
|----------|------|-------------|
| `isLoggedIn` | boolean | User authenticated? |
| `isInChannel` | boolean | WebRTC connected (LIVE)? |
| `user` | Object | Current user info |
| `currentMemberId` | string | Your member ID in current channel |
| `currentRoomCode` | string | Shareable room code |
| `selectedChannel` | Object | Current channel info |
| `localMediaState` | Object | `{ audio, video, audioMuted, videoMuted }` |
| `screencastState` | Object | `{ video, videoMuted }` |

### Event Callbacks

Set these to receive notifications:

```javascript
// Authentication
chat.onLogin = (user) => { };
chat.onLoginError = (error) => { };
chat.onLogout = () => { };

// Data Loading
chat.onChannelsLoaded = (channels) => { };
chat.onUsersLoaded = (users) => { };
chat.onChannelSelected = (channel) => { };

// Connection State
chat.onJoined = () => { };    // You went LIVE
chat.onLeft = () => { };      // You returned to LOBBY

// Member Events
chat.onMemberJoined = (memberId) => { };              // Someone went LIVE
chat.onMemberLeft = (memberId) => { };                // Someone left LIVE (may be in LOBBY)
chat.onMemberUpdate = (memberId) => { };              // Status/info changed (includes self!)
chat.onMemberStreamStart = (memberId, streamType) => { };  // Video available
chat.onMemberStreamEnd = (memberId, streamType) => { };    // Video stopped
chat.onMemberMediaChange = (memberId, streamType) => { };  // Mute/unmute

// Local State
chat.onLocalMediaChange = () => { };

// Errors
chat.onError = (context, error) => { };
```

**Important:** `streamType` is either `"camera"` or `"screencast"`.

**Important:** `onMemberUpdate` fires for ALL members including yourselfâ€”useful for updating your own status badge.

---

## UI Patterns & Design Suggestions

> **ðŸ“ Design Guidelines**: For comprehensive visual design standards including layout patterns, color systems, and interaction principles, see:
> - [Appendix A: Video Layout & Interaction Guide](#appendix-a-video-layout--interaction-guide)
> - [Appendix B: VibeLive Color System & Design Tokens](#appendix-b-vibelive-color-system--design-tokens)

### Design Decision: When to Show Members

**Option A: Video-Only Grid**
Show tiles only when video streams exist. Simplest approach.
```javascript
chat.onMemberStreamStart = (id, type) => { createTile(id, type); };
chat.onMemberStreamEnd = (id, type) => { removeTile(id, type); };
```

**Option B: Member-Aware Grid** *(Recommended)*
Show all members immediately with placeholders, replace with video when available.
```javascript
chat.onChannelSelected = async (channel) => {
    const members = await chat.getCurrentChannelMembers();
    members.forEach(m => createPlaceholder(m.id, m.display_name));
};
chat.onMemberStreamStart = (id, type) => { showVideoInTile(id, type); };
```

**Option C: Sidebar + Video**
Member list in sidebar, main area for active video.

### Design Decision: Status Indicators

Raw status codes â†’ display labels:
```javascript
const status = chat.getDisplayStatus(member.status);
// Returns: 'ACTIVE' (streaming), 'LOBBY' (observing), 'INACTIVE' (exited)
```

Consider visual indicators:
- ðŸŸ¢ ACTIVE â€” green dot or "LIVE" badge
- ðŸŸ¡ LOBBY â€” yellow dot or "Waiting" text  
- âšª INACTIVE â€” grayed out or hidden

### Design Decision: Mute Indicators

Show audio/video state per member:
```javascript
const states = chat.getMemberMediaStates(memberId);
// states.cam_audio_detail: 'ON' | 'MUTED' | 'OFF'
// states.cam_video_detail: 'ON' | 'HIDDEN' | 'OFF'
```

Common patterns:
- ðŸŽ¤ / ðŸ”‡ for audio
- ðŸ“¹ / ðŸ“· (with slash) for video
- Color coding: green=on, yellow=muted, red=off

### Design Decision: LOBBY vs LIVE UI

**Unified View**: Same UI, just enable/disable controls based on `chat.isInChannel`.

**Two-Phase View**: Show lobby "waiting room" first, transition to video grid after `join()`.

### Design Decision: Mobile Considerations

- Use `playsInline` on all `<video>` elements (required for iOS)
- Consider portrait vs landscape tile layouts
- Touch-friendly button sizes (44px minimum)
- Handle visibility changes (browser tab switches)

### Design Decision: Self-View Position

Two main approaches for displaying your own camera:

#### Approach A: Unified Grid *(Recommended)*

Your local video appears as the first tile in the same grid as remote members. This creates a natural, symmetric layout where everyone is equal.

```javascript
chat.onChannelSelected = async (channel) => {
    const members = await chat.getCurrentChannelMembers();
    
    // Create self tile FIRST so it appears at the start of the grid
    const selfMember = members.find(m => m.id === chat.currentMemberId);
    if (selfMember) {
        createTile(selfMember.id, 'You', 'LOBBY', true);  // isLocal = true
    }
    
    // Then create tiles for other members
    members.forEach(m => {
        if (m.id === chat.currentMemberId) return;  // Skip self, already created
        createTile(m.id, m.display_name, 'LOBBY', false);
    });
};

function createTile(memberId, name, status, isLocal) {
    const tile = document.createElement('div');
    tile.id = `tile-${memberId}`;
    
    const placeholder = document.createElement('div');
    placeholder.className = 'placeholder';
    placeholder.innerHTML = `<span class="initials">${getInitials(name)}</span>`;
    
    const cameraVideo = document.createElement('video');
    cameraVideo.autoplay = true;
    cameraVideo.playsInline = true;
    cameraVideo.muted = isLocal;  // Always mute self to prevent echo
    
    tile.appendChild(placeholder);
    tile.appendChild(cameraVideo);
    document.getElementById('videoGrid').appendChild(tile);
    
    // KEY: Register local video element with placeholder
    if (isLocal) {
        chat.setLocalVideoElement(cameraVideo, placeholder);
    }
}
```

**Key Points:**
- Pass **both** the video element AND placeholder to `setLocalVideoElement(videoEl, placeholderEl)`
- MiniChatCore automatically shows video and hides placeholder when camera is enabled
- Always set `muted = true` on the self video to prevent audio feedback
- Skip self in `onMemberStreamStart` - local video is attached directly, not via WebRTC events

```javascript
chat.onMemberStreamStart = (memberId, streamType) => {
    // Self video is handled by setLocalVideoElement, not this callback
    if (memberId === chat.currentMemberId) return;
    
    // Handle remote member streams...
};
```

**Pro**: Symmetric, intuitive UI. Self and remote tiles share the same layout.  
**Con**: Self tile uses grid space (consider CSS adjustments for 1-on-1 calls).

#### Approach B: Separate Section

Local video in a dedicated area outside the grid. Remote members in a separate grid.

```html
<div id="localPreview">
    <video id="localVideo" autoplay playsinline muted></video>
</div>
<div id="remoteGrid">
    <!-- Remote tiles dynamically created here -->
</div>
```

```javascript
// Register once at startup
chat.setLocalVideoElement(document.getElementById('localVideo'));

// Skip self when creating tiles
chat.onMemberJoined = (memberId) => {
    if (memberId === chat.currentMemberId) return;  // Don't create self tile
    createRemoteTile(memberId);
};
```

**Pro**: Clear separation, easy picture-in-picture styling.  
**Con**: Asymmetric layout, requires separate CSS for local vs remote.

#### Important: Local Video is Direct, Not Event-Driven

Unlike remote streams which arrive via WebRTC events, local video is attached **directly** when you enable your camera:

```
toggleVideo() â†’ createLocalMedia() â†’ stream attached to #localVideoElement
```

This happens immediately - no waiting for `onMemberStreamStart`. The callback only fires for **remote** member streams.

### Design Decision: Screen Sharing

When a member shares their screen, `onMemberStreamStart` fires with `streamType: 'screencast'`. You need to decide how to display it.

**Approach A: Single Tile, Two Video Elements** *(Recommended for simple UIs)*

Each tile has both a camera and screencast `<video>` element. Screencast takes visual priority.

```javascript
function createTile(memberId, name, isLocal) {
    const tile = document.createElement('div');
    tile.id = `tile-${memberId}`;
    
    const cameraVideo = document.createElement('video');
    cameraVideo.className = 'camera-video';
    
    const screenVideo = document.createElement('video');
    screenVideo.className = 'screen-video';
    
    // ... append both to tile
}

chat.onMemberStreamStart = (memberId, streamType) => {
    const tile = document.getElementById(`tile-${memberId}`);
    const cameraVideo = tile.querySelector('.camera-video');
    const screenVideo = tile.querySelector('.screen-video');
    
    if (streamType === 'camera') {
        const hasScreencast = screenVideo.style.display === 'block';
        cameraVideo.style.display = hasScreencast ? 'none' : 'block';
        chat.setMemberVideoElement(memberId, 'camera', cameraVideo);
    } else {
        // Screencast takes priority
        cameraVideo.style.display = 'none';
        screenVideo.style.display = 'block';
        chat.setMemberVideoElement(memberId, 'screencast', screenVideo);
    }
};

chat.onMemberStreamEnd = (memberId, streamType) => {
    const tile = document.getElementById(`tile-${memberId}`);
    const cameraVideo = tile.querySelector('.camera-video');
    const screenVideo = tile.querySelector('.screen-video');
    const member = chat.getMember(memberId);
    
    if (streamType === 'screencast') {
        screenVideo.style.display = 'none';
        // Show camera if still active
        if (member?.hasCamera) cameraVideo.style.display = 'block';
    } else {
        cameraVideo.style.display = 'none';
    }
};
```

**Pro**: Camera stream stays attached; instant switch when screencast ends.  
**Con**: Can't view both simultaneously.

**Approach B: Separate Tiles per Stream Type**

Create distinct tiles: `tile-${memberId}-camera` and `tile-${memberId}-screencast`.

```javascript
function createTile(memberId, streamType, name) {
    const tile = document.createElement('div');
    tile.id = `tile-${memberId}-${streamType}`;
    // ... single video element
}

chat.onMemberStreamStart = (memberId, streamType) => {
    let tile = document.getElementById(`tile-${memberId}-${streamType}`);
    if (!tile) {
        createTile(memberId, streamType, member?.display_name);
        tile = document.getElementById(`tile-${memberId}-${streamType}`);
    }
    // ... show video
};
```

**Pro**: See camera and screen share side-by-side.  
**Con**: More tiles, more complex layout.

**Approach C: Click to Toggle** *(Enhancement for Approach A)*

Let users click a tile to switch between camera and screencast views:

```javascript
tile.addEventListener('click', () => {
    const member = chat.getMember(memberId);
    if (!member?.hasCamera || !member?.hasScreencast) return;
    
    // Toggle visibility
    const showingCamera = cameraVideo.style.display === 'block';
    cameraVideo.style.display = showingCamera ? 'none' : 'block';
    screenVideo.style.display = showingCamera ? 'block' : 'none';
});
```

---

## Complete Examples

### Minimal Example

~50 lines. Anonymous join by room code, camera toggle, basic video display.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>MiniChat - Minimal</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        video { width: 300px; height: 200px; background: #000; margin: 5px; }
        button { padding: 10px 20px; margin: 5px; }
        input { padding: 10px; margin: 5px; width: 150px; }
    </style>
</head>
<body>
    <h1>Minimal Video Chat</h1>
    
    <div id="setup">
        <input id="username" placeholder="Your name">
        <input id="roomCode" placeholder="Room code">
        <button onclick="joinRoom()">Join Room</button>
        <button onclick="createRoom()">Create Room</button>
    </div>
    
    <div id="controls" style="display:none;">
        <button onclick="chat.toggleVideo()">Toggle Camera</button>
        <button onclick="chat.toggleAudio()">Toggle Mic</button>
        <button onclick="chat.goLive()">Go Live</button>
        <span id="status"></span>
    </div>
    
    <div id="videos" style="display:flex; flex-wrap:wrap;"></div>

    <script type="importmap">
    {
        "imports": {
            "minichat-core": "https://proto2.makedo.com:8883/v03/scripts/minichat-core.js",
            "config": "https://proto2.makedo.com:8883/v03/scripts/configServer.js"
        }
    }
    </script>

    <script type="module">
        import MiniChatCore from 'minichat-core';

        const chat = window.chat = new MiniChatCore({
            contextId: 'YOUR_CONTEXT_ID',
            contextAuthToken: 'YOUR_CONTEXT_TOKEN'
        });

        // Wire up local video
        const localVideo = document.createElement('video');
        localVideo.autoplay = true;
        localVideo.muted = true;
        localVideo.playsInline = true;
        document.getElementById('videos').appendChild(localVideo);
        chat.setLocalVideoElement(localVideo);

        // Events
        chat.onJoined = () => { 
            document.getElementById('status').textContent = 'ðŸ”´ LIVE'; 
        };
        
        chat.onLeft = () => { 
            document.getElementById('status').textContent = 'ðŸ‘¥ LOBBY'; 
        };

        chat.onMemberStreamStart = (memberId, streamType) => {
            const video = document.createElement('video');
            video.id = `video-${memberId}-${streamType}`;
            video.autoplay = true;
            video.playsInline = true;
            document.getElementById('videos').appendChild(video);
            chat.setMemberVideoElement(memberId, streamType, video);
        };

        chat.onMemberStreamEnd = (memberId, streamType) => {
            const video = document.getElementById(`video-${memberId}-${streamType}`);
            if (video) video.remove();
        };

        chat.onError = (ctx, err) => alert(`Error: ${err.message}`);

        // Actions
        window.joinRoom = async () => {
            const name = document.getElementById('username').value || 'Guest';
            const code = document.getElementById('roomCode').value;
            if (!code) return alert('Enter a room code');
            
            await chat.signupAnonymous(name);
            await chat.enterByRoomCode(code);
            
            document.getElementById('setup').style.display = 'none';
            document.getElementById('controls').style.display = 'block';
            document.getElementById('status').textContent = 'ðŸ‘¥ LOBBY';
        };

        window.createRoom = async () => {
            const name = document.getElementById('username').value || 'Guest';
            await chat.signupAnonymous(name);
            const channel = await chat.createChannel({ title: `${name}'s Room` });
            alert(`Room created! Code: ${channel.room_code}`);
            await chat.enterByRoomCode(channel.room_code);
            
            document.getElementById('setup').style.display = 'none';
            document.getElementById('controls').style.display = 'block';
            document.getElementById('status').textContent = 'ðŸ‘¥ LOBBY';
        };
    </script>
</body>
</html>
```

---

### Standard Example

~150 lines. Adds member placeholders, status badges, mute indicators, proper LOBBY/LIVE flow.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>MiniChat - Standard</title>
    <style>
        body { font-family: sans-serif; padding: 20px; max-width: 900px; margin: 0 auto; }
        .controls { margin: 15px 0; padding: 15px; background: #f5f5f5; border-radius: 8px; }
        button { padding: 10px 20px; margin: 5px; cursor: pointer; }
        button:disabled { opacity: 0.5; }
        input { padding: 10px; margin: 5px; width: 150px; }
        
        .video-grid { display: flex; flex-wrap: wrap; gap: 10px; }
        .video-tile { 
            width: 280px; height: 200px; 
            position: relative; 
            border-radius: 8px; 
            overflow: hidden;
        }
        .video-tile video { 
            width: 100%; height: 100%; 
            object-fit: cover; 
            background: #000; 
        }
        .video-tile .placeholder {
            position: absolute; inset: 0;
            display: flex; flex-direction: column;
            align-items: center; justify-content: center;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .video-tile .initials { font-size: 48px; font-weight: bold; }
        .video-tile .name { font-size: 14px; margin-top: 8px; }
        .video-tile .badge {
            position: absolute; top: 8px; right: 8px;
            padding: 4px 8px; border-radius: 4px;
            font-size: 11px; font-weight: bold;
        }
        .badge.active { background: #22c55e; color: white; }
        .badge.lobby { background: #eab308; color: black; }
        .badge.inactive { background: #6b7280; color: white; }
        
        .video-tile .media-state {
            position: absolute; bottom: 8px; left: 8px;
            background: rgba(0,0,0,0.7); color: white;
            padding: 4px 8px; border-radius: 4px; font-size: 12px;
        }
        .on { color: #4ade80; }
        .muted { color: #facc15; }
        .off { color: #f87171; }
    </style>
</head>
<body>
    <h1>ðŸŽ¥ Standard Video Chat</h1>

    <!-- Entry -->
    <div id="entry" class="controls">
        <input id="username" placeholder="Your name">
        <input id="roomCode" placeholder="Room code (or leave blank)">
        <button onclick="startChat()">Join or Create Room</button>
    </div>

    <!-- Media & Connection Controls -->
    <div id="mediaControls" class="controls" style="display:none;">
        <button onclick="chat.toggleVideo()">ðŸ“¹ Camera</button>
        <button onclick="chat.toggleAudio()">ðŸŽ¤ Mic</button>
        <button onclick="chat.toggleMuteVideo()">Hide Video</button>
        <button onclick="chat.toggleMuteAudio()">Mute Mic</button>
        <button onclick="chat.toggleScreencast()">ðŸ–¥ï¸ Screen</button>
        <span style="margin-left: 20px;">
            Local: ðŸŽ¤ <span id="localAudio" class="off">OFF</span> 
            ðŸ“¹ <span id="localVideo" class="off">OFF</span>
        </span>
    </div>

    <div id="connectionControls" class="controls" style="display:none;">
        <strong id="roomInfo"></strong>
        <button id="joinBtn" onclick="chat.goLive()">â–¶ï¸ Go Live</button>
        <button id="lobbyBtn" onclick="chat.returnToLobby()" disabled>â¸ï¸ Return to Lobby</button>
        <button onclick="exitFully()">ðŸšª Exit</button>
        <strong id="connectionStatus">LOBBY</strong>
    </div>

    <!-- Video Grid -->
    <div id="videoGrid" class="video-grid"></div>

    <script type="importmap">
    {
        "imports": {
            "minichat-core": "https://proto2.makedo.com:8883/v03/scripts/minichat-core.js",
            "config": "https://proto2.makedo.com:8883/v03/scripts/configServer.js"
        }
    }
    </script>

    <script type="module">
        import MiniChatCore from 'minichat-core';

        const chat = window.chat = new MiniChatCore({
            contextId: 'YOUR_CONTEXT_ID',
            contextAuthToken: 'YOUR_CONTEXT_TOKEN'
        });

        // ===== EVENT HANDLERS =====

        chat.onChannelSelected = async (channel) => {
            document.getElementById('roomInfo').innerHTML = 
                `Room: <strong>${channel.room_code}</strong>`;
            
            // Fetch all members and create placeholder tiles
            const members = await chat.getCurrentChannelMembers();
            members.forEach(m => {
                const isMe = m.id === chat.currentMemberId;
                createTile(m.id, m.display_name || 'Unknown', 
                          chat.getDisplayStatus(m.status), isMe);
            });
        };

        chat.onJoined = () => {
            document.getElementById('connectionStatus').textContent = 'ðŸ”´ LIVE';
            document.getElementById('joinBtn').disabled = true;
            document.getElementById('lobbyBtn').disabled = false;
        };

        chat.onLeft = () => {
            document.getElementById('connectionStatus').textContent = 'ðŸ‘¥ LOBBY';
            document.getElementById('joinBtn').disabled = false;
            document.getElementById('lobbyBtn').disabled = true;
        };

        chat.onMemberJoined = (memberId) => {
            const m = chat.getMember(memberId);
            if (!document.getElementById(`tile-${memberId}`)) {
                createTile(memberId, m?.display_name || 'Unknown', 'ACTIVE', false);
            }
        };

        chat.onMemberUpdate = (memberId) => {
            const m = chat.getMember(memberId);
            if (!m) return;
            
            // Create tile if it doesn't exist (e.g., new lobby member)
            if (!document.getElementById(`tile-${memberId}`)) {
                const isMe = memberId === chat.currentMemberId;
                createTile(memberId, m.display_name || 'Unknown', 
                          chat.getDisplayStatus(m.member_status), isMe);
                return;
            }
            
            // Update existing tile's badge
            const badge = document.querySelector(`#tile-${memberId} .badge`);
            if (badge) {
                const status = chat.getDisplayStatus(m.member_status);
                badge.textContent = status;
                badge.className = `badge ${status.toLowerCase()}`;
            }
        };

        chat.onMemberLeft = (memberId) => {
            const m = chat.getMember(memberId);
            const status = chat.getDisplayStatus(m?.member_status);
            if (status === 'INACTIVE') {
                document.getElementById(`tile-${memberId}`)?.remove();
            }
        };

        chat.onMemberStreamStart = (memberId, streamType) => {
            const tile = document.getElementById(`tile-${memberId}`);
            if (!tile) return;
            
            const placeholder = tile.querySelector('.placeholder');
            const video = tile.querySelector('video');
            if (placeholder) placeholder.style.display = 'none';
            if (video) {
                video.style.display = 'block';
                chat.setMemberVideoElement(memberId, streamType, video);
            }
            updateMediaIndicator(memberId, streamType);
        };

        chat.onMemberStreamEnd = (memberId, streamType) => {
            const tile = document.getElementById(`tile-${memberId}`);
            if (!tile) return;
            
            const placeholder = tile.querySelector('.placeholder');
            const video = tile.querySelector('video');
            if (video) video.style.display = 'none';
            if (placeholder) placeholder.style.display = 'flex';
        };

        chat.onMemberMediaChange = (memberId, streamType) => {
            updateMediaIndicator(memberId, streamType);
        };

        chat.onLocalMediaChange = () => {
            const s = chat.localMediaState;
            const audioEl = document.getElementById('localAudio');
            const videoEl = document.getElementById('localVideo');
            
            audioEl.textContent = s.audio ? (s.audioMuted ? 'MUTED' : 'ON') : 'OFF';
            audioEl.className = s.audio ? (s.audioMuted ? 'muted' : 'on') : 'off';
            
            videoEl.textContent = s.video ? (s.videoMuted ? 'HIDDEN' : 'ON') : 'OFF';
            videoEl.className = s.video ? (s.videoMuted ? 'muted' : 'on') : 'off';
        };

        chat.onError = (ctx, err) => console.error(`[${ctx}]`, err);

        // ===== HELPER FUNCTIONS =====

        function createTile(memberId, name, status, isLocal) {
            if (document.getElementById(`tile-${memberId}`)) return;
            
            const tile = document.createElement('div');
            tile.className = 'video-tile';
            tile.id = `tile-${memberId}`;
            
            // Placeholder
            const placeholder = document.createElement('div');
            placeholder.className = 'placeholder';
            placeholder.innerHTML = `
                <div class="initials">${getInitials(name)}</div>
                <div class="name">${isLocal ? 'You' : name}</div>
            `;
            
            // Video (hidden initially)
            const video = document.createElement('video');
            video.autoplay = true;
            video.playsInline = true;
            video.muted = isLocal;
            video.style.display = 'none';
            
            // Status badge
            const badge = document.createElement('div');
            badge.className = `badge ${status.toLowerCase()}`;
            badge.textContent = status;
            
            // Media state indicator
            const mediaState = document.createElement('div');
            mediaState.className = 'media-state';
            mediaState.innerHTML = 'ðŸŽ¤ <span class="audio off">--</span> ðŸ“¹ <span class="video off">--</span>';
            mediaState.style.display = 'none';
            
            tile.appendChild(placeholder);
            tile.appendChild(video);
            tile.appendChild(badge);
            tile.appendChild(mediaState);
            document.getElementById('videoGrid').appendChild(tile);
            
            // Register local video element
            if (isLocal) {
                chat.setLocalVideoElement(video);
            }
        }

        function updateMediaIndicator(memberId, streamType) {
            const tile = document.getElementById(`tile-${memberId}`);
            if (!tile) return;
            
            const mediaState = tile.querySelector('.media-state');
            const states = chat.getMemberMediaStates(memberId);
            if (!states) return;
            
            mediaState.style.display = 'block';
            const audioSpan = mediaState.querySelector('.audio');
            const videoSpan = mediaState.querySelector('.video');
            
            const ad = states.cam_audio_detail || 'OFF';
            const vd = states.cam_video_detail || 'OFF';
            
            audioSpan.textContent = ad;
            audioSpan.className = `audio ${ad === 'ON' ? 'on' : ad === 'MUTED' ? 'muted' : 'off'}`;
            videoSpan.textContent = vd;
            videoSpan.className = `video ${vd === 'ON' ? 'on' : vd === 'HIDDEN' ? 'muted' : 'off'}`;
        }

        function getInitials(name) {
            if (!name) return '?';
            const parts = name.trim().split(' ');
            return parts.length >= 2 
                ? (parts[0][0] + parts[parts.length-1][0]).toUpperCase()
                : name.substring(0, 2).toUpperCase();
        }

        // ===== ACTIONS =====

        window.startChat = async () => {
            const name = document.getElementById('username').value || 'Guest';
            const code = document.getElementById('roomCode').value.trim();
            
            await chat.signupAnonymous(name);
            
            if (code) {
                await chat.enterByRoomCode(code);
            } else {
                const channel = await chat.createChannel({ title: `${name}'s Room` });
                await chat.enterByRoomCode(channel.room_code);
            }
            
            document.getElementById('entry').style.display = 'none';
            document.getElementById('mediaControls').style.display = 'block';
            document.getElementById('connectionControls').style.display = 'block';
        };

        window.exitFully = () => {
            chat.exitChannel();
            document.getElementById('videoGrid').innerHTML = '';
            document.getElementById('entry').style.display = 'block';
            document.getElementById('mediaControls').style.display = 'none';
            document.getElementById('connectionControls').style.display = 'none';
        };
    </script>
</body>
</html>
```

---

### Full-Featured Example

The complete reference implementation with all three entry paths (Login / Create / Join), user lists, channel selection, and comprehensive UI handling.

See **[minichat-example.html](minichat-example.html)** for the ~800 line implementation that demonstrates:

- Three-block entry UI (Login / Create Room / Join Room)
- Channel dropdown for logged-in users
- User list for quick 1:1 chats
- Full placeholder tile system with status badges
- Proper LOBBY â†’ LIVE â†’ EXIT flow with button state management
- Event log for debugging
- Complete error handling

Key patterns from the full example:

**Three Entry Blocks:**
```javascript
// Login with existing account
await chat.login(email, password);
await chat.loadChannels();
// â†’ triggers onChannelsLoaded

// Create room (anonymous or logged-in)
if (!chat.isLoggedIn) await chat.signupAnonymous(username);
const channel = await chat.createChannel({ title: roomTitle });
await chat.enterByRoomCode(channel.room_code);

// Join existing room
if (!chat.isLoggedIn) await chat.signupAnonymous(username);
await chat.enterByRoomCode(roomCode);
```

**Fetching Fresh Channel Data:**
```javascript
chat.onChannelSelected = async (channel) => {
    // Always fetch fresh - cached data may be stale
    const freshChannel = await chat.getChannelById(channel.id);
    freshChannel.members.forEach(member => {
        createMemberPlaceholder(member.id, member.display_name, 
                               chat.getDisplayStatus(member.status));
    });
};
```

**Handling LOBBY vs EXIT in onMemberLeft:**
```javascript
chat.onMemberLeft = (memberId) => {
    const member = chat.getMember(memberId);
    const status = chat.getDisplayStatus(member?.member_status);
    
    if (status === 'LOBBY') {
        // Keep tile visible - they're still observing
    } else if (status === 'INACTIVE') {
        // Fully exited - remove tile
        document.getElementById(`tile-${memberId}`)?.remove();
    }
};
```

---

## Tips & Best Practices

1. **Always `await` authentication** â€” `signupAnonymous()` and `login()` must complete before other calls.

2. **Call `goLive()` after entering a room** â€” `enterByRoomCode()` puts you in LOBBY; `goLive()` connects WebRTC.

3. **Create video elements in `onMemberStreamStart`** â€” Don't pre-create them; let the event tell you when streams are ready.

4. **Use `onMemberUpdate` for status badges** â€” It fires for ALL members including yourself.

5. **Check status in `onMemberLeft`** â€” Members may have returned to LOBBY, not exited entirely.

6. **Use `exitChannel()` for full cleanup** â€” Ensures camera light turns off immediately.

7. **Display `chat.currentRoomCode`** â€” Let users share the room code.

8. **Fetch fresh data on channel select** â€” Cached channel data from `loadChannels()` may be stale.

9. **Handle `onError`** â€” During development, log everything to understand the flow.

10. **Use `playsInline` on all video elements** â€” Required for iOS compatibility.

---

## Getting Context Credentials

To use MiniChatCore, you need a `contextId` and `contextAuthToken`. These identify your application and control access.

Contact the Makedo team to obtain your credentials.

---

# Appendix A: Video Layout & Interaction Guide

This appendix defines how to design **video chat experiences** so the result feels *visually balanced, intuitive, and wow-worthy* by default.

---

## A.1 Core Philosophy

1. **Good defaults beat configuration**  
   If the user does not specify a layout, choose the most natural, human-friendly option.

2. **Every participant deserves visual fairness**  
   No one should feel visually secondary unless explicitly designed (e.g., host view).

3. **Space-aware layouts**  
   Video tiles should always feel intentional â€” never cramped, never floating.

4. **Mobile-first logic, desktop-enhanced**  
   Vertical clarity on mobile, spatial balance on larger screens.

---

## A.2 Entry Screen Pattern (Default)

The entry screen should use **separated cards** for clarity:

### Card Structure

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Name Card                  â”‚
â”‚  â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”  â”‚
â”‚  â”‚ Your name             â”‚  â”‚
â”‚  â”‚ [___________________] â”‚  â”‚
â”‚  â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜  â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜

â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Action Card                â”‚
â”‚                             â”‚
â”‚  [ Start a Room ]  (primary)â”‚
â”‚                             â”‚
â”‚  â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€ OR â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€   â”‚
â”‚                             â”‚
â”‚  Join with room code        â”‚
â”‚  [_________] [ Join ]       â”‚
â”‚                             â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

### Dynamic Button Priority

- **No room code entered**: "Start a Room" = primary (accent color), "Join" = secondary
- **Room code entered**: "Join" = primary (accent color), "Start a Room" = secondary

This guides users naturally toward the most relevant action.

### URL Deep Linking

If a room code is in the URL (`?code=XXXXX`):
- Auto-fill the room code input
- Swap button priority to emphasize "Join"

---

## A.3 Core Live Room Actions (Required)

Every video chat room must support the following actions:

| Action | Behavior | Visual Feedback |
|--------|----------|-----------------|
| Join Room | User enters live room | Smooth fade-in + tile expansion |
| Leave Room | User exits room | Tile collapses + reflow animation |
| Video On / Off | Toggle camera | Video â†’ avatar / initials tile |
| Audio On / Off | Toggle mic | Mic icon + subtle muted indicator |
| Screen Share | Share screen | Shared content becomes primary tile |
| Create Room | Start new room | Immediate preview with self-tile |

---

## A.4 Video Tile Rules (Universal)

### Video Tile Anatomy
Each participant tile includes:
- Live video OR fallback avatar
- Name (bottom-left, muted text)
- Audio status icon (mic / muted)
- Optional role indicator (host)

**Never clutter tiles with controls.** Controls live outside the tile.

---

## A.5 Auto Layout Logic (Default Behavior)

When layout preferences are not specified, apply the following rules.

### 1 Participant
- Full screen video
- Centered
- No grid

### 2 Participants

**Desktop / Tablet (Landscape)**
- Two equal tiles
- Split horizontally (50% / 50%)
- Full width usage

**Mobile (Portrait)**
- Vertical stack
- Each tile ~50% height

### 3 Participants

**Desktop**
- One large tile (top)
- Two smaller tiles below (50% / 50%)

**Mobile**
- Vertical stack
- Active speaker slightly emphasized

### 4 Participants

**All Devices**
- 2 Ã— 2 grid
- Equal tile sizes
- Maximize screen usage

### 5â€“6 Participants

**Desktop**
- 3 Ã— 2 grid

**Mobile**
- 2 Ã— 2 visible
- Remaining users accessible via swipe / pagination

### 7+ Participants

- Grid with paging
- Active speaker auto-emphasized
- No tile smaller than readability threshold

---

## A.6 Active Speaker Logic (Soft Emphasis)

By default:
- Detect active speaker via audio
- Apply **subtle emphasis**, not dominance

Examples:
- Slight scale (1.03x)
- Soft border glow
- Gentle background contrast

**Never jump aggressively between speakers.**

---

## A.7 Screen Share Priority Rules

When screen sharing starts:

1. Shared content becomes **primary tile**
2. Participants move to secondary row or strip
3. On mobile:
   - Shared content = main view
   - Speaker tile floats (picture-in-picture)

When screen sharing stops:
- Smoothly restore previous layout

---

## A.8 Video Off / Audio Off States

### Video Off
- Replace video with:
  - User avatar OR
  - Initials on neutral background
- Maintain tile size (no collapsing)

### Audio Muted
- Mic-off icon visible
- No color alarm (stay calm, muted gray)

---

## A.9 Room Code Sharing (Invite Flow)

### Room Header Display

When in a room, display the room code with **two sharing options**:

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚  Room  [ABC123]  [ðŸ“‹]  [ðŸ”—]         Status       â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
         â†‘          â†‘     â†‘
         Code    Copy   Share Link
```

### Sharing Actions

| Button | Action | Feedback |
|--------|--------|----------|
| Copy Code (ðŸ“‹) | Copy room code to clipboard | Checkmark icon briefly |
| Share Link (ðŸ”—) | Copy full URL with `?code=` param | Checkmark icon briefly |

### Visual Feedback

- On copy success: swap icon to checkmark for 2 seconds
- Use subtle color change (e.g., success green)
- No intrusive toast notifications

---

## A.10 Controls Placement

### Primary Controls (Always Visible)
- Mic toggle
- Camera toggle
- Leave room

### Secondary Controls
- Screen share
- Settings
- Participants list

**Placement Guidelines**
- Desktop: bottom center bar
- Mobile: bottom floating bar
- Leave button visually separated (danger affordance)

---

## A.11 Motion & Transitions (WOW Factor)

Use motion sparingly but intentionally:

- Join / leave: smooth scale + fade
- Layout reflow: animated grid adjustment
- Active speaker: gentle pulse or glow

Avoid:
- Hard cuts
- Sudden tile jumps

Motion should feel *alive, not noisy*.

---

## A.12 Accessibility & Comfort

- Never rely on color alone for state
- Ensure readable names at small sizes
- Avoid flashing or aggressive animations
- Respect reduced-motion preferences

---

## A.13 Layout Summary (TL;DR)

- **Entry screen**: Use separated cards (Name card + Action card with dynamic button priority)
- **Room sharing**: Always show both copy code AND share link buttons
- **URL deep linking**: Auto-fill room code from URL params, swap button priority
- If no layout specified â†’ choose the most human, balanced option
- Keep participant tiles visually equal by default
- Optimize for mobile first, enhance for desktop
- Prioritize calm clarity over flashy effects
- Make joining feel welcoming, leaving feel graceful

---

# Appendix B: VibeLive Color System & Design Tokens

**Theme / Brand color:** `#0EA5A4` (VibeLive Teal)  
**Style:** modern, minimal, calm surfaces + teal accents  
**Default radius:** `16px`  
**Max content width:** `980px`  
**Font stack:** `ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial`

---

## B.1 Core Principles

1. **Neutral first, accent second:** most UI is gray/white; teal is for actions + meaning.
2. **One primary action per screen:** only one primary (teal) button in a view when possible.
3. **Soft feedback:** hover/selected states use subtle tints, not loud colors.
4. **Readable hierarchy:** clear headline/body/muted layers.

---

## B.2 Essential Color Tokens

Use **only these** unless you have a special reason.

### Surfaces
| Token | Value | Usage |
|-------|-------|-------|
| `--bg` | `#f7f9f8` | Main background |
| `--card` | `#ffffff` | Cards, panels, modals |
| `--soft` | `#f0f4f3` | Hover surface, subtle fill |
| `--border` | `#e2e8f0` | Dividers, outlines |
| `--borderSoft` | `#dbe2ea` | Hover/emphasis borders |

### Text
| Token | Value | Usage |
|-------|-------|-------|
| `--text` | `#1e293b` | Primary text |
| `--muted` | `#64748b` | Secondary text, labels |
| `--muted2` | `#9ca3af` | Tertiary / helper text |

### Brand Accent (Primary)
| Token | Value | Usage |
|-------|-------|-------|
| `--accent` | `#0EA5A4` | Primary buttons, key links |
| `--accentHover` | `#0d9488` | Hover state |
| `--accentSoft` | `rgba(14,165,164,.10)` | Selected background tint |
| `--accentBorder` | `rgba(14,165,164,.35)` | Focus/outline |

### Status (Semantic)

**Live / Online / Enabled** (reserved for presence)
| Token | Value |
|-------|-------|
| `--liveBg` | `#e8f4ed` |
| `--liveBorder` | `#a6c5a3` |
| `--liveText` | `#324252` |

**Disabled / Offline**
| Token | Value |
|-------|-------|
| `--disabledBg` | `#f3f4f6` |
| `--disabledBorder` | `#e5e7eb` |
| `--disabledText` | `#6b7280` |

---

## B.3 Component Rules

### Buttons

**Primary (CTA)**
- Background: `--accent`
- Text: `#ffffff`
- Hover: `--accentHover`
- Use for: "Join Room", "Create Room", "Start Session", "Save", "Confirm"

**Secondary**
- Background: `--soft`
- Text: `--text`
- Border: `--border`
- Use for: "Cancel", "Back", "Close"

**Danger (Rare)**
- Background: `#FEF2F2`
- Border: `#fca5a5`
- Text: `#b91c1c`
- Use for: destructive confirmations like "End session", "Remove participant"

### Links
- Default: `--accent`
- Hover: underline (don't change color dramatically)

### Inputs
- Background: `--card`
- Border: `--border`
- Focus ring: `--accentBorder` (soft teal glow)
- Error state: use the Danger colors above

### Cards
- Background: `--card`
- Border: `1px solid --border`
- Hover: border becomes `--borderSoft` OR add a very soft shadow

### Pills / Badges
Keep it **semantic**, not decorative:
- **Live**: `--liveBg / --liveBorder / --liveText`
- **Primary tag**: `--accentSoft` background + `--accent` text
- **Disabled**: `--disabledBg / --disabledBorder / --disabledText`

---

## B.4 Modals, Toasts, Alerts

**Overlay/backdrop:** `rgba(30, 41, 59, 0.5)`  
**Container:** `--card` with `--border`, radius `16px`

### Alert types
- **Success / Live:** use Live palette
- **Error:** use Danger palette
- **Info:** `--soft` background + `--border` + `--text`

---

## B.5 Iconography

- No emojis in production UI
- Use simple stroke icons (Lucide style)
- Sizes: 20px / 24px
- Default icon color: `currentColor` (inherits text color)
- Use `--muted` for inactive icons, `--accent` for active icons

---

## B.6 Copy-Paste CSS Variables

```css
:root {
  /* layout */
  --radius: 16px;
  --max: 980px;
  --sans: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;

  /* surfaces */
  --bg: #f7f9f8;
  --card: #ffffff;
  --soft: #f0f4f3;
  --border: #e2e8f0;
  --borderSoft: #dbe2ea;

  /* text */
  --text: #1e293b;
  --muted: #64748b;
  --muted2: #9ca3af;

  /* brand */
  --accent: #0EA5A4;
  --accentHover: #0d9488;
  --accentSoft: rgba(14,165,164,.10);
  --accentBorder: rgba(14,165,164,.35);

  /* status */
  --liveBg: #e8f4ed;
  --liveBorder: #a6c5a3;
  --liveText: #324252;

  --disabledBg: #f3f4f6;
  --disabledBorder: #e5e7eb;
  --disabledText: #6b7280;

  /* overlay */
  --overlay: rgba(30, 41, 59, 0.5);
}
```

---

## B.7 Do / Don't Guidelines

**Do**
- Use teal primarily for key actions and emphasis
- Keep backgrounds neutral and clean
- Use Live green only for real "live/online/enabled" states

**Don't**
- Add random new colors for every feature
- Use multiple teal CTAs on the same screen
- Use emojis as UI icons in production

---

*MiniChatCore v0.29 | For questions, contact the Makedo team.*