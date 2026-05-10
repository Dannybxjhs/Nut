# Nut Cracker - Tapping Game Specification

## 1. Project Overview

**Project Name:** Nut Cracker
**Type:** Mobile and Desktop Tapping Game
**Core Functionality:** Players tap on a nut positioned on a plate to score points, with a global leaderboard tracking top scores and user authentication system.
**Target Users:** Casual gamers on both mobile and desktop browsers.

## 2. Technology Stack

- **Frontend:** HTML5, CSS3, Vanilla JavaScript
- **Backend:** Node.js with Express.js
- **Database:** In-memory storage with JSON file persistence (leaderboard.json, users.json)
- **Authentication:** bcrypt password hashing (10 salt rounds), express-session with HTTP-only cookies
- **Real-time:** Server-Sent Events (SSE) for live leaderboard updates
- **Audio:** Web Audio API with procedural sound generation
- **Rate Limiting:** express-rate-limit for API protection

## 3. Visual & Rendering Specification

### Scene Setup

- **Canvas:** Full-viewport responsive canvas
- **Background:** Warm wooden table texture gradient with subtle pattern
- **Layout:** Centered nut/plate with score display at top, controls at bottom

### Visual Elements

- **Nut:** SVG-based 3D-styled walnut with realistic crack texture
  - Size: \~120px diameter on desktop, scales with viewport on mobile
  - Animation: Scale pulse on tap, crack texture overlay increases
- **Plate:** Simple ceramic plate SVG beneath the nut
  - Size: \~180px diameter
  - Color: Off-white/cream with subtle shadow
