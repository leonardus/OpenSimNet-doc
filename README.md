# OpenSimNet

This specification is an **initial draft**. It is incomplete and subject to change. **It is not recommended that you refer to this document in its current state to implement OpenSimNet!**

## Preface

OpenSimNet (OSN) is offered as a modern alternative to the [FSD](https://studentweb.uvic.ca/~norrisng/fsd-doc/) protocol used in current flight simulation networks. OpenSimNet is an open protocol: Anyone has the freedom to implement and operate an OpenSimNet-based server or client. Extensive and detailed documentation is offered to encourage these implementations.

This documentation uses keywords ("MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "MAY") as defined in [RFC 2119](https://tools.ietf.org/html/rfc2119).
A portion of the OpenSimNet protocol and documentation is inspired by the Internet Relay Chat (IRC) protocol and [ircdocs](https://github.com/ircdocs/modern-irc).

## Connection Setup

OpenSimNet uses TCP and UDP port 12280. OpenSimNet connections MUST use TLS.

## TCP Messages

Messages are JSON objects terminated with a single newline (`\n`) character. Newlines are included in this documentation for readability. Every message has the key `command` [string], and may have keys `source` [string] and `args` [object]. Example:

```json
{
    "source": "User01",
    "command": "BROADCAST",
    "args": {
        "frequency": 12280,
        "message": "KPBI tfc on final for rwy 10"
    }
}
```

`source` is the username of the client that the message originates from, if applicable. When sending a message from the client to the server, the client MUST NOT include the source. If the message originates from the server itself, the source MUST NOT be included. `command` is the action that the message is describing. `args` is a list of arguments provided for the command. `args` MAY be ommitted if no arguments are accepted for the command (e.g. the `DISCONNECT` message).
Empty messages (including messages with no `command` specified) and undefined arguments SHOULD be silently ignored. Unrecognized commands are handled by an `UNKNOWN_COMMAND` error response. Invalid JSON is handled by the `COULD_NOT_PARSE` error response.

## Registration

Registration begins with the TCP connection, on port 12280. To register with the server, a client sends the `USER` message.

The client SHOULD send a `STATUS`, `FREQUENCY`, and `SQUAWK` message upon completion of registration. The server MUST send a `SERVER` and `LIST` message upon completion of registration, and a `STATUS` message for all other connected clients. The server will then send a `JOIN` message to all clients, including the client connecting, to indicate that a new client has successfully joined. Once registration has completed, the client and server may begin exchanging UDP packets immediately. Until registration has completed, the server MUST silently ignore any UDP packets from the client.

## TCP

The following messages may be sent over the TCP connection.

### USER message

Command: `USER`  
Arguments accepted:

* `username` [string]: The user's username
* `password` [string]: The user's password
* `plane` [string]: The ICAO aircraft type designator of the plane being operated by the user

The server may respond with one of the following errors:

* `LOGIN_FAILED`
* `UNAUTHORIZED`
* `SERVER_FULL`
* `ALREADY_REGISTERED`

If the login is successful, the server will respond with an `ACK` message.  
Whitespace at the beginning and end of usernames MUST be silently truncated.

### ACK message

Command: `ACK`  
Arguments accepted:

* `token` [int]: A unique token assigned by the server that the client will use to uniquely identify itself in UDP packets

The server sends the `ACK` message to a newly-registered client to indicate that the login was successful.

### SERVER message

Command: `SERVER`  
Arguments accepted:

* `callsign_min` [int]: The minimum amount of characters the server will accept in a callsign
* `callsign_max` [int]: The maximum amount of characters the server will accept in a callsign
* `slots` [int]: The maximum amount of clients the server will accept
* `clients` [int]: The amount of clients the server currently has connected
* `motd` [string] *(optional)*: A message to be displayed to all clients upon connecting
* `max_msg_len` [string] *(optional)*: The maximum length for a `PRIVMSG` or `BROADCAST` message

The `SERVER` message is only sent from the server to the client.

### LIST message

Command: `LIST`  
Arguments accepted:

* `range` [float] *(optional)*: A maximum range, in nautical miles, of clients to return
* `clients` [array\<object\>]: A list of clients online - each entry in the array is a JSON object with the keys `username`, `callsign` (may or may not exist), and `plane` - **only sent from the server to the client**

### JOIN message

Command: `JOIN`  
Arguments accepted:

* `user` [string]: The username of the user that has joined the server
* `plane` [string]: The plane being operated by the user

The server sends the `JOIN` message to indicate that a new user has joined the server. This message is relayed to all clients, including the client that joins.

### PART message

Command: `PART`  
Arguments accepted:

* `user` [string]: The username of the user that has left the server

The server sends the `PART` message to indicate that a user has left the server.

### STATUS message

Command: `STATUS`  
Arguments accepted:

* `gear` [bool]: The status of the landing gear - `false` is up, `true` is down
* `landing_ret` [bool]: The status of the retractable landing lights
* `landing_fix` [bool]: The status of the fixed landing lights
* `wiper_l` [int]: The status of the left windshield wiper - `0` is OFF, `1` is INTERMITTENT, `2` is LOW, `3` IS HIGH.
* `wiper_r` [int]: The status of the right windshield wiper
* `turnoff_l` [int]: The status of the left runway turnoff light
* `turnoff_r` [int]: The status of the right runway turnoff light
* `taxi` [bool]: The status of the taxi light
* `logo` [bool]: The status of the logo light
* `position` [int]: The status of the position indicator light - `0` is OFF, `1` is STEADY, `2` is STROBE AND STEADY
* `ac` [bool]: The status of the anti-collision light
* `wing` [bool]: The status of the wing light
* `wwell` [bool]: The status of the weel well light
* `eng1` [bool]: The status of engine 1
* `eng2` [bool]: The status of engine 2
* `eng3` [bool]: The status of engine 3
* `eng4` [bool]: The status of engine 4
* `apu` [bool]: The status of the APU

### FREQUENCY message

Command: `FREQUENCY`  
Arguments accepted:

* `frequency` [int]: The frequency desired by the user (6-digit integer)
* `radio` [int]: A unique identifier for the radio that the frequency is tuned to

### SQUAWK message

Command: `SQUAWK`  
Arguments accepted:

* `code` [int]: The squawk code entered into the plane's transponder
* `mode` [string]: The squawk mode - possible values are `OFF`, `A`, `AC` and `S`

The server may respond with an `INVALID_SQUAWK_MODE` error.

### CALLSIGN message

Command: `CALLSIGN`  
Arguments accepted:

* `callsign` [string]: The user's requested callsign, a string with a minimum and maximum length as defined by the server

The server may respond with one of the following errors:

* `INVALID_CALLSIGN`
* `RESERVED_CALLSIGN`
* `CALLSIGN_MAX`
* `CALLSIGN_MIN`

Until the server receives a CALLSIGN message from the the user, the user's username should be used as a substitute.
The CALLSIGN message may be sent from the server to the client to indicate that another client has changed their callsign.

### FLIGHTPLAN message

Command: `FLIGHTPLAN`  
Arguments accepted:

* `rules` [string]: Flight rules, which may be: SVFR, VFR, or IFR
* `depart` [string]: The departure airport
* `arrive` [string]: The arrival airport
* `alternate` [string] *(optional)*: The alternate airport
* `cruise` [string] *(optional)*: The cruising altitude
* `tas` [int] *(optional)*: The true airspeed
* `route` [string] *(optional)*: The planned route
* `remarks` [string] *(optional)*: The flight plan's remarks

### BROADCAST message

Command: `BROADCAST`  
Arguments accepted:

* `frequency` [int]: The frequency the message will be broadcast on
* `message` [string]: The message to be broadcast on the frequency

The server may respond with a `MSG_TOO_LONG` error.

### PRIVMSG message

Command: `PRIVMSG`  
Arguments accepted:

* `target` [string]: The username of the intended recipient of the message
* `message` [string]: The message to be broadcast on the frequency

The server may respond with a `MSG_TOO_LONG` error.

Clients SHOULD omit arguments that have already been sent or that are not applicable to the plane.

### METAR message

Command: `METAR`  
Arguments accepted:

* `icao` [string]: The ICAO airport code of the requested METAR information
* `metar` [string]: The METAR information of the airport - **only sent from the server to the client**

### DISCONNECT message

Command: `DISCONNECT`  
Arguments accepted: None  
Upon receiving a `DISCONNECT` message from a client, the sever must send a response `DISCONNECT` message, close the TCP connection, and immediately cease sending UDP messages. The server may send a `DISCONNECT` message to the client at any given time, to forcibly remove the client from the server. The server will then close the TCP connection. When receiving a `DISCONNECT` message, the client must immediately cease sending UDP messages. When a client has successfully disconnected from the server, the server sends a `PART` message to all clients on the server.

### ERR message

This message is sent from the server to the client to indicate that the client has provided an invalid message.
Command: `ERR`  
Arguments accepted:

* `type` [string]: A unique identifier for the error
* `message` [string] *(optional)*: A human-readable message detailing the nature of the error

### List of possible error messages

* `LOGIN_FAILED`: The username or password provided by the client was incorrect
* `UNAUTHORIZED`: The user is not authorized to join the server, or is banned from the server
* `SERVER_FULL`: The server is full, and is unable to accept any more connections
* `INVALID_CALLSIGN`: The callsign contains illegal characters
* `RESERVED_CALLSIGN`: The user is not authorized to use the provided callsign
* `CALLSIGN_MAX`: The callsign exceeds the maximum length configured by the server
* `CALLSIGN_MIN`: The callsign is shorter than the minimum length configured by the server
* `MSG_TOO_LONG`: The message sent exceeds the server's maximum message length
* `INVALID_SQUAWK_MODE`: The squawk mode requested is not one of `OFF`, `A`, `AC`, `S`
* `NOT_ENOUGH_ARGS`: The client has omitted required arguments for the command
* `ALREADY_REGISTERED`: The client tried to send the USER message, but it was already sent previously
* `UNKNOWN_COMMAND`: The command sent by the client was not recognized by the server
* `COULD_NOT_PARSE`: The server was unable to parse the JSON sent by the client
