# 133 - Allow over-long authentication sessions in `SaslTermination`

Allow over-long authentication sessions in the `SaslTermination` filter added by [Proposal 124](124-sasl-termination.md).

## Current situation

Proposal 124 added the `SaslTermination` filter with support for OAUTHBEARER, SCRAM-SHA-256 and SCRAM-SHA-512.
The filter supports [KIP-368 reauthentication](https://cwiki.apache.org/confluence/spaces/KAFKA/pages/89068981/KIP-368+Allow+SASL+Connections+to+Periodically+Re-Authenticate), and for OAUTHBEARER access tokens the reauthentication time advertized to the client is based on the access token's own expiry (`expires_in`).

## Motivation

A contributor has [commented](https://github.com/kroxylicious/kroxylicious/issues/4391#issuecomment-5413561636) asking about configurability of the session lifetime we advertise to clients in the `SaslAuthenticate` response and which drives subsequent reauthentication:

> Ideally JWT tokens will be short lived, in a streaming scenario, it may not be ideal to re-establish the connection every 10 minutes or so. It might be sufficient to authenticate the JWT only when the connection request is made.

This cannot always be acheived by configuring a longer token lifetime on the Oauth authorization server. In general there is no upper bound on  connection lifetime. 

The filter already supports a `maxTimeBeforeReauth` configuration parameter, but it cannot be used to extend a session lifetime, as described in the documentation: "the effective session expiry is the earlier of this value and the token’s own expiry."
    
## Proposal

To enable client authentication sessions that are longer-lived than token expiry we would add a new configuration:

| Name                                    | Type    | Default   |
|-----------------------------------------|---------|-----------|
| `maxTimeBeforeReauthIgnoresTokenExpiry` | boolean | `false`   |

When this is `true` the reauthentication time advertized to the Kafka client will be `maxTimeBeforeReauth`, rather than the earlier of `maxTimeBeforeReauth` and the token's own expiry. 
Setting `maxTimeBeforeReauth` to an implausibly long time, e.g. `3650d`, would have the effect that the Kafka session would last for the duration of the connection. 


## Affected/not affected projects

This affects only the `SaslTermination` filter in the `kroxylicious` project.

## Compatibility

This change is backwards compatible.

## Rejected alternatives


