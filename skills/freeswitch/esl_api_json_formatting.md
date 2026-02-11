# FreeSWITCH ESL API JSON Formatting Implementation

Native JSON output support in FreeSWITCH's Event Socket Layer API commands, specifically for show commands.

**Key Finding**: JSON formatting is entirely server-side in FreeSWITCH, not client-side formatting in fs_cli.

## Core Implementation

### Source Location
- **File**: `/src/mod/applications/mod_commands/mod_commands.c`
- **JSON Callback**: `show_as_json_callback` (lines 5671-5712)
- **Format Detection**: Command parser (lines 6148-6179)
- **Library**: Native cJSON integration

### Command Syntax
```bash
# Correct ESL API command syntax
api show channels as json

# NOT: show channels as json (missing api prefix)
```

### Supported Output Formats
- **Default**: `api show channels` (tabular format)
- **JSON**: `api show channels as json`
- **XML**: `api show channels as xml`
- **CSV/Delimited**: `api show channels as csv` or `api show channels as delim`

## JSON Output Structure

### Response Format
```json
{
  "row_count": N,
  "rows": [
    {
      // Channel object with all channel fields
    }
  ]
}
```

### Empty Channels Response
```json
{"row_count": 0}
```

## Implementation Details

### Server-Side Processing
```c
// Line 5915-5917: Command parsing with "as" parameter
if (argv[2] && !strcasecmp(argv[1], "as")) {
    as = argv[2];
}

// Line 6148-6150: JSON callback invocation
} else if (!strcasecmp(as, "json")) {
    switch_cache_db_execute_sql_callback(db, sql, show_as_json_callback, &holder, &errmsg);
```

### ESL Protocol Notes
- No special Content-Type headers required
- JSON formatting happens at application layer within FreeSWITCH
- Not at ESL message transport layer

## Common Issues & Solutions

### Issue: "show channels as json" Returns Tabular Format

**Root Causes:**
1. **Missing "api" prefix** - Use `api show channels as json`
2. **Wrong ESL command type** - Must be sent as API command
3. **FreeSWITCH version** - Ensure recent version with JSON support

**Client Implementation:**
```python
# Correct approach
response = esl_connection.api("show channels as json")
# Returns: {"row_count": N, "rows": [...]}

# Incorrect approach
response = esl_connection.api("show channels")
# Returns: Tabular format
```

### Issue: Hardcoded Command Mapping in Client

**Problem**: Client code transforming commands unnecessarily:
```rust
// Wrong approach - hardcoded mappings
match subcommand {
    "channels" => "show channels",
    // ... more hardcoded mappings
}
```

**Solution**: Pass through commands directly:
```rust
// Correct approach - direct passthrough  
let command = format!("show {}", parts.join(" "));
```

## Universal Applicability

This JSON formatting support extends beyond just `show channels`:
- `api show calls as json`
- `api show registrations as json` 
- `api show modules as json`
- Any FreeSWITCH show command with `as json` parameter

## ESL Client Implementation Recommendations

1. **Pass through commands directly** - Don't hardcode command mappings
2. **Use proper ESL API syntax** - Always prefix with `api`
3. **Let FreeSWITCH handle formatting** - No client-side JSON parsing needed
4. **Support all formats** - json, xml, csv available for most show commands

## Version Compatibility

JSON support has been available in FreeSWITCH for many years across all modern versions. This is a stable, production-ready feature implemented in mod_commands core functionality.