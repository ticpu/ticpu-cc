FreeSWITCH Sofia supports `profile::gateway` syntax to explicitly specify which profile a gateway belongs to. Recommended when you have gateways with the same name across multiple profiles.

### Gateway Registration (sofia_reg.c:3686-3709)

Gateways are registered in the hash table twice:

```c
char *pkey = switch_mprintf("%s::%s", profile->name, key);  // Line 3686
status = switch_core_hash_insert(mod_sofia_globals.gateway_hash, key, gateway);      // Line 3708
status |= switch_core_hash_insert(mod_sofia_globals.gateway_hash, pkey, gateway);    // Line 3709
```

### Gateway Lookup (sofia_reg.c:3606)

```c
gateway = (sofia_gateway_t *) switch_core_hash_find(mod_sofia_globals.gateway_hash, key);
```

Works with either `gateway_name` or `profile::gateway_name`.

### Gateway Creation (sofia.c:3822)

```c
char *pkey = switch_mprintf("%s::%s", profile->name, name);  // Line 3822
```

## Valid Syntax

```xml
<!-- Gateway name only (if unique across profiles) -->
<action application="bridge" data="sofia/gateway/gateway_name/5551234"/>

<!-- profile::gateway (recommended - explicit) -->
<action application="bridge" data="sofia/gateway/internal::gateway_name/5551234"/>

<!-- Profile with gateway parameter (alternative) -->
<action application="bridge" data="sofia/profile/5551234@domain;gw=gateway_name"/>
```

## Invalid Syntax

```xml
<!-- ❌ WRONG: :: only valid with sofia/gateway/, not sofia/profile/ -->
<action application="bridge" data="sofia/internal::gateway_name/5551234"/>
```

## When to Use

- Multiple profiles with same gateway name
- Explicit configuration for clarity
- Production environments with complex routing
