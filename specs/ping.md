# Ping/pong protocol

The ping/pong protocol is used to test connections between 2 peers and ensure the Hobbits implementation is passing conformance tests.

When a ping message is received, the hobbits implementer should respond with the body of the ping message as a pong message.

## Ping

Headers: `ping` as UTF-8 bytes

Body: `random 32 bytes`

## Pong

Headers: `pong` as UTF-8 bytes

Body: `32 bytes sent by the ping packet`
