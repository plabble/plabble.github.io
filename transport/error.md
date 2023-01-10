# ERROR
> &larr; Back to [Home](../index.md)

---
The ERROR response packet is used to tell the client that something went wrong while processing its request.

## Response

![Error response bytemap](../img/transport-error-res.drawio.png)

_Figure A: ERROR response byte-map_

The ERROR packet contains of two fields:
- **Status**: a 1-byte number (uint8) between 0 and 255 indicating what kind of error occured. There are some [error codes](#error-codes) reserved by Plabble, but other error codes can be used freely and will never be reserved.
- **Error message**: an UTF-8 encoded string that contains a message to tell exactly what went wrong. This message can differ per Plabble implementation and might also contain some metadata like JSON or anything the server likes. It can also be an empty string.

## Error codes
The following error codes are reserved by Plabble:

|Code| Meaning     | Type   |
| - | ----------- | ------ |
| 0 | _-- reserved --_ | _Global_ |
| 1 | Internal Server Error | _Global_ |
| 2 | Unsupported protocol version | _Global_ |
| 3 | Invalid permissions (permission denied) | _Global_ |
| 4 | Authentication failed | _Global_ |
| 5 | Payload too large | _Global_ |
| 6 | Bad request		| _Global_ |
| 7 - 9 | _-- reserved --_ | _Global_ |
| 10,11 | _-- reserved --_ | _Global_ |
| 12 | Bucket update (partial response content) | _Global_ |
| 13 - 20 | _-- reserved --_ | _Global_ |
| 21 | Bucket does not exist | _Global_ |
| 22 - 29 | _-- reserved --_ | _Global_ |
| |
| 30 | _-- reserved for CONNECT --_ | connect |
| 31 | No certificate available | connect |
| 32 | Could not upgrade to encrypted connection | connect |
| 33 - 39 | _-- reserved for CONNECT --_ | connect |
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
| 70 | _-- reserved for WIPE --_ | wipe |
| 71 | It is not allowed to delete this bucket | wipe |
| 72 - 79 | _-- reserved for WIPE --_ | wipe |
| |
| 80 - 89 | _-- reserved for REQUEST--_ | request |
| |
| 90 - 99 | _-- reserved for SUBSCRIBE --_ | subscribe |
| |
| 100 - 109 | _-- reserved for UNSUBSCRIBE --_ | unsubscribe |
| |
| 110 - 119 | _-- reserved for other types --_ | other |
| 120 - 127 | _-- reserved --_ | _Global_ |
| 128 - 255 | Free to use. Can differ per underlying protocol | _Global_ |

---
> &larr; Back to [Home](../index.md) - To [Transport](./index.md)