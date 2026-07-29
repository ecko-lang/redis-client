# redis-client

## `encode(parts)`

encode(parts) -> the RESP bytes for a command. Public so you can build a
custom command or inspect the wire format without sending it.

## `connect(host, port)`

Open a plain TCP connection to a Redis server. Needs the `net` capability.

## `connect_tls(host, port)`

Open a TLS connection to a Redis server - use this for anything over a
network you do not control. Needs the `net` capability.

## `close(sock)`

Close the connection.

## `command(sock, parts)`

command(sock, parts) -> one decoded reply. Every helper is a wrapper over it,
and you can call it directly for commands without a helper.

## `ping(sock)`

Check the server is answering. Returns "PONG".

```ecko
sock = connect("127.0.0.1", 6379)
ping(sock)   # "PONG"
```

## `set(sock, key, value)`

Store a string value under `key`, replacing anything already there.

## `get(sock, key)`

The value at `key`, or null if nothing is stored there.

## `del(sock, key)`

Remove `key`. Returns 1 if it was there, 0 if it was not.

## `incr(sock, key)`

Add one to the integer at `key` and return the result. An unset key counts
as 0, so the first call gives 1.

## `decr(sock, key)`

Subtract one from the integer at `key` and return the result.

## `exists(sock, key)`

1 if the key is set, 0 if it is not.

## `expire(sock, key, secs)`

Give `key` a lifetime in seconds, after which Redis drops it. Returns 1 if
the key was there to expire.

## `ttl(sock, key)`

Seconds left before `key` expires. -1 means it has no expiry set, -2 means
there is no such key.

## `keys(sock, pattern)`

Every key matching a glob pattern. This walks the entire keyspace, so treat
it as a debugging tool rather than something to call in a request.

## `lpush(sock, key, value)`

Prepend a value to the list at `key`. Returns the list's new length.

## `rpush(sock, key, value)`

Append a value to the list at `key`. Returns the list's new length.

## `lrange(sock, key, start, stop)`

Items from the list at `key`, between `start` and `stop` inclusive.
Negative indices count back from the end, so 0 and -1 gets everything.
