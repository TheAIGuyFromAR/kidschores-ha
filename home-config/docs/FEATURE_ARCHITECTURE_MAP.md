# Feature Architecture Map
**Visual System Overview**

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KIDSCHORES SYSTEM                           │
│                                                                     │
│  ┌────────────────────┐        ┌────────────────────┐             │
│  │  KidsChores Core   │        │  Custom Additions  │             │
│  │                    │        │                    │             │
│  │ • Chores (36)      │───────▶│ • Required Sensor  │             │
│  │ • Kids (4)         │        │ • Allowance Calc   │             │
│  │ • Points Tracking  │        │ • Banking System   │             │
│  │ • Claim/Approve    │        │ • Notifications    │             │
│  └────────────────────┘        └────────────────────┘             │
│           │                              │                         │
│           │                              │                         │
│           ▼                              ▼                         │
│  ┌──────────────────────────────────────────────────────┐         │
│  │              AUTOMATION LAYER                        │         │
│  │                                                      │         │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │         │
│  │  │ Custody  │  │  Payout  │  │ Notific. │          │         │
│  │  │  Toggle  │  │ Automation│  │  System  │          │         │
│  │  └──────────┘  └──────────┘  └──────────┘          │         │
│  │                                                      │         │
│  └──────────────────────────────────────────────────────┘         │
│           │                              │                         │
│           │                              │                         │
│           ▼                              ▼                         │
│  ┌──────────────────────────────────────────────────────┐         │
│  │              PRESENTATION LAYER                      │         │
│  │                                                      │         │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │         │
│  │  │   Kid    │  │  Parent  │  │  Mobile  │          │         │
│  │  │Dashboard │  │Dashboard │  │  Notif.  │          │         │
│  │  └──────────┘  └──────────┘  └──────────┘          │         │
│  │                                                      │         │
│  └──────────────────────────────────────────────────────┘         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: Weekly Allowance Cycle

```
SUNDAY 00:00 ─────────────────────────────────────────────┐
│                                                          │
│ 1. KidsChores resets weekly points                      │
│    └─▶ chore_approvals_weekly = {}                      │
│                                                          │
▼                                                          │
MONDAY - SATURDAY ────────────────────────────────────────┤
│                                                          │
│ 2. Kids complete chores                                 │
│    ├─▶ Claim chore (kid)                                │
│    ├─▶ Approve chore (parent)                           │
│    └─▶ Points accumulate                                │
│                                                          │
│ 3. Required Sensor tracks progress                      │
│    ├─▶ Counts "Required" chores completed               │
│    ├─▶ Updates all_required_done boolean                │
│    └─▶ Calculates completion_rate                       │
│                                                          │
│ 4. Allowance Sensor calculates earnings                 │
│    ├─▶ IF all_required_done: base = $10                 │
│    ├─▶ ELSE: base = $0                                  │
│    └─▶ bonus = points × $0.25                           │
│                                                          │
│ 5. SATURDAY 16:00 - Warning if base locked              │
│    └─▶ Notification: "Complete X more chores!"          │
│                                                          │
▼                                                          │
SUNDAY 18:00 ─────────────────────────────────────────────┘
│
│ 6. Payout Automation triggers
│    ├─▶ Read allowance amount
│    ├─▶ Deposit to checking account
│    ├─▶ Auto-save 20% to savings
│    └─▶ Send payment notification
│
▼
CYCLE REPEATS
```

---

## Feature Dependency Graph

```
                    ┌─────────────────┐
                    │  KidsChores     │
                    │  Integration    │
                    └────────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
      ┌─────────▼─────────┐    ┌─────────▼─────────┐
      │ RequiredChores    │    │    Allowance      │
      │ CompletedSensor   │    │  Template Sensor  │
      └─────────┬─────────┘    └─────────┬─────────┘
                │                         │
                │         ┌───────────────┴───────────────┐
                │         │                               │
                │    ┌────▼────┐                    ┌─────▼─────┐
                │    │ Banking │                    │Dashboards │
                │    │ Accounts│                    │           │
                │    └────┬────┘                    └───────────┘
                │         │
                │    ┌────▼────────────────┐
                │    │                     │
                │    ▼                     ▼
                │ ┌──────┐            ┌────────┐
                │ │Payout│            │ Goal   │
                │ │System│            │Tracking│
                │ └──┬───┘            └────────┘
                │    │
                │    ▼
                │ ┌────────────┐
                │ │Auto-Savings│
                │ │ (20% rule) │
                │ └─────┬──────┘
                │       │
                │       ▼
                │ ┌────────────┐
                │ │ Monthly    │
                │ │ Interest   │
                │ └────────────┘
                │
                ▼
      ┌─────────────────┐
      │   Notifications │
      │   - Daily       │
      │   - Saturday    │
      │   - Achievements│
      └─────────────────┘

      ┌─────────────────┐
      │ Custody Toggle  │  ◀── Independent feature
      │  (Lilly only)   │
      └─────────────────┘
```

---

## Entity Relationship Map

