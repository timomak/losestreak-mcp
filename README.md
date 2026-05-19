# LoseStreak MCP

Remote MCP server for [LoseStreak](https://losestreak.com), an iOS competitive fitness app. Connects AI assistants — Claude Desktop, Claude Code, ChatGPT, Cursor, OpenCode, and any MCP-compatible client — to your LoseStreak account.

**Connector URL:** `https://mcp.losestreak.com/mcp`

**Hosting:** Cloudflare Worker. The server source is closed; this repository is the public registry/documentation entry. Authentication and per-tool authorization are enforced server-side; all user data is stored in Supabase and protected by row-level security.

---

## Setup

Per-client setup snippets. All flows go through the LoseStreak OAuth 2.1 consent screen and require an active LoseStreak subscription on the account.

### Claude Desktop / claude.ai
1. Settings → Connectors → **Add custom connector**
2. Paste `https://mcp.losestreak.com/mcp`
3. Click **Connect**, sign in with the email tied to your LoseStreak account

### Claude Code
```bash
claude mcp add losestreak https://mcp.losestreak.com/mcp
```

### ChatGPT (Plus / Team / Enterprise)
1. Settings → **Connectors** → **New connector**
2. Paste `https://mcp.losestreak.com/mcp`
3. Complete the OAuth flow

> Free-tier ChatGPT does not currently support custom MCP connectors.

### Cursor
1. Settings → **MCP Servers** → **Add server**
2. Type: **HTTP**, URL: `https://mcp.losestreak.com/mcp`

### OpenCode
```bash
opencode mcp add losestreak https://mcp.losestreak.com/mcp
```

### Other MCP-compatible clients
Any client that supports remote MCP servers with OAuth 2.1 will work. Point it at `https://mcp.losestreak.com/mcp` and complete the consent flow.

---

## OAuth scopes

The consent screen lists scopes as human-readable rows. You may approve any subset.

| Scope | Grants |
|-------|--------|
| `profile:read` | Read display name, level, XP, streak, badge count, win/loss record |
| `health:read` | Read workouts (HealthKit-synced), weight history, meal logs, daily summary; submit food images for AI scanning |
| `health:write` | Log meals (text or image) and weigh-ins |
| `competitions:read` | List competitions, read competition details, battle activity feed, AI-generated verdicts |
| `competitions:write` | Post comments to a competition's live battle feed |
| `friends:read` | List friends, search users, list pending friend requests |
| `friends:write` | Send / accept / decline friend requests, remove friends |

Scopes are bound to the OAuth grant. To change scopes, re-authorize (creates a new grant).

---

## Tools (18)

### Read tools

#### `get_profile`
Scope: `profile:read`. Rate limit: 60/hour.
Returns the authenticated user's profile: id, username, display_name, avatar_url, level, xp, xp_to_next_level, current_streak, longest_streak, badge_count, competitions_won, competitions_lost.

#### `get_workouts`
Scope: `health:read`. Rate limit: 200/hour.
Recent workouts synced from HealthKit. Input: `{ start_date?, end_date?, limit? }`. Output: `{ workouts[], has_more }` — includes activity type, duration, calories, heart rate, distance, source app.

#### `get_weight_history`
Scope: `health:read`. Rate limit: 100/hour.
Daily weight entries. Input: `{ start_date?, end_date?, limit? }`. Output: `{ entries: { date, weight_kg, weight_lbs }[] }`.

#### `get_meals`
Scope: `health:read`. Rate limit: 100/hour.
Meal log with name, calories, macros, scanned-or-manual flag, timestamp. Does not return photo URLs.

#### `get_daily_summary`
Scope: `health:read`. Rate limit: 100/hour.
Pre-computed daily totals: calories in/out, workouts, weight, sleep. Input: `{ date? }` (defaults today).

#### `list_competitions`
Scope: `competitions:read`. Rate limit: 100/hour.
List the user's competitions. Input: `{ status?: 'active' | 'completed' | 'pending' | 'all', limit? }`.

#### `get_competition`
Scope: `competitions:read`. Rate limit: 200/hour.
Full competition details: participants, teams, mode, dates, current standings.

#### `get_battle_feed`
Scope: `competitions:read`. Rate limit: 200/hour.
Live activity feed for a competition: votes, comments, score updates. Input: `{ competition_id, since?, limit? }`.

#### `get_verdict`
Scope: `competitions:read`. Rate limit: 100/hour.
AI-generated verdict for a completed competition: winners, losers, full verdict text.

#### `get_friends`
Scope: `friends:read`. Rate limit: 60/hour.
Accepted friends with display name, avatar, level, current streak. No PII beyond what is listed.

#### `search_users`
Scope: `friends:read`. Rate limit: 30/hour.
Find LoseStreak users by username or display name (case-insensitive substring, 2+ chars).

#### `list_friend_requests`
Scope: `friends:read`. Rate limit: 60/hour.
Pending friend requests in both directions (incoming/outgoing).

#### `get_badges`
Scope: `profile:read`. Rate limit: 30/hour.
Earned badges with unlock dates.

#### `scan_food_image`
Scope: `health:read`. Rate limit: 30/day.
Scan a food image to estimate calories and macros. Returns a `scan_id` valid for 5 minutes — pass to `log_meal` to save after user confirmation. Input: `{ image: { type: 'base64', data, mime_type } }`. Output: `{ scan_id, name, calories, protein_g, carbs_g, fat_g, confidence, warning? }`.

### Write tools

#### `log_meal`
Scope: `health:write`. Rate limit: 30/day.
Three input modes (exactly one required):
- `scan_id` from a prior `scan_food_image` call
- `description` — free-text 1–500 chars; server estimates macros inline
- `manual` — caller provides `{ name, calories, protein_g, carbs_g, fat_g }`

Optional: `consumed_at`, `meal_type`.

#### `log_weight`
Scope: `health:write`. Rate limit: 5/day.
Today's weigh-in or a back-dated correction. One entry per day — re-logging the same date overwrites. Input: `{ weight_kg, logged_at? }`.

#### `post_battle_comment`
Scope: `competitions:write`. Rate limit: 10/hour.
Post a comment to a competition's battle feed. Max 280 characters. Caller must be a participant or accepted friend of one. Comments are tagged with the connected client name (e.g., "via Claude") in the iOS UI.

#### `send_friend_request`
Scope: `friends:write`. Rate limit: 20/day.
Send a friend request by username.

#### `accept_friend_request`
Scope: `friends:write`. Rate limit: 30/hour.
Accept a pending request by `relationship_id`.

#### `decline_friend_request`
Scope: `friends:write`. Rate limit: 30/hour.
Decline a pending request by `relationship_id`.

#### `remove_friend`
Scope: `friends:write`. Rate limit: 10/day.
Remove a friend or retract a pending request by the friend's `user_id`.

---

## Resources (3)

Attachable docs. Fetched only when the client/user explicitly attaches one.

| URI scheme | Description |
|------------|-------------|
| `verdict://{competition_id}` | Full AI verdict text as markdown |
| `competition://{id}/summary` | Title, mode, dates, participants, current standings |
| `weekly-recap://current` | Last 7 days: workouts, meal totals, weight delta, active competitions, badge unlocks |

---

## Security

- **OAuth 2.1** with PKCE (S256), Dynamic Client Registration (RFC 7591), audience-bound tokens (RFC 8707).
- **Access tokens:** 1 hour, signed JWT (HS256), audience pinned to `https://mcp.losestreak.com`.
- **Refresh tokens:** sliding expiration with rotation on every use; replay of an invalidated refresh token kills the entire grant.
- **Subscription gate:** every tool call verifies an active LoseStreak subscription (5-minute KV cache TTL).
- **Per-tool rate limits** (see table above) plus a global 2000 calls/hour ceiling.
- **Audit log:** every tool call writes a row visible to the user in *iOS app → Profile → Settings → Integrations*. PII in args is redacted before logging.
- **Per-client revoke:** users can revoke any connected client from the iOS app; tokens are rejected on the next use.

---

## LoseStreak

LoseStreak is an iOS competitive fitness app: users challenge friends to 1v1, free-for-all, or team battles across 23 challenge types — bodyweight reps counted on-device with the Vision framework's pose estimation; runs, lifts, cycling, and steps auto-synced from HealthKit; meals logged via AI image scanning. An AI verdict crowns the winner at competition end. Spectators vote and comment in a live battle feed.

- App Store: <https://apps.apple.com/app/losestreak>
- Website: <https://losestreak.com>

---

## Status

The MCP server is in active development. Spec compliance targets MCP 2025-06-18. Issues and feedback welcome here.
