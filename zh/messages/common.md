<!-- THIS FILE IS AUTO-GENERATED: https://github.com/mavlink/mavlink/blob/master/doc/mavlink_gitbook.py -->

# MAVLINK通用消息集

The MAVLink *common* message set contains *standard* definitions that are managed by the MAVLink project. The definitions cover functionality that is considered useful to most ground control stations and autopilots. MAVLink-compatible systems are expected to use these definitions where possible (if an appropriate message exists) rather than rolling out variants in their own [dialects](../messages/README.md).

The original definitions are defined in [common.xml](https://github.com/mavlink/mavlink/blob/master/message_definitions/v1.0/common.xml).

> **Tip** The common set `includes` [minimal.xml](minimal.md), which contains the *minimal set* of definitions for any MAVLink system. These definitions are [reproduced at the end of this topic](#minimal).

<span></span>

> **Note** MAVLink 2 messages have an ID > 255 and are marked up using **(MAVLink 2)** in their description.

<span id="mav2_extension_field"></span>

> **Note** MAVLink 2 extension fields that have been added to MAVLink 1 messages are displayed in blue. 

<style>
td {
    vertical-align:top;
}
</style>

xxx



### MAV_CMD_REQUEST_STORAGE_INFORMATION (525)

**DEPRECATED** Replaced by [MAV_CMD_REQUEST_MESSAGE](#MAV_CMD_REQUEST_STORAGE_INFORMATION) (2019-08).</p>

Request storage information ([STORAGE_INFORMATION](#STORAGE_INFORMATION)). Use the command's target_component to target a specific component's storage.

  <table class="sortable">
   <thead>
    <tr>
     <th>Param (:Label)</th>
     <th>Description</th>
     <th>Values</th>
    </tr>
   </thead>
   <tbody>
    <tr>
     <td>1: Storage ID</td>
     <td>Storage ID (0 for all, 1 for first, 2 for second, etc.)</td>
     <td>
      <em>min:</em>0 <em>increment:</em>1</td>
    </tr>
    <tr>
     <td>2: Information</td>
     <td>0: No Action 1: Request storage information</td>
     <td>
      <em>min:</em>0 <em>max:</em>1 <em>increment:</em>1</td>
    </tr>
    <tr>
     <td>3</td>
     <td>Reserved (all remaining params)</td>
     <td>
     </td>
    </tr>
   </tbody>
  </table>


### MAV_CMD_REQUEST_MESSAGE


XXX

### STORAGE_INFORMATION

# Minimal.xml {#minimal}

The minimal set of definitions required for any MAVLink system are included from [minimal.xml](minimal.md). These are listed below.

MD

[common](_html/minimal.html ':include type=iframe width=100% height=800px')

