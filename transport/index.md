# Transport packets
> &larr; Back to [Home](../index.md)

---

The Plabble Transport Protocol (**PTP**) uses a special transport codec for the delivery of the packets between clients and servers. There are two types of transport packets:
[Request Packets](#request-packet) and [Response Packets](#response-packet). The packets are sized dynamically using [flags](#request-flags) and length attributes to reduce the size of the packets to a minimum.

### **Bucket storage**
A Plabble server stores the data in **buckets**. You can see a bucket as a key-value slot on a server with an authorization system to prevent access to users that do not own the secret key. Every bucket consists of a list with a maximum of 65,536 slots. You can add, edit or remove slots from/to a bucket. Thanks to a **permission system** it is possible to add access restrictions to a bucket, so you can make buckets immutable or unreadable for users without the authorization key. The bucket storage system can optionally be used for multiple purposes on top of multiple protocols because the server acts as a key-value storage system.

---

## **Request packet**
![Transport Packet](/img/transport-request-bytemap.drawio.png)

_Figure A: Plabble Request Packet_
- *: The length is [dynamically sized](#dynamically-sized-length) and not always included
- **: The type and flags use 4 bits each from 1 byte. Notice these are the Most Significant Bits of this byte
- ***: The [MAC](#authentication) is optional and is toggled with the flag on the 5th bit on the type/flag byte.

![Type-flag bitmap](/img/transport-typeflag-bitmap.png)

_Figure B: Type/Flag bitmap. First 4 bits are the type, the last 4 bits are the flags_
- *: bit-flag #5 is reserved for indicating if the MAC is present.

---

In _Figure A_ the byte map of the Plabble Request Packet is shown. The packets consists of the following parts:
- The **Length**, a [dynamically sized](#dynamically-sized-length) 7 - 28 bits unsigned number that indicates the size of the packet in bytes. (this does not include the length of the _Length_ field itself, just the size of the following data). The length is excluded if a protocol is used that already uses a length (like _WebSocket_)
- The **header**, which consists of the following fields:
    - **Type** and **Flag**, 4 bits each (first 4 bits indicate the [packet type](#packet-types) and the last 4 bits are the packet flags, see _Figure B_. The flags are different per packet type, except flag #5 which is reserved for the MAC. See [request flags](#request-flags).)
    - The dynamic **header fields** which depend on the packet type, see [packet types](#packet-types)
- The **body**, which is a variable length field that differs per [type](#packet-types).
- Optionally a 16-byte **MAC**, see [Authentication](#authentication).

---

## **Response packet**
![Transport Packet](/img/transport-response-bytemap.drawio.png)

_Figure C: Plabble Response Packet_
- *: The length is [dynamically sized](#dynamically-sized-length) and not always included
- **: The last bit of the status is used to toggle the MAC. The first 7 bits are the [status code](#response-codes)
- ***: The MAC is optional, it's toggled with the last bit on the status byte.

![Status bitmap](/img/transport-status-bitmap.drawio.png)

_Figure D: Status code bitmap. Last bit is used to indicate if MAC is present (used as flag), first 7 bits are used for the 7-bit status code_

---

In _Figure C_ the byte map of the Plabble Response Packet is shown. The packets consists of the following parts:
- The **Length**, a [dynamically sized](#dynamically-sized-length) 7 - 28 bits unsigned number that indicates the size of the packet in bytes. (this does not include the length of the _Length_ field itself, just the size of the following data). The length is excluded if a protocol is used that already uses a length (like _WebSocket_)
- The **header**, which consists of the following fields:
    - The 1-byte **status** field, which consists of a 7-bit [status code](#response-codes) and a 1-bit flag that toggles the MAC (see _Figure D_).
    - The unsigned 2-byte (16 bits) **message counter** is used to keep track on the sended and received messages and it is used to randomize the keys used for the encryption of the packet or MAC. The client and the server both keep two [counters](#counters) (client and server counter). The counter field in the response is the count of the packet it is a response to, so you can keep track on the messages you've sent to map requests and responses.
    - The dynamic **header fields** which depend on the packet type, see [packet types](#packet-types)
- The **body**, which is a variable length field that differs per [type](#packet-types).
- Optionally a 16-byte **MAC**, see [Authentication](#authentication).

---

## Packet types
There are several types of packets specified in the protocol, with also a bunch reserved for future use. The packets are used to create, read, update and delete [storage buckets](#bucket-storage) on a server.
**All packets are wrapped in a [Request packet](#request-packet) or a [Response packet](#response-packet)**.

| Name | Value | Description |
|---|---|--|
| **[CONNECT](./connect.md)** | 0 | Start session on a  server
| **[CREATE](./create.md)** | 1 | Create a new bucket
| **[PUT](./put.md)** | 2 | Put a value to a bucket
| **[APPEND](./append.md)** | 3 | Append a value to the first free slot in the bucket
| **[WIPE](./wipe.md)** | 4 | Wipe one or more values from the bucket, or delete entire bucket
| **[REQUEST](./request.md)** | 5 | Read one or more values from the bucket
| **[SUBSCRIBE](./subscribe.md)** | 6 | Subscribe to bucket updates
| **[UNSUBSCRIBE](./unsubscribe.md)** | 7 | Unsubscribe from bucket updates
| _reserved_ | 8 - 15 | Reserved types for future use

### Request flags
In a Plabble [Request Packet](#request-packet) there are 4 bits in the type/flags field that are used to set some properties on a packet. **The first bit (flag #5) is global** and is the same for every packet type: when this flag is set, a MAC is included in the request. The other flags differ per type:

|[Packet type](#packet-types)|Flag #6|Flag #7|Flag #8|
| --------- | ----- | ----- | ----- |
| **[CONNECT](./connect.md)**   | Upgrade to encrypted connection | Send certificate in response | _reserved_
| **[CREATE](./create.md)**    | Subscribe to the created bucket with an optional range | Do not persist bucket (Create RAM bucket) | _reserved_
| **[PUT](./put.md)**       | _reserved_ | _reserved_ | _reserved_ |
| **[APPEND](./append.md)** | _reserved_ | _reserved_ | _reserved |
| **[WIPE](./wipe.md)**      | Delete entire bucket | _reserved_ | _reserved_
| **[REQUEST](./request.md)**   | Also subscribe to the bucket or to the selected range | _reserved_ | _reserved_ |
| **[SUBSCRIBE](./subscribe.md)** | _reserved_ | _reserved_ | _reserved_ |
| **[UNSUBSCRIBE](./unsubscribe.md)** | _reserved_ | _reserved_ | _reserved_ |

### Response codes
In a Plabble [Response Packet](#response-packet) a status code is included. Some of these status codes are global, others are specific for a packet type. The status code is a 7-bit number and the following codes are in use:

|Code| Meaning     | Type   |
| - | ----------- | ------ |
| 0 | Success | _Global_ |
| 1 | Internal Server Error | _Global_ |
| 2 | Unsupported protocol version | _Global_ |
| 3 | Invalid permissions (permission denied) | _Global_ |
| 4 | Authentication failed | _Global_ |
| 5 | Payload too large | _Global_ |
| 6 | Bad request		| _Global_ |
| 7 - 9 | _-- reserved --_ | _Global_ |
| 10 | Subscription success | _Global_ |
| 11 | Success, partial response (means more data on same request can follow) | _Global_ |
| 12 | Bucket update (partial response content) | _Global_ |
| 13 - 20 | _-- reserved --_ | _Global_ |
| 21 | Bucket does not exist | _Global_ |
| 22 - 29 | _-- reserved --_ | _Global_ |
| |
| 30 | Upgraded to encrypted connection | connect |
| 31 | No certificate available | connect |
| 32 - 39 | _-- reserved for CONNECT --_ | connect |
| |
| 40 | _-- reserved for CREATE --_ | create |
| 41 | Bucket already exists | create |
| 42 - 49 | _-- reserved for CREATE --_ | create |
| |
| 50 | _-- reserved for PUT --_ | put |
| 51 | Selected index in bucket already taken (only if no write permissions and try to append with index) | put |
| 52 - 59 | _-- reserved for PUT --_ | put |
| |
| 60 | _-- reserved for APPEND --_ | append |
| 61 | Bucket full | append |
| 62 - 69 | _-- reserved for APPEND --_ | append |
| |
| 70 | _-- reserved for WIPE --_ | 
| 71 | It is not allowed to delete this bucket |
| 72 - 79 | _-- reserved for WIPE --_ | wipe |
| |
| 80 | Subscription success and partial (server might send 10 or 11 instead of this one) | request |
| 81 - 89 | _-- reserved for REQUEST--_ | request |
| |
| 90 - 99 | _-- reserved for SUBSCRIBE --_ | subscribe |
| |
| 100 - 109 | _-- reserved for UNSUBSCRIBE --_ | unsubscribe |
| |
| 110 - 119 | _-- reserved for other types --_ | other |
| 120 - 127 | _-- reserved --_ | _Global_ |

## Packet encryption and authentication
Encryption of packets in PTP is optional, because Plabble also allows the usage of TLS. However, Plabble still wants to maintain authenticity on protocol-level because the user has to prove it has access to a bucket. To solve this, Plabble has basically two modes:
1. **Not encrypted**: The packets include a [MAC](#authentication) to ensure authenticity
2. **Encrypted**: Plabble uses an Authenticated Encryption with Associated Data (AEAD) algorithm to ensure both confidentiality and authenticity.

### Authentication

![Authentication data byte map](/img/plabble-transport-ad.drawio.png)

_Figure E: Authentication data bytemap_

The data hashed in the **MAC** (Message Authentication Code) or **AD** (Associated Data) contains the following fields (see _Figure E_):

- **Header values**: The packet header: type/flag and header fields for a request and the status, counter and header fields for the response. Size can differ
- Optionally a **body hash**, a `SHA-256` hash of the body (32 bytes). This body hash is only included when **no encryption is used** to ensure integrity on the body data
- Optionally a **bucket key** to prove that you are allowed to perform specific operations on a bucket.

If a packet _is encrypted_, the **header** is encrypted with the `chacha20` algorithm while the body is encrypted with `chacha20-poly1305` AEAD algorithm like shown in [_Figure A_](#request-packet) and [_Figure C_](#response-packet). The authentication data poly hash is thus passed to the body while the header does not contain authentication data. Because the header data is included in the body authentication hash it can still be verified safely. No MAC is included.

If _no encryption_ is used, the a 16-byte Message Authentication Code (MAC) is appended to the packet. For the generation of the MAC `poly1305` is used to generate a 16-byte authentication hash. When a MAC is included a flag must be set, for the request packet flag #5 in the _type/flag_ field will be set and for a response the first bit of the _status_ field.

When sending a [CONNECT](./connect.md) packet, encryption is not used and no MAC is included because the keys are not exchanged yet.

### MAC and Encryption keys
The keys used to encrypt the packet are generated using a Key Derivation Function (KDF). We use `HKDF-SHA256` to derive keys from a shared secret generated in the [CONNECT](./connect.md) step, referred to as the _session key_. For generating encryption or authorization keys we use this _session key_. We generate 2 keys every request:
- Key 0: HKDF( _key_: session key, _info_: [counter](#counters) (2 bytes, [LE](#endianness)) + `0x00` )
- Key 1: HKDF( _key_: session key, _info_: [counter](#counters) (2 bytes, [LE](#endianness)) + `0x01` )

These two keys are generated because it is not secure to reuse keys with the same nonce when using `chacha20` (we use an empty byte array of 12 bytes as nonce at this point). When encrypting the packet we use _Key 0_ to encrypt the header and _Key 1_ to encrypt the body. If no encryption is used, _Key 0_ is used to generate the MAC. Because the counter differs every time, different encryption keys are used for every packet.

### Counters
The client and the server both keep two counters which are stored in a 2-byte unsigned integer (uint-16) each, a **client counter** and a **server counter**. These counters are used as a _nonce_ in the Key Derivation Function (KDF) to generate a shared secret that differs every time and to keep track on requests/responses. To encrypt the data or create a MAC the counters are used to [generate a key](#mac-and-encryption-keys). A client or server always uses his own counter when sending a message to the other party. Because they both keep the two counters, the counter of the other party is also known. After each message sent to the other party own counter is incremented. After each message that is received from the other party, the other counter is incremented, so a counter increments after the processing of a message.

#### Counter overflow
If a counter reaches the maximum value _65535_ (because it is an u16), the connection must be closed because we don't want the risk of reusing encryption keys. The client needs to reconnect to the server.

## Packet data

### Dynamically sized length
![Dynlen integer bitmap](/img/transport-dynlen-bitmap.drawio.png)

_Figure F: Dynamic length bitmap_

Plabble uses dynamically sized length unsigned integers (see _Figure F_). Plabble uses this system to avoid unnecessary bytes to be send, what makes Plabble faster and more efficient in its transport system. The length can have 1 to 4 bytes, and every last bit in every byte is used to indicate if the next byte is present. So if you have the first byte and the last bit equals 1, than a second byte is present. If on the second byte the last bit is set, a third byte is present etc. So the range of this integer is between 0 and 268435455.

Below is a code snippet that indicates how the Plabble dynamic length integer could be implemented (Go):
```go
func encode(num int) (result []byte) {
	result = make([]byte, 0, 4)
	for num > 0 { // while loop
        // in dynamic typed languages, remember to Floor the divisions!
		encodedByte := byte(num % 128)
		num /= 128
		if num > 0 {
			encodedByte |= 128
		}
		result = append(result, encodedByte)
	}
	return //result
}

func decode(arr []byte) (num int) {
	multiplier := 1
	for _, encodedByte := range arr {
		num += (int(encodedByte) & 127) * multiplier
		multiplier *= 128
	}
	return //num
}
```

### Endianness
Everywhere throughout the protocol **little endian** encoding is used for consistency when encoding numbers. In the past, big endian was preferred because most devices at the time were big endian. Nowadays, it doesn't really matter so Plabble decided to use little endian because most devices nowadays are little endian.

---
> &larr; Back to [Home](../index.md) - Next: [CONNECT packet](./connect.md) &rarr;
