# Redis Client - Ecko Std Lib Package

A Redis client (RESP2) for [Ecko](https://ecko.sh), written in Ecko. It frames the wire protocol over `std.net`'s raw sockets
(`connect` / `connect_tls`), read and written with `std.io`'s `write`,
`read_until` and `read_exact`, and the `bytes` type.

## Install

```bash
ecko get github.com/ecko-lang/redis-client --as redis
```

`ecko get` vendors the package under
`./vendor/github.com/ecko-lang/redis-client/` and pins a file-tree hash in
`ecko.sum`.

Without `--as`, `ecko get` records this dependency under the alias
`redis_client` (the repo name with `-` mapped to `_`); `--as redis` is what makes
`import redis` work. It writes the entry as
`"redis": "github.com/ecko-lang/redis-client@0.37.0"`. Expand it to the map form in
your `ecko.json` to grant the network capability the client needs:

```json
{
  "dependencies": {
    "redis": {
      "path": "github.com/ecko-lang/redis-client",
      "version": "0.37.0",
      "grant": ["net"]
    }
  }
}
```

```ecko
import redis
```

## Usage

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