```
┌────────────────────────────────────────────────────────────┐
│                         ENTITIES                           │
└────────────────────────────────────────────────────────────┘

KidsChores Integration Entities:
├── sensor.kc_bella_points
├── sensor.kc_bella_chores_completed_weekly
├── sensor.kc_bella_required_chores_completed_weekly  ◀── Custom
├── sensor.kidschores_chore_* (36 chores)
└── button.kc_bella_claim_chore

Custom Template Entities:
├── sensor.bella_weekly_allowance  ◀── Depends on Required sensor
└── sensor.bella_goal_progress

Helper Entities:
├── input_boolean.lilly_home_this_week  ◀── Custody toggle
├── input_number.bella_checking         ◀── Banking
├── input_number.bella_savings          ◀── Banking
├── input_number.bella_goal_amount      ◀── Goal tracking
└── input_text.bella_goal_name          ◀── Goal tracking

Automation Entities:
├── automation.custody_toggle_weekly
├── automation.manage_lilly_chores
├── automation.weekly_payout
├── automation.auto_savings_transfer
├── automation.saturday_allowance_warning
└── automation.daily_chore_reminder
```

---

## State Machine: Chore Lifecycle

```
┌─────────┐
│ CREATED │  ◀── Import script creates chore
└────┬────┘
     │
     ▼
┌─────────┐
│ PENDING │  ◀── Due date set, waiting for kid
└────┬────┘
     │
     │ Kid clicks "Claim"
     ▼
┌─────────┐
│ CLAIMED │  ◀── Waiting for parent approval
└────┬────┘
     │
     ├─────────────────┐
     │                 │
     │ Parent          │ Parent
     │ Approves        │ Disapproves
     │                 │
     ▼                 ▼
┌─────────┐       ┌─────────┐
│APPROVED │       │ PENDING │  ◀── Back to start
└────┬────┘       └─────────┘
     │
     │ Points awarded
     │ Required sensor updates
     │ Allowance recalculates
     │
     ▼
┌─────────┐
│COMPLETED│  ◀── Shows in history
└─────────┘
     │
     │ Next due date set (if recurring)
     │
     ▼
┌─────────┐
│ PENDING │  ◀── Cycle repeats
└─────────┘
```

---

## Automation Trigger Timeline

```
DAILY SCHEDULE:
├── 00:00:01 - Sunday custody toggle check
├── 09:00    - Morning chores due (blinds, plant lights)
├── 15:00    - Daily chore reminder notification
├── 16:00    - Saturday allowance warning (if needed)
├── 17:00    - Evening chores due (litter, dog)
├── 18:00    - Sunday payout automation
└── 23:59    - Most chores default due time

WEEKLY EVENTS:
├── Sunday 00:00    - KidsChores resets weekly points
├── Sunday 00:00:01 - Custody toggle switches
├── Sunday 18:00    - Weekly payout
└── Saturday 16:00  - Allowance warning

MONTHLY EVENTS:
├── 1st of month 00:00:01 - Interest calculation
└── Various - Monthly chores due (bathroom, sheets, etc.)
```

---

## Critical Integration Points

### 1. Chore → Required Sensor

```python
# KidsChores approves chore
def approve_chore(chore_id, kid_id):
    kid_info.chore_approvals_weekly[chore_id] += 1
    kid_info.points += chore.default_points

    # Triggers state change
    update_kid_sensor(kid_id)

# Custom sensor listens
def required_chores_sensor_state():
    completed = 0
    for chore_id, chore in chores.items():
        if "Required" in chore.labels:
            if chore_approvals_weekly[chore_id] > 0:
                completed += 1
    return completed
```

### 2. Required Sensor → Allowance Sensor

```jinja
{# Template sensor depends on Required sensor #}
{% set all_required_done =
    state_attr('sensor.kc_bella_required_chores_completed_weekly',
               'all_required_done') %}

{% set base = 10 if all_required_done else 0 %}
```

### 3. Allowance Sensor → Payout Automation

```yaml
variables:
  allowance: "{{ states('sensor.bella_weekly_allowance') | float }}"

action:
  - service: input_number.set_value
    data:
      value: "{{ current_balance + allowance }}"
```

### 4. Payout → Auto-Savings

```yaml
trigger:
  - platform: state
    entity_id: input_number.bella_checking

# Triggers when payout increases balance
```

---

## Testing Strategy Map

```
UNIT TESTS (Test individual components):
├── test_required_sensor_counts_correctly()
├── test_allowance_base_locked_when_required_incomplete()
├── test_allowance_base_unlocked_when_required_complete()
├── test_bonus_calculation()
└── test_custody_toggle_disables_chores()

INTEGRATION TESTS (Test component interactions):
├── test_chore_approval_updates_required_sensor()
├── test_required_sensor_updates_allowance()
├── test_payout_deposits_to_checking()
├── test_checking_increase_triggers_auto_save()
└── test_notifications_sent_when_expected()

END-TO-END TESTS (Test complete workflows):
├── test_weekly_cycle_complete_all_required()
├── test_weekly_cycle_missing_required()
├── test_lilly_custody_full_cycle()
└── test_goal_achievement_flow()

REGRESSION TESTS (Ensure features don't break):
├── test_existing_chores_not_modified()
├── test_parent_chores_stay_zero_points()
└── test_seasonal_chores_toggle_correctly()
```

