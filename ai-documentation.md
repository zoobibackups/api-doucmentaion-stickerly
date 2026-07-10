# Stickerly API Documentation

REST API for the Stickerly backend (Express, port `3001`).

All curl examples use Postman-style variables so they can be imported directly:

| Variable | Meaning |
|---|---|
| `{{stickerly}}` | Base URL, e.g. `http://localhost:3001` |
| `{{accessToken}}` | JWT access token from `POST /auth/google` / `POST /auth/anonymous` |
| `{{refreshToken}}` | Refresh token from login / refresh |

---

## Conventions

### Authentication
Every endpoint marked **Auth: Bearer** requires the header:

```
Authorization: Bearer {{accessToken}}
```

- A missing/invalid token returns `401 {"status": false, "message": "You are not logged in!"}` or `"Invalid or expired token"`.
- Endpoints marked **Auth: Admin** additionally require the user's `role` to be `admin` in the DB (`403 Admin only` otherwise).
- The access token payload carries `{ id, username, email }`; most v2 endpoints take the acting user's identity from the token, **not** from the body.

### Response envelope
All endpoints return JSON in the shape:

```json
{ "status": true|false, "message": "...", "data": ... }
```

List endpoints may also return `total`, `page`, `limit`.

### Pagination
Two styles are in use (noted per endpoint):

- **`limit` + `offset`** — `offset` defaults to `0`.
- **`page` + `limit`** — `page` defaults to `1`.

All pagination params are optional; defaults and maximums are listed per endpoint.

### Validation errors
Body validation uses Joi. Invalid input returns `400` with the validation message.

### Deprecated endpoints
Legacy routes still work but respond with headers `Deprecation: true` and `X-Deprecated-Use: <successor route>`. New clients must use the v2 routes. Deprecated routes are listed in compact tables at the end of each section.

---

