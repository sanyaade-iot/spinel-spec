# Security Considerations #

EDITOR: Insert verbiage here conforming to RFC Editor guidelines.

## Network Name Normalization {#security-network-name}

TODO: Discuss UTF8 verification and normalization considerations related to
`PROP_NET_NETWORK_NAME`.

## Raw Application Access ##

Spinel **MAY** be used as an API boundary for allowing processes to configure the NCP. However, such a system **MUST NOT** give unprivileged processess the ability to send or receive arbitrary command frames to the NCP. Only the specific commands and properties that are required should be allowed to be passed, and then only after being checked for proper format.