---

## Error Handling Map

```
COMMON FAILURE POINTS:

1. Sensor unavailable
   ├── Cause: HA restart, integration error
   ├── Detection: state == "unavailable"
   └── Mitigation: Default to 0, notify admin

2. Allowance calculation error
   ├── Cause: Missing attribute, template error
   ├── Detection: state == "unknown"
   └── Mitigation: Log error, show $0.00

3. Payout automation fails
   ├── Cause: Sensor unavailable, bank entity missing
   ├── Detection: Automation error in logs
   └── Mitigation: Retry next week, manual payout

4. Chores disappear after reload
   ├── Cause: Integration overwrites storage
   ├── Detection: Chore count drops to 0
   └── Mitigation: Disable integration, re-import, enable

5. Custody toggle doesn't disable chores
   ├── Cause: Due dates not cleared
   ├── Detection: Chores still show as pending
   └── Mitigation: Manual clear via UI, fix automation
```

---

## Performance Considerations

```
SENSOR UPDATE FREQUENCY:

Fast (< 1s):
├── sensor.kc_bella_points (on chore approval)
├── sensor.kc_bella_required_chores_completed_weekly (on approval)
└── sensor.bella_weekly_allowance (template, depends on above)

Medium (1-5s):
├── Dashboard updates (UI rendering)
└── Automation triggers (depends on sensor updates)

Slow (5-60s):
├── Integration reload (loads all chores)
└── HA restart (reloads everything)

OPTIMIZATION TIPS:
├── Don't poll sensors unnecessarily
├── Use state triggers, not time_pattern
├── Batch multiple actions in one automation
└── Cache template results when possible
```

---

## Monitoring Dashboard

```yaml
# Admin view for monitoring system health

type: entities
title: 🔧 System Health
entities:
  # Integration Status
  - entity: binary_sensor.kidschores_integration_loaded
    name: KidsChores Integration

  # Sensor Status
  - entity: sensor.kc_bella_required_chores_completed_weekly
    name: Required Sensor (Bella)
    secondary_info: last-changed

  - entity: sensor.bella_weekly_allowance
    name: Allowance Sensor (Bella)
    secondary_info: last-changed

  # Automation Status
  - entity: automation.weekly_payout
    name: Payout Automation

  - entity: automation.custody_toggle_weekly
    name: Custody Toggle

  # Data Integrity
  - type: custom:template-entity-row
    entity: sensor.kidschores_chore_count
    name: Total Chores
    state: >
      {{ states.sensor | selectattr('entity_id', 'search', 'kidschores_chore_')
         | list | count }}
    secondary: Expected: 36

  - type: custom:template-entity-row
    entity: sensor.required_chores_count
    name: Required Chores
    state: >
      {{ state_attr('sensor.kc_bella_required_chores_completed_weekly',
                    'total_required') }}
    secondary: Expected: 20-30
```

---

## Feature Implementation Priority

```
CRITICAL PATH (Must work for system to function):
1. ✅ KidsChores Integration
2. ✅ Chore Import
3. ✅ Required Sensor
4. ✅ Allowance Calculation
5. ⏳ Basic Dashboard (kids can't use without this)
6. ⏳ Weekly Payout (no money flow otherwise)

HIGH PRIORITY (Greatly improves UX):
7. ⏳ Notifications (keeps kids engaged)
8. ⏳ Custody Toggle (Lilly needs this)
9. ⏳ Auto-Savings (teaches financial habits)

MEDIUM PRIORITY (Nice to have):
10. ⏳ Goal Tracking (motivational)
11. ⏳ Parent Dashboard (oversight)
12. ⏳ Monthly Interest (rewards saving)

LOW PRIORITY (Future enhancements):
13. ⏳ Advanced Dashboards (polish)
14. ⏳ Achievement System (gamification)
15. ⏳ Historical Tracking (analytics)
```

---

## Quick Reference: Entity Names

```
SENSORS (read-only):
sensor.kc_bella_points
sensor.kc_bella_chores_completed_weekly
sensor.kc_bella_required_chores_completed_weekly
sensor.bella_weekly_allowance
sensor.bella_goal_progress

HELPERS (read-write):
input_boolean.lilly_home_this_week
input_number.bella_checking
input_number.bella_savings
input_number.bella_goal_amount
input_text.bella_goal_name

AUTOMATIONS:
automation.custody_toggle_weekly
automation.manage_lilly_chores
automation.weekly_payout
automation.auto_savings_transfer
automation.saturday_allowance_warning
automation.daily_chore_reminder

SERVICES:
kidschores.claim_chore
kidschores.approve_chore
kidschores.set_chore_due_date
kidschores.skip_chore_due_date
```

---

**This architecture map provides:**
- Visual system overview
- Data flow understanding
- Feature dependencies
- Integration points
- Testing strategy
- Error handling
- Performance tips
- Monitoring setup

Use this alongside FEATURE_IMPLEMENTATION_BLUEPRINTS.md for complete implementation guide! 🎯
