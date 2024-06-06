# The Bussin Binary protocol spec
The bussin protocol is encoded in binary to ensure efficient
transfer of request and responses.

## Base data types
The following are the base data types and their size as used in
BBP.

- u8  (1 byte)
- u16 (2 bytes)
- u32 (4 bytes)
- u64 (8 bytes)
- double
- float
- strings -> {u4: length, u8[length]}

## Endianness
BBP uses Big Endian (Network byte order)

## Format
The general valid request/response of bussin protocol is as follows.

|       | Header |Path         | Settings    | body
|-------|--------|-------------|-------------|---
|Size   |8 bytes |string length| n           | n
|Offset |0 bytes | +8 bytes    | +path.length| +settings.length bytes
|Type   |[Header](#header-type)| [String](#string) | [Settings](#settings) | raw binary bytes

## Data types
### Header <a name="header-type"></a>
The header is a 8byte sequence which is used to indicate this TCP/UDP
request speaks bussin.

```c
struct Header {
  u32 magic_number;
  u8 version_major;
  u8 version_minor;
  Action action;
  u8 padding;
}
```

- `magic_number`: Set to 0x00042069. The server/client should check if the first 4 bytes matches these sequence.
  Otherwise should cancel the request and respond with an error.
- `version_major`: Bussin protocol version number. e.g. if currently at version 69.1 it will be the 69 part
- `version_minor`: Bussing protocol version number minor. e.g. if currently at version 69.1 it will be the 1 part
- `action`: [Action](#action-type) type representing what action the client wants executed or what response code the server
  replied with.
- `padding`: Currently unused byte. Maybe it will be used to expand possible response codes to 65536.

### Action(u8) <a name="action-type"></a>
This is an enum of values that tell the server what action to perform and the client what was the response code of
the action performed. There are possible 256 different actions possible and response codes of between 0 and 255.

#### Action request
These are the actions the server should handle.

| hex | name   | Description
|-----|--------|------------
| 00  | Noop   | perform no operation, possibly wasting CPU cycles and network resources
| 01  | Read   | Read from a server (GET)
| 02  | Write  | Write to the server (POST)
| 03  | Modify | Modify operation (PUT)
| 04  | Remove | Delete operation (DELETE)

#### Action response
These are the response values to clients

| hex | name   | Description
|-----|--------|------------
| 00  | Seen   | The request was seen by the server but no action was performed. Useful for debugging connections.
| 01  | Success | The Action was successful (OK)