- **Score Display:** Large, bold font at top center
  - Font: "Fredoka" or similar playful rounded font
  - Color: Dark brown (#3E2723) on light background
- **Volume Control:** Speaker icon with slider at bottom

### Color Palette

- **Primary (Wood):** #8B4513 (saddle brown)
- **Secondary (Nut):** #D2691E (chocolate)
- **Accent (Score):** #FFD700 (gold)
- **Background:** Linear gradient #DEB887 to #A0522D
- **Plate:** #FFF8DC (cornsilk) with #D2B48C shadow

### Animations

- **Tap Feedback:** 0.1s scale down (0.9) then spring back
- **Score Increment:** +1 floats up from nut and fades
- **Crack Effect:** SVG crack lines appear progressively
- **Idle Animation:** Subtle nut wobble

## 4. Audio Specification

### Sound Effects

- **Crack Sound:** Short (0.15s) crunchy crack audio
  - Format: Procedurally generated using Web Audio API oscillators
  - Triggered on each successful tap
- **Volume Range:** 0% to 100% with mute option

### Implementation

- Preload audio on game start
- Use Web Audio API for low-latency playback
- Procedural generation using noise + sine wave combination

## 5. Game Mechanics

### Core Loop

1. Player taps/clicks on nut
2. Score increments by 1
3. Crack sound plays
4. Visual feedback (scale animation)
5. Crack texture on nut increases slightly
6. If logged in and score > 0, auto-submit to leaderboard (debounced 500ms)
7. Leaderboard refreshes immediately after successful score submission
8. After login the player can start clicking the nut and The leaderboard automatically updates immediately

### Hit Detection

- Circular hitbox centered on nut
- Radius: nut size / 2 + 15px padding
- Tolerance for touch devices: additional 15px padding

### Game Session

- Game starts immediately on page load
- No time limit (endless mode)
- Restart button clears current session score but not submitted score

## 6. Authentication System

### Login/Signup Page

- **URL:** /login.html
- **Code Name Field:** 3-15 characters, alphanumeric and underscore only
- **Password Field:** Minimum 4 characters, maximum 50 characters
- **Confirm Password:** Required for signup, must match password exactly
- **Real-time Validation:** Username availability check on input (300ms debounce)
- **Loading States:** Spinner during submission

### Security Implementation

- **Password Hashing:** bcrypt with 10 salt rounds
- **Rate Limiting:**
  - Signup: 5 attempts per IP per minute
  - Login: 10 attempts per IP per minute
- **Session Management:** express-session with HTTP-only, sameSite: lax cookies
- **Input Sanitization:** All user inputs sanitized before storage using sanitizeString()
- **CORS Configuration:** Explicit Allow-Credentials header for cross-origin session cookies

### Session Configuration (server.js)

```javascript
const sessionMiddleware = session({
  secret: process.env.SESSION_SECRET || 'nut-cracker-static-secret-key-2024',
  resave: false,
  saveUninitialized: true,
  cookie: {
    secure: false,
    httpOnly: true,
    maxAge: 7 * 24 * 60 * 60 * 1000,  // 7 days
    sameSite: 'lax'
  }
});
```

### CORS Middleware (server.js)

```javascript
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', 'http://localhost:3000');
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept, Authorization');
  res.header('Access-Control-Allow-Credentials', 'true');
  if (req.method === 'OPTIONS') {
    return res.sendStatus(200);
  }
  next();
});
```

### User Data Model

```json
{
  "id": "UUID v4 (e.g., xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx)",
  "username": "string (3-15 chars, sanitized, alphanumeric + underscore)",
  "password": "bcrypt hashed string",
  "createdAt": "ISO 8601 date string"
}
```

### API Endpoints

| Method | Endpoint                           | Description                 | Rate Limit |
| ------ | ---------------------------------- | --------------------------- | ---------- |
| POST   | /api/auth/signup                   | Create new account          | 5/min/IP   |
| POST   | /api/auth/login                    | Login with credentials      | 10/min/IP  |
| POST   | /api/auth/logout                   | Logout current session      | None       |
| GET    | /api/auth/me                       | Get current user info       | None       |
| GET    | /api/auth/check-username/:username | Check username availability | None       |

### Validation Functions

```javascript
function validateUsername(username) {
  if (!username || typeof username !== 'string') return false;
  const cleaned = sanitizeString(username, 15);
  return cleaned.length >= 3 && /^[a-zA-Z0-9_]+$/.test(cleaned);
}

function validatePassword(password) {
  if (!password || typeof password !== 'string') return false;
  return password.length >= 4 && password.length <= 50;
}
```

### Username Availability Check

- Triggered on input blur in signup mode
- 300ms debounce on input
- Real-time feedback: "✓ Available" / "✗ Already taken" / "Invalid format"

## 7. Leaderboard System

### API Endpoints

| Method | Endpoint                   | Description                          | Rate Limit |
| ------ | -------------------------- | ------------------------------------ | ---------- |
| GET    | /api/leaderboard           | Retrieve top 100 scores              | None       |
| POST   | /api/score                 | Submit new score                     | 30/min/IP  |
| GET    | /api/leaderboard-stream    | SSE stream for real-time updates     | None       |
| GET    | /api/leaderboard/:playerId | Get specific player's score and rank | None       |

### Score Submission Flow

1. User taps nut → score increments
2. After 500ms debounce, auto-submit triggers (if logged in)
3. Server validates score and user authentication
4. Score updates leaderboard if higher than existing
5. Server broadcasts update to all SSE clients
6. Client receives update via SSE or explicit fetch call
7. Leaderboard UI refreshes immediately showing new score

### Auto-Submit Behavior

```javascript
// In game.js - incrementScore function
if (currentUser && score > 0 && !playerSubmitted) {
  debouncedSubmitScore();
}

// In submitScore function - after successful submission
loadLeaderboard();  // Immediate refresh after submission
```

### Real-time Updates (SSE)

- Server maintains array of connected SSE clients
- On score submission, `broadcastLeaderboard()` sends update to all clients
- Client EventSource receives update and calls `renderLeaderboard()`
- Auto-reconnect on connection loss with 3 second delay

### Data Model

```json
{
  "id": "user_id (UUID)",
  "playerName": "string (sanitized, max 20 chars)",
  "score": "number (positive integer < 1,000,000)",
  "timestamp": "ISO 8601 date string"
}
```

### Score Validation

```javascript
function validateScore(score) {
  const num = parseInt(score, 10);
  return !isNaN(num) && num > 0 && num < 1000000;
}
```

### Score Update Logic

- Only updates if new score is higher than existing score
- Returns current rank even if score not high enough to update
- Updates timestamp on score change

### Display

- Show top 10 scores by default
- Highlight current player's score (CSS class 'highlight')
- Medal emojis for top 3: 🥇🥈🥉
- CSS rank classes: gold, silver, bronze

## 8. UI Components

### Header Section

- Game title "NUT CRACKER"
- Current score display (large, centered)
- Personal best indicator
- Login/Logout link
- User display (when logged in)

### Login Page

- Toggle between Login and Sign Up modes
- Nut logo SVG
- Form validation with error messages
- Real-time username availability feedback
- "Play as Guest" link

### Game Area

- Nut on plate (centered)
- Tap instruction text below nut
- Crack texture overlays (4 levels based on score)

### Controls Section

- Volume slider with icon (0-100%)
- Mute toggle button
- Leaderboard button
- Restart button

### Leaderboard Modal

- Overlay with score list
- Close button (X and click-outside)
- Current score display
- Submit score button (for logged-in users only)
- Player rank display
- Scrollable list (top 10 visible)

## 9. Responsive Design

### Breakpoints

- **Mobile:** < 600px (touch-optimized, larger tap targets)
- **Tablet:** 600px - 1024px
- **Desktop:** > 1024px

### Adaptations

- Nut size scales with viewport
- Touch targets minimum 48px
- Font sizes adjust for readability
- Modal width adapts to screen (90% on mobile, max 500px on desktop)

## 10. File Structure

```
/Nut Cracker/
├── SPEC.md                 (this file)
├── package.json
├── server.js               (Express server with all API routes)
├── users.json              (auto-generated, user accounts)
├── leaderboard.json        (auto-generated, scores)
└── public/
    ├── index.html          (main game page)
    ├── login.html          (authentication page)
    ├── css/
    │   └── style.css       (game styling)
    └── js/
        └── game.js         (game logic, auth, leaderboard)
```

## 11. Acceptance Criteria

### Core Gameplay

1. ✅ Game loads without errors on Chrome, Firefox, Safari, Edge
2. ✅ Tapping nut increments score by exactly 1
3. ✅ Crack sound plays on each tap (procedural Web Audio)
4. ✅ Volume control affects sound (0-100%)
5. ✅ Visual feedback is immediate (< 100ms)

### Authentication

1. ✅ User can create account with unique code name (3-15 chars, alphanumeric/underscore)
2. ✅ User can login with existing credentials
3. ✅ Passwords are securely hashed with bcrypt
4. ✅ Session persists across page navigation via HTTP-only cookies
5. ✅ Form validation prevents invalid input (username format, password length)
6. ✅ Real-time username availability checking during signup
7. ✅ Rate limiting prevents brute force (5 signup/10 login per minute per IP)

### Leaderboard

1. ✅ Leaderboard displays top 10 scores
2. ✅ Scores are automatically submitted when logged in (debounced 500ms)
3. ✅ Leaderboard updates in real-time via SSE broadcast
4. ✅ Leaderboard refreshes immediately after score submission (explicit loadLeaderboard call)
5. ✅ Only higher scores update existing entries
6. ✅ Authentication required for score submission

### Bug Fixes (v2)

1. ✅ Registration no longer redirects to guest mode (CORS credentials + session config fixed)
2. ✅ Session cookies properly sent with cross-origin requests
3. ✅ Auto-update leaderboard after each user click action (not just SSE passive updates)

## 12. Bug Fix History

### Issue: Registration Redirects to Guest Mode

**Root Cause:** Session cookies not being properly sent with cross-origin requests due to missing CORS credentials configuration.

**Solution:**

1. Added explicit CORS middleware with `Access-Control-Allow-Credentials: true`
2. Set `Access-Control-Allow-Origin` to specific origin (not wildcard)
3. Changed session secret from dynamic UUID to static string for consistency
4. Set `saveUninitialized: true` to ensure session is saved on login
5. Added `sameSite: 'lax'` cookie option for proper session handling
6. Added `credentials: 'include'` to all fetch calls in frontend

### Issue: Leaderboard Not Updating After Score Submission

**Root Cause:** Leaderboard only updated via SSE passive broadcasts, not immediately after user actions.

**Solution:**
Added explicit `loadLeaderboard()` call in `submitScore()` function immediately after successful score submission:

```javascript
playerSubmitted = true;

loadLeaderboard();  // Immediate refresh

if (!isAuto) {
  submitScoreBtn.textContent = 'Submitted!';
}
```

## 13. Implementation Notes

### CSS Adjustments Made During Implementation Review

1. **Nut Size Adjustment**: The nut-wrapper dimensions were updated from `clamp(100px, 30vw, 140px)` to `clamp(100px, 25vw, 120px)` to better match the spec's ~120px desktop size while maintaining responsiveness.

2. **Tap Animation Scale**: The nut tap animation scale was corrected from 0.85 to 0.9 to match the spec's "0.1s scale down (0.9)" requirement.

### Verified Implementations

The following game mechanics were verified to be correctly implemented per specification:

- **Core Loop**: Score increment, sound playback, visual feedback, crack level progression
- **Auto-submit**: Debounced 500ms score submission for logged-in users
- **Immediate Leaderboard Refresh**: Explicit `loadLeaderboard()` call after successful score submission
- **Session Authentication**: HTTP-only cookies with proper CORS configuration
- **SSE Real-time Updates**: Broadcast to all connected clients on score changes
- **Hit Detection**: Circular hitbox with radius = nut size/2 + 15px padding
- **Responsive Design**: Mobile-first approach with breakpoints at 600px and 1024px

## 14. Daily Quests System

### Overview

The Daily Quests system provides players with daily goals to complete for rewards. Each day, players receive a set of quests that reset after a 24-hour cycle.

### Quest Types

| Quest ID | Name | Description | Difficulty | Target | Reward | Type |
|----------|------|-------------|------------|--------|--------|------|
| q1 | First Crack | Score your first point | Easy | 1 score | 1 Speed Potion, 1 Gem | score |
| q2 | Nutcracker | Score 50 points | Easy | 50 score | 2 Speed Potions, 2 Gems | score |
| q3 | Speed Demon | Score 200 points | Medium | 200 score | 3 Speed Potions, 5 Gems | score |
| q4 | Master Cracker | Score 500 points | Hard | 500 score | 5 Speed Potions, 10 Gems | score |
| q5 | Legend | Score 1000 points | Expert | 1000 score | 10 Speed Potions, 25 Gems | score |
| q6 | Tap Master | Reach 100 total taps | Easy | 100 taps | 2 Speed Potions, 3 Gems | taps |
| q7 | League Climber | Reach top 3 in your league | Hard | Rank ≤3 | 5 Speed Potions, 15 Gems | league_rank |
| q8 | Persistence | Login 3 days in a row | Medium | 3 day streak | 3 Speed Potions, 8 Gems | streak |
| q9 | Social Butterfly | Make your first friend | Easy | 1 friend | 2 Speed Potions, 5 Gems | friends |
| q10 | Influencer | Receive 5 likes on your feed posts | Medium | 5 likes | 3 Speed Potions, 10 Gems | likes |
| q11 | Champion's Path | Reach the Champion league | Expert | Champion league | 10 Speed Potions, 30 Gems | league_legend |
| q12 | Achievement Hunter | Unlock your first achievement | Easy | 1 achievement | 2 Speed Potions, 5 Gems | achievements |
| q13 | Speed Runner | Score 300 points in one game | Hard | 300 high score | 5 Speed Potions, 15 Gems | high_score |

### Data Model

```json
{
  "userId": "UUID",
  "quests": {
    "q1": { "completed": false, "progress": 0, "claimed": false },
    "q2": { "completed": false, "progress": 0, "claimed": false },
    "q3": { "completed": false, "progress": 0, "claimed": false },
    "q4": { "completed": false, "progress": 0, "claimed": false },
    "q5": { "completed": false, "progress": 0, "claimed": false },
    "q6": { "completed": false, "progress": 0, "claimed": false },
    "q7": { "completed": false, "progress": 0, "claimed": false },
    "q8": { "completed": false, "progress": 0, "claimed": false }
  },
  "lastResetDate": "YYYY-MM-DD",
  "loginStreak": 0,
  "lastLoginDate": "YYYY-MM-DD",
  "totalTaps": 0,
  "inventory": {
    "speedPotions": 0,
    "gems": 0
  }
}
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/quests | Get user's daily quests |
| POST | /api/quests/progress | Update quest progress |
| POST | /api/quests/claim/:questId | Claim quest reward |
| POST | /api/quests/reset | Manually reset quests (admin) |

### Quest Progress Tracking

- Progress automatically updates when player scores
- Progress is cumulative throughout the day
- Completed quests show checkmark and glow effect
- Unclaimed rewards show pulsing indicator

### Daily Reset

- Quests reset at midnight UTC
- Previous day's progress is discarded
- Unclaimed rewards are forfeited
- New quests generated with same objectives

### UI Components

- **Quest Log Button**: Opens quest modal
- **Quest Modal**: Shows all 5 quests with progress bars
- **Progress Bar**: Visual indicator of completion percentage
- **Reward Badge**: Shows potion and gem icons with quantities
- **Claim Button**: Appears when quest is complete, claims reward

## 15. Customization System

### Overview

Players can personalize their game experience with custom backgrounds and avatars.

### Background Customization

- **Upload**: Players can upload JPG, PNG, or WebP images (max 5MB)
- **Preview**: Real-time preview before saving
- **Cropping**: Center-crop to fit viewport
- **Persistence**: Saved to localStorage as base64
- **Default**: Wooden table gradient background

### Avatar Customization

- **Initial Display**: Single character letter
- **Background Colors**: 8 preset color options
- **Custom Image Upload**: Optional avatar image (max 2MB)
- **Image Cropping**: Square crop with circular mask
- **Preview**: Live preview in modal

### Moderation System

- **AI Content Filter**: Basic image analysis for inappropriate content
- **Blacklist**: Server-side blacklist of inappropriate keywords
- **File Validation**: Check file magic bytes, not just extension
- **Size Limits**: Enforce max file sizes strictly
- **Auto-Reject**: Images failing moderation show error message

### Data Model

```json
{
  "userId": "UUID",
  "background": {
    "type": "default" | "custom",
    "data": "base64 encoded image or null"
  },
  "avatar": {
    "initial": "string (1 char)",
    "bgColor": "hex color code",
    "customImage": "base64 encoded or null"
  },
  "moderationStatus": "approved" | "pending" | "rejected"
}
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/customization | Get user's customization settings |
| POST | /api/customization/background | Upload background image |
| POST | /api/customization/avatar | Upload avatar image |
| DELETE | /api/customization/background | Reset to default background |
| DELETE | /api/customization/avatar | Reset to default avatar |

### Image Processing (Client-side)

```javascript
// Canvas-based cropping
function cropImage(img, width, height) {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  const size = Math.min(img.width, img.height);
  canvas.width = width;
  canvas.height = height;
  ctx.drawImage(img,
    (img.width - size) / 2, (img.height - size) / 2,
    size, size,
    0, 0, width, height
  );
  return canvas.toDataURL('image/jpeg', 0.8);
}
```

## 16. Social Features

### Overview

Players can connect with friends, view activity feeds, and interact with other players' achievements.

### Friend System

#### Friend Data Model

```json
{
  "userId": "UUID",
  "friends": ["friendId1", "friendId2"],
  "friendRequests": {
    "incoming": [{ "fromUserId": "uuid", "username": "name", "timestamp": "ISO" }],
    "outgoing": [{ "toUserId": "uuid", "username": "name", "timestamp": "ISO" }]
  }
}
```

#### Friend Search

- Search by exact username match
- Minimum 3 characters to search
- Rate limited: 20 searches per minute

#### Friend Request Flow

1. User searches for friend by username
2. System sends friend request
3. Recipient sees notification badge
4. Recipient accepts or declines
5. Both users added to each other's friends list

### Activity Feed

#### Feed Item Types

| Type | Trigger | Content |
|------|---------|---------|
| score | New high score | "{username} scored {n} points!" |
| quest | Quest completed | "{username} completed {quest name}!" |
| achievement | Achievement unlocked | "{username} unlocked {achievement}!" |
| milestone | Score milestone | "{username} reached {n} total score!" |

#### Feed Data Model

```json
{
  "id": "UUID",
  "userId": "UUID",
  "username": "string",
  "type": "score" | "quest" | "achievement" | "milestone",
  "content": "string",
  "metadata": { "score": 100, "questId": "q3" },
  "timestamp": "ISO 8601",
  "likes": ["userId1", "userId2"],
  "comments": [{ "userId": "uuid", "username": "name", "text": "comment", "timestamp": "ISO" }]
}
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/friends | Get user's friends list |
| POST | /api/friends/request | Send friend request |
| POST | /api/friends/accept/:requestId | Accept friend request |
| POST | /api/friends/decline/:requestId | Decline friend request |
| DELETE | /api/friends/:friendId | Remove friend |
| GET | /api/feed | Get activity feed |
| POST | /api/feed/:itemId/like | Like a feed item |
| DELETE | /api/feed/:itemId/like | Unlike a feed item |
| POST | /api/feed/:itemId/comment | Add comment to feed item |

### Privacy Settings

| Setting | Options | Default |
|---------|---------|---------|
| Show in public feed | on/off | on |
| Show achievements | on/off | on |
| Show score | on/off | on |
| Allow friend requests | on/off | on |

```json
{
  "userId": "UUID",
  "privacy": {
    "publicFeed": true,
    "showAchievements": true,
    "showScore": true,
    "allowFriendRequests": true
  }
}
```

### Notification System

#### Notification Types

| Type | Trigger | Message |
|------|---------|---------|
| friend_request | New friend request | "{username} sent you a friend request" |
| friend_accepted | Request accepted | "{username} accepted your friend request" |
| quest_complete | Daily quest completed | "You completed {quest name}!" |
| milestone | Score milestone | "You reached {n} total points!" |
| feed_like | Someone liked post | "{username} liked your post" |
| feed_comment | Someone commented | "{username} commented on your post" |

#### Notification Data Model

```json
{
  "userId": "UUID",
  "notifications": [
    {
      "id": "UUID",
      "type": "friend_request" | "friend_accepted" | "quest_complete" | "milestone" | "feed_like" | "feed_comment",
      "message": "string",
      "read": false,
      "timestamp": "ISO 8601",
      "data": {}
    }
  ]
}
```

#### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/notifications | Get user notifications |
| POST | /api/notifications/read/:id | Mark notification as read |
| POST | /api/notifications/read-all | Mark all as read |
| GET | /api/notifications/count | Get unread count |

## 17. File Structure (Updated)

```
/Nut Cracker/
├── SPEC.md                 (this file)
├── package.json
├── server.js               (Express server with all API routes)
├── quests.json             (auto-generated, quest progress)
├── users.json              (auto-generated, user accounts)
├── leaderboard.json        (auto-generated, scores)
├── friends.json            (auto-generated, friends data)
├── feed.json               (auto-generated, activity feed)
├── notifications.json      (auto-generated, notifications)
├── leagues.json            (auto-generated, league data)
├── achievements.json       (auto-generated, achievement progress)
├── chests.json            (auto-generated, chest claiming data)
└── public/
    ├── index.html          (main game page)
    ├── login.html          (authentication page)
    ├── css/
    │   └── style.css       (game styling)
    └── js/
        ├── game.js         (game logic, auth, leaderboard)
        ├── quests.js        (daily quests system)
        ├── customization.js  (avatar/background customization)
        ├── social.js        (friends, feed, notifications)
        ├── league.js        (league system UI and logic)
        ├── achievements.js  (achievements UI and logic)
        ├── shop.js          (shop UI and purchase logic)
        ├── chest.js         (daily chest system UI and logic)
        └── imageProcessor.js (client-side image processing)
```

## 18. Acceptance Criteria (Updated)

### Daily Quests

1. ✅ 13 distinct daily quests with varying difficulty
2. ✅ Clear completion criteria displayed for each quest
3. ✅ Visible progress tracking with progress bars
4. ✅ Rewards (speed potions and gems) awarded on completion
5. ✅ Automatic 24-hour reset cycle
6. ✅ Quest log interface showing active/completed/failed quests
7. ✅ Quest types: score-based, taps-based, league rank, and login streak
8. ✅ Dynamic quest generation to prevent duplicates
9. ✅ Validation checks for quest progress tracking

### Customization System

1. ✅ Custom background image upload
2. ✅ Avatar customization with image upload
3. ✅ Image cropping and scaling functionality
4. ✅ Live preview before saving
5. ✅ Basic moderation to prevent inappropriate content
6. ✅ Custom assets persist across sessions

### Social Features

1. ✅ Friend search by username
2. ✅ Friend request/accept/decline functionality
3. ✅ Public activity feed with player achievements
4. ✅ Like and comment on feed posts
5. ✅ Privacy settings for public profile
6. ✅ Notification system for social interactions

## 18. Daily Chest System

### Overview

Players can claim up to 5 treasure chests per day, with the limit resetting at midnight UTC. Each chest contains random gem rewards (1-1000) and at least one magic item from the magic items pool.

### Chest Mechanics

- **Daily Limit**: 5 chests per day
- **Reset Time**: Midnight UTC (00:00)
- **Gem Reward**: Random value between 1 and 1000 (inclusive)
- **Magic Items**: 1-2 items per chest based on rarity

### Magic Items Pool

| Item | Name | Description | Rarity | Duration |
|------|------|-------------|--------|----------|
| lucky_charm | Lucky Charm | Increases gem drops by 10% | common | 1 hour |
| speed_potion | Speed Potion | Doubles tap score for 1 hour | common | 1 hour |
| golden_nut | Golden Nut | 2x score multiplier | rare | 30 min |
| crystal_shard | Crystal Shard | Instant 500 bonus points | rare | - |
| diamond | Diamond | Instant 1000 bonus points | epic | - |
| crown | Royal Crown | Triple score for 30 minutes | legendary | 30 min |
| phoenix_feather | Phoenix Feather | Revive bonus: +50% on next game | rare | - |
| dragon_scale | Dragon Scale | Shield: protects score for 1 hour | epic | 1 hour |
| unicorn_dust | Unicorn Dust | Lucky: 3x gem chance for 30 min | common | 30 min |
| magic_wand | Magic Wand | Auto-tap once every 5 seconds | legendary | 5 min |
| enchanted_hammer | Enchanted Hammer | Crack 3 nuts at once | epic | 10 min |
| mystic_orbs | Mystic Orbs | Orb copies your last 10 taps | rare | - |
| star_fragment | Star Fragment | Instant 200 bonus points | common | - |
| moonstone | Moonstone | Night bonus: +25% score after 6PM | rare | - |
| sapphire_gem | Sapphire Gem | Instant 300 bonus points | common | - |

### Rarity Distribution

| Rarity | Drop Chance | Items per Chest |
|--------|-------------|-----------------|
| Common | 50% | 1-2 |
| Rare | 30% | 1-2 |
| Epic | 15% | 1 |
| Legendary | 5% | 1 |

### Chest Data Model

```json
{
  "userId": "UUID",
  "claimsToday": 0,
  "lastResetDate": "YYYY-MM-DD",
  "totalChestsClaimed": 0,
  "lastClaimTime": "ISO-8601",
  "chestHistory": [],
  "gemsTotal": 0,
  "magicItemsAcquired": []
}
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/chest | Get user's chest status and stats |
| POST | /api/chest/claim | Claim a chest and receive rewards |
| GET | /api/chest/stats | Get detailed chest analytics |
| GET | /api/magic-items | Get user's acquired magic items |

### Chest UI Features

1. **Visual countdown timer** showing time until next chest/reset
2. **Animated chest opening sequence** with bounce and glow effects
3. **Reward reveal popup** with gem count and magic item display
4. **Rarity-based item styling** (common=gray, rare=blue, epic=purple, legendary=gold)
5. **Sound effects** for chest opening and reward reveal
6. **Badge indicator** showing remaining chests on button
7. **Chest history** showing recent chest openings

### Acceptance Criteria

1. ✅ 5 chests available per day with midnight UTC reset
2. ✅ Random gem rewards between 1-1000
3. ✅ At least 1 magic item per chest
4. ✅ Visual countdown timer for next availability
5. ✅ Animated chest opening sequence
6. ✅ Notification when chests are unclaimed (badge indicator)
7. ✅ Error handling for daily limit reached
8. ✅ Error handling for network failures
9. ✅ Analytics tracking for chests claimed and gems earned
10. ✅ Clear UI indicators for remaining claims

## 19. League System

### Overview

Players compete in weekly leagues based on their performance. Each league has exactly **50 participants**. At the end of each weekly cycle, **top 8 ranked players** are promoted to the next higher league, while players outside the promotion zone are subject to demotion based on league capacity.

### League Tiers

| Tier | Name | Min Score | Max Score | Icon | Color | Max Participants | Promotion Cutoff |
|------|------|-----------|-----------|------|-------|------------------|------------------|
| 1 | Wood | 0 | 99 | 🪵 | #8B4513 | 50 | 8 |
| 2 | Stone | 100 | 299 | 🪨 | #808080 | 50 | 8 |
| 3 | Bronze | 300 | 599 | 🥉 | #CD7F32 | 50 | 8 |
| 4 | Silver | 600 | 999 | 🥈 | #C0C0C0 | 50 | 8 |
| 5 | Crystal | 1000 | 1499 | 💎 | #00CED1 | 50 | 8 |
| 6 | Elite | 1500 | 2499 | ⭐ | #FFD700 | 50 | 8 |
| 7 | Champion | 2500 | 3999 | 🏆 | #FF4500 | 50 | 8 |
| 8 | Legend | 4000+ | ∞ | 👑 | #9B59B6 | 50 | 0 (Top players stay) |

### Weekly Cycle

- **Start**: Monday 00:00 UTC
- **End**: Next Monday 00:00 UTC
- **Score Reset**: All weekly scores reset to 0 at cycle start
- **Promotion**: Top 8 players in each league are promoted to the next tier
- **Demotion**: Players outside top 8 in leagues above Wood are demoted (except Legend which has no demotion)

### League Data Model

```json
{
  "userId": "UUID",
  "weeklyScore": 0,
  "totalScore": 0,
  "currentLeague": "wood",
  "highestLeague": "wood",
  "rank": 0,
  "cycleStartScore": 0,
  "lastCycleRank": 0,
  "cycleStart": "YYYY-MM-DD",
  "wasPromoted": false,
  "wasDemoted": false,
  "participantsCount": 50
}
```

### League Leaderboard

The league leaderboard displays:
- Rank position (1st, 2nd, 3rd get medals 🥇🥈🥉)
- Player username
- Weekly score
- Current user's entry highlighted
- Players in promotion zone (top 8) are highlighted in green

### Promotion/Demotion Rules

1. **Promotion Zone**: Top 8 players in each league automatically promoted
2. **Demotion Zone**: Players ranked 9th or lower may be demoted if league is full
3. **Legend League**: No demotion - top players stay permanently
4. **Wood League**: No demotion - entry point for new players
5. **Overflow Handling**: If a league exceeds 50 participants, lower-ranked players are demoted first

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/league | Get user's current league info with rank and participant count |
| GET | /api/league/leaderboard | Get league-specific leaderboard with top 100 players |
| POST | /api/league/update | Update weekly score and recalculate rank |

## 20. Achievements System

### Overview

25 distinct achievements that players can unlock by meeting specific criteria.

### Achievement List

| ID | Name | Description | Criteria | Icon | Points |
|----|------|-------------|----------|------|--------|
| a1 | First Tap | Tap the nut for the first time | 1 tap | 👆 | 10 |
| a2 | Century Cracker | Score 100 points total | 100 total score | 💯 | 50 |
| a3 | Century Cracker II | Score 500 points total | 500 total score | 💯💯 | 100 |
| a4 | Century Cracker III | Score 1000 points total | 1000 total score | 💯💯💯 | 200 |
| a5 | Century Cracker IV | Score 5000 points total | 5000 total score | 💯💯💯💯 | 500 |
| a6 | Century Cracker V | Score 10000 points total | 10000 total score | 💯💯💯💯💯 | 1000 |
| a7 | Daily Dedication | Complete 1 daily quest | 1 quest | 📜 | 25 |
| a8 | Daily Dedication II | Complete 10 daily quests | 10 quests | 📜📜 | 100 |
| a9 | Daily Dedication III | Complete 50 daily quests | 50 quests | 📜📜📜 | 250 |
| a10 | Quest Master | Complete all daily quests in one day | 5 quests in day | 👑 | 500 |
| a11 | Social Butterfly | Add your first friend | 1 friend | 🦋 | 50 |
| a12 | Popular | Have 10 friends | 10 friends | 👥 | 200 |
| a13 | Legendary Collector | Collect 100 gems total | 100 gems collected | 💎 | 150 |
| a14 | Speed Collector | Collect 50 speed potions | 50 potions | ⚗️ | 150 |
| a15 | League Climber | Reach Crystal league | Crystal | 💎 | 300 |
| a16 | Elite Status | Reach Elite league | Elite | ⭐ | 500 |
| a17 | Champion | Reach Champion league | Champion | 🏆 | 750 |
| a18 | Legend | Reach Legend league | Legend | 👑 | 1000 |
| a19 | Social Star | Get 10 likes on feed posts | 10 likes | ❤️ | 100 |
| a20 | Commentator | Post 25 comments | 25 comments | 💬 | 100 |
| a21 | Lucky | Find a hidden bonus | random | 🍀 | 50 |
| a22 | Combo Master | Score 10 times in 5 seconds | 10 taps/5sec | 🔥 | 200 |
| a23 | Night Owl | Play after midnight | after 12am | 🦉 | 25 |
| a24 | Early Bird | Play before 6am | before 6am | 🐦 | 25 |
| a25 | Dedicated Player | Login 7 days in a row | 7 day streak | �7️ | 350 |

### Achievement Data Model

```json
{
  "userId": "UUID",
  "achievements": {
    "a1": { "unlocked": true, "unlockedAt": "ISO timestamp" },
    "a2": { "unlocked": false, "unlockedAt": null }
  },
  "totalPoints": 0,
  "achievementCount": 0
}
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/achievements | Get all achievements and user's progress |
| POST | /api/achievements/check | Check and unlock any new achievements |

## 21. Shop System

### Overview

Players can spend gems to purchase cosmetic and utility items.

### Shop Items

| ID | Name | Description | Type | Cost | Effect |
|----|------|-------------|------|------|--------|
| s1 | Golden Crack Effect | Crack effects turn golden | Cosmetic | 50 💎 | +gold particles |
| s2 | Sparkle Trail | Nut leaves sparkle trail | Cosmetic | 100 💎 | +sparkle effect |
| s3 | Lucky Charm | +10% bonus points for 1 hour | Utility | 75 💎 | 1.1x score |
| s4 | Double XP Weekend | Double points this weekend | Utility | 150 💎 | 2x score |
| s5 | Custom Title | Set a custom title | Cosmetic | 200 💎 | title badge |
| s6 | Animated Avatar | Animated avatar border | Cosmetic | 150 💎 | glow effect |
| s7 | Particle Explosion | Explosion on crack | Cosmetic | 125 💎 | +particles |
| s8 | VIP Badge | Permanent VIP badge | Cosmetic | 500 💎 | badge display |
| s9 | Rainbow Trail | Colorful rainbow follows your nut | Cosmetic | 175 💎 | rainbow effect |
| s10 | Frost Aura | Ice-cold frost surrounds the nut | Cosmetic | 120 💎 | frost effect |
| s11 | Fire Burst | Flames erupt on each tap | Cosmetic | 165 💎 | fire effect |
| s12 | Neon Glow | Vibrant neon lighting effect | Cosmetic | 140 💎 | neon glow |
| s13 | Time Warp | Slow motion effect for 30 seconds | Utility | 50 💎 | time slow |
| s14 | Mega Crack | Crack multiplier x3 for 5 minutes | Utility | 100 💎 | 3x crack |
| s15 | Lucky Star | Stars rain down on cracks | Cosmetic | 130 💎 | star particles |
| s16 | Cosmic Background | Space-themed background | Background | 250 💎 | space BG |
| s17 | Forest Background | Nature-themed background | Background | 250 💎 | forest BG |
| s18 | Ocean Background | Underwater-themed background | Background | 250 💎 | ocean BG |

### Shop Data Model

```json
{
  "userId": "UUID",
  "inventory": {
    "goldenCrack": false,
    "sparkleTrail": false,
    "luckyCharm": false,
    "doubleXP": false,
    "customTitle": "",
    "animatedAvatar": false,
    "particleExplosion": false,
    "vipBadge": false
  },
  "purchaseHistory": [
    { "itemId": "s1", "purchasedAt": "ISO timestamp" }
  ],
  "activeEffects": {
    "luckyCharm": { "expiresAt": "ISO timestamp" },
    "doubleXP": { "expiresAt": "ISO timestamp" }
  }
}
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/shop | Get all items and user's inventory |
| POST | /api/shop/purchase | Purchase an item |
| POST | /api/shop/activate | Activate a consumable item |
| DELETE | /api/shop/item/:id | Use/equip an item |

## 22. File Structure (Updated)

```
/Nut Cracker/
├── SPEC.md                 (this file)
├── package.json
├── server.js               (Express server with all API routes)
├── quests.json             (auto-generated, quest progress)
├── users.json              (auto-generated, user accounts)
├── leaderboard.json        (auto-generated, scores)
├── friends.json            (auto-generated, friends data)
├── feed.json               (auto-generated, activity feed)
├── notifications.json      (auto-generated, notifications)
├── leagues.json           (auto-generated, league data)
├── achievements.json       (auto-generated, achievement data)
├── shop.json              (auto-generated, shop inventory)
└── public/
    ├── index.html          (main game page)
    ├── login.html          (authentication page)
    ├── css/
    │   └── style.css       (game styling)
    └── js/
        ├── game.js         (game logic, auth, leaderboard)
        ├── quests.js        (daily quests system)
        ├── customization.js  (avatar/background customization)
        ├── social.js       (friends, feed, notifications)
        ├── league.js       (league system)
        ├── achievements.js  (achievements system)
        └── shop.js         (shop system)
```

## 23. Acceptance Criteria (Updated)

### League System

1. ✅ 8 league tiers with increasing score requirements
2. ✅ Weekly cycle starting Monday 00:00 UTC
3. ✅ Score resets at beginning of each cycle
4. ✅ Promotion/demotion based on performance
5. ✅ League-specific leaderboard
6. ✅ Proper error handling for unauthenticated users
7. ✅ Loading states and retry functionality
8. ✅ 50 participants per league limit
9. ✅ Top 8 promotion mechanism at cycle end
10. ✅ Demotion mechanism for non-top participants
11. ✅ Participant count display
12. ✅ Promotion zone indicators
13. ✅ Logging for error diagnosis and debugging

### Achievements

1. ✅ 25 distinct achievements
2. ✅ Clear criteria for each achievement
3. ✅ Progress tracking toward goals
4. ✅ Unlock notifications
5. ✅ Achievement points system
6. ✅ Proper error handling for unauthenticated users
7. ✅ Loading states and retry functionality

### Shop System

1. ✅ 8 unique items with different costs
2. ✅ Gem-based economy
3. ✅ Cosmetic and utility item types
4. ✅ Inventory persistence
5. ✅ Effect activation system
6. ✅ Proper error handling for unauthenticated users
7. ✅ Loading states and retry functionality

### Bug Fixes (Recent)

1. ✅ Fixed points update discrepancy between leaderboard and leagues
2. ✅ Fixed shop page blank screen when not authenticated
3. ✅ Fixed trophies/achievements page blank screen when not authenticated
4. ✅ Fixed league score update to properly track score increments
5. ✅ Added proper authentication prompts for restricted features

- Consider switching to database (SQLite/PostgreSQL) for production
- Add password reset functionality
- Implement email verification for accounts
- Add social login (OAuth)
- Consider WebSocket instead of SSE for more efficient real-time updates
- Add unit tests for API endpoints
- Implement refresh token rotation for sessions
- Add push notifications for mobile
- Implement chat system between friends
- Add achievement badges system

