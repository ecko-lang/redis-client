# Redis Client

A Redis client (RESP2) for [Ecko](https://ecko.sh), written in Ecko. It frames the wire protocol over `std.net`'s raw sockets
(`connect` / `connect_tls`, `send`, `recv_until`, `recv_exact`) and the `bytes`
type.

## Install

```bash
ecko add https://github.com/ecko-sh/redis-client
```

`ecko add` vendors the package into your project's `./vendor/redis/` and pins it
by SHA-256 in `ecko.lock`. Grant it the network capability in your `ecko.json`:

```json
{
  "dependencies": {
    "redis": {
      "source": "https://github.com/ecko-sh/redis-client",
      "grant": ["net"]
    }
  }
}
```

## Use

```ecko
import redis

r = redis.connect("127.0.0.1", 6379)   # or redis.connect_tls(host, port)

redis.set(r, "greeting", "hello")
redis.get(r, "greeting")               # -> "hello"
redis.incr(r, "counter")               # -> 1
redis.lrange(r, "mylist", 0, -1)       # a RESP array -> an Ecko list
redis.close(r)
```

## API

| function | notes |
|----------|-------|
| `connect(host, port)` / `connect_tls(host, port)` | open a connection (handle) |
| `command(sock, [parts])` | send any command as a RESP array, decode one reply |
| `encode([parts])` | the RESP bytes for a command (build/inspect without sending) |
| `ping` `set` `get` `del` `incr` `decr` `exists` `expire` `ttl` `keys` `lpush` `rpush` `lrange` | thin wrappers over `command` |
| `close(sock)` | close the connection |

### Reply mapping

| RESP | Ecko |
|------|------|
| simple string (`+OK`) | `String` |
| integer (`:1`) | `Int` |
| bulk string (`$5\r\nvalue`) | `String` (UTF-8) |
| nil (`$-1` / `*-1`) | `null` |
| array (`*2...`) | `List` |
| error (`-ERR ...`) | raised as `{ kind: "redis", message }` |

Bulk replies decode as UTF-8 text (the common case). For binary values, use
`command` and decode the bytes yourself.

## Testing

```bash
ecko test          # offline: RESP encoding (redis_test.ecko), no server needed
ecko example.ecko  # live round trip against a Redis on 127.0.0.1:6379
```

## License

MIT
