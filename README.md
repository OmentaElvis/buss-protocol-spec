# The Bussin Binary protocol spec


## Networking
The Bussin protocol is designed to be transport-agnostic, supporting both User Datagram Protocol (UDP) 
and Transmission Control Protocol (TCP) connections. The standard ports for Bussin protocol are 420 for UDP and 42069 for TCP.
When choosing between UDP and TCP, consider the application's requirements. UDP is well-suited for time-sensitive applications
 that prioritize low latency, such as streaming, whereas TCP is ideal for reliable, lossless data transfer where guaranteed delivery is essential.

## Security

To ensure the confidentiality and integrity of data, all Bussin protocol 
communications should be encrypted using public key cryptography.

### Live Servers
Live servers, which respond to active operations and CRUD requests, 
require an active SSL/TLS key encryption between the browser 
and server. This enables secure, encrypted communication between the client and server.


### Static Servers
Static servers serve pre-compiled and minified Bussin sites as a 
single file archive. Since these servers only serve static files, 
they do not process Bussin protocol requests or generate dynamic 
content. Examples of static server hosting options include local 
storage, cloud storage services (e.g., Dropbox, Google Drive), Git services 
(e.g., Gitlab, Github through releases page), and other file hosting platforms. 
To ensure the authenticity and integrity of the served file, the author should 
digitally sign the file using their private key. The corresponding 
public key should be served alongside the file, allowing the user's browser 
to verify the signature and ensure that the file has not been tampered 
with during transmission and originates from the trusted author.

## Compilation
Static websites can be compiled into a single, self-contained archive that 
includes all necessary data to run the site successfully in a browser. This 
compilation enables flexible hosting options, such as hosting the site locally, 
sharing the file with others, or storing it on public file storage platforms.

The compilation process also involves Just-In-Time (JIT) compilation, which 
reduces the code size and improves performance. The specific optimizations 
applied depend on the compiler specifications and settings.

One of the benefits of static sites is their caching friendliness. 
Browsers can compile and cache the site, allowing for faster loading 
times on subsequent requests.

Additionally, static sites can be configured to request connections to 
external APIs or parent Bussin servers through Bussin protocol settings. 
This feature is particularly useful for compiled WebX games that require 
central servers for gameplay or data synchronization.

# Binary format

The bussin protocol is encoded in binary to ensure efficient
transfer of request and responses. Why binary? Its easier and more
easier to pack more information into a single byte that ASCII key value pairs.
Its also faster.

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

|       | Header |Path         | Setting count|  Settings    | body
|-------|--------|-------------|--------------|-------------|---
|Size   |8 bytes |string length| 2 bytes      | n           | n
|Type   |[Header](#header-type)| [String](#string) | u16 | [Settings](#settings) | raw binary bytes

```c
struct Bussin {
  Header header;
  String path;
  u16 settings_count;
  Settings[settings_count] settings_pool;
  u8[] body;
}
```

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
|00000010| 0a | state | Treat this connection as stateful. This demands the server to keep track of the current session.

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
| 05  | UpgradeState | Requests the server to upgrade the connection to a[ stateful](#stateful) protocol, allowing the client to save settings for the current session.
| 06  | SaveSettings | Instructs the server to store the current settings in the session store, available only in [stateful](#statefulness) mode. This allows the client to save commonly repeated settings that the server can reference for the remainder of the session.

#### Action response
These are the response values to clients

The following are the range of response codes and their types.

- **Informational (0x00-0x1F)**
  This group of response codes provides information about the processing of a request. They indicate that the request has been received and is being processed, but do not indicate the outcome of the request. These codes are used to provide interim information about the request.

- **Success (0x20-0x3F)**
  This group of response codes indicates that a request has been successfully processed. They confirm that the requested action has been completed and that the response contains the expected outcome.

- **Redirection (0x40-0x5F)**
  This group of response codes indicates that the requested resource is not available at the expected location. They provide information about the new location of the resource or alternative actions that can be taken.

- **Client Errors (0x60-0x7F)**
  This group of response codes indicates that the request contains errors or cannot be processed due to client-side issues. They provide information about the nature of the error and may suggest corrections or alternative actions.

- **Server Errors (0x80-0x9F)**
  This group of response codes indicates that the request cannot be processed due to server-side issues. They provide information about the nature of the error and may suggest alternative actions or provide information about when the service may be available again.

- **Reserved (0xA0-0xFF)**
  This range of response codes is reserved for future use or custom implementation. They may be used to provide additional information or functionality in specific implementations.

| hex | name   | Description
|-----|--------|------------
| 00  | Seen   | The request was seen by the server but no action was performed. Useful for debugging connections.
| 20  | Success | The Action was successful (OK)
| 21  | Stateful mode enabled | The server has successfully upgraded the connection to a stateful protocol and is ready to store settings for the current session.
| 60  | NotFound | The requested resource does not exist or is not available for the requested action.
| 61  | AccessDenied | The client is not authorized to perform the requested action on the specified resource.
| 62  | MalformedRequest | The request contains invalid or malformed data, preventing the server from processing it.
| 63  | <a name="nostate">No state</a> | The server declines to upgrade the protocol to a stateful mode, indicating that it is unable or unwilling to maintain protocol state information
| 64  | State storage full | The server cannot store any more settings as the session store is full.
| 80  | InternalError | An unexpected error occurred on the server, preventing it from processing the request.
| 81  | Session termination | The server is terminating the current session due to internal reasons, which will cause the connection to revert to a stateless protocol. The client should resume sending full settings with each request.

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
| 0a  |     4       | State storage size | u32 | The allowed storage size for active session. This is the space the server is willing to allocate for bussin settings in stateful mode. | true | true
| 0b  |     8       | StateId | u64 | Unique identifier for the current session/state, sent by the server and echoed back by the client in every request when the stateful bit flag is set. | true | true


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
  
## Statefulness <a name="stateful"></a>
By default, Bussin connections are stateless, meaning each request is processed independently without retaining any information from previous requests. As a result, the server requires the client to resubmit identifying headers with every request and response.

However, clients can opt to upgrade to a stateful protocol, provided the server supports it. If the server does not support stateful mode, it should reject the request using the [No State (0x0a)](#nostate) return code.

In stateful mode, the server is responsible for storing and tracking client-specified settings for future reference. The server can define limits on the maximum memory allocated for storing settings and the duration for which a state remains valid before it is deleted.

While stateful connections may enhance communication speed between the server and client, they come at the cost of increased server memory usage to maintain state information.
