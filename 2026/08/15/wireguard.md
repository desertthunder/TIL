---
title: The WireGuard Protocol
description: Peers, cryptokey routing, handshakes, key rotation, and protocol boundaries
tags:
  - networking
  - security
  - cryptography
  - vpn
---

WireGuard is a layer-3 tunnel protocol. It takes an IPv4 or IPv6 packet, encrypts and
authenticates the whole packet, and sends the result to another peer over UDP. To the
operating system, the tunnel appears as an ordinary network interface such as `wg0`.[^1]

Each peer has a Curve25519 static key pair. A configuration associates the local private
key with remote public keys, the tunnel addresses each remote key may use, and optionally
an initial UDP endpoint. The public key is the peer's protocol identity. WireGuard does
not use certificates, usernames, or a negotiation protocol for distributing keys and
configuration.[^1]

## Cryptokey routing

WireGuard binds routing and authentication through each peer's `AllowedIPs`:

- On send, the destination address of the inner IP packet selects a peer. WireGuard
  encrypts the packet for that peer and sends it to the peer's current UDP endpoint.
- On receive, WireGuard authenticates the outer packet as coming from a peer, decrypts
  the inner packet, and accepts it only if the inner source address belongs to that
  peer's `AllowedIPs`.

The same table is therefore a routing table in the outbound direction and an access
control list in the inbound direction. WireGuard calls the association between a public
key and its permitted IP prefixes a **cryptokey routing table**.[^1]

This table belongs to WireGuard, while the operating system's routing table decides which
packets enter the WireGuard interface in the first place. A full-tunnel configuration may
associate `0.0.0.0/0` and `::/0` with one peer, but the host must still route the desired
traffic to that interface.

An endpoint is only a delivery address, not a peer's identity. After WireGuard
successfully authenticates a packet, it records that packet's source IP and UDP port as
the peer's latest endpoint. Future packets go there. This lets a phone move between Wi-Fi
and cellular networks without acquiring a new tunnel identity or running a separate
reconnection protocol.[^1]

## Handshake

WireGuard uses a modified form of the Noise `IK` pattern. `I` means the initiator sends
its static public key during the handshake. `K` means the initiator already knows the
responder's static public key. The static keys authenticate the peers, while fresh
ephemeral keys produce new session keys.[^2]

The normal exchange takes one round trip:

```text
initiator                                  responder
    |                                          |
    |  handshake initiation                    |
    |  ephemeral key, encrypted static key,    |
    |  encrypted timestamp, MACs               |
    | ---------------------------------------> |
    |                                          |
    |                    handshake response    |
    |              ephemeral key, empty AEAD,  |
    |                                  MACs    |
    | <--------------------------------------- |
    |                                          |
    |  first encrypted transport packet        |
    |  (confirms the derived keys)             |
    | ---------------------------------------> |
    |                                          |
```

The initiation authenticates the sender in the first packet. Its encrypted TAI64N
timestamp must be greater than the last timestamp accepted from that peer, preventing an
old initiation from disrupting a live session. The response contributes another
ephemeral key. Both sides then derive separate sending and receiving keys and erase the
ephemeral private keys and intermediate handshake state.[^2]

The responder waits for the initiator's first transport packet before using the new
session to send data. That packet proves that the initiator derived a key incorporating
the responder's ephemeral key. If the initiator has no IP packet ready, it can send an
encrypted empty packet for confirmation.[^2]

WireGuard has four packet types:

| Type | Packet               | Purpose                                           |
| ---- | -------------------- | ------------------------------------------------- |
| 1    | Handshake initiation | Authenticate the initiator and begin rekeying     |
| 2    | Handshake response   | Authenticate the responder and derive keys        |
| 3    | Cookie reply         | Make a sender prove its source address under load |
| 4    | Transport data       | Carry an encrypted inner IP packet or keepalive   |

## Cryptographic construction

WireGuard fixes one cryptographic suite instead of negotiating algorithms:[^2]

