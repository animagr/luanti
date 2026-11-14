# Fork changes

This fork backs the game Luanti Builders and is not meant to go upstream.
This file lists every difference from upstream Luanti (`master`, 5.19.0-dev).

## Server-settable third-person camera distance

**What:** `ObjectRef:set_camera(params)` accepts a number field
`third_person_distance`: how far the third-person camera may move away from
the player, in nodes. It is clamped to 1 to 20, and the default is 2.75 (the
value upstream hardcodes). A table without the field leaves the distance
unchanged. `set_camera(nil)` resets it to the default. Values that are not
numbers, and NaN, are ignored. `get_camera()` returns the field.
`core.features.camera_third_person_distance` is `true`.

The camera still stops half a node in front of the first walkable node, as
upstream does.

**Network:** `TOCLIENT_CAMERA` (0x48) now carries an `f32` distance after the
mode byte. The protocol version is unchanged:

- A stock client reads the mode byte and ignores the extra bytes.
- A fork client reads the distance only when the packet has bytes left, so
  it still works with a stock server (which sends the mode byte only).
- A fork client clamps the received value and ignores NaN.

**Files:**

- `src/player.h`: `Player::third_person_camera_distance` and the constants
  `THIRD_PERSON_DISTANCE_DEFAULT`, `_MIN` and `_MAX`.
- `src/script/lua_api/l_object.cpp`: `l_set_camera` and `l_get_camera`.
- `src/server.cpp`: `Server::SendCamera` appends the distance.
- `src/network/clientpackethandler.cpp`: `Client::handleCommand_Camera`
  reads it when present.
- `src/network/networkprotocol.h`: packet comment.
- `src/client/camera.cpp`: `Camera::update` uses the distance as the bound
  of the third-person collision loop instead of the hardcoded 2.75.
- `builtin/game/features.lua`: the feature flag.
- `doc/lua_api.md`: `set_camera`, `get_camera` and `core.features`.

**Why:** Luanti Builders mods need to set, per player, how far the
third-person camera pulls back; upstream fixes it at 2.75 nodes.