## 1. Health / Static

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/` | None | Health check → `{"status":true,"message":"Server running"}` |
| GET | `/apple-app-site-association` | None | iOS Universal Links file (JSON) |
| GET | `/.well-known/*` | None | App-link verification files (Android + iOS) |
| GET | `/uploads/*` | None | Static access to locally uploaded files |

```bash
curl {{stickerly}}/
```

---

## 2. Auth — `/auth`

### POST `/auth/google` — Login / register with Google
**Auth:** None

Verifies the Google ID token, creates the user on first login (username derived from the email), and returns tokens. Blocked or deleted accounts are rejected with `400`.

Body (JSON):

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `idToken` | string | **Yes** | — | Google ID token (verified against `GOOGLE_CLIENT_ID`) |
| `fcmToken` | string | No | `null` | FCM push token, stored/updated on the user |
| `countryId` | number | No | `null` | Country FK, stored/updated on the user |

```bash
curl -X POST {{stickerly}}/auth/google \
  -H "Content-Type: application/json" \
  -d '{"idToken":"{{googleIdToken}}","fcmToken":"{{fcmToken}}","countryId":1}'
```

Response `200`: `{ status, message, data: { user, accessToken, refreshToken } }`

### POST `/auth/anonymous` — Anonymous / email login
**Auth:** None

Creates (or reuses) a user by email with `loginType: 'anonymous'`.

Body (JSON):

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `email` | string | **Yes** | — | Lookup / creation key |
| `firstName` | string | No | `''` | |
| `lastName` | string | No | `''` | |
| `pic` | string | No | `null` | Profile image URL |
| `fcmToken` | string | No | `null` | |
| `countryId` | number | No | `null` | |

```bash
curl -X POST {{stickerly}}/auth/anonymous \
  -H "Content-Type: application/json" \
  -d '{"email":"{{email}}","firstName":"John","lastName":"Doe"}'
```

### POST `/auth/refresh` — Rotate tokens
**Auth:** None (refresh token in body)

Revokes the presented refresh token and issues a new access + refresh pair. Returns `401` if the token is invalid, expired, or already revoked.

| Param | Type | Required | Description |
|---|---|---|---|
| `refreshToken` | string | **Yes** | Previously issued refresh token |

```bash
curl -X POST {{stickerly}}/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"{{refreshToken}}"}'
```

Response `200`: `{ status: true, data: { accessToken, refreshToken } }`

### POST `/auth/logout` — Revoke one session
**Auth:** None

| Param | Type | Required | Description |
|---|---|---|---|
| `refreshToken` | string | No | Token to revoke; always returns `200` |

```bash
curl -X POST {{stickerly}}/auth/logout \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"{{refreshToken}}"}'
```

### POST `/auth/logout-all` — Revoke all sessions
**Auth:** Bearer. No body.

```bash
curl -X POST {{stickerly}}/auth/logout-all \
  -H "Authorization: Bearer {{accessToken}}"
```

---

## 3. Users — `/users`

### GET `/users/me` — My account
**Auth:** Bearer. No params.

```bash
curl {{stickerly}}/users/me -H "Authorization: Bearer {{accessToken}}"
```

### PUT `/users/me` — Update my account
**Auth:** Bearer

All body fields optional; send only what changes.

| Param | Type | Constraints |
|---|---|---|
| `firstName` | string | |
| `lastName` | string | |
| `username` | string | |
| `mobileNumber` | string | nullable |
| `gender` | string | nullable |
| `dateOfBirth` | date | nullable |
| `profileImg` | string | nullable (URL) |
| `countryId` | integer | nullable |
| `fcmToken` | string | nullable |
| `bio` | string | max 500, nullable |
| `website` | string | valid URI, max 500, nullable |
| `coverImg` | string | nullable (URL) |

```bash
curl -X PUT {{stickerly}}/users/me \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"firstName":"John","bio":"Sticker maker","website":"https://example.com"}'
```

### DELETE `/users/me` — Delete my account
**Auth:** Bearer. No params. Soft-deletes the account.

```bash
curl -X DELETE {{stickerly}}/users/me -H "Authorization: Bearer {{accessToken}}"
```

### POST `/users/me/avatar` — Upload profile image
**Auth:** Bearer. Multipart form.

| Field | Type | Required | Description |
|---|---|---|---|
| `profileImg` | file | **Yes** | Image file; stored under `/uploads/` |

```bash
curl -X POST {{stickerly}}/users/me/avatar \
  -H "Authorization: Bearer {{accessToken}}" \
  -F "profileImg=@{{filePath}}"
```

### GET `/users/me/tags` — My interest tags
**Auth:** Bearer. No params.

### PUT `/users/me/tags` — Replace my interest tags
**Auth:** Bearer

| Param | Type | Required | Description |
|---|---|---|---|
| `tag_ids` | integer[] | **Yes** | Full replacement set (empty array clears) |

```bash
curl -X PUT {{stickerly}}/users/me/tags \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"tag_ids":[1,2,3]}'
```

### GET `/users/me/blocks` — Users I've blocked
**Auth:** Bearer

| Query | Type | Default | Max |
|---|---|---|---|
| `limit` | number | 50 | 200 |
| `offset` | number | 0 | — |

```bash
curl "{{stickerly}}/users/me/blocks?limit=50&offset=0" \
  -H "Authorization: Bearer {{accessToken}}"
```

### POST `/users/me/blocks` — Block a user (personal block)
**Auth:** Bearer

| Param | Type | Required | Constraints |
|---|---|---|---|
| `blocked_user_id` | integer | **Yes** | |
| `reason` | string | No | max 255, nullable |

```bash
curl -X POST {{stickerly}}/users/me/blocks \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"blocked_user_id":{{userId}},"reason":"spam"}'
```

Response `201` on success.

### DELETE `/users/me/blocks/:userId` — Unblock a user
**Auth:** Bearer. Path param `userId` (numeric).

```bash
curl -X DELETE {{stickerly}}/users/me/blocks/{{userId}} \
  -H "Authorization: Bearer {{accessToken}}"
```

### GET `/users/search` — Search users
**Auth:** Bearer

| Query | Type | Required | Default | Max |
|---|---|---|---|---|
| `q` | string | **Yes** | — | search text |
| `limit` | number | No | 20 | 100 |
| `offset` | number | No | 0 | — |

```bash
curl "{{stickerly}}/users/search?q={{query}}&limit=20&offset=0" \
  -H "Authorization: Bearer {{accessToken}}"
```

### Profile endpoints
**Auth:** Bearer for all.

| Method | Path | Description |
|---|---|---|
| GET | `/users/me/profile` | My full profile (counts, follow state) |
| GET | `/users/me/favourites` | My favourites |
| GET | `/users/me/packs` | My packs (created or liked) |
| GET | `/users/:id/profile` | Another user's profile (numeric `id`) |
| GET | `/users/:id/packs` | Another user's packs |
| GET | `/users/:id/favourites` | Another user's favourites |
| GET | `/users/:id` | Public profile (basic info) |

Query params for the two `/packs` endpoints:

| Query | Type | Default | Allowed / Max |
|---|---|---|---|
| `type` | string | `created` | `created` \| `liked` |
| `limit` | number | 20 | max 100 |
| `offset` | number | 0 | — |

```bash
curl "{{stickerly}}/users/{{userId}}/packs?type=created&limit=20&offset=0" \
  -H "Authorization: Bearer {{accessToken}}"
```

### Deprecated user routes

| Method | Path | Body / params | Use instead |
|---|---|---|---|
| GET | `/users/getById/:id` | — | `GET /users/:id` |
| POST | `/users/login` | login body | `POST /auth/google` |
| POST | `/users/` | `username`, `email`, `firstName`, `lastName` (all required) | `POST /auth/google` |
| POST | `/users/upload-image/:id` | multipart `profileImg` | `POST /users/me/avatar` |
| PUT | `/users/updateById/:id` | user fields | `PUT /users/me` |
| DELETE | `/users/removeById/:id` | — | `DELETE /users/me` |
| POST | `/users/create-user-tag` | `user_id` (number), `tag_id` (string) — required | `PUT /users/me/tags` |
| PUT | `/users/update-user-tag/:id` | `user_id`, `tag_id` | `PUT /users/me/tags` |
| PUT | `/users/update-pack/:id` | pack fields | `PATCH /pack/:id` |
| POST | `/users/add-pack-tag` | `pack_id` (number), `tag_id` (string) — required | `PUT /pack/:id/tags` |
| GET | `/users/get_user_packs` | query `user_id`, `is_public` | `GET /pack?user_id=...` |
| GET | `/users/get_user_likes_packs_and_stickers` | query `user_id` | `GET /users/me/favourites` |

---

## 4. Packs — `/pack`

### POST `/pack` — Create a pack
**Auth:** Bearer (owner = token user)

| Param | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `title` | string | **Yes** | — | 1–255 chars, trimmed |
| `type` | string | No | `regular` | `regular` \| `animated` |
| `is_public` | boolean \| 0/1 | No | `1` | |
| `tag_ids` | integer[] | No | `[]` | |

```bash
curl -X POST {{stickerly}}/pack \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"title":"My Pack","type":"regular","is_public":1,"tag_ids":[1,2]}'
```

Response `201`: created pack.

### GET `/pack` — List packs
**Auth:** Bearer

| Query | Type | Required | Default | Allowed / Max |
|---|---|---|---|---|
| `scope` | string | No | `public` | `public` \| `mine` \| `user` |
| `type` | string | No | — | `regular` \| `animated` filter |
| `user_id` | number | Required when `scope=user` | — | Target user |
| `page` | number | No | 1 | — |
| `limit` | number | No | 20 | max 100 |

```bash
curl "{{stickerly}}/pack?scope=public&type=regular&page=1&limit=20" \
  -H "Authorization: Bearer {{accessToken}}"
```

### GET `/pack/:id` — Pack detail
**Auth:** Bearer. Path param `id` (numeric). Returns counts, tags, tray icon, `isLiked` for the viewer. `404` if not found/visible.

```bash
curl {{stickerly}}/pack/{{packId}} -H "Authorization: Bearer {{accessToken}}"
```

### PATCH `/pack/:id` — Update a pack (owner only)
**Auth:** Bearer

All body fields optional:

| Param | Type | Constraints |
|---|---|---|
| `title` | string | 1–255 chars |
| `type` | string | `regular` \| `animated` |
| `is_public` | boolean \| 0/1 | |
| `status` | string | `active` \| `not-active` |

```bash
curl -X PATCH {{stickerly}}/pack/{{packId}} \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"title":"New title","is_public":0}'
```

### PATCH `/pack/:id/publish` — Toggle publish
**Auth:** Bearer

| Param | Type | Required | Description |
|---|---|---|---|
| `is_public` | boolean \| 0/1 | **Yes** | New visibility |

### PUT `/pack/:id/tray-icon` — Set tray icon from an existing sticker
**Auth:** Bearer (owner only)

| Param | Type | Required |
|---|---|---|
| `sticker_id` | integer | **Yes** |

```bash
curl -X PUT {{stickerly}}/pack/{{packId}}/tray-icon \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"sticker_id":{{stickerId}}}'
```

### PUT `/pack/:id/tray-icon/upload` — Upload a tray icon file
**Auth:** Bearer (owner only). Multipart form.

| Field | Type | Required | Constraints |
|---|---|---|---|
| `tray_icon` | file | **Yes** | png / jpeg / webp, max **500 KB** |

Errors: `413` file too large, `415` unsupported type.

```bash
curl -X PUT {{stickerly}}/pack/{{packId}}/tray-icon/upload \
  -H "Authorization: Bearer {{accessToken}}" \
  -F "tray_icon=@{{filePath}}"
```

### PUT `/pack/:id/tags` — Replace pack tags (owner only)
**Auth:** Bearer

| Param | Type | Required |
|---|---|---|
| `tag_ids` | integer[] | **Yes** (empty array clears) |

### DELETE `/pack/:id` — Delete a pack (owner only)
**Auth:** Bearer. Soft delete.

```bash
curl -X DELETE {{stickerly}}/pack/{{packId}} -H "Authorization: Bearer {{accessToken}}"
```

### Pack engagement
**Auth:** Bearer for all. Actor = token user. No body needed for like/unlike/views/shares/downloads.

| Method | Path | Description |
|---|---|---|
| POST | `/pack/:id/like` | Like pack |
| POST | `/pack/:id/unlike` | Remove like |
| POST | `/pack/:id/views` | Record a view |
| POST | `/pack/:id/shares` | Record a share |
| POST | `/pack/:id/downloads` | Record a download |
| GET | `/pack/user/:userId/liked-summary` | Counts of packs liked by user |
| GET | `/pack/user/:userId/liked` | Packs liked by user — `page` (default 1), `limit` (default 20, max 100) |

```bash
curl -X POST {{stickerly}}/pack/{{packId}}/like -H "Authorization: Bearer {{accessToken}}"
```

### Pack comments
**Auth:** Bearer

| Method | Path | Params | Defaults |
|---|---|---|---|
| GET | `/pack/:id/comments` | query `limit`, `offset` | `limit=50` (max 200), `offset=0` |
| POST | `/pack/:id/comments` | body `comment` (string, **required**, 1–255) | — |
| DELETE | `/pack/:id/comments/:cid` | path `cid` = comment id | commenter or pack owner |

```bash
curl -X POST {{stickerly}}/pack/{{packId}}/comments \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"comment":"Nice pack!"}'
```

### Pack reporting
**Auth:** Bearer

| Method | Path | Params |
|---|---|---|
| POST | `/pack/:id/report` | body `reason` (string, **required**, 1–255) → `201` |
| GET | `/pack/user/:userId/reported` | own reports only (`403` if `userId` ≠ token user); query `page` (default 1), `limit` (default 20, max 100) |

```bash
curl -X POST {{stickerly}}/pack/{{packId}}/report \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"reason":"inappropriate content"}'
```

### Deprecated pack routes

All require Bearer auth. `Like/share/download/view` bodies: `user_id` (number, required), `pack_id` (number, required). Comment body adds `comment` (string, required).

| Method | Path | Body / query | Use instead |
|---|---|---|---|
| POST | `/pack/create` | `title`, `user_id`, `is_public`, `type`, `tags` (all required) | `POST /pack` |
| PUT | `/pack/:id` | same as create | `PATCH /pack/:id` |
| POST | `/pack/create-tray-icon` | tray icon body | `PUT /pack/:id/tray-icon` |
| GET | `/pack/pack-type` | query `type` | `GET /pack?type=...` |
| GET | `/pack/get_pack_by_id/:id` | — | `GET /pack/:id` |
| GET | `/pack/get_sticker_count_by_pack_id` | query `pack_id`, `user_id` | `GET /pack/:id` |
| GET | `/pack/get_pack_by_user_id` | query `user_id`, `type` | `GET /pack?scope=user&user_id=...` |
| GET | `/pack/get_pack_by_user_interests/:user_id` | — | interest feed TBD |
| POST | `/pack/like` / `/pack/remove_like` | `user_id`, `pack_id` | `POST /pack/:id/like` / `unlike` |
| POST | `/pack/download` / `/pack/share` / `/pack/views` | `user_id`, `pack_id` | `POST /pack/:id/downloads` / `shares` / `views` |
| POST | `/pack/comments` / `/pack/update_comments` | `user_id`, `pack_id`, `comment` | `POST /pack/:id/comments` |
| POST | `/pack/remove_comments` | — | `DELETE /pack/:id/comments/:cid` |
| POST | `/pack/add_tags` / `update_tags` / `delete_tags` | `pack_id`, `tag_id` | `PUT /pack/:id/tags` |

---

## 5. Stickers — `/sticker`

Upload constraints (all sticker/tray-icon uploads): **png / jpeg / webp only**, max **500 KB** per file (WhatsApp sticker spec), max **30 files** per bulk request. Errors: `413` too large, `415` wrong type, `400` too many files.

### POST `/sticker/pack/:packId` — Upload one sticker
**Auth:** Bearer (pack owner). Multipart form.

| Field | Type | Required | Description |
|---|---|---|---|
| `sticker` | file | **Yes** | png/jpeg/webp ≤ 500 KB |
| `tags` | string / array | No | Tag names as JSON array, comma-separated string, or array; created if new |

```bash
curl -X POST {{stickerly}}/sticker/pack/{{packId}} \
  -H "Authorization: Bearer {{accessToken}}" \
  -F "sticker=@{{filePath}}" \
  -F "tags=funny,cat"
```

Response `201`: created sticker with its tags.

### POST `/sticker/pack/:packId/bulk` — Upload up to 30 stickers
**Auth:** Bearer (pack owner). Multipart form.

| Field | Type | Required | Description |
|---|---|---|---|
| `stickers` | file[] | **Yes** | Repeat the field per file, max 30 |

```bash
curl -X POST {{stickerly}}/sticker/pack/{{packId}}/bulk \
  -H "Authorization: Bearer {{accessToken}}" \
  -F "stickers=@{{filePath1}}" \
  -F "stickers=@{{filePath2}}"
```

### GET `/sticker/pack/:packId` — List stickers in a pack
**Auth:** Bearer. Path param `packId` (numeric).

```bash
curl {{stickerly}}/sticker/pack/{{packId}} -H "Authorization: Bearer {{accessToken}}"
```

### GET `/sticker/:id` — Sticker detail
**Auth:** Bearer. `404` if not found.

### PATCH `/sticker/:id` — Update sticker (owner only)
**Auth:** Bearer

| Param | Type | Required | Description |
|---|---|---|---|
| `status` | boolean \| 0/1 | No | Enable/disable the sticker |

### DELETE `/sticker/:id` — Delete sticker (owner only)
**Auth:** Bearer. Soft delete.

### Sticker engagement
**Auth:** Bearer. Actor = token user. No body needed.

| Method | Path | Description |
|---|---|---|
| POST | `/sticker/:id/like` | Like |
| POST | `/sticker/:id/unlike` | Remove like |
| POST | `/sticker/:id/views` | Record a view |
| POST | `/sticker/:id/shares` | Record a share |
| POST | `/sticker/:id/downloads` | Record a download |
| GET | `/sticker/user/:userId/liked` | Stickers liked by user — `page` (default 1), `limit` (default 20, max 100) |

### Sticker comments
**Auth:** Bearer

| Method | Path | Params | Defaults |
|---|---|---|---|
| GET | `/sticker/:id/comments` | query `limit`, `offset` | `limit=50` (max 200), `offset=0` |
| POST | `/sticker/:id/comments` | body `comment` (string, **required**, 1–255) | — |
| DELETE | `/sticker/:id/comments/:cid` | path `cid` = comment id | — |

### Sticker reporting
**Auth:** Bearer

| Method | Path | Params |
|---|---|---|
| POST | `/sticker/:id/report` | body `reason` (string, **required**, 1–255) → `201` |
| GET | `/sticker/user/:userId/reported` | own reports only (`403` otherwise); `page` (default 1), `limit` (default 20, max 100) |

### Deprecated sticker routes

All require Bearer auth.

| Method | Path | Body / params | Use instead |
|---|---|---|---|
| POST | `/sticker/create` | multipart `file` + `pack_id` (string, required), `user_id` (number, required) | `POST /sticker/pack/:packId` |
| PUT | `/sticker/update/:sticker_id` | sticker fields | `PATCH /sticker/:id` |
| DELETE | `/sticker/delete/:id` | — | `DELETE /sticker/:id` |
| GET | `/sticker/get_stikcer_by_id/:sticker_id` | — | `GET /sticker/:id` |
| GET | `/sticker/get_stikcer_by_user_interests/:user_id` | — | `GET /pack?scope=...` |
| POST | `/sticker/like` / `remove_like` / `download` / `share` / `views` | `user_id`, `sticker_id` | `POST /sticker/:id/<action>` |
| POST | `/sticker/comments` / `remove_comments` / `update_comments` | comment body | `/sticker/:id/comments` routes |
| POST | `/sticker/add_tags` / `update_tags` / `delete_tags` | tag body | not in scope |

---

## 6. Home feed — `/home`

All feed endpoints return pack cards:
```
{ id, title, type, is_public, trayIconUrl, stickerCount,
  likeCount, viewCount, shareCount, downloadCount, isLiked,
  owner: { id, username, profileImg } }
```
(`/home/creators` returns users instead of packs.)

**Auth:** Bearer for all. Common query params:

| Query | Type | Default | Max |
|---|---|---|---|
| `limit` | number | 15 | 100 |
| `offset` | number | 0 | — |

| Method | Path | Description |
|---|---|---|
| GET | `/home/trending` | Trending packs |
| GET | `/home/explore` | Explore feed |
| GET | `/home/following` | Packs from creators the user follows |
| GET | `/home/creators` | Suggested creators (users) |

```bash
curl "{{stickerly}}/home/trending?limit=15&offset=0" \
  -H "Authorization: Bearer {{accessToken}}"
```

### Deprecated home routes

| Method | Path | Query | Use instead |
|---|---|---|---|
| GET | `/home/foryou` | `page`, `limit` | `GET /home/explore` |
| GET | `/home/artists` | `page`, `limit` | `GET /home/creators` |
| GET | `/home/tags` | `page`, `limit`, `tag` | `GET /home/explore` |
| GET | `/home/search_packs/` | `search_text`, `limit` | search TBD |

---

## 7. Followers — `/followers`

**Auth:** Bearer for all. Acting user = token user.

List query params (all list endpoints):

| Query | Type | Default | Max |
|---|---|---|---|
| `limit` | number | 50 | 200 |
| `offset` | number | 0 | — |

| Method | Path | Description |
|---|---|---|
| GET | `/followers/me/following` | Users I follow |
| GET | `/followers/me/followers` | Users following me |
| GET | `/followers/:userId/following` | Users `:userId` follows |
| GET | `/followers/:userId/followers` | Followers of `:userId` |
| GET | `/followers/:userId/status` | `{ isFollowing, followsMe }` between me and `:userId` |
| POST | `/followers/:userId` | Follow `:userId` → `201` |
| DELETE | `/followers/:userId` | Unfollow `:userId` |

```bash
curl -X POST {{stickerly}}/followers/{{userId}} \
  -H "Authorization: Bearer {{accessToken}}"
```

### Deprecated follower routes

| Method | Path | Body | Use instead |
|---|---|---|---|
| GET | `/followers/get_followings/:id` | — | `GET /followers/:userId/following` |
| GET | `/followers/get_followers/:id` | — | `GET /followers/:userId/followers` |
| POST | `/followers/` | `user_id`, `followed_user_id` (both required) | `POST /followers/:userId` |
| DELETE | `/followers/:id` | — | Returns **410 Gone** (was broken); use `DELETE /followers/:userId` |

---

## 8. Collections — `/collections`

Personal sticker collections (favourites folders). **Auth:** Bearer for all; all operations act on the token user's collections.

### GET `/collections` — List my collections
No params. Pinned collections first.

```bash
curl {{stickerly}}/collections -H "Authorization: Bearer {{accessToken}}"
```

### POST `/collections` — Create a collection

| Param | Type | Required | Constraints |
|---|---|---|---|
| `name` | string | **Yes** | 1–100 chars, trimmed |

```bash
curl -X POST {{stickerly}}/collections \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"name":"Favourites"}'
```

Response `201`.

### GET `/collections/:id` — Collection with its stickers
Path param `id` (numeric).

### PATCH `/collections/:id` — Rename / pin
Both fields optional:

| Param | Type | Constraints |
|---|---|---|
| `name` | string | 1–100 chars |
| `isPinned` | boolean | |

### DELETE `/collections/:id` — Delete a collection

### Pin / stickers

| Method | Path | Body | Description |
|---|---|---|---|
| POST | `/collections/:id/pin` | — | Pin |
| DELETE | `/collections/:id/pin` | — | Unpin |
| POST | `/collections/:id/stickers` | `sticker_id` (positive integer, **required**) | Add a sticker |
| DELETE | `/collections/:id/stickers/:stickerId` | — | Remove a sticker |

```bash
curl -X POST {{stickerly}}/collections/{{collectionId}}/stickers \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"sticker_id":{{stickerId}}}'
```

---

## 9. Notifications — `/notifications`

**Auth:** Bearer for all. Notification `type` values: `follow`, `pack_like`, `pack_share`, `profile_view`, `pack_removed`.

### GET `/notifications` — My notifications

| Query | Type | Default | Max |
|---|---|---|---|
| `limit` | number | 50 | 100 |
| `offset` | number | 0 | — |

```bash
curl "{{stickerly}}/notifications?limit=50&offset=0" \
  -H "Authorization: Bearer {{accessToken}}"
```

### GET `/notifications/unread-count`
No params. Returns `{ status: true, data: { count } }`.

### PATCH `/notifications/:id/read` — Mark one as read
Path param `id` (numeric). No body.

### PATCH `/notifications/read-all` — Mark all as read
No params.

### GET `/notifications/settings` — Notification preferences
No params. Returns every type with `is_enabled` (**default `true`** when never set).

### PUT `/notifications/settings/:type` — Toggle a preference
Path param `type` — one of the type values above (`400` otherwise).

| Param | Type | Required |
|---|---|---|
| `is_enabled` | boolean | **Yes** |

```bash
curl -X PUT {{stickerly}}/notifications/settings/pack_like \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"is_enabled":false}'
```

---

## 10. Tags — `/tags`

**Auth:** Bearer for all.

### GET `/tags` — All tags (with user's selection state)

| Query | Type | Default |
|---|---|---|
| `page` | number | 1 |
| `limit` | number | 100 |

```bash
curl "{{stickerly}}/tags?page=1&limit=100" -H "Authorization: Bearer {{accessToken}}"
```

### POST `/tags` — Create tags (bulk)
Body is a **JSON array**:

| Param | Type | Required |
|---|---|---|
| `[].tag` | string | **Yes** |

```bash
curl -X POST {{stickerly}}/tags \
  -H "Authorization: Bearer {{accessToken}}" \
  -H "Content-Type: application/json" \
  -d '[{"tag":"funny"},{"tag":"animals"}]'
```

### Other tag routes

| Method | Path | Params | Description |
|---|---|---|---|
| PUT | `/tags/:id` | body: tag fields | Update a tag |
| GET | `/tags/get_by_id/:id` | — | Tag by id |
| DELETE | `/tags/:id` | — | Delete a tag |

---

## 11. Pack tags — `/pack_tags`

**Auth:** Bearer for all. Prefer `PUT /pack/:id/tags` for managing a pack's tags.

| Method | Path | Body / params |
|---|---|---|
| GET | `/pack_tags/` | — (all pack↔tag links) |
| POST | `/pack_tags/` | `pack_id` (number, **required**), `tag_id` (string, **required**) |
| PUT | `/pack_tags/:id` | `user_id`, `tag_id` |
| GET | `/pack_tags/getById/:id` | — |
| DELETE | `/pack_tags/:id` | — |

---

## 12. User tags — `/user_tags` (all DEPRECATED)

**Auth:** Bearer for all. Use `GET /users/me/tags` and `PUT /users/me/tags` instead.

| Method | Path | Body / params | Use instead |
|---|---|---|---|
| GET | `/user_tags/` | — | `GET /tags` |
| POST | `/user_tags/` | `user_id` (number, **required**), `tag_id` (string, **required**) | `PUT /users/me/tags` |
| PUT | `/user_tags/:id` | `user_id`, `tag_id` | `PUT /users/me/tags` |
| GET | `/user_tags/getById/:id` | — | `GET /users/me/tags` |
| DELETE | `/user_tags/:id` | — | `PUT /users/me/tags` |
| GET | `/user_tags/getTagsByUserId/:UserId` | — | `GET /users/me/tags` |

---

## 13. Admin — `/admin`

**Auth: Admin** — Bearer token **and** `role = 'admin'` (re-checked against the DB on every request; `403 Admin only` otherwise).

### Bans (admin ban ≠ personal block)

| Method | Path | Params | Defaults |
|---|---|---|---|
| GET | `/admin/bans` | query `limit`, `offset` | `limit=50` (max 200), `offset=0` |
| POST | `/admin/bans` | body `user_id` (integer, **required**), `reason` (string, **required**, 1–255) → `201` | — |
| DELETE | `/admin/bans/:userId` | path `userId` | — |

```bash
curl -X POST {{stickerly}}/admin/bans \
  -H "Authorization: Bearer {{adminAccessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"user_id":{{userId}},"reason":"terms violation"}'
```

### Content reports

| Method | Path | Params | Defaults |
|---|---|---|---|
| GET | `/admin/reports/packs` | query `limit`, `offset` | `limit=50` (max 200), `offset=0` — pending only |
| GET | `/admin/reports/stickers` | query `limit`, `offset` | same |
| PATCH | `/admin/reports/packs/:id` | body `status` (**required**) | `pending` \| `reviewed` \| `actioned` \| `dismissed` |
| PATCH | `/admin/reports/stickers/:id` | body `status` (**required**) | same values |

```bash
curl -X PATCH {{stickerly}}/admin/reports/packs/{{reportId}} \
  -H "Authorization: Bearer {{adminAccessToken}}" \
  -H "Content-Type: application/json" \
  -d '{"status":"actioned"}'
```

---

## 14. User-to-user blocking — `/blocked_user_by_user`

**Auth:** Bearer. Prefer the `/users/me/blocks` endpoints — these are the older equivalents.

| Method | Path | Body / params | Description |
|---|---|---|---|
| GET | `/blocked_user_by_user/:user_id` | path param is **ignored** — always returns the token user's blocks | List my blocks |
| POST | `/blocked_user_by_user/` | body `blocked_user_id` (number, required), `reason` (string, optional) | Block a user |
| DELETE | `/blocked_user_by_user/:blockedUserId` | path `blockedUserId` | Unblock |

---

## 15. Report user — `/report_user_by_user`

⚠️ **No auth middleware** is currently applied to these routes.

| Method | Path | Body / params | Description |
|---|---|---|---|
| GET | `/report_user_by_user/:user_id` | path `user_id` | Reports made by a user |
| POST | `/report_user_by_user/` | `user_id` (number, **required**), `reported_user_id` (number, **required**), `reason` (string, **required**) | Report a user |
| PUT | `/report_user_by_user/` | `user_id` (number, **required**), `reported_user_id` (number, **required**) | Un-report a user |
| PUT | `/report_user_by_user/:id` | — | Stub (returns placeholder) |
| DELETE | `/report_user_by_user/:id` | — | Stub (returns placeholder) |

---

## 16. Admin blocked user — `/admin_blocked_user` (all DEPRECATED)

**Auth:** Bearer (but **not** admin-checked — another reason these are deprecated). Use `/admin/bans` instead.

| Method | Path | Body / params | Use instead |
|---|---|---|---|
| GET | `/admin_blocked_user/` | — returns `405` | `GET /admin/bans` |
| POST | `/admin_blocked_user/` | `user_id` (number, required), `reason` (string) | `POST /admin/bans` |
| GET | `/admin_blocked_user/:id` | path `id` = user id — **unblocks** the user | `DELETE /admin/bans/:userId` |
| PUT | `/admin_blocked_user/:id` | — stub | `POST /admin/bans` |
| DELETE | `/admin_blocked_user/:id` | — stub | `DELETE /admin/bans/:userId` |

---

## 17. Image upload — `/upload_image`

### POST `/upload_image` — Generic file upload
⚠️ **No auth.** Stores the file on local disk under `./uploads/` and returns its path. Prefer the purpose-built uploads (`/users/me/avatar`, `/sticker/pack/:packId`, `/pack/:id/tray-icon/upload`).

| Field | Type | Required |
|---|---|---|
| `file` | file (multipart) | **Yes** |

```bash
curl -X POST {{stickerly}}/upload_image -F "file=@{{filePath}}"
```

Response: `{ "status": true, "path": "uploads/<filename>" }`

---

## Appendix: HTTP status codes used

| Code | Meaning here |
|---|---|
| 200 | Success |
| 201 | Created (packs, stickers, comments, reports, blocks, bans, follows, collections) |
| 400 | Validation failure or business-rule rejection |
| 401 | Missing/invalid/expired token, revoked refresh token, inactive account |
| 403 | Not allowed (admin-only, or accessing another user's reports) |
| 404 | Resource not found |
| 405 | Method not allowed (deprecated admin block list) |
| 410 | Gone (legacy broken unfollow) |
| 413 | Uploaded file exceeds 500 KB |
| 415 | Unsupported file type (only png/jpeg/webp allowed) |
| 500 | Server error → `{ "status": false, "message": "<error>" }` |
