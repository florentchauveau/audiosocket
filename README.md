# AudioSocket

[![](https://godoc.org/github.com/florentchauveau/audiosocket?status.svg)](http://godoc.org/github.com/florentchauveau/audiosocket)

AudioSocket is a simple TCP-based protocol for sending and receiving realtime
audio streams.

There exists a protocol definition (below), a Go library, and Asterisk
application and channel interfaces.

**NOTE:** as of 2020-01-15, AudioSocket has been included in the upstream Asterisk
system.  

**NOTE** this is a fork of https://github.com/CyCoreSystems/audiosocket, with:
- support for DTMF
- support for Asterisk 22.X
- updated code for Go 1.23
- updated code using native Go error wrapping
- fixed example

**NOTE:** I have added [support for DTMF](https://github.com/florentchauveau/asterisk/commit/be399d1cef22be8398bb994b6be69319f821889c) 
in the AudioSocket protocol. Here is the [patch](https://github.com/florentchauveau/asterisk/commit/be399d1cef22be8398bb994b6be69319f821889c.patch) 
for Asterisk 22.X.

## Protocol definition

The singular design goal of AudioSocket is to present the simplest possible
audio streaming protocol, initially based on the constraints of Asterisk audio.
Each packet contains a three-byte header and a variable payload.  The header is
composed of a one-byte type and a two-byte length indicator.

The minimum message length is three bytes:  type and payload-length.  Hangup
indication, for instance, is `0x00 0x00 0x00`.

### Types

  - `0x00` - Terminate the connection (socket closure is also sufficient)
  - `0x01` - Payload will contain the UUID (16-byte binary representation) for the audio stream
  - `0x03` - Payload is 1 byte (ascii) DTMF (dual-tone multi-frequency) digit
  - `0x10` - Payload is signed linear, 16-bit, 8kHz, mono PCM (little-endian)
  - `0xff` - An error has occurred; payload is the (optional)
    application-specific error code.  Asterisk-generated error codes are listed
    below.

### Payload length

The payload length is a 16-bit unsigned integer (big endian) indicating how many bytes are
in the payload.

### Payload

The content of the payload is defined by the header: type and length.

### Asterisk error codes

Error codes are application-specific.  The error codes for Asterisk are
single-byte, bit-packed error codes:

  - `0x01` - hangup of calling party
  - `0x02` - frame forwarding error
  - `0x04` - memory (allocation) error

## Asterisk usage

There are two Asterisk implementations: a channel interface and a dialplan
application interface.  Each of these lends itself to simplify a different
use-case, but they work in exactly the same way.

The following examples demonstrate an AudioSocket connection to a server at
`server.example.com` running on TCP port 9092.  The UUID (which is chosen
arbitrarily) of the call is `40325ec2-5efd-4bd3-805f-53576e581d13`.

Dialplan application:

```
exten = 100,1,Verbose("Call to AudioSocket via Dialplan Application")
 same = n,Answer()
 same = n,AudioSocket(40325ec2-5efd-4bd3-805f-53576e581d13,server.example.com:9092)
 same = n,Hangup()
```

Channel interface:

```
exten = 101,1,Verbose("Call to AudioSocket via Channel interface")
 same = n,Answer()
 same = n,Dial(AudioSocket/server.example.com:9092/40325ec2-5efd-4bd3-805f-53576e581d13/c(slin))
 same = n,Hangup()
```
