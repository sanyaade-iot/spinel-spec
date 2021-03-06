# Commands

## CMD 9: (Host->NCP) CMD_NET_SAVE

Bytes: |     1    |      1
:-------|:---------|:-------------
Fields: | `HEADER` | `CMD_NET_SAVE`

Save network state command. Saves any current network credentials and state necessary to reconnect to the current network to non-volatile memory.

This operation affects non-volatile memory only. The current network information stored in volatile memory is unaffected.

The response to this command is always a `CMD_PROP_VALUE_IS` for `PROP_LAST_STATUS`, indicating the result of the operation.

This command is only available if the `CAP_NET_SAVE` capability is set.

## CMD 10: (Host->NCP) CMD_NET_CLEAR

Bytes: |     1    |        1
:-------|:---------|:---------------
Fields: | `HEADER` | `CMD_NET_CLEAR`

Clear saved network settings command. Erases all network credentials and state from non-volatile memory. The erased settings include any data saved automatically by the network stack firmware and/or data saved by `CMD_NET_SAVE` operation.

This operation affects non-volatile memory only. The current network information stored in volatile memory is unaffected.

The response to this command is always a `CMD_PROP_VALUE_IS` for `PROP_LAST_STATUS`, indicating the result of the operation.

This command is always available independent of the value of `CAP_NET_SAVE` capability.


## CMD 11: (Host->NCP) CMD_NET_RECALL

Bytes: |    1   |      1
--------|--------|----------------
Fields: | HEADER | CMD_NET_RECALL

Recall saved network state command. Recalls any previously saved network credentials and state previously stored by `CMD_NET_SAVE` from non-volatile memory.

This command will typically generated several unsolicited property updates as the network state is loaded. At the conclusion of loading, the authoritative response to this command is always a `CMD_PROP_VALUE_IS` for `PROP_LAST_STATUS`, indicating the result of the operation.

This command is only available if the `CAP_NET_SAVE` capability is set.
