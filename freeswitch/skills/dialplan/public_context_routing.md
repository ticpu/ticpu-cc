# Public Context Routing

The public context receives inbound calls from external providers. TLS,
digest auth, and IP ACLs already verify the caller before the dialplan
runs. The public context's only job is to route the destination number
to an internal context via transfer or bridge.

## Keep Conditions Minimal

A phone number may failover between providers (e.g. Telus goes down, calls
arrive via Vidéotron instead). Gating on `sofia_profile_name` or other
ingress metadata causes silent routing failures during provider failover.

```xml
<!-- WRONG: profile gate blocks calls arriving on a failover provider -->
<extension name="inbound_telus">
    <condition field="${sofia_profile_name}" expression="^(telus)$" break="on-false">
        <action application="log" data="INFO INBOUND CALL FROM ${sofia_profile_name}"/>
    </condition>
    <condition field="destination_number" expression="^(4182282865)$" break="on-true">
        <action application="bridge" data="sofia/gateway/pbx/$1"/>
    </condition>
</extension>

<!-- CORRECT: route on destination_number only -->
<extension name="inbound_telus">
    <condition field="destination_number" expression="^(4182282865)$" break="on-true">
        <action application="log" data="INFO bridging call from ${sofia_profile_name} to PBX"/>
        <action application="bridge" data="sofia/gateway/pbx/$1"/>
    </condition>
</extension>
```

The first condition on `sofia_profile_name` with `break="on-false"` acts as a
gate for the entire extension. When Telus is down and the same DID arrives on
the Vidéotron profile, the gate fails and the destination routing is never
reached — the call drops with NO_ROUTE_DESTINATION.

## Principle

The public context should only match on destination_number. Authentication
and authorization are handled at the SIP profile level (TLS, ACLs, digest
auth) before the call ever reaches the dialplan.
