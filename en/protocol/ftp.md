# File Transfer Protocol (FTP)

The FTP Protocol enables an FTP-like file transfer protocol over MAVLink.
It supports common FTP operations including: reading, truncating, writing, removing and creating files, listing and removing directories etc.

The protocol follows a client-server pattern, where all commands are sent by the GCS (server), 
and the Drone (client) responds either with the requested information in an ACK or an error in a NAK. 
The GCS sets a timeout after most commands, and may resend the command if it is triggered.
(similarly, the drone must re-send its response if a request with the same sequence number is received).
This ensures that the GCS and drone can never get out of sync in normal operation.

All messages (commands, ACK, NAK) are exchanged inside [FILE_TRANSFER_PROTOCOL](../messages/common.md#FILE_TRANSFER_PROTOCOL) packets.
This message type definition is minimal, with fields for specifying the target network, system and component, and for an "arbitrary" variable-length payload. 

The different commands and other information required to implement the protocol are encoded *within* in the `FILE_TRANSFER_PROTOCOL` payload.
This topic explains the encoding, packing format, commands and errors, and the order in which the commands are sent to implement the core FTP functionality. 

> **Note** The encoding and content of the payload field are not mandated by the specification - and can be extension specific.
  The encoding explained here has been used by *QGroundControl* and PX4.

A MAVLink system that supports this protocol should also set the [MAV_PROTOCOL_CAPABILITY_FTP](../messages/common.md#MAV_PROTOCOL_CAPABILITY_FTP) flag in indicate support in the [AUTOPILOT_VERSION.capability](../messages/common.html#AUTOPILOT_VERSION) field.



## Payload Format {#payload}

The `FILE_TRANSFER_PROTOCOL` payload is encoded with the information required for the various FTP messages. 
This includes fields for specifying which message is being sent, the sequence number of the current FTP message (for multi-message data transfers), 
the size of information in the data part of the message etc..

> **Tip** Readers will note that the FTP payload format is very similar to the packet format used for serializing MAVLink itself.

Below is the over-the-wire format for the payload part of the [FILE_TRANSFER_PROTOCOL](../messages/common.md#FILE_TRANSFER_PROTOCOL) message on PX4/*QGroundControl*.

![FILE_TRANSFER_PROTOCOL Payload format - QGC](../../assets/packets/ftp_transfer_payload_data_qgc.jpg)

Byte Index | C version | Content | Value | Explanation
--- | --- | --- | --- | ---
0 to 1 | `uint16_t seq_number` | Sequence number for message | 0&nbsp;-&nbsp;65535 | All *new* messages between the GCS and drone iterate this number. Re-sent commands/ACK/NAK should use the previous response's sequence number.
2 | `uint8_t session`   | Session id | 0 - 255 | Session id for read/write operations (the server may use this to reference the file handle and information about the progress of read/write operations).
3 | `uint8_t opcode`    | [OpCode](#opcodes) (id) | 0 - 255 | Ids for particular commands and ACK/NAK messages.
4 | `uint8_t size`      | Size         | 1 - 255 | Depends on [OpCode](#opcodes). For Reads/Writes this is the size of the `data` transported. For an ACK to `OpenFileRO` it is the size of the file that has been opened (and must be read). For NAK it is the number of bytes used for [error information](#error_codes) (1 or 2).
5 | `uint8_t req_opcode`| Request [OpCode](#opcodes) | 0 - 255 | OpCode (of original message) returned in an ACK or NAK response. 
6 | `uint8_t burst_complete` | Burst complete | 0, 1 | Code to indicate if a burst is complete. 1: set of burst packets complete, 0: More burst packets coming.<br>- Only used if `req_opcode` is [BurstReadFile](#BurstReadFile).
7 | `uint8_t padding` | Padding | | 32 bit alignment padding.
8 to 11 | `uint32_t offset` | Content offset | | Offsets into data to be sent for [ListDirectory](#ListDirectory) and [ReadFile](#ReadFile) commands.
12 to (max) 251| `uint8_t data[]` | Data | | Command/response data. Varies by [OpCode](#opcodes). For Reads/Writes this is the buffer transported. For a NAK the first byte is the [error code](#error_codes) and the (optional) second byte may be an error number.


## OpCodes/Command {#opcodes}

The opcodes defined/implemented in the server are:

<!--  uint8_t enum Opcode: https://github.com/PX4/Firmware/blob/master/src/modules/mavlink/mavlink_ftp.h -->
     
Opcode | Name | Description
--- | --- | ---
0   | None | Ignored, always ACKed
1   | TerminateSession | Terminates open Read `session`.<br>- Closes the file associated with (`session`) and frees the session ID for re-use. 
<span id="ResetSessions"></span>2   | ResetSessions | Terminates *all* open read sessions.<br>- Clears all state held by the server; closes all open files, etc.<br>- Sends an ACK reply with no data. <!-- Note, is same as Terminate, but does not check if file session exists -->
<span id="ListDirectory"></span>3   | ListDirectory | List files and directories in `<path>` from `<offset>`.<br>- Opens the directory (`path`), seeks to (`offset`) and fills the result buffer with `NULL`-separated filenames (files also include tab-separated file size) and directory names. Sends an ACK packet with the result buffer on success, otherwise a NAK packet with an error code.<br>- The directory is closed after the operation, so this leaves no state on the server.
<span id="OpenFileRO"></span>4   | OpenFileRO | Opens file at `<path>` for reading, returns `<session>`<br>- Opens the file (`path`) and allocates a *session number*. The file must exist.<br>- Sends an ACK packet with the allocated *session number* on success and the data size of the file to be opened (`size`), otherwise a NAK packet with an error code. Typical error codes for this command are `NoSessionsAvailable`, `FileExists`. <br>- The file remains open after the operation, and must eventually be closed by `Reset` or `Terminate`.
<span id="ReadFile"></span>5   | ReadFile | Reads `<size>` bytes from `<offset>` in `<session>`.<br>- Seeks to (`offset`) in the file opened in (session) and reads (`size`) bytes into the result buffer.<br>- Sends an ACK packet with the result buffer on success, otherwise a NAK packet with an error code. For short reads or reads beyond the end of a file, the (`size`) field in the ACK packet will indicate the actual number of bytes read.<br>- Reads can be issued to any offset in the file for any number of bytes, so reconstructing portions of the file to deal with lost packets should be easy.<br>- For best download performance, try to keep two `Read` packets in flight.
6   | CreateFile | Creates file at `<path>` for writing, returns `<session>`.<br>- Creates the file (path) and allocates a *session number*. The file must not exist, but all parent directories must exist.<br>- Sends an ACK packet with the allocated session number on success, or a NAK packet with an error code on error (i.e. [FileExists](#FileExists) if the `path` already exists).<br>- The file remains open after the operation, and must eventually be closed by `Reset` or `Terminate`.
7   | WriteFile | Writes `<size>` bytes to `<offset>` in `<session>`.<br>- Sends an ACK reply with no data on success, otherwise a NAK packet with an error code.
8   | RemoveFile | Remove file at `<path>`.<br>- Sends an ACK reply with no data on success, otherwise a NAK packet with an error code.
9   | CreateDirectory | Creates directory at `<path>`.<br>- Sends an ACK reply with no data on success, otherwise a NAK packet with an error code.
10  | RemoveDirectory | Removes directory at `<path>`. The directory must be empty. <br>- Sends an ACK reply with no data on success, otherwise a NAK packet with an error code.
11  | OpenFileWO | Opens file at `<path>` for writing, returns `<session>`. <br>- Opens the file (`path`) and allocates a *session number*. The file must exist.<br>- Sends an ACK packet with the allocated *session number* on success, otherwise a NAK packet with an error code.<br>- The file remains open after the operation, and must eventually be closed by `Reset` or `Terminate`.
12  | TruncateFile | Truncate file at `<path>` to `<offset>` length.<br>- Sends an ACK reply with no data on success, otherwise a NAK packet with an error code.
13  | Rename | Rename `<path1>` to `<path2>`.<br>- Sends an ACK reply the no data on success, otherwise a NAK packet with an error code (i.e. if the source path does not exist).
14  | CalcFileCRC32 | Calculate CRC32 for file at `<path>`.<br>- Sends an ACK reply with the checksum on success, otherwise a NAK packet with an error code.
<span id="BurstReadFile"></span>15  | BurstReadFile | Burst download session file.
128 | ACK | ACK response.
129 | NAK | NAK response.


Notes:
* All methods respond with an ACK response on success or a NAK in the event of an error.
* The ACK response may additionally return requested data in the payload (e.g. `OpenFileRO` returns the session and file size, `ReadFile` returns the requested file data, etc.). 
* The NAK response includes [error information](#error_code). 


## NAK Error Information {#error_codes}

NAK responses contain one of the errors codes listed below in the [payload](#payload) `data[0]` field.
If the error code is `FailErrno`, then `data[1]` will additionally contain file-system specific error number (understood by the server). 
The payload `size` field must be updated with either 1 or 2, depending on whether or not `FailErrno` is specified. 

> **Note** These are **errors**. Normally if the GCS receives an error it should not attempt to continue the FTP operation, but instead return to an idle state.

<!--  uint8_t enum ErrorCode: https://github.com/PX4/Firmware/blob/master/src/modules/mavlink/mavlink_ftp.h -->

Error | Name | Description
--- | --- | ---
<span id="None"></span>1                | None            | No error
<span id="Fail"></span>2                | Fail            | Unknown failure
<span id="FailErrno"></span>3           | FailErrno       | Command failed, Err number sent back in `PayloadHeader.data[1]`. This is a file-system error number understood by the server operating system.
<span id="InvalidDataSize"></span>4     | InvalidDataSize | Payload `size` is invalid
<span id="InvalidSession"></span>5      | InvalidSessionn | Session is not currently open
<span id="NoSessionsAvailable"></span>6 | NoSessionsAvailable | All available sessions are already in use.
<span id="EOF"></span>7                 | EOF             | Offset past end of file for `ListDirectory` and `ReadFile` commands.
<span id="UnknownCommand"></span>8      | UnknownCommand  | Unknown command / opcode
<span id="FileExists"></span>9          | FileExists      | File already exists
<span id="FileProtected"></span>10      | FileProtected   | File is write protected






## Timeouts/Resending {#timeouts}

The GCS (server) starts a timeout after most commands are sent (these are cleared if an ACK/NAK is received).

> **Note** Timeouts may not be set for some messages. For example, a timeout need not set for [ResetSessions](#ResetSessions) as the message should always succeed. 

If a timeout activates either the command or its response is assumed to have been lost, 
and the command should be re-sent with the same sequence number etc.
A number of retries are allowed, after which the GCS should fail the whole download and reset to an idle state.

If the drone (client) receives a message with the same sequence number then it assumes that its ACK/NAK response was lost.
In this case it should resend the response (the sequence number is not iterated, because it is as though the previous response was not sent).

The drone has no timeout mechanism; it only ever responds to commands and does not expect any responses.

GCS recommended settings:
- ACK/NAK timeout: 50 milliseconds
- Command retries: 6


## FTP Operations


### Reading a File

The sequence of operations for downloading (reading) a file, assuming there are no timeouts and all operations/requests succeed, is shown below.

{% mermaid %}
sequenceDiagram;
    participant GCS
    participant Drone
    Note right of GCS: Open file
    GCS->>Drone:  OpenFileRO(path)
    Drone-->>GCS: ACK(session, size=filesize)
    Note right of GCS: Read file in chunks<br>(repeat until done)
    GCS->>Drone:  ReadFile(session, size, offset)
    Drone-->>GCS: ACK(session, size=data_size, data=buffer)
    Note right of GCS: Close session
    GCS->>Drone:  TerminateSession(session)
    Drone-->>GCS: ACK()
{% endmermaid %}

The sequence of operations is:
1. GCS (server) sends [OpenFileRO](#OpenFileRO) command specifying the file path to open.
1. Drone (client) responds with either 
   - ACK with the [payload](#payload) `session` (representing the file) and `size` field (containing the size of the file that has been opened). 
     The `size` field is used by the GCS to determine when it has fully downloaded the file.
   - NAK with [error information](#error_codes), e.g. `NoSessionsAvailable`, `FileExists`. 
     The server may cancel the operation, depending on the error.
1. GCS sends [ReadFile](#ReadFile) commands to download a chunk of data from the file. 
   This operation can be repeated at different offsets to download the whole file.
1. Drone responds to each message with either 
   - ACK with the `data` in the [payload](#payload) `data` field (and the size of data returned in the `size` field)
   - NAK with [error information](#error_codes). Generally errors are unrecoverable, but in some case they may indicate that an operation is complete - e.g. EOF error when all the data is downloaded.
1. Once all the data is downloaded the GCS can send [TerminateSession](#TerminateSession) to close the file. Generally speaking the ACK/NAK can be ignored.


The GSC should create a timeout after `OpenFileRO` and `ReadFile` commands are sent and resend the messages as needed (and [described above](#timeouts)).
A timeout is not set for `TerminateSession` (the server may ignore failure of the command or the ACK).



### Writing a File



### List Directory



## C Implementation

The FTP Protocol has been implemented (minimally) in C by PX4 <!-- and ArduPilot Flight Stacks, --> and *QGroundControl*.  
This implementation can be used in your own code within the terms of their software licenses.

PX4 Implementation::
* [src/modules/mavlink/mavlink_ftp.cpp](https://github.com/PX4/Firmware/blob/master/src/modules/mavlink/mavlink_ftp.cpp)
* [src/modules/mavlink/mavlink_ftp.h](https://github.com/PX4/Firmware/blob/master/src/modules/mavlink/mavlink_ftp.h)

*QGroundControl* implementation:
* [src/uas/FileManager.cc](https://github.com/mavlink/qgroundcontrol/blob/master/src/uas/FileManager.cc)
* [/src/uas/FileManager.h](https://github.com/mavlink/qgroundcontrol/blob/master/src/uas/FileManager.h)


Everything is run by the master (QGC in this case); the slave simply responds to packets in order as they arrive. There’s buffering in the server for a little overlap (two packets in the queue at a time). This is a tradeoff between memory and link latency which may need to be reconsidered at some point.

The MAVLink receiver thread copies an incoming request verbatim from the MAVLink buffer into a request queue, and queues a low-priority work item to handle the packet. This avoids trying to do file I/O on the MAVLink receiver thread, as well as avoiding yet another worker thread. The worker is responsible for directly queuing replies, which are sent with the same sequence number as the request.

The implementation on PX4 only supports a single session.