| Primitive         | Role                                          |
| ----------------- | --------------------------------------------- |
| Curve25519        | Static and ephemeral Diffie-Hellman exchanges |
| ChaCha20-Poly1305 | Authenticated encryption                      |
| BLAKE2s           | Hashing and keyed hashing                     |
| HKDF              | Deriving handshake and transport keys         |
| SipHash24         | Protecting implementation hash tables         |

Fixing the suite removes downgrade paths and makes implementations easier to compare and
audit, but it also means a new suite requires a new protocol version rather than a
negotiated option.[^3]

An optional 256-bit pre-shared key can be mixed into the handshake. It adds a symmetric
secret on top of Curve25519. It does not replace either peer's static key or provide key
distribution. Its protection depends on securely provisioning and retaining that extra
secret.[^2]

## Transport and key rotation

A transport packet contains a receiver index, a 64-bit counter, and an authenticated
ciphertext. The counter supplies the ChaCha20-Poly1305 nonce. A receiver rejects a reused
counter but retains a sliding window so legitimate UDP packets may arrive out of order.
WireGuard does not add reliability, ordering, or congestion control to the inner traffic.
The encapsulated protocols retain those responsibilities.[^2]

Sessions rotate according to time and message-count limits. An active peer begins a new
handshake before its current keys expire, while old keys remain briefly available for
packets already in flight. Expired session keys and handshake material are erased. This
rotation gives transport data forward secrecy: compromising a static key later does not
recover erased session keys from previously recorded data packets.[^3]

The protocol is otherwise quiet. Idle peers do not exchange periodic negotiation
traffic. An administrator may enable persistent keepalives when a peer behind NAT needs
to preserve a mapping. A keepalive is simply a transport packet with no inner packet.
Ordinary authenticated traffic also refreshes the peer's endpoint.[^3]

## Denial-of-service handling

A responder normally stores no state and sends no reply until the first handshake packet
authenticates as belonging to a configured peer. Random probes therefore receive silence.
An attacker who knows the responder's public key can still force expensive Curve25519
work, so a responder under load may demand a cookie before completing that work.[^2]

The cookie binds a short-lived server secret to the apparent source IP. The initiator
uses it to authenticate a retried handshake, giving the responder evidence that the
sender can receive packets at that address and allowing meaningful rate limiting. Cookie
replies are encrypted so an observer cannot copy the cookie into its own requests.[^2]

## Boundaries and limitations

WireGuard deliberately leaves deployment policy to other software. It does not distribute
public keys, assign tunnel addresses, push routes, configure DNS, authenticate human
users, or decide whether traffic should traverse the tunnel. A VPN application or system
administrator must supply those pieces.[^1]

Its security properties also have limits:

- The payload is confidential and authenticated, but observers can still see UDP packet
  sizes, timing, and endpoints. WireGuard is not a traffic-obfuscation protocol.
- Transport data has forward secrecy. If an attacker records old handshakes and later
  obtains the responder's static private key, the attacker can identify which initiator
  public keys produced those handshakes, though the recorded transport payloads remain
  protected.[^4]
- Curve25519 is not post-quantum secure. A pre-shared key can add protection against a
  future break of Curve25519, but it does not by itself create forward-secure
  post-quantum key exchange.[^4]
- Replay protection for handshake initiations depends on a monotonically increasing
  system clock. Large backward or forward clock jumps can prevent later handshakes.[^4]
- Roaming trusts the source address of a correctly authenticated packet as the latest
  endpoint. An active attacker cannot decrypt or forge its contents, but may be able to
  influence where later encrypted packets are sent.[^4]

WireGuard's small protocol surface comes from this division of work. It owns authenticated
IP encapsulation, peer identity, session-key rotation, and the binding between keys and
tunnel addresses. Everything from enrollment to DNS remains outside the tunnel protocol.

[^1]: WireGuard, [“Conceptual Overview”](https://www.wireguard.com/#conceptual-overview).

[^2]: Jason A. Donenfeld, [“WireGuard: Next Generation Kernel Network Tunnel”](https://www.wireguard.com/papers/wireguard.pdf).

[^3]: WireGuard, [“Protocol & Cryptography”](https://www.wireguard.com/protocol/).

[^4]: WireGuard, [“Known Limitations”](https://www.wireguard.com/known-limitations/).
