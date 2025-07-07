This is a specification that defines all values for **NCP**. While the examples are written in Python, you may implement peer or nodeserver logic in any language. The Python code snippets serve purely as implementation guidelines.

default ports:
- client: 63924
- server: 63925

peer definition spec: {ip} {port} {node type}
(node type denoted by server / client or s / c)
version notation: 1.0 and not v1
# communication
the seperator is a single backslash anything else used and other nodes wont be able to decode your message
## peer nodeserver communication
- header: pp2pn\\{node type}\\{version}
- get peers: get\\peers
- add peers: add\\peers\\{data}
- close conn: close
## p2p communication

- broadcast: cast\\{id}\\{ip}:{port}\\{data}
    - `{ip}:{port}` is the sender's address so receiving peers can add them to their peer list.
- dead peer announcement: dead{ip}:{port}
	    - This informs peers that the given peer failed to respond or connect. Peers receiving this can choose to remove the specified peer from their lists.
# data transmission
If you get a message that is the same as a message you have already received, discard it.  
`id` = a unique identifier appended to a broadcasted message to make messages unique and avoid duplicates being discarded.
## generating a message id
using the peer uuid generate a message id:
```python
counter = 0 # on startup

def generate_msg_id():
    global counter
    counter += 1
    return f"{peer_id}-{counter}"
```
it is also allowed (but not preferred) to generate a uuid per message:
```python
import uuid

def generate_msg_id():
    return uuid.uuid4().hex
```
or using the timestamp with random numbers (not recommended):
```python
import time, random

def generate_msg_id():
    return f"{int(time.time()*1000)}-{random.randint(0, 99999)}"
```
example cast with id: ``cast\9f8c3a5b7d1e4f6a8b9c0d12ef34ab56-42\Hello peers!``
if you want to make any of these more secure add the timestamp like in the third example:
```python
def generate_msg_id():
    global counter
    counter += 1
    timestamp = int(time.time() * 1000)
    return f"{peer_id}-{timestamp}-{counter}"
```

## peer list encoding / decoding
### peer list encoding
convert the IP of the peer to 4 bytes using the socket package:
```python
ip_bytes = socket.inet_aton(ip_str)
```
pack the port (unsigned short, big endian):
```python
port_bytes = struct.pack('!H', port)
```
encode the server / client char (one ASCII byte):
```python
sc_byte = sc_char.encode('ascii')
```
### peer list decoding
extract 4 IP bytes and convert to a string using the socket package:
```python
ip_bytes = chunk[:4]
ip_str = socket.inet_ntoa(ip_bytes)
```
extract 2 port bytes and unpack:
```python
port_bytes = chunk[4:6]
(port,) = struct.unpack('!H', port_bytes)
```
extract 1 char byte and decode:
```python
sc_char = chunk[6:7].decode('ascii')
```
### full encoding / decoding functions
```python
import struct
import socket

def encode_peers(tuples_list):
    encoded = bytearray()
    for ip_str, port, sc_char in tuples_list:
        ip_bytes = socket.inet_aton(ip_str)

        port_bytes = struct.pack('!H', port)

        sc_byte = sc_char.encode('ascii')

        encoded.extend(ip_bytes)
        encoded.extend(port_bytes)
        encoded.extend(sc_byte)

    return bytes(encoded)

def decode_peers(encoded_bytes):
    tuples_list = []
    tuple_size = 7

	if len(encoded_bytes) % tuple_size != 0:
		raise ValueError("Invalid encoded peers length")

    for i in range(0, len(encoded_bytes), tuple_size):
        chunk = encoded_bytes[i:i+tuple_size]

        ip_bytes = chunk[:4]
        ip_str = socket.inet_ntoa(ip_bytes)

        port_bytes = chunk[4:6]
        (port,) = struct.unpack('!H', port_bytes)

        sc_char = chunk[6:7].decode('ascii')

        tuples_list.append((ip_str, port, sc_char))

    return tuples_list
```
# peers
## uuid
every peer generates a unique uuid at startup:
```python
import uuid

peer_id = uuid.uuid4().hex
```
this is not persistent to make duplicates harder

# Network Architecture and Node Servers

### Node Servers
Node servers are specialized peers that maintain and share lists of known peers to facilitate peer discovery across the network. These servers:
- Hold large peer lists containing hundreds or more peers, including both clients and other servers.
- Provide a peer list service to help new or isolated peers find others in the network.
- Enable peers to add their known peers to the server’s list and request the current peer list for discovery purposes.

### Peer Discovery and Messaging

- When a peer connects to a node server, it can:
    - Add peers to the server’s peer list.
    - Request the server’s peer list to find other peers to connect with.
- Message broadcasting is **fully decentralized**:
    - Peers send broadcast messages **directly to all peers they know**, not through node servers.
    - Each peer forwards received messages to its known peers (excluding duplicates), allowing messages to propagate organically throughout the network.

This hybrid approach balances:

- **Centralized peer discovery** via node servers for easy bootstrapping and peer list sharing.
- **Decentralized message propagation**, preventing bottlenecks and single points of failure while maximizing scalability.
# error and state handling

- If a peer fails to connect to another peer, it may broadcast a message indicating that the peer is unreachable, allowing others to remove it from their lists.
- This approach assumes nodes re-announce themselves via cast messages including their IP and port, so temporary removal is not catastrophic.
- There is no built-in authentication. Any peer can join the network.
- Version negotiation is not currently implemented since only version 1.0 exists.
- Message size should ideally be capped around 1KB to reduce fragmentation or abuse potential.
- Peer list expiration is optional and should only apply to peers (not nodeservers).
## standard error messages

- `error\10\invalid message format`
- `error\11\unsupported version`
- `error\12\unknown request type`
- `error\13\missing data field`
- `error\20\peer not found`
- `error\21\internal node error`
- `error\30\timeout while connecting to peer`
- `error\31\connection refused`
- `error\32\network unreachable`
- `error\40\too many requests`
- `error\41\peer list capacity full`
- `error\42\message size exceeds limit`
- `error\50\unexpected header format`
- `error\51\reserved keyword used`
- `error\52\malformed broadcast id`
- `error\60\deprecated endpoint`
- `error\61\version mismatch with node`

Each node can define additional error codes as needed, but they should follow the same `error\{code}\{description}` pattern where the first number defines type:

| Type | Meaning                                 |
| ---- | --------------------------------------- |
| 1xx  | Receiver errors (bad request)           |
| 2xx  | Sender errors (missing resources, etc.) |
| 3xx  | Network/connectivity failures           |
| 4xx  | Rate/resource limits                    |
| 5xx  | Protocol/security enforcement           |
| 6xx  | Deprecation/version handling            |
Some error codes may appear similar but are contextually distinct.  
For example:

- `error\11\unsupported version` is used when a higher-version node encounters a lower-version peer that does not support necessary features.
- `error\61\version mismatch with node` is used when a lower-version node attempts to communicate with a higher-version node that requires features it does not support.
