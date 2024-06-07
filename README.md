# The Bussin Binary protocol spec
The bussin protocol is encoded in binary to ensure efficient
transfer of request and responses.

**Note**: All numerical values will be explicitly denoted with their base, 
unless they are decimal (base 10). For example, hexadecimal numbers will 
be prefixed with `'0x'`, binary numbers with `'0b'`, and so on. 
Only decimal numbers will be presented without a prefix, 
unless otherwise specified in a table or similar context.

## Base data types
The following are the base data types and their size as used in
BBP.

- u8  (1 byte)
- u16 (2 bytes)
- u32 (4 bytes)
- u64 (8 bytes)
- double
- float
- strings -> [String](#string)

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
  u8 flags;
}
```

- `magic_number`: Set to 0x00042069. The server/client should check if the first 4 bytes matches these sequence.
  Otherwise should cancel the request and respond with an error.
- `version_major`: Bussin protocol version number. e.g. if currently at version 69.1 it will be the 69 part
- `version_minor`: Bussing protocol version number minor. e.g. if currently at version 69.1 it will be the 1 part
- `action`: [Action](#action-type) type representing what action the client wants executed or what response code the server
  replied with.
- `flags`: Data format flags. Various flags that describe the binary content.

#### Flags
Flags that describe the binary data contained in this protocol. These are just bit flags. There are only 8 flags we can
squeeze on this field.

| value  | hex| name  | description
|--------|----|-------|------------
|00000001| 01 | utf16 | Treat strings as utf16 (default utf8)

##### How do i use flags.
The flags are bit encoded. You can set a flag value by doing OR operation on it.

```c
   u8 flags = 0;
   // set utf16 flag.
   flags = flags | 0b00000001; // or 0x01
```
Check if flag was set with AND operation:

```c
   u8 flags = 0b10110011; // various other flags
   u8 utf16_flag = 0b00000001;
   if ((flags & utf16_flag) == utf16_flag) {
      // do utf16 stuff
   }
```

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

### String <a name="string"></a>
Strings are represented as:

- for utf8
  
  ```c
  struct String {
    u32: length;
    u8[length] chars;
  }
  ```
- for utf16
  
  ```c
  struct String {
    u32: length;
    u16[length] chars;
  }
  ```

unlike c, strings are not null terminated. The strings are in utf8 format.

- `length` the number of characters(bytes) in the string
- `chars array` the characters representing the string. not null terminated.

Avoiding null termination allows for efficient reading of strings. The software
knows earlier how long the string is and allocate enough memory for it before reading.

### Settings <a name="settings"></a>
Settings are values passed during both request and response that are supposed to alter
the default behavior of the client/server. They can also carry extra information for the
protocol. The settings type/flag are of type u8 (1 byte). This means there are 256 possible
settings. Values from `0 to 254(0xfe)` are reserved for standard settings values. For your custom
settings, value `0xff(255)` indicates custom settings.

#### Standard settings (0x00 to 0xfe)

Settings marked NOT optional indicates that the server/client must parse the settings i.e. its mandatory to know how this field is structured.
To facilitate ignoring of optional settings, Every optional settings should have 4 bytes
after the tag indicating what is the number bytes on the value fields. Implementers that don't care
about a particular settings can infer the number of bytes to skip from the field length.

```c
struct OptionalSettings {
  u8 tag; // The settings type
  u32 length; // Number of bytes present in this settings excluding the length of tag (1byte) and length field(4bytes).
}
```
For mandatory settings you don't care about, you can check its size below and skip those number of
bytes to the next tag.

| hex(tag) | size(bytes) | name | type | description | Server(optional) | Client(optional)
|-----|-------------|------|------|-------------|------------------|-----------------
| 00  |     4       | BodyLength| u32 | The number of bytes contained in body content| false | false
| 01  | string.length| Host | string | The host domain name that this request was requested for| true | false


#### Custom settings (0xff)
These are user defined settings that the client and the server know about but it is not defined in the standard spec.
Since its arbitrary user defined encoding, there is no way for other servers/clients to know how to
deal with the encountered settings. This forces custom settings to define their length similar to optional settings so that
other servers/clients can ignore the next n bytes of the custom settings. If an implementation encounters a tag (0xff), It can read the next 4 bytes which 
represent length (n) of the custom settings and choose to ignore it by skipping the next n bytes or parse it.

```c
struct CustomSettings {
  u8 tag;
  u8 custom_type;
  u32 length;
  u8[length] bytes;
}
```

- `tag`: Equal to `0xff`. Indicates this is a non standard tag.
- `custom_type`: More 256 custom tags to play with depending on your needs.
- `length`: The number of bytes the value of the custom tag takes. You can skip
  this number of bytes if you don't recognize this custom tag.
  
