# CBOR-RPC

This specification is a sketch of a simple protocol, intended for anyone to build upon.

## Goals :

- Focus on being relatively small.
- Use CBOR format.
- Can transfer anything.
- Can be streamed and bidirectional.

## Data Transmission

A general approach to rpc protocol is to use
a [Type-Length-Value](https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value) format.
We already know the format, CBOR, so we can skip it. A CBOR message describe itself, there is no need to indicate the
size in advance.

Unlike the [json-rpc](https://www.jsonrpc.org/specification) which contains a "data" object **inside** the header,
The CBOR-RPC format is sequential. It allows a parser to fully parse the header before proceeding to the message.

An envelope is composed of 3 parts:

1. CBOR list: The **Header**, it contains packet information.
2. CBOR data: The **Message**, any kind of encoded CBOR data or Null (`0xF6`)
3. CBOR uint + bytes: The **Payload**, any raw data of the size described with the preceding CBOR uint or Null (`0xF6`)

Envelopes can be sent successively on the same communication stream. The channel can be anything. Websocket, bluetooth,
TCP/IP socket, etc.

<details>
<summary>Why is there a Message <b>AND</b> a Payload ?</summary>

The **Message** should describe data for the function to call as parameter. The **Payload**, should be like an input
stream in a command line utility.

Also, from a parser perspective, a CBOR encoded data is something finite. The [cbor.io](https://cbor.io/impls.html) page
say the basic is to have `data = CBOR.decode(encoded)`. However, sometimes you want to transfer raw bytes, an image, a
file, etc… In these cases, you might want to handle the data as a stream. Yes you could have a major 3 byte string, but
it would not be the raw data, it would be the raw data encapsulated in a CBOR message. Adding complexity to the parser.

</details>

### Envelope

The **Header** is a tuple looking like this `[vrpc, vapi, id, event, name]`.

Where:

- `vrpc (uint) (1)`: Version of this protocol. As of writing `1`.
- `vapi (uint)`: This field can be used to display the version of the provided api.
- `id (uint)`: Represents the packet identifier. This field can't be null.
- `event (uint)`: The type of action to execute. It can be anything, but you can use this set:
    - `query (0)`: One side ask something to the other side. This implies a response from the other side.
    - `success (1)`: Success response from a query.
    - `error (2)`: Error response from a query.
    - `notification (3)`: Same as `query` without the other side responding
- `name(text)`: The name of the procedure to call.

Notes about the header:

- One api can add as much `event` types as necessary. It is like the status code of an HTTP request, if you prefer, you
  can use the same values as HTTP.
- `event` types are just semantic cues to transit packets. Nothing disallow `success` and `error` events to be fired
  multiple times to the same `query`. Or a `query` to have a `notification` as response. You make sense of what is doing
  what in your api.
- Both client and server can use the same `id` in different messages. Rule of thumb is:
    - When receiving a `query` or a `notification` the `id` represent the other side.
    - When receiving a `success` or `error` the `id` should be one of previously sent messages.

The **Message** can be any kind of CBOR data. A list, an object, an int, a text string, etc… The message can be
Null (`0xF6`) to indicate that there is no message.

The **Payload** consist of one CBOR uint and the corresponding number of bytes afterward. The CBOR uint can be replaced
with Null (`0xF6`) if there is no payload.

Nothing disallow an envelope to have both **Message** and **Payload** set to Null. A method call without parameter might
be sufficient.

### Example

When encoding, you write the header as tuple (list) following the fields orders `vrpc`, `vapi`, `id`, `event`, `name`.

```yaml
# [vrpc, vapi, id, event, name]
[ 1, 0, 33, 0, "my.function.name" ]
```

With everything combined, an envelope can look like this:

```
85 01000000646563686F (10) # Header: [vrpc: 1, vapi: 0, id: 0, event: 0, name: "echo"]
65 48656C6C6F        (6)  # Message: text("Hello")             
05 576F726C64        (6)  # Payload: uint(5) + 'World'
   

Message : 8501000000646563686F6548656C6C6F05576F726C64
Total : 22 bytes
```

And a response like this:

```
85 01000001646563686F (10) # Header: [vrpc: 1, vapi: 0, id: 0, event: 1, name: "echo"]
65 48656C6C6F        (6)  # Message: text("Hello")             
05 576F726C64        (6)  # Payload: uint(5) + 'World'
   

Message : 8501000001646563686F6548656C6C6F05576F726C64
Total : 22 bytes
```

## FAQ

- I want to add metadata to the Header, so I can handle the Message and Payload correctly. </br>
  No, you want to have different hedears for different messages. Think of RPC as a single function in your favorite
  programming language. When you call it, the type of the arguments are known.

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
- [ 1, 0, 0, 0, "add" ] [ 10, 5 ] 0xF6 # A
- [ 1, 0, 0, 2, "internal" ] "add function does not exists" 0xF6 # A
```

- What if I want to bulk queries or notifications? </br>
  Buffer your queries and send them all at once or add a special `event` to the protocol `bulk_query`,
  `bulk_success` (with any custom values like 53 and 54).

```yaml
- [ 1, 0, 0, 53, "sum" ] [ [ 10, 5 ], [ 33, 2, 5 ] ] 0xF6 # A
- [ 1, 0, 0, 54, "sum" ] [ 15, 40 ] 0xF6 # B
```

- What happens if two distinct messages have the same id at the same time during communication? </br>
  That's undefined behavior. One implementation can check this and send an error back.
