# redis-client

A Redis client (RESP2) for [Ecko](https://ecko.sh), written in Ecko over
`std.net` sockets, plain or TLS, with password and ACL-user login. Needs `net`;
grant it when you import the package.

## Install

```bash
ecko get github.com/ecko-lang/redis-client --as redis
```

Then grant the network capability in your `ecko.json`:

```json
{
  "dependencies": {
    "redis": {
      "path": "github.com/ecko-lang/redis-client",
      "version": "0.38.0",
      "grant": ["net"]
    }
  }
}
```

```ecko norun
import redis   # granted net in ecko.json
```

## Usage

```ecko norun
import redis   # granted net in ecko.json

r = redis.connect("127.0.0.1", 6379)   # add password: (and username:) for a protected server

print(redis.set(r, "greeting", "hello"))              # OK
print(redis.get(r, "greeting"))                       # hello
print(redis.set(r, "session:42", "tok", ex: 3600))    # OK - gone in an hour
print(redis.incr(r, "counter"))                       # 1
redis.rpush(r, "jobs", "a")
redis.rpush(r, "jobs", "b")
print(redis.lrange(r, "jobs", 0, -1))                 # [a, b]
print(redis.get(r, "missing"))                        # null
redis.close(r)
```

The output comments assume an empty database.

### Logging in

```ecko norun
import redis   # granted net in ecko.json
import std.os

# A password only (`requirepass`), over TLS:
r = redis.connect_tls("cache.example.com", 6380, password: secret(os.env("REDIS_PASSWORD")))

# An ACL user (Redis 6+), or log in on an open connection:
r2 = redis.connect("127.0.0.1", 6379, password: secret(os.env("REDIS_PASSWORD")), username: "app")
redis.auth(r2, secret(os.env("REDIS_PASSWORD")), username: "app")   # OK
```

`password` may be a String or a `secret`; a secret is revealed only into the
`AUTH` command.

### Binary values

```ecko norun
import redis   # granted net in ecko.json

r = redis.connect("127.0.0.1", 6379)
redis.set(r, "blob", b"\xff\x00\x01")
print(redis.get(r, "blob"))                    # b"\xff\x00\x01" - not UTF-8, so bytes
redis.set(r, "word", "hi")
print(redis.get(r, "word", binary: true))      # b"hi" - always bytes
```

### Building a command without sending it

```ecko
import redis

print(redis.encode(["SET", "k", 60]))   # b"*3\r\n$3\r\nSET\r\n$1\r\nk\r\n$2\r\n60\r\n"
```

## API

### Connections

| function | returns |
|---|---|
| `connect(host, port, password = null, username = null)` | a connection over plain TCP, logged in when `password` is given |
| `connect_tls(host, port, password = null, username = null)` | the same over TLS, with the certificate verified |
| `auth(sock, password, username = null)` | `"OK"` once the server accepts the credentials |
| `close(sock)` | `null`; closes the connection |

### Commands

| function | returns |
|---|---|
| `command(sock, parts, binary = false)` | the decoded reply to any command |
| `encode(parts)` | the RESP bytes for a command, without sending it |
| `ping(sock)` | `"PONG"` |
| `set(sock, key, value, ex = null, px = null)` | `"OK"`; `ex` seconds or `px` milliseconds sets an expiry |
| `get(sock, key, binary = false)` | the value, or `null` when the key is not set |
| `del(sock, key)` | `1` if the key was removed, `0` if it was not there |
| `incr(sock, key)` | the integer after adding one |
| `decr(sock, key)` | the integer after subtracting one |
| `exists(sock, key)` | `1` if the key is set, `0` if not |
| `expire(sock, key, secs)` | `1` if the key was there to expire, `0` if not |
| `ttl(sock, key)` | seconds left; `-1` for no expiry, `-2` for no key |
| `keys(sock, pattern)` | a List of keys matching a glob (walks the whole keyspace) |
| `lpush(sock, key, value)` | the list's new length after prepending |
| `rpush(sock, key, value)` | the list's new length after appending |
| `lrange(sock, key, start, stop, binary = false)` | the list items from `start` to `stop` inclusive |

## Notes

### Errors

| kind | reason | when |
|---|---|---|
| `redis_client` | `server_error` | the server answered with an error; `code` holds its first word (`ERR`, `WRONGTYPE`, ...) |
| `redis_client` | `auth_failed` | the server refused the credentials (`code` as above); the password is never in the message |
| `redis_client` | `closed` | the server closed the connection before replying in full |
| `redis_client` | `protocol` | the reply was malformed; close the connection |
| `net` | - | the server could not be reached, or the TLS handshake failed |
| `bug` | `bad_argument` | an argument has no wire form (`null`, a Bool, a list, a map, a secret), `parts` is not a non-empty list, or `ex`/`px` is invalid |

```ecko norun
import redis   # granted net in ecko.json

r = redis.connect("127.0.0.1", 6379)
try {
    redis.incr(r, "greeting")
} catch (e) {
    match get(e, "reason") {
        "server_error" => print("Redis said: {e.message}")
        _ => error(e)
    }
}
```

After `closed` or `protocol` the connection is out of step with the server:
close it and open a new one. A `server_error` leaves it usable.

### Reply mapping

| RESP | Ecko |
|---|---|
| simple string (`+OK`) | String |
| integer (`:1`) | Int |
| bulk string (`$5\r\nvalue`) | String when valid UTF-8, bytes otherwise; always bytes with `binary: true` |
| nil (`$-1` / `*-1`) | `null` |
| array (`*2...`) | List |
| error (`-ERR ...`) | raised as `redis_client`/`server_error` |

An error inside an array, such as one failing command in a `MULTI`/`EXEC`
block, is raised after the whole array has been read. The error carries the
full reply as `reply`, with each failed element as an error map, so the other
results are not lost and the connection stays in step.

### Arguments

Strings are sent as their UTF-8 bytes, Ints, Floats and decimals as their
decimal text, and bytes unchanged. Anything else is refused rather than sent
as its display text, so `set(r, "k", null)` raises instead of storing `"null"`.

### Limits

RESP2 only, one command at a time (no pipelining or pub/sub). `SELECT`, hash
commands and other commands without a helper go through `command`. Socket
deadlines are `std.net`'s (`ECKO_NET_TIMEOUT_MS`, 30 s by default).

## Testing

```bash
ecko test
```

The tests are offline: encoding is checked against known RESP bytes, and
decoding, `AUTH` and `SET` options run against a scripted fake server on
loopback. `ecko example.ecko` is a live round trip against a Redis on
127.0.0.1:6379.

## License

[MIT](LICENSE)
