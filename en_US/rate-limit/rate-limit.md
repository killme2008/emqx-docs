# Rate Limit

EMQX allows for specifying limits on connection speed and messaging speed, using a backpressure scheme that avoids system overload at the entry point and guarantees system stability with predictable throughput.

## Limiter Types

EMQX uses the following types of limiters to specify the rate limits:

| Type          | Description                               | Post-Overload Behavior          |
| :------------ | :---------------------------------------- | :------------------------------ |
| bytes_rate    | Incoming message size in bytes per second per client | Pause receiving client messages |
| messages_rate | Incoming messages per second per client             | Pause receiving client messages |
| max_conn_rate | Connections per second per listener                     | Pause receiving new connections |

For example, to set a limiter for the default TCP listener, you can configure it in emqx.conf as follows:

```bash
listeners.tcp.default {
  bind = "0.0.0.0:1883"
  max_conn_rate = "1000/s"
  messages_rate = "1000/s"
  bytes_rate = "1MB/s"
}
```

## Rate Unit

### Time Unit

The supported time unit in the rate value could be:

- **s** :: Second
- **m** :: Minute
- **h** :: Hour
- **d** :: Day

The time unit also can be an interval value, like `1000/10s` means setting the limit to 1000 per every 10 seconds.

### Size Unit

The supported size unit in the rate value could be:

- **KB** :: Kilobyte
- **MB** :: Megabyte
- **GB** :: Gigabyte

