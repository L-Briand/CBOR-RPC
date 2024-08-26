# CBOR-RPC

This project is just a sketch of mine, of a simple protocol so that anyone can build upon it.

## Goals :

- Focus on being relatively small.
- Use CBOR format.
- Can transfer anything.
- Can be streamed and bidirectional.

## Data Transmission

An envelope is composed of four parts:

1. CBOR uint: Size of the header
2. CBOR list: The header
3. CBOR uint: Size of the payload
4. The payload

Envelopes can be sent successively on the same communication stream. The channel can be anything. Websocket, bluetooth,
TCP/IP, etc.

### Envelope

A `header` is a tuple looking like this `[id, vrpc, vapi, event, name]`.

Where:

- `id (uint)`: Represents the packet identifier. This field can't be null.
- `vrpc (uint) (1)`: Version of this protocol. As of writing `1`.
- `vapi (uint)`: This field can be used to display the version of the provided api.
- `event (uint)`: The type of action to execute. It can be anything, but you can use this set:
    - `query (0)`: One side ask something to the other side. This implies a response from the other side.
    - `success (1)`: Success response from a query.
    - `error (2)`: Error response from a query.
    - `notification (3)`: Same as `query` without the other side responding
- `name(text)`: The name of the procedure to call.

A `payload` can be anything. Cbor encoded value or not.

Notes:

- One api can add as much `event` types as necessary. It is like the status code of an HTTP request, if you prefer, you
  can use the same values as HTTP.
- `event` types are just semantic cues to transit packets. Nothing disallow `success` and `error` events to be fired
  multiple times to the same `query`. Or a `query` to have a `notification` as response. You make sense of what is doing
  what in your api.
- Both client and server can use the same `id` in different messages. Rule of thumb is:
    - When receiving a `query` or a `notification` the `id` represent the other side.
    - When receiving a `success` or `error` the `id` should be one of previously sent messages.

### Example

When encoding, you write the header as tuple (list) following the fields orders `id`, `vrpc`, `vapi`, `event`, `name`.

```yaml
# [id, vrpc, vapi, event, name]
[ 33, 1, 0, 0, "my.function.name" ]
```

With everything combined, an envelope can look like this:

```
0A (1)                        # Size of header
8500010000646563686F (10)     # [ 0, 1, 0, 0, "echo"]
0C (1)                        # Size of payload
6B48656C6C6F20776F726C64 (12) # "Hello world" (CBOR)

Message : 0A8500010000646563686F0C6B48656C6C6F20776F726C64
Total : 24 bytes
```

And a response like this:

```
0A (1)                        # Size of header
8500010001646563686F (10)     # [ 0, 1, 0, 1, "echo"]
0C (1)                        # Size of payload
6B48656C6C6F20776F726C64 (12) # "Hello world" (CBOR)

Message : 0A8500010001646563686F0C6B48656C6C6F20776F726C64
Total : 24 bytes
```

## FAQ

- I want to add metadata to the header, so I can handle the payload correctly. </br>
  No, you want to have different header names or a payload containing parameters. Think of RPC as a single function in
  your favorite programming language. When you call it, you only pass arguments.

- Why does the `name` field is a String and not an Int like the `event` field? </br>
  `event` is not meant to change often; once you have a set of events meeting your requirements, you will not change it.
  However, function names are something that changes and are added all the time. Logging can be much easier in the
  long term with string as name. Strings also allows to be more lenient when building things.

- Why the `id` field is required for notification? </br>
  So the server can still send an error back if the notification fails to run or some restrictions are applied.
  The client can omit it.

- What if an error occurs because of network, serialization or any other cause? </br>
  There are no default error messages. An implementation can choose to have special `event` + `name` field to handle
  such cases. Like:

```yaml
- [ 0, 1, 0, 0, "add"] [ 10, 5 ] # A
- [ 0, 1, 0, 2, "internal"] "add function does not exists" # A
```

- What if I want to bulk queries or notifications? </br>
  Buffer your queries and send them all at once or add a special `event` to the protocol `bulk_query`,
  `bulk_success` (with any custom values like 53 and 54).

```yaml
- [ 0, 1, 0, 53, "sum" ] [ [ 10, 5 ], [ 33, 2, 5 ] ] # A
- [ 0, 1, 0, 54, "sum" ] [ 15, 40 ] # B
```

- What happens if two distinct messages have the same id at the same time during communication? </br>
  That's undefined behavior. One implementation can enforce such a case and send an error.
