# Changelog

## 0.38.0

### Fixed

- An error inside an array reply (one failing command in `MULTI`/`EXEC`) no
  longer leaves the connection out of step: the whole array is read before the
  error is raised, so the next command gets its own reply instead of a
  leftover from the last one.
- A bulk reply that is not valid UTF-8 (a binary value) is returned as bytes
  instead of crashing the read.
- `null`, Bools, lists, maps and secrets are refused as command arguments
  instead of being sent as their display text (`set(r, "k", null)` used to
  store the string `"null"`).
- `encode("PING")` with a string instead of a list is refused instead of
  sending each character as an argument.
- A malformed reply or a connection dropped mid-reply raises a
  `redis_client` error instead of an internal one.

### Added

- `connect` and `connect_tls` take `password:` and `username:` and log in
  before returning; `auth(sock, password, username:)` logs in on an open
  connection. A `secret` password is accepted.
- `set` takes `ex:` (seconds) or `px:` (milliseconds) to set an expiry.
- `binary: true` on `command`, `get` and `lrange` returns every bulk value as
  bytes.
- Server errors carry `code`, the error's first word (`ERR`, `WRONGTYPE`), and
  every error carries a `reason`.

### Breaking

- Errors are kind `redis_client` with a `reason` (`server_error`,
  `auth_failed`, `closed`, `protocol`) instead of kind `redis`.
  Migration: match `get(e, "kind") == "redis_client"`, or better
  `get(e, "reason")`, where you matched `"redis"`.
- A bulk reply that is not valid UTF-8 comes back as bytes, where it used to
  raise.
  Migration: pass `binary: true` if you want bytes every time, or check
  `type_of(v) == "bytes"`.
- `null`, Bool, list, map and secret arguments now raise kind `bug`.
  Migration: convert the value yourself (`string(flag)`, `json.encode(m)`,
  `reveal(s)`) before passing it.
