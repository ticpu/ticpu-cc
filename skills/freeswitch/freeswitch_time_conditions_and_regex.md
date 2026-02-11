# FreeSWITCH Time Conditions and Regex Elements Documentation

## Table of Contents
1. [Time-Based Conditions](#time-based-conditions)
2. [Regex Elements](#regex-elements)
3. [Combining Time and Regex Conditions](#combining-time-and-regex-conditions)
4. [Best Practices](#best-practices)

---

## Time-Based Conditions

FreeSWITCH provides powerful time-based condition matching for implementing business hours, holidays, and scheduled routing.

### Time Condition Attributes

#### 1. `wday` (Weekday)
Matches specific days of the week (1-7, where 1=Sunday, 7=Saturday)

```xml
<!-- Open on weekends only -->
<condition wday="1"/>  <!-- Sunday -->
<condition wday="7"/>  <!-- Saturday -->

<!-- Monday through Friday -->
<condition wday="2-6"/>  <!-- Monday-Friday range -->
```

#### 2. `time-of-day`
Matches specific time ranges within a day (24-hour format HH:MM-HH:MM)

```xml
<!-- Business hours: 8:00 AM to 5:30 PM -->
<condition time-of-day="08:00-17:30"/>

<!-- Lunch break: noon to 1 PM -->
<condition time-of-day="12:00-12:59"/>

<!-- After hours: 5:30 PM to midnight -->
<condition time-of-day="17:30-23:59"/>

<!-- Early morning: midnight to 8 AM -->
<condition time-of-day="00:00-07:59"/>
```

#### 3. `year`
Matches specific year(s)

```xml
<!-- Single year -->
<condition year="2025"/>

<!-- Multiple years (requires multiple conditions) -->
<condition year="2024"/>
<condition year="2025"/>
```

#### 4. `mon` (Month)
Matches specific month(s) (01-12)

```xml
<!-- January -->
<condition mon="01"/>

<!-- Summer months -->
<condition mon="06"/>  <!-- June -->
<condition mon="07"/>  <!-- July -->
<condition mon="08"/>  <!-- August -->
```

#### 5. `mday` (Day of Month)
Matches specific day(s) of the month (01-31)

```xml
<!-- First day of month -->
<condition mday="01"/>

<!-- Christmas -->
<condition mday="25"/>

<!-- Date range -->
<condition mday="24-26"/>  <!-- Dec 24-26 -->

<!-- Multiple specific days -->
<condition mday="01-02"/>  <!-- 1st and 2nd -->
```

#### 6. `week`
Matches specific week(s) of the year (1-53)

```xml
<condition week="1"/>  <!-- First week of year -->
<condition week="52-53"/>  <!-- Last weeks of year -->
```

#### 7. `mweek` (Week of Month)
Matches specific week(s) within a month (1-6)

```xml
<condition mweek="1"/>  <!-- First week of month -->
<condition mweek="-1"/>  <!-- Last week of month -->
```

#### 8. `hour`
Matches specific hour(s) (0-23)

```xml
<condition hour="9-17"/>  <!-- 9 AM to 5 PM -->
```

#### 9. `minute`
Matches specific minute(s) (0-59)

```xml
<condition minute="0-29"/>  <!-- First half of any hour -->
```

#### 10. `minute-of-day`
Matches specific minute of the day (0-1439, where 0=midnight)

```xml
<condition minute-of-day="540-1020"/>  <!-- 9:00 AM to 5:00 PM -->
```

### Complex Time Condition Examples

#### Business Hours with Lunch Break
```xml
<extension name="business_hours">
    <!-- Monday-Friday, 8:00 AM - 12:00 PM -->
    <condition wday="2-6" time-of-day="08:00-11:59" break="on-true">
        <action application="set" data="office_status=open"/>
    </condition>

    <!-- Monday-Friday, 1:00 PM - 5:30 PM -->
    <condition wday="2-6" time-of-day="13:00-17:30" break="on-true">
        <action application="set" data="office_status=open"/>
    </condition>

    <!-- All other times -->
    <condition>
        <action application="set" data="office_status=closed"/>
    </condition>
</extension>
```

#### Holiday Schedule
```xml
<extension name="holiday_routing">
    <!-- Christmas 2025 -->
    <condition year="2025" mon="12" mday="24-26" break="on-true">
        <action application="playback" data="holiday_greeting.wav"/>
        <action application="transfer" data="holiday_voicemail"/>
    </condition>

    <!-- New Year 2026 -->
    <condition year="2026" mon="01" mday="01-02" break="on-true">
        <action application="playback" data="holiday_greeting.wav"/>
        <action application="transfer" data="holiday_voicemail"/>
    </condition>
</extension>
```

---

## Regex Elements

The `<regex>` element provides powerful pattern matching capabilities with OR/AND logic for complex condition evaluation.

### Basic Regex Structure

```xml
<condition regex="any|all">
    <regex field="variable_name" expression="pattern"/>
    <regex field="another_variable" expression="pattern2"/>
    <action application="..." data="..."/>
</condition>
```

### Regex Condition Attributes

#### 1. `regex="any"` (OR Logic)
Matches if ANY of the regex patterns match

```xml
<condition regex="any">
    <regex field="destination_number" expression="^911$"/>
    <regex field="destination_number" expression="^112$"/>
    <regex field="destination_number" expression="^999$"/>
    <action application="transfer" data="emergency"/>
</condition>
```

#### 2. `regex="all"` (AND Logic)
Matches only if ALL regex patterns match

```xml
<condition regex="all">
    <regex field="caller_id_number" expression="^1000$"/>
    <regex field="destination_number" expression="^2\d{3}$"/>
    <action application="set" data="authorized_internal_call=true"/>
</condition>
```

### Time Conditions Within Regex

You can combine time conditions with field matching in regex blocks:

```xml
<condition regex="any">
    <!-- Weekday business hours -->
    <regex wday="2-6" time-of-day="08:00-17:00"/>
    <!-- Saturday morning -->
    <regex wday="7" time-of-day="08:00-12:00"/>
    <!-- Specific holidays -->
    <regex year="2025" mon="12" mday="25"/>
    <action application="set" data="office_open=true"/>
</condition>
```

### Advanced Regex Examples

#### Multiple Field Matching
```xml
<extension name="authorized_transfer">
    <condition regex="all">
        <regex field="${authorized}" expression="^true$"/>
        <regex field="${transfer_enabled}" expression="^yes$"/>
        <regex field="destination_number" expression="^\*\*(\d+)$"/>
        <action application="transfer" data="$1"/>
    </condition>
</extension>
```

#### Complex OR Logic with Different Fields
```xml
<extension name="vip_routing">
    <condition regex="any">
        <regex field="caller_id_number" expression="^(1001|1002|1003)$"/>
        <regex field="${sip_h_X-VIP}" expression="^true$"/>
        <regex field="${customer_tier}" expression="^(gold|platinum)$"/>
        <action application="set" data="vip_caller=true"/>
        <action application="transfer" data="vip_queue"/>
    </condition>
</extension>
```

---

## Combining Time and Regex Conditions

### Real-World Example: Municipal Emergency Services

This example from the CITAM dialplan shows sophisticated time and regex combination:

```xml
<extension name="sainte_marguerite_urgences">
    <condition field="destination_number" expression="^(8778549847)$" break="on-false">
        <!-- Set initial variables -->
        <action inline="true" application="set" data="city=ste_marguerite"/>
        <action inline="true" application="set" data="priority=80"/>

        <!-- Temporary closure -->
        <condition regex="any" break="on-true">
            <regex year="2025" mon="10" mday="16" time-of-day="16:30-20:59"/>
            <regex year="2025" mon="10" mday="17" time-of-day="16:30-20:59"/>
            <regex year="2025" mon="10" mday="18" time-of-day="09:30-17:59"/>
            <action inline="true" application="set" data="citam_status=closed"/>
        </condition>

        <!-- Regular business hours -->
        <condition regex="any" break="on-true">
            <!-- Weekends -->
            <regex wday="1"/>  <!-- Sunday all day -->
            <regex wday="7"/>  <!-- Saturday all day -->

            <!-- Weekday after-hours -->
            <regex wday="2" time-of-day="00:00-08:29"/>  <!-- Monday early -->
            <regex wday="2" time-of-day="16:30-00:00"/>  <!-- Monday evening -->
            <regex wday="3" time-of-day="00:00-08:29"/>
            <regex wday="3" time-of-day="16:30-00:00"/>
            <regex wday="4" time-of-day="00:00-08:29"/>
            <regex wday="4" time-of-day="16:30-00:00"/>
            <regex wday="5" time-of-day="00:00-08:29"/>
            <regex wday="5" time-of-day="16:30-00:00"/>
            <regex wday="6" time-of-day="00:00-08:29"/>
            <regex wday="6" time-of-day="16:30-00:00"/>

            <action inline="true" application="set" data="citam_status=opened"/>
        </condition>

        <!-- Holiday schedule -->
        <condition regex="any" break="on-true">
            <regex year="2024" mon="12" mday="24-26"/>
            <regex year="2024" mon="12" mday="31"/>
            <regex year="2025" mon="01" mday="01-02"/>
            <regex year="2025" mon="04" mday="18"/>  <!-- Good Friday -->
            <regex year="2025" mon="04" mday="21"/>  <!-- Easter Monday -->
            <regex year="2025" mon="05" mday="19"/>  <!-- Victoria Day -->
            <regex year="2025" mon="06" mday="24"/>  <!-- Quebec National Holiday -->
            <regex year="2025" mon="09" mday="01"/>  <!-- Labour Day -->
            <regex year="2025" mon="10" mday="13"/>  <!-- Thanksgiving -->
            <action inline="true" application="set" data="citam_status=opened"/>
        </condition>

        <action application="transfer" data="${destination_number} XML citam"/>
    </condition>
</extension>
```

### Pattern: Time-Based Service Availability

```xml
<extension name="service_availability">
    <condition field="destination_number" expression="^(8005551234)$">
        <!-- Set default -->
        <action inline="true" application="set" data="service_status=closed"/>

        <!-- Check if we're in business hours -->
        <condition regex="any" break="on-true">
            <!-- Normal business hours -->
            <regex wday="2-6" time-of-day="08:00-17:00"/>
            <!-- Extended support on Tuesday/Thursday -->
            <regex wday="3,5" time-of-day="17:00-20:00"/>
            <action inline="true" application="set" data="service_status=open"/>
        </condition>

        <!-- Override for holidays - always closed -->
        <condition regex="any" break="on-true">
            <regex year="2025" mon="01" mday="01"/>
            <regex year="2025" mon="07" mday="04"/>
            <regex year="2025" mon="12" mday="25"/>
            <action inline="true" application="set" data="service_status=holiday"/>
        </condition>

        <!-- Route based on status -->
        <condition field="${service_status}" expression="^open$">
            <action application="transfer" data="service_queue"/>
            <anti-action application="transfer" data="voicemail"/>
        </condition>
    </condition>
</extension>
```

---

## Best Practices

### 1. Order Conditions from Most to Least Specific
Place more specific conditions (like temporary closures) before general ones:

```xml
<!-- Specific temporary closure -->
<condition regex="any" break="on-true">
    <regex year="2025" mon="03" mday="15" time-of-day="14:00-16:00"/>
    <action inline="true" application="set" data="status=maintenance"/>
</condition>

<!-- General business hours -->
<condition regex="any" break="on-true">
    <regex wday="2-6" time-of-day="08:00-17:00"/>
    <action inline="true" application="set" data="status=open"/>
</condition>
```

### 3. Use `break` Attributes Wisely
- `break="on-true"`: Stop checking conditions once matched
- `break="on-false"`: Stop if condition fails (useful for required checks)
- `break="never"`: Always continue (for setting multiple variables)

```xml
<!-- Must be authorized user -->
<condition field="${authorized}" expression="^true$" break="on-false">
    <!-- Then check time -->
    <condition regex="any" break="on-true">
        <regex wday="2-6" time-of-day="08:00-17:00"/>
        <action application="bridge" data="internal_support"/>
    </condition>
</condition>
```

### 4. Group Related Time Conditions
Use regex="any" to group related time periods:

```xml
<!-- All after-hours periods -->
<condition regex="any">
    <!-- Weekday evenings -->
    <regex wday="2-6" time-of-day="17:00-23:59"/>
    <regex wday="2-6" time-of-day="00:00-08:00"/>
    <!-- Weekends -->
    <regex wday="1,7"/>
    <action application="set" data="after_hours=true"/>
</condition>
```

### 5. Document Holiday Logic
Always comment holiday conditions for future maintenance:

```xml
<condition regex="any">
    <regex year="2025" mon="04" mday="18"/>  <!-- Good Friday 2025 -->
    <regex year="2025" mon="05" mday="19"/>  <!-- Victoria Day 2025 -->
    <regex year="2025" mon="06" mday="24"/>  <!-- Saint-Jean-Baptiste Day -->
    <regex year="2025" mon="07" mday="01"/>  <!-- Canada Day -->
    <action application="set" data="holiday=true"/>
</condition>
```

### 6. Test Time Conditions
Use FreeSWITCH console to test time conditions:

```bash
# Test if current time matches condition
fs_cli> strftime_tz UTC %Y-%m-%d %H:%M:%S

# Evaluate dialplan with simulated time
fs_cli> dialplan <context> <destination> [caller_id] [time_string]
```

### 7. Consider Timezones
When dealing with multiple timezones, set the appropriate timezone:

```xml
<action application="set" data="timezone=America/Montreal"/>
<action application="set" data="tod_tz_offset=-5"/>
```

### 8. Combine with Variable Conditions
Mix time conditions with other variable checks:

```xml
<condition regex="all">
    <regex field="${emergency_mode}" expression="^false$"/>
    <regex wday="2-6" time-of-day="08:00-17:00"/>
    <regex field="${queue_available}" expression="^true$"/>
    <action application="transfer" data="normal_queue"/>
</condition>
```

---

## Common Patterns

### Pattern 1: Business Hours with Lunch
```xml
<condition regex="any">
    <regex wday="2-6" time-of-day="08:00-11:59"/>  <!-- Morning -->
    <regex wday="2-6" time-of-day="13:00-17:00"/>  <!-- Afternoon -->
    <action inline="true" application="set" data="business_hours=true"/>
</condition>
```

### Pattern 2: 24/7 Except Holidays
```xml
<!-- Default: always open -->
<action inline="true" application="set" data="service_available=true"/>

<!-- Close on holidays -->
<condition regex="any">
    <regex year="2025" mon="01" mday="01"/>
    <regex year="2025" mon="12" mday="25"/>
    <action inline="true" application="set" data="service_available=false"/>
</condition>
```

### Pattern 3: Graduated Service Levels
```xml
<!-- Premium support hours -->
<condition regex="any" break="on-true">
    <regex wday="2-6" time-of-day="08:00-17:00"/>
    <action inline="true" application="set" data="support_level=premium"/>
</condition>

<!-- Basic support hours -->
<condition regex="any" break="on-true">
    <regex wday="2-6" time-of-day="17:00-21:00"/>
    <regex wday="7" time-of-day="09:00-17:00"/>
    <action inline="true" application="set" data="support_level=basic"/>
</condition>

<!-- Emergency only -->
<condition>
    <action inline="true" application="set" data="support_level=emergency"/>
</condition>
```

### Pattern 4: Maintenance Windows
```xml
<!-- Regular maintenance window -->
<condition regex="any">
    <!-- Every Sunday 2-4 AM -->
    <regex wday="1" time-of-day="02:00-04:00"/>
    <!-- First Tuesday of month, 3-4 AM -->
    <regex wday="3" mweek="1" time-of-day="03:00-04:00"/>
    <action application="playback" data="maintenance_message.wav"/>
    <action application="hangup"/>
</condition>
```

---

## Troubleshooting Time Conditions

### 1. Time Not Matching Expected
- Check server timezone: `date` command in Linux
- Verify FreeSWITCH timezone settings
- Use `inline="true"` for variable assignments that affect routing

### 2. Complex Conditions Not Working
- Break down into simpler conditions for testing
- Use `info` application to display variables
- Enable debug logging for dialplan module

### 3. Regex Patterns Not Matching
- Test regex patterns separately
- Check field variable expansion with `info`
- Verify `break` attribute logic

### Example Debug Extension
```xml
<extension name="debug_time">
    <condition field="destination_number" expression="^9999$">
        <action application="answer"/>
        <action application="log" data="NOTICE Current time: ${strftime(%Y-%m-%d %H:%M:%S)}"/>
        <action application="log" data="NOTICE Weekday: ${strftime(%w)}"/>
        <action application="log" data="NOTICE Testing conditions..."/>

        <condition wday="2-6" time-of-day="08:00-17:00">
            <action application="log" data="NOTICE Business hours: YES"/>
            <anti-action application="log" data="NOTICE Business hours: NO"/>
        </condition>

        <action application="playback" data="tone_stream://%(1000,0,500)"/>
        <action application="hangup"/>
    </condition>
</extension>
```

