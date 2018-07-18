# File Transfer Protocol (FTP)

The FTP Protocol enables an FTP-like file transfer protocol over MAVLink.

All messages are exchanged inside [FILE_TRANSFER_PROTOCOL](../messages/common.md#FILE_TRANSFER_PROTOCOL) packets (i,e commands, success responses (ACK) and error responses (NAK)).
This message type definition is minimal, with fields for specifying the target network, system and component. The only other field is the variable length payload. 
The encoding and content of the payload field is not mandated by the specification - and can be extension specific.


A MAVLink system that supports this protocol should also indicate support in the [AUTOPILOT_VERSION > capability](../messages/common.html#AUTOPILOT_VERSION) field, 
by setting the [MAV_PROTOCOL_CAPABILITY_FTP](../messages/common.md#MAV_PROTOCOL_CAPABILITY_FTP) flag. 

This document explains the payload encoding implemented by PX4 and *QGroundControl*.


## Payload Message Format

The payload encodes a "package header" that defines the information required for the various FTP messages. 
This includes fields for specifying which message is being sent, the FTP sequence number of the current FTP message (for multi-message data transfers), 
the size of information in the data part of the message, and for response messages (ACK/NAK) the original opcode that the message is a response to.

> **Tip** Readers will note that the FTP payload package format is very similar to the packet format used for serializing MAVLink itself.


Below is the over-the-wire format for the payload part of the [FILE_TRANSFER_PROTOCOL](../messages/common.md#FILE_TRANSFER_PROTOCOL) message on PX4/*QGroundControl* FTP.

![FILE_TRANSFER_PROTOCOL Payload  format - QGC](../../assets/packets/ftp_transfer_payload_data_qgc.jpg)

Byte Index | C version | Content | Value | Explanation
--- | --- | --- | --- | ---
0 to 1| `uint16_t seq_number` | sequence number for message | | ...
2 | `uint8_t session`   | Session id          | 0&nbsp;-&nbsp;65535 | Session id for read and write commands.
3 | `uint8_t opcode`    | Command [OpCode](#opcodes) (id) | 0 - 255 | Commands ids and ids for ACK/NAK messages.
4 | `uint8_t size`      | Size         | 1 - 255 | Depends on opcode. For Reads/Writes this is the size of the `data` transported. For an ACK to `OpenFileRO` it is the size of the file that has been opened (and must be read).
5 | `uint8_t req_opcode`| Request [OpCode](#opcodes) | 0 - 255 | OpCode (of original message) returned in an ACK or NAK response. 
6 | `uint8_t burst_complete` | Burst complete | 0 - 255 | Only used if `req_opcode` is [BurstReadFile](#BurstReadFile). - 1: set of burst packets complete, 0: More burst packets coming.
7 | `uint8_t padding` | Padding | | 32 bit alignment padding.
8 to 11 | `uint32_t offset` | Content offset | | Offsets into data to be sent for [ListDirectory](#ListDirectory) and [ReadFile](#ReadFile) commands.
12 to (max) 251| `uint8_t data[]` | Data | | Command/response data. Varies by OpCode.


## Command OpCodes {#opcodes}

The opcodes defined/implemented in the server are:

<!--  uint8_t enum Opcode: https://github.com/PX4/Firmware/blob/master/src/modules/mavlink/mavlink_ftp.h -->
     
Opcode | Name | Description
--- | --- | ---
0   | None | Ignored, always ACKed
1   | TerminateSession | Terminates open Read `session`.<br>- Closes the file associated with (`session`) and frees the session ID for re-use. 
2   | ResetSessions | Terminates *all* open read sessions.<br>- Clears all state held by the server; closes all open files, etc.<br>- Sends an ACK reply with no data. <!-- Note, is same as Terminate, but does not check if file session exists -->
<span id="ListDirectory"></span>3   | ListDirectory | List files and directories in `<path>` from `<offset>`.<br>- Opens the directory (`path`), seeks to (`offset`) and fills the result buffer with `NULL`-separated filenames (files also include tab-separated file size) and directory names. Sends an ACK packet with the result buffer on success, otherwise a NAK packet with an error code.<br>- The directory is closed after the operation, so this leaves no state on the server.
4   | OpenFileRO | Opens file at `<path>` for reading, returns `<session>`<br>- Opens the file (`path`) and allocates a *session number*. The file must exist.<br>- Sends an ACK packet with the allocated *session number* on success and the data size of the file to be opened (), otherwise a NAK packet with an error code.<br>- The file remains open after the operation, and must eventually be closed by `Reset` or `Terminate`.
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
* The ACK response may additionally return requested data in the payload (e.g. `OpenFileRO` returns the file session, `ReadFile` returns the requested file data, etc.). 
* The NAK response includes [error code](#error_code). Some common errors include:
  - [InvalidSession](#InvalidSession) if the session is invalid.
  - [FileExists](#FileExists) for file creation commands where the file already exists (`CreateFile`, `CreateDirectory`)
  - [FailErrno](#FailErrno) for file operations that fail for other reasons. The payload include an additional error number from the underlying file system. 
<!-- Looks like most errors are just normal file system numbers - e.g. EROFS. Only very few mapped. FileExists is, but FileProtected is not -->

I would expect that you can run all of the protocol from an asynchronous state machine; once you have 
initiated a process, you can kick the state machine from either packet reception or a timeout event.


/// BELOW HERE IS TESTING
Opcode
Description of function
Description of ACK and what ack can contain
Description of NAK and what nak can contain
Description of timeouts
Description of next steps following NAK> 


### Read File


Open File 
- ACK with session and file size of file to be opened.
- NAK with which errors?
ReadFile - 
- 
- 
TerminateSession
- 
-


Timeout waiting for ACK. 
So basically it does retry a number of times. Then if it can't retry it sends reset command. 
Do sequence for that.



{% mermaid %}
sequenceDiagram;
    participant GCS
    participant Drone
    GCS->>Drone:  OpenFileRO(path)
    GCS->>GCS: Start timeout
    Drone -->> GCS: ACK(session, size=filesize)
    Note right of GCS: Read data in chunks

    GCS->>Drone:  ReadFile(session, size, offset)
    GCS->>GCS: Start timeout
    Drone -->> GCS: ACK(session, size=data_size, data=buffer)
    Note right of GCS: When all chunks read...
    GCS->>Drone:  TerminateSession(session)
    GCS->>GCS: Start timeout
    Drone -->> GCS: ACK()
    Note right of Drone: When all chunks read... blah blah
    Note right of GCS: When all chunks read... blah blah
{% endmermaid %}

### Write File







### List Directory

OpCode: 3
Description: List files and directories in <path> from <offset>.


- Opens the directory (path), seeks to (offset) and fills the result buffer with NULL-separated filenames (files also include tab-separated file size) and directory names. Sends an ACK packet with the result buffer on success, otherwise a NAK packet with an error code.
- The directory is closed after the operation, so this leaves no state on the server.


//// ABOVE HERE IS TESTING



## Error codes {#error_code}

<!--  uint8_t enum ErrorCode: https://github.com/PX4/Firmware/blob/master/src/modules/mavlink/mavlink_ftp.h -->

Error codes returned in NAK response `PayloadHeader.data[0]`

Error | Name | Description
--- | --- | ---
<span id="None"></span>1                | None            | No error
<span id="Fail"></span>2                | Fail            | Unknown failure
<span id="FailErrno"></span>3           | FailErrno       | Fail with an error number (sent back in `PayloadHeader.data[1]`)
<span id="InvalidDataSize"></span>4     | InvalidDataSize | `PayloadHeader.size` is invalid
<span id="InvalidSession"></span>5      | InvalidSessionn | Session is not currently open
<span id="NoSessionsAvailable"></span>6 | NoSessionsAvailable | All available sessions are already in use.
<span id="EOF"></span>7                 | EOF             | Offset past end of file for `ListDirectory` and `ReadFile` commands.
<span id="UnknownCommand"></span>8      | UnknownCommand  | Unknown command / opcode
<span id="FileExists"></span>9          | FileExists      | File exists already
<span id="FileProtected"></span>10      | FileProtected   | File is write protected



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

The MAVLink receiver thread copies an incoming request verbatim from the MAVLink buffer into a request queue, and queues a low-priority work item to handle the packet. This avoids trying to do file I/O on the MAVLink receiver thread, as well as avoiding yet another worker thread. The worker is responsible for directly queueing replies, which are sent with the same sequence number as the request.

The implementation on PX4 only supports a single session.



///// XXX HOW

Implementation notes
PX4 - Is a client. Always handles requests for opcodes from GCS. Then returns ack or nack. Protocol does not require client to request anything.

MESSAGE RECEIVED
MavlinkFTP::handle_message 
- receive message in server
- check if is MAVLINK_MSG_ID_FILE_TRANSFER_PROTOCOL
- if so, decode to a mavlink_file_transfer_protocol_t ftp_request
- check if ftp_request.target_system in request matches this server
- if so, pass request and sysid of source to _process_request()

/// @brief Processes an FTP message
void MavlinkFTP::_process_request(message, source sysid)
- set stream_send false
- set error code to none
- check buffers for holding message exist [_ensure_buffers_exist()]
- check payload  size is within max lenght
- cast message payload into our Payload Header
- Resend last reply (cached) if this is a resend request (ie if ACK/NAK was lost)
  - This checks payload sequence number of this message is one more than last message
  
- CASE switch to workout opcode we are handling.
  - All opcodes are handled in separate functions that take payload and return a code. 
  - e.g. errorCode = _workRename(payload);
  - The payload is updated correctly for the message to be returned (ACK)

- We then iterate the sequence number [SEQUENCE number iterated at both ends every time a NEW message is sent]. Ie If request is seq_number then response will be sent with seq_number+1, and next request will be seq_number+2 ....
- If no error (ACK) then 
  - set payload respond opcode to ACK
  - set payload req_opcode to the opcode from message we are responding to. 
  - Message already has whatever data from the "_workXX" function.
- If error (NAK) then 
  - set payload respond opcode to ACK
  - set payload req_opcode to the opcode from message we are responding to. [NOTE, duplicated work, should just do this once near payload->seq_number++;.]
  - set payload size to 1 (for error code)
  - Set error code based on error number (in this case, just exists) and write error code into zero'th data position.
  - If error has error number then add that to second byte, and set data size of data to 2.
  
- set _last_reply_valid=false (reset, as we are about to send a new message)

- IF this is not being streamed (or if it is but there is an error) then 
- set the target system id based on the stored session
- send the message reply using _reply()

void MavlinkFTP::_reply(mavlink_file_transfer_protocol_t *ftp_req)
- Take a copy of the last (N)ACK so it can be resent (only for small messages.) 
- Set arget network and component to zero (Not used)
- Send the message.

LOTS OF STUFF FOR HANDLING MESSAGE TYPES.


QGC has state engine:     enum OperationState
        {
            kCOIdle,		// not doing anything
            kCOAck,			// waiting for an Ack
            kCOList,		// waiting for List response
            kCOOpenRead,    // waiting for Open response followed by Read download
			kCOOpenBurst,   // waiting for Open response, followed by Burst download
            kCORead,		// waiting for Read response
			kCOBurst,		// waiting for Burst response
            kCOWrite,       // waiting for Write response
            kCOCreate,      // waiting for Create response
            kCOCreateDir,   // waiting for Create Directory response
  


