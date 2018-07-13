# Frequently Asked Questions (FAQ)

## Users

<dl>
  <dt>How efficient is MAVLink?</dt>
  <dd>MAVLink is a very efficient protocol. MAVLink 1 has just 8 bytes overhead per packet, including start sign and packet drop detection. MAVLink 2 has just 14 bytes of overhead (27 if signing is used), but is now a much more extensible protocol.</dd>

  <dt>How many vehicles does MAVLink support?</dt>
  <dd>255 vehicles, with system IDs ranging from 1 to 255 (0 is not a valid vehicle ID).
    <br><b>Note:</b> Strictly speaking MAVLink supports 255 concurrent <em>systems</em>, and these can include a mix of vehicles, GCS, antenna trackers and other hardware.</dd>

  <dt>Where can I use MAVLink?</dt>
  <dd>MAVLink has been shown to work on multiple microcontrollers and operating systems, including ARM7, ATMega, dsPic, STM32 and Windows, Linux, MacOS, Android and iOS.</dd>

  <dt>How reliable is MAVLink?</dt>
  <dd>Very. MAVLink has been used since 2009 to communicate between many different vehicles, ground stations (and other nodes) over varied and challenging communication channels (high latency/noise). It provides methods for detecting packet drops, and the well-established ITU X.25 checksum for packet corruption detection.</dd>
</dl>


## Developers

<dl>
  <dt>Can I use MAVLink in a closed source application without copyright issues?</dt>
  <dd>Yes, without any limitations. The generated MAVLink library headers are made available under the *MIT license* (for more information see: <a href="../README.md#license">Introduction > License</a>).
  </dd>

  <dt>How does MAVLink detect and decode messages in the byte stream?</dt>
  <dd>MAVLink waits for the packet start sign, then reads the packet length and matches the checksum after n bytes. If the checksum matches, it returns the decoded packet and waits again for the start sign. If bytes are altered or lost, it will drop the current message and continue the next try on the following message.</dd>

  <dt>MAVLink uses only one start sign - isn't this less safe than using two or three start signs?</dt>
  <dd>No. We use the CRC check to reliably determine whether a complete message has been received. Using additional start signs may increase likelihood of detecting the start point, but would provide no greater certainty of message validity. Since extra signs would increase bytes on the communication link, we choose not to use them.</dd>

  <dt>What are the system and component IDs for?</dt>
  <dd>The system ID represents the identity of a particular <em>MAVLink system</em> (vehicle, GCS, etc.). MAVLink can be used with up to 255 systems at the same time. The component ID reflects a component that is part of a larger system - for example a system might include an autopilot, companion computer and/or camera, which can be separately addressed. The component ID therefore lets MAVLink be used for both on- and off-board communication.</dd>

  <dt>Why is the sequence number in the MAVLink header needed?</dt>
  <dd>MAVLink is part of the safety critical components of an unmanned air system. A bad communication link dropping many packets can endanger the flight safety of the aircraft and has to be monitored. Having the sequence in the header allows MAVLink to continuously provide feedback about the packet drop rate and thus allows the aircraft or ground control station to take action.</dd>

  <dt>I would like to help improve the decoding/encoding routines or other features. Can MAVLink be changed?</dt>
  <dd>Yes, but only very, very carefully with safety testing. 
  <br>MAVLink is used as a safety-critical component in many autopilot systems and has undergone many years of testing. Please suggest new features on the MAVLink <a href="../README.md#support">support channels</a>.</dd>
</dl>