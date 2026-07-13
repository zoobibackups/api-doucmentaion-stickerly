# Muthat Caller ID — API Documentation

Backend API for the Muthat Caller ID app.

- **Base URL (local):** `http://localhost:3035`
- **Content type:** `application/json` for all request bodies
- **Auth:** a Firebase Bearer-token middleware exists (`src/_middleware/auth.js`) but is currently **not applied** to any route in `app.js`. When enabled, send `Authorization: Bearer <firebase_id_token>`.
- **Validation errors:** invalid bodies are passed to the global error handler and returned as `{ "status": false, "errorMessage": "..." }`.
- **Postman:** import `Muthat-Caller-ID-APP.postman_collection.json` from the repo root.

---

## Contents

1. [Health & Utility](#1-health--utility)
2. [Users](#2-users)
3. [Countries](#3-countries)
4. [User Friends (contact sync)](#4-user-friends-contact-sync)
5. [Spam Report](#5-spam-report)
6. [Number Name Suggestion](#6-number-name-suggestion)
7. [Network (carrier lookup)](#7-network-carrier-lookup)
8. [Number Search (caller ID)](#8-number-search-caller-id)
9. [Agora (call tokens)](#9-agora-call-tokens)

---

## 1. Health & Utility

### GET `/`
Simple liveness check.

**Response `200`**
```json
{ "status": true, "successMessage": "Server is running" }
```

### GET `/check_db`
Verifies the MySQL pool by running `SELECT 1`.

**Response `200`** — `{ "status": true, "message": "Database connection successful" }`
**Response `500`** — `{ "status": false, "error": "Database connection failed: <reason>" }`

### GET `/check_all`
Checks the database and the Firebase Admin SDK in one call.

**Response `200`**
```json
{ "status": true, "db": "ok", "firebase": "ok" }
```
Failed components contain `"FAILED: <reason>"` instead of `"ok"`.

### GET `/check_async`
Verifies async request handling works. Returns `{ "status": true, "async": "works" }`.

### POST `/translate`
Translates text using Google Translate.

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | yes | Text to translate |
| `targetLang` | string | yes | Target language code, e.g. `ur`, `en`, `ar` |

**Response `200`** — `{ "status": true, "translatedText": "..." }`
**Response `400`** — when `text` or `targetLang` is missing.

---

## 2. Users

### POST `/users/register`
Find-or-create a user account. **Lookup order:**

1. If `mobile_number` is present → search by `mobile_number` + `countryiso2` (skipped when mobile is missing).
2. If not found and `email` is present → search by `email` (skipped when email is missing).
3. If still not found → a new account is created.

| Field | Type | Required | Notes |
|---|---|---|---|
| `first_name` | string | no | defaults to `"New"` |
| `last_name` | string | no | defaults to `"User"` |
| `mobile_number` | number | no | unique per country |
| `email` | string | no | must be a valid email when present |
| `type` | string | no | e.g. `"verified"` |
| `fcm_token` | string | no | push-notification token |
| `countryiso2` | string | no | e.g. `"PK"`; unknown codes fall back to `US` |

**Response `200`** — the `message` tells you which path was taken (`"User Found successfully"` vs `"User Created successfully"`); the `user` shape is identical in both cases:
```json
{
	"status": true,
	"message": "User Created successfully",
	"user": {
		"id": 2089101,
		"first_name": "Aftab",
		"last_name": "Khan",
		"mobile_number": null,
		"email": "user@example.com",
		"profile_url": null,
		"date_of_birth": null,
		"gender": null,
		"type": "verified",
		"fcm_token": "fcm-token-here",
		"country_name": "Pakistan",
		"country_iso2": "PK",
		"state": " ",
		"zip": " ",
		"city": " "
	}
}
```
Fields not sent are returned as `null`. On failure: `{ "status": false, "user": null, "message": "<reason>" }` (still HTTP 200).

### POST `/users/findbynumber`
Looks a user up by number and country.

| Field | Type | Required |
|---|---|---|
| `mobile_number` | number | yes |
| `countryiso2` | string | yes |

**Response `200`** — `{ "status": true, "user": { ... }, "message": "User has been found Successfully" }`
**Response `404`** — `{ "status": false, "user": null, "message": "User does not exist" }`

### POST `/users/finUserByNumberForCallLogAndVideoCall`
Lightweight existence check used by the call-log / video-call screens. Matches only users whose `type` is `APL938CallerIDUser` or `verified`.

Body: same as `/users/findbynumber`.

**Response `200`** — `true`
**Response `404`** — `false`

### GET `/users/:id`
Fetch one user by id.

**Response `200`** — `{ "status": true, "user": { ... }, "message": "User has been found Successfully" }`
When the id does not exist: `{ "status": false, "user": null, "message": "User does not exist" }`.

### PUT `/users/:id`
Updates a user profile. Accepts the same fields as register plus `profile_url`, `date_of_birth`, `gender` (extra fields are allowed by the validator).

**Response `200`** — `{ "status": true, "user": { ...updated user... }, "message": "User Updated successfully" }`
**Response `404`** — on failure.

### POST `/users/update_fcm`
Updates the FCM device token (and type / country / mobile) for a user. If a user with the given `mobile_number` exists, that row is updated; otherwise the row with `user_id` is updated.

| Field | Type | Required |
|---|---|---|
| `countryIso` | string | yes |
| `fcm_token` | string | yes |
| `user_id` | string | yes |
| `mobile_number` | number | yes |
| `type` | string | yes |

**Response `200`** — `{ "status": true, "message": "Token Updated", "user": { ... } }`
On failure: `{ "status": false, "message": "User not found to update", "user": null }`.

### DELETE `/users/:id`
Deletes a user by id.

**Response `200`** — `{ "status": true, "user": null, "message": "User has been deleted successfully" }`

---

## 3. Countries

Read-only reference data served from `src/data/countries.json` (no database). `POST`, `PUT` and `DELETE` are rejected.

### GET `/countries`
Returns all 247 countries.

**Response `200`**
```json
{
	"status": true,
	"successMessage": "Countires Data get successfully",
	"countries": [ { "id": "167", "name": "Pakistan", "iso2": "PK", "phonecode": "92", ... } ]
}
```

### GET `/countries/:iso2`
Returns one country by its iso2 code (e.g. `/countries/PK`). Unknown codes fall back to the United States.

**Response `200`** — a single country object.

---

## 4. User Friends (contact sync)

### POST `/user_friends`
Syncs the user's phone-book contacts. Each contact is stored (or matched by its unique mobile number) in `filtered_contacts`, then linked to the user in `user_friends`. Re-syncing the same contacts is idempotent.

| Field | Type | Required | Notes |
|---|---|---|---|
| `userId` | string \| number | yes | the syncing user's id |
| `contacts` | array | yes | max **50** per request; extras are ignored |
| `contacts[].name` | string | no | defaults to `"Unknown"` |
| `contacts[].mobile_number` | number | no | contacts **without** a number are skipped |
| `contacts[].country_iso2` | string | no | unknown codes fall back to `US` |

**Response `200`** (always, sync is fire-and-forget)
```json
{ "status": true, "successMessage": "Your friend list is updated", "contacts": [] }
```

### GET `/user_friends/:id`
Stub kept for app compatibility — always returns an empty list:
```json
{ "status": true, "successMessage": "Your friend list get successfully.", "contacts": [] }
```

`GET /`, `PUT /:id` and `DELETE /:id` return `405`.

---

## 5. Spam Report

### POST `/spam_report`
Reports a number as spam.

| Field | Type | Required |
|---|---|---|
| `userId` | number | yes |
| `mobile_number` | number | yes |
| `country_iso2` | string | yes |
| `type` | string | yes |
| `comment` | string | yes |
| `name` | string | no |

**Response `200`** — `{ "status": true, ... }` on success, `{ "status": false, "errorMessage": "..." }` on failure.

### POST `/spam_report/byNumber`
Returns all spam reports filed against a number.

| Field | Type | Required |
|---|---|---|
| `mobile_number` | number | yes |
| `country_iso2` | string | yes |

**Response `200`**
```json
{
	"status": true,
	"data": [ { "id": 12, "name": "Unknown Caller", "mobile_number": 3001234567, "type": "spam", "comment": "Telemarketing" } ],
	"successMessage": "data found"
}
```
When nothing is found: `{ "status": false, "data": null, "errorMessage": "data not found" }`.

`GET /` is rejected; `PUT /:id` and `DELETE /:id` return `405`.

---

## 6. Number Name Suggestion

### POST `/number_name_suggestion`
Stores a user's suggested name for a number. Each user has **one** suggestion per number (unique index on `filtered_contact_id` + `user_id`) — suggesting again overwrites the previous name. Unlike other endpoints, an unknown `country_iso2` is **rejected** here (no US fallback).

| Field | Type | Required |
|---|---|---|
| `userId` | number | yes |
| `mobile_number` | number | yes |
| `country_iso2` | string | yes |
| `name` | string | yes |

**Response `200`**
```json
{
	"status": true,
	"data": { "id": 3, "name": "Aftab Office", "filtered_contact_id": 25, "user_id": 1 },
	"successMessage": "Number Suggestion Successfully"
}
```
Unknown country: `{ "status": false, "data": null, "errorMessage": "Country Not Found" }`.

---

## 7. Network (carrier lookup)

### POST `/network`
Offline carrier / line-type lookup via libphonenumber — no database involved.

| Field | Type | Required |
|---|---|---|
| `mobile_number` | string \| number | yes |
| `iso2` | string | yes |

**Response `200`** — a plain string: `"<carrier or line type>, <country name>"`
```json
"Jazz, Pakistan"
```
Unparseable numbers fall back to `"Mobile, <country name>"`.

---

## 8. Number Search (caller ID)

### POST `/search`
The main caller-ID lookup. **Search order:**

1. **App users** — registered users matching the number (returned in `appuserresults`).
2. **db1** — the community `filtered_contacts` table with suggested names and spam counts.
3. **db2** — the external zoobidev contact databases (checked one by one until a hit).

| Field | Type | Required |
|---|---|---|
| `mobile_number` | string | yes |
| `iso2` | string | yes |

**Response `200` — app user hit** (`appuser: true`): user record in `appuserresults`, `results` is `null`.

**Response `200` — db1 / db2 hit** (`appuser: false`):
```json
{
	"status": true,
	"db": "db2",
	"appuser": false,
	"message": "Contact found successfully",
	"appuserresults": null,
	"results": {
		"filtered_contact_id": 465900001,
		"filtered_contact_name": "Aftab Khan",
		"spam_check": 0,
		"suggested_names": [ { "suggested_name": "Aftab Khan", "count": 1 } ]
	},
	"location_info": "Jazz, Pakistan"
}
```

**Response `200` — nothing found**
```json
{ "status": false, "db": "db2", "message": "Not Data Found", "appuser": false, "appuserresults": null, "results": null, "location_info": "Jazz, Pakistan" }
```

`location_info` is always populated (same logic as `/network`).

---

## 9. Agora (call tokens)

Token generation for Agora audio/video calls and chat. All tokens are valid for **8 hours**.

### POST `/agora/rtc`
| Field | Type | Required |
|---|---|---|
| `channelName` | string | yes |
| `uid` | number | yes |

**Response `200`** — `{ "status": true, "token": "...", "uid": 12345, "channelName": "...", "expireAt": 1760000000 }`
**Response `400`** — when `channelName` or `uid` is missing.

### GET `/agora/rtc/channel/:channelName/agorauid/:uid`
Same as above but via URL params; `uid` must be a non-negative number. Optional `?userAccount=<name>` switches to account-based tokens.

**Response `200`** — includes **`accessToken`** (the field the Android client parses) plus `token` (compat), `channelName`, `uid`, `userAccount`, `expireAt`.

### POST `/agora/generateVideoCallToken`
Identical body and response to `POST /agora/rtc` (kept as a separate route for the video-call flow).

### POST `/agora/generateUserToken`
| Field | Type | Required |
|---|---|---|
| `uid` | string | yes |

**Response `200`** — `{ "status": "success", "token": "..." }` (Agora Chat user token).

### GET `/agora/generateAppToken`
No parameters. Returns `{ "status": "success", "token": "..." }` (Agora Chat app token).
