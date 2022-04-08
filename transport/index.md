# Transport packets
The Plabble Transport Protocol (**PTP**) uses a special transport codec for the delivery of the packets between clients and servers. There are two types of transport packets:
[Request Packets](#request-packet) and [Response Packets](#response-packet). The packets are sized dynamically using flags and length attributes to reduce the size of the packets to a minimum.

### **Bucket storage**
A Plabble server stores the data in **buckets**. You can see a bucket as a key-value slot on a server with an authorization system to prevent access to users that do not own the secret key. Every bucket consists of a list with a maximum of 65,535 slots. You can add, edit or remove slots from/to a bucket. Thanks to a **permission system** it is possible to add access restrictions to a bucket, so you can make buckets immutable or unreadable for users without the authorization key. The bucket storage system can optionally be used for multiple purposes on top of multiple protocols because the server acts as a key-value storage system.

---

## **Request packet**
![Transport Packet](/img/transport-request-bytemap.drawio.png)

_Figure A: Plabble Request Packet_
- *: The length is [dynamically sized](#dynamically-sized-length)
- **: The type and flags use 4 bits each from 1 byte
- ***: The [MAC](#authentication) is optional and is toggled with the flag on the 5th bit on the type/flag byte.

![Type-flag bitmap](/img/transport-typeflag-bitmap.png)

_Figure B: Type/Flag bitmap. First 4 bits are the length, the last 4 bits are the flag_
- *: bit-flag #5 is reserved for indicating if the MAC is present.

---

In _Figure A_ the byte map of the Plabble Request Packet is shown. The packets consists of the following parts:
- The **Length**, a [dynamically sized](#dynamically-sized-length) 7 - 28 bits number that indicates the size of the packet in bytes. (this does not include the length of the _Length_ field itself, just the size of the following data)
- The **header**, which consists of the following fields:
    - **Type** and **Flag**, 4 bits each (first 4 bits indicate the [packet type](#packet-types) and the last 4 bits are the packet flags, see _Figure B_. The flags are different per packet type, except flag #5 which is reserved for the MAC)
    - The dynamic **header fields** which depend on the packet type, see [packet types](#packet-types)
- The **body**, which is a variable length field
- Optionally a 16-byte **MAC**, see [Authentication](#authentication).

### Packet types
There are several types of packets specified in the protocol, with also a bunch reserved for future use. The packets are used to create, read, update and delete [storage buckets](#bucket-storage) on a server.

| Name | Value | Description |
|---|---|--|
| **[CONNECT]()** | 0 | Start session on a  server
| **[CREATE]()** | 1 | Create a new bucket
| **[PUT]()** | 2 | Put a value to a bucket
| **[WIPE]()** | 3 | Wipe one or more values from the bucket
| **[REQUEST]()** | 4 | Read one or more values from the bucket
| **[SUBSCRIBE]()** | 5 | Subscribe to bucket updates
| **[UNSUBSCRIBE]()** | 6 | Unsubscribe from bucket updates
| _reserved_ | 7 - 15 | Reserved types for future use

---

## **Response packet**
![Transport Packet](/img/transport-response-bytemap.drawio.png)

_Figure C: Plabble Response Packet_
- *: The length is [dynamically sized](#dynamically-sized-length)
- **: The first bit of the status is used to toggle the MAC. The resulting 7 bits are the status code
- ***: The MAC is optional, it's toggled with the first bit of the status byte.

![Status bitmap](/img/transport-status-bitmap.drawio.png)

_Figure D: Status code bitmap. First bit is used to indicate if MAC is present (used as flag), other 7 bits are used for the 7-bit status code_

---

In _Figure C_ the byte map of the Plabble Response Packet is shown. The packets consists of the following parts:
- The **Length**, a [dynamically sized](#dynamically-sized-length) 7 - 28 bits number that indicates the size of the packet in bytes. (this does not include the length of the _Length_ field itself, just the size of the following data)
- The **header**, which consists of the following fields:
    - The 1-byte **status** field, which consists of a 1-bit flag that toggles the MAC and a 7-bit status code (see _Figure D_).
    - The unsigned 2-byte (16 bits) **message counter** is used to keep track on the sended and received messages and it is used to randomize the keys used for the encryption of the packet or MAC. The client and the server both keep two [counters](#counters) (client and server counter). The counter field in the response is the count of the packet it is a response to, so you can keep track on the messages you've sent to map requests and responses.
    - The dynamic **header fields** which depend on the packet type, see [packet types](#packet-types)
- The **body**, which is a variable length field
- Optionally a 16-byte **MAC**, see [Authentication](#authentication).

---

## Packet encryption and authentication
Encryption of packets in PTP is optional, because Plabble also allows the usage of TLS. However, Plabble still wants to maintain authenticity on protocol-level because the user has to prove it has access to a bucket. To solve this, Plabble has basically two modes:
1. **Not encrypted**: The packets include a [MAC](#authentication) to ensure authenticity
2. **Encrypted**: Plabble uses an Authenticated Encryption with Associated Data (AEAD) algorithm to ensure both confidentiality and authenticity.

### Authentication

![Authentication data byte map](/img/plabble-transport-ad.drawio.png)

_Figure E: Authentication data bytemap_

The data passed in the **MAC** (Message Authentication Code) or **AD** (Associated Data) contains the following fields (see _Figure E_):

- **Header values**: The packet header: type/flag and header fields for a request and the status, counter and header fields for the response. Size can differ
- **Body hash**: A `SHA-256` hash of the body (32 bytes)
- Optionally a **bucket key** to prove that you are allowed to perform specific operations on a bucket.

If a packet is encrypted, the **header** is encrypted with the `chacha20` algorithm while the body is encrypted with `chacha20-poly1305` AEAD algorithm like shown in [_Figure A_](#request-packet) and [_Figure C_](#response-packet). The authentication data poly hash is thus passed to the body while the header does not contain authentication data. Because the header data is included in the body authentication hash it can still be verified safely. No MAC is included.

If no encryption is used, the a 16-byte Message Authentication Code (MAC) is appended to the packet. For the generation of the MAC `poly1305` is used to generate a 16-byte authentication hash. When a MAC is included a flag must be set, for the request packet flag #5 in the _type/flag_ field will be set and for a response the first bit of the _status_ field.

When sending a [CONNECT]() packet, encryption is not used and no MAC is included because the keys are not exchanged yet.

### MAC and Encryption keys
The keys used to encrypt the packet are generated using a Key Derivation Function (KDF). We use `HKDF-SHA256` to derive keys from a shared secret generated in the [CONNECT]() step, referred as the _session id_ and _session key_. For generating encryption or authorization keys we use the _session key_. We generate 2 keys every request:
- Key 0: HKDF( _key_: session key, _info_: [counter](#counters) (2 bytes, [LE](#endianness)) + `0x00` )
- Key 1: HKDF( _key_: session key, _info_: [counter](#counters) (2 bytes, [LE](#endianness)) + `0x01` )

These two keys are generated because it is not secure to reuse keys when using chacha20. When encrypting the packet we use _Key 0_ to encrypt the header and _Key 1_ to encrypt the body. If no encryption is used, _Key 0_ is used to generate the MAC. Because the counter differs every time, different encryption keys are used for every packet.

### Counters
The client and the server both keep two counters which are stored in a 2-byte unsigned integer each, a **client counter** and a **server counter**. These counters are used as a _nonce_ in the Key Derivation Function (KDF) to generate a shared secret that differs every time. To encrypt the data or create a MAC the counters are used to [generate a key](#mac-and-encryption-keys). A client or server always uses his own counter when sending a message to the other party. Because they both keep the two counters, the counter of the other party is also known. After each message sent to the other party own counter is incremented. After each message that is received from the other party, the other counter is incremented.

## Packet data

### Dynamically sized length
![Dynlen integer bitmap](/img/transport-dynlen-bitmap.drawio.png)

_Figure F: Dynamic length bitmap_

Plabble uses dynamically sized length integers (see _Figure F_). Plabble uses this system to avoid unnecessary bytes to be send, what makes Plabble faster and more efficient in its transport system. The length can have 1 to 4 bytes, and every last bit in every byte is used to indicate if the next byte is present. So if you have the first byte and the last bit equals 1, than a second byte is present. If on the second byte the last bit is set, a third byte is present etc. So the range of this integer is between 0 and 268435455.

### Endianness
Everywhere throughout the protocol **little endian** encoding is used for consistency when encoding numbers. In the past, big endian was preferred because most devices at the time were big endian. Nowadays, it doesn't really matter so Plabble decided to use little endian because most devices nowadays are little endian.