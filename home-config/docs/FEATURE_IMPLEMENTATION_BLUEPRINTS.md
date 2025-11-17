# Feature Implementation Blueprints
**For Local Claude Integration**

Comprehensive pseudo code, flowcharts, and mock tests for remaining features.

---

## Table of Contents
1. [Custody Toggle System](#1-custody-toggle-system)
2. [Weekly Payout Automation](#2-weekly-payout-automation)
3. [Notification System](#3-notification-system)
4. [Dashboard Definitions](#4-dashboard-definitions)
5. [Banking System](#5-banking-system)
6. [Goal Tracking](#6-goal-tracking)
7. [Bathroom Rotation](#7-bathroom-rotation)
8. [Integration Test Suite](#8-integration-test-suite)

---

## 1. Custody Toggle System

### Purpose
Lilly is gone every other week. Disable her chores during away weeks, re-enable when home.

### Entities Required

```yaml
# Helper Entity
input_boolean:
  lilly_home_this_week:
    name: "Lilly Home This Week"
    icon: mdi:home-account
    initial: true  # Start as home
```

### Flow Chart

```
┌─────────────────────────────────────────────┐
│         Sunday 00:00:01 Trigger             │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│    Toggle input_boolean.lilly_home_this_week│
└────────────────┬────────────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
┌──────────────┐  ┌──────────────┐
│  Home (ON)   │  │  Away (OFF)  │
└──────┬───────┘  └──────┬───────┘
       │                 │
       ▼                 ▼
┌─────────────┐   ┌──────────────┐
│Enable Chores│   │Disable Chores│
└─────────────┘   └──────────────┘
       │                 │
       ▼                 ▼
┌──────────────────────────────────┐
│For each Lilly chore:             │
│  - Get chore frequency           │
│  - Calculate next due date       │
│  - Set/Clear due date            │
└──────────────────────────────────┘
```

### Pseudo Code

```python
# automation.yaml
automation_custody_toggle_weekly:
  trigger:
    - platform: time
      at: "00:00:01"
    - condition: time
      weekday: [sun]

  action:
    # Toggle Lilly's custody status
    service: input_boolean.toggle
    target:
      entity_id: input_boolean.lilly_home_this_week

automation_manage_lilly_chores:
  trigger:
    - platform: state
      entity_id: input_boolean.lilly_home_this_week

  variables:
    lilly_chores:
      - "Laundry - Lilly"
      - "Scoop Litter - Lilly"
      - "Dog Out After School - Lilly"
      - "Room Reset - Lilly"
      - "Room Trash Check - Lilly"
      - "Towels - Lilly"
      - "Bed Sheets Change - Lilly"

  action:
    - choose:
        # CASE 1: Lilly is HOME - Enable chores
        - conditions:
            - condition: state
              entity_id: input_boolean.lilly_home_this_week
              state: 'on'
          sequence:
            - repeat:
                for_each: "{{ lilly_chores }}"
                sequence:
                  # Get chore info to calculate next due date
                  - service: kidschores.skip_chore_due_date
                    data:
                      chore_name: "{{ repeat.item }}"
                  # This advances to next occurrence based on frequency

        # CASE 2: Lilly is AWAY - Disable chores
        - conditions:
            - condition: state
              entity_id: input_boolean.lilly_home_this_week
              state: 'off'
          sequence:
            - repeat:
                for_each: "{{ lilly_chores }}"
                sequence:
                  # Clear due date = chore won't recur
                  - service: kidschores.set_chore_due_date
                    data:
                      chore_name: "{{ repeat.item }}"
                      due_date: null
```

### Mock Test

```yaml
# Test Case 1: Toggle to Away
GIVEN:
  - Lilly has 7 active chores with due dates
  - input_boolean.lilly_home_this_week = ON

WHEN:
  - Toggle switches to OFF (away)

THEN:
  - All 7 Lilly chores have due_date = null
  - Chores don't appear in "pending" lists
  - Required chores sensor still counts them (but as incomplete)

VERIFY:
  assert states('sensor.kc_lilly_required_chores_completed_weekly').attributes.total_required == 7
  assert all(chore.due_date is None for chore in lilly_chores)

# Test Case 2: Toggle to Home
GIVEN:
  - Lilly's chores have due_date = null (disabled)
  - input_boolean.lilly_home_this_week = OFF

WHEN:
  - Toggle switches to ON (home)
  - skip_chore_due_date service called for each chore

THEN:
  - All 7 chores have valid due dates (next occurrence)
  - Daily chores due: tomorrow at their scheduled time
  - Weekly chores due: next applicable day

VERIFY:
  assert all(chore.due_date is not None for chore in lilly_chores)
  assert chore["Scoop Litter - Lilly"].due_date == next_tuesday_at_17:00
```

### Edge Cases

```python
# Edge Case 1: Toggle mid-week
# Problem: If Lilly comes home Wednesday, chores shouldn't all pile up at once
# Solution: Use skip_chore_due_date which calculates NEXT occurrence, not past ones

# Edge Case 2: Completed chore before leaving
# If Lilly completes "Laundry" Sunday morning, then leaves Sunday evening:
#   - Chore is marked complete
#   - Due date cleared when toggle switches
#   - When she returns, skip_chore_due_date sets it to next Tuesday (correct)

# Edge Case 3: Seasonal chores
# Water plants is seasonal AND assigned to both kids
# When Lilly is away, only clear HER instance, not Bella's
# Solution: Chores are per-kid, so this works automatically
```

---

## 2. Weekly Payout Automation

### Purpose
Every Sunday at 6pm:
- Transfer allowance to checking account
- Show notification of payment
- (Optional) Reset points for new week

### Entities Required

```yaml
# Banking Entities
input_number:
  bella_checking:
    name: "Bella Checking Account"
    min: 0
    max: 10000
    step: 0.01
    unit_of_measurement: "$"
    icon: mdi:cash
    mode: box

  bella_savings:
    name: "Bella Savings Account"
    min: 0
    max: 10000
    step: 0.01
    unit_of_measurement: "$"
    icon: mdi:piggy-bank
    mode: box

  lilly_checking:
    name: "Lilly Checking Account"
    # Same config as Bella

  lilly_savings:
    name: "Lilly Savings Account"
    # Same config as Bella

# Transaction History (optional)
input_text:
  last_payout_bella:
    name: "Bella Last Payout"
    max: 255
    initial: "Never"

  last_payout_lilly:
    name: "Lilly Last Payout"
    max: 255
    initial: "Never"
```

### Flow Chart

```
┌──────────────────────────────┐
│  Sunday 18:00:00 Trigger     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Get Bella's weekly allowance │
│ allowance = sensor.bella_    │
│   weekly_allowance.state     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Add to checking account:     │
│ bella_checking =             │
│   current + allowance        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Log transaction:             │
│ "Paid $17.50 on 2025-01-19"  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Send notification to Bella   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Repeat for Lilly             │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ (Optional) Reset points      │
│ Note: KidsChores does this   │
│ automatically Sunday 00:00   │
└──────────────────────────────┘
```

### Pseudo Code

```python
automation_weekly_payout:
  trigger:
    - platform: time
      at: "18:00:00"
    - condition: time
      weekday: [sun]

  action:
    # Pay Bella
    - variables:
        bella_allowance: "{{ states('sensor.bella_weekly_allowance') | float(0) }}"
        bella_current: "{{ states('input_number.bella_checking') | float(0) }}"
        bella_new_balance: "{{ bella_current + bella_allowance }}"

    - service: input_number.set_value
      target:
        entity_id: input_number.bella_checking
      data:
        value: "{{ bella_new_balance }}"

    - service: input_text.set_value
      target:
        entity_id: input_text.last_payout_bella
      data:
        value: >
          Paid ${{ bella_allowance }} on {{ now().strftime('%Y-%m-%d') }}
          New balance: ${{ bella_new_balance }}

    - service: notify.mobile_app_bellas_phone
      data:
        title: "💰 Allowance Paid!"
        message: >
          Your weekly allowance of ${{ bella_allowance }} has been deposited!
          New balance: ${{ bella_new_balance }}
        data:
          tag: "weekly_payout"
          group: "allowance"

    # Pay Lilly (same logic)
    - variables:
        lilly_allowance: "{{ states('sensor.lilly_weekly_allowance') | float(0) }}"
        lilly_current: "{{ states('input_number.lilly_checking') | float(0) }}"
        lilly_new_balance: "{{ lilly_current + lilly_allowance }}"

    - service: input_number.set_value
      target:
        entity_id: input_number.lilly_checking
      data:
        value: "{{ lilly_new_balance }}"

    # ... (repeat notification for Lilly)
```

### Mock Test

```yaml
# Test Case 1: Normal Payout
GIVEN:
  - Bella completed all Required chores (30 points)
  - sensor.bella_weekly_allowance = 17.50 ($10 base + $7.50 bonus)
  - input_number.bella_checking = 25.00

WHEN:
  - Sunday 18:00 automation triggers

THEN:
  - input_number.bella_checking = 42.50 (25.00 + 17.50)
  - input_text.last_payout_bella = "Paid $17.50 on 2025-01-19. New balance: $42.50"
  - Notification sent to Bella's phone

VERIFY:
  assert states('input_number.bella_checking') == 42.50
  assert '$17.50' in states('input_text.last_payout_bella')

# Test Case 2: Base Locked (Missing Required)
GIVEN:
  - Lilly completed 18/20 Required chores (27 points)
  - sensor.lilly_weekly_allowance = 6.75 ($0 base + $6.75 bonus)
  - input_number.lilly_checking = 10.00

WHEN:
  - Sunday 18:00 automation triggers

THEN:
  - input_number.lilly_checking = 16.75 (10.00 + 6.75)
  - Lilly gets ONLY bonus money (base was locked)

VERIFY:
  assert states('input_number.lilly_checking') == 16.75
  assert notification.message contains "$6.75" (not $10+)

# Test Case 3: Lilly Away This Week
GIVEN:
  - input_boolean.lilly_home_this_week = OFF
  - Lilly completed 0 chores (chores were disabled)
  - sensor.lilly_weekly_allowance = 0.00

WHEN:
  - Sunday 18:00 automation triggers

THEN:
  - input_number.lilly_checking unchanged
  - Notification: "No allowance this week (away)"

VERIFY:
  assert states('input_number.lilly_checking') == previous_value
```

### Edge Cases

```python
# Edge Case 1: Allowance sensor unavailable
if allowance == "unavailable":
    # Don't pay, send error notification
    notify("Error: Allowance sensor not working")
    return

# Edge Case 2: Checking account at max
if current_balance + allowance > 10000:
    # Overflow to savings
    overflow = (current_balance + allowance) - 10000
    bella_checking = 10000
    bella_savings += overflow
    notify("Checking full! $" + overflow + " moved to savings")

# Edge Case 3: Negative balance (shouldn't happen, but...)
if current_balance < 0:
    # Log error, fix manually
    logger.error("Negative balance detected!")
    current_balance = 0
```

---

## 3. Notification System

### Purpose
Keep kids engaged and aware:
- Daily chore reminders
- Saturday warning if base allowance at risk
- Achievement notifications

### Flow Chart: Base Allowance Warning

```
┌───────────────────────────────┐
│ Saturday 16:00 Trigger        │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│ Check: base_unlocked == True? │
└───────┬───────────────────────┘
        │
   ┌────┴────┐
   ▼         ▼
┌─────┐   ┌─────────────────────┐
│ YES │   │ NO - Send Warning   │
│Skip │   └──────────┬──────────┘
└─────┘              │
                     ▼
          ┌──────────────────────┐
          │ Calculate missing:   │
          │ total - completed    │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ Send urgent message: │
          │ "Only X hours left!" │
          │ "Complete Y chores"  │
          └──────────────────────┘
```

### Pseudo Code

```python
automation_saturday_allowance_warning:
  trigger:
    - platform: time
      at: "16:00:00"
    - condition: time
      weekday: [sat]

  action:
    # Check Bella
    - choose:
        - conditions:
            - condition: template
              value_template: >
                {{ not state_attr('sensor.bella_weekly_allowance', 'base_unlocked') }}
          sequence:
            - variables:
                total: >
                  {{ state_attr('sensor.kc_bella_required_chores_completed_weekly', 'total_required') }}
                completed: >
                  {{ states('sensor.kc_bella_required_chores_completed_weekly') | int }}
                missing: "{{ total - completed }}"
                hours_left: 26  # Until Sunday 6pm payout

            - service: notify.mobile_app_bellas_phone
              data:
                title: "⚠️ Allowance Alert!"
                message: >
                  You still need {{ missing }} Required chores to unlock your $10 base!
                  Only {{ hours_left }} hours until payout!

                  Missing chores earn you ONLY ${{ (completed * 0.25) | round(2) }}
                  Complete all Required = ${{ (total * 0.25 + 10) | round(2) }}
                data:
                  actions:
                    - action: "VIEW_CHORES"
                      title: "View Chores"
                    - action: "VIEW_ALLOWANCE"
                      title: "Check Progress"
                  tag: "allowance_warning"
                  group: "reminders"
                  priority: high

    # Repeat for Lilly (if home)
    - choose:
        - conditions:
            - condition: state
              entity_id: input_boolean.lilly_home_this_week
              state: 'on'
            - condition: template
              value_template: >
                {{ not state_attr('sensor.lilly_weekly_allowance', 'base_unlocked') }}
          sequence:
            # Same logic as Bella

automation_daily_chore_reminder:
  trigger:
    - platform: time
      at: "15:00:00"  # After school

  action:
    # Bella's reminder
    - variables:
        bella_pending: >
          {{ states('select.kc_bella_chore_list') }}
        bella_count: >
          {{ bella_pending.split(',') | count if bella_pending != 'None' else 0 }}

    - condition: template
      value_template: "{{ bella_count > 0 }}"

    - service: notify.mobile_app_bellas_phone
      data:
        title: "📋 Daily Reminder"
        message: "You have {{ bella_count }} chores due today!"
        data:
          tag: "daily_reminder"

automation_achievement_unlock:
  trigger:
    - platform: template
      value_template: >
        {{ state_attr('sensor.bella_weekly_allowance', 'base_unlocked') }}
      for:
        seconds: 5  # Debounce

  condition:
    - condition: time
      weekday: [mon, tue, wed, thu, fri, sat]  # Not Sunday (that's payout)

  action:
    - service: notify.mobile_app_bellas_phone
      data:
        title: "🎉 Achievement Unlocked!"
        message: >
          All Required chores complete! Your $10 base is unlocked!
          Keep earning bonus points for more money!
        data:
          tag: "achievement"
```

### Mock Test

```yaml
# Test Case 1: Saturday Warning - Base at Risk
GIVEN:
  - Saturday 16:00
  - Bella completed 17/20 Required chores
  - base_unlocked = False

WHEN:
  - Automation triggers

THEN:
  - Notification sent with:
    - Title: "⚠️ Allowance Alert!"
    - Missing count: 3
    - Hours left: 26
    - Current earnings: $4.25 (17 × $0.25)
    - Potential earnings: $15.00 (20 × $0.25 + $10)
  - Actions: VIEW_CHORES, VIEW_ALLOWANCE
  - Priority: HIGH

VERIFY:
  assert notification.title == "⚠️ Allowance Alert!"
  assert "3" in notification.message
  assert "$4.25" in notification.message

# Test Case 2: No Warning - Base Already Unlocked
GIVEN:
  - Saturday 16:00
  - Bella completed 20/20 Required chores
  - base_unlocked = True

WHEN:
  - Automation triggers

THEN:
  - No notification sent (condition fails)

VERIFY:
  assert notification_count == 0

# Test Case 3: Lilly Away - No Notification
GIVEN:
  - Saturday 16:00
  - input_boolean.lilly_home_this_week = OFF

WHEN:
  - Automation triggers

THEN:
  - Lilly receives no notification (condition fails)

VERIFY:
  assert lilly_notifications == 0
```

---

## 4. Dashboard Definitions

### Kid Dashboard (Bella's View)

```yaml
# Mock Structure - Lovelace YAML

title: Bella's Chores
type: vertical-stack
cards:
  # Card 1: Allowance Summary
  - type: entities
    title: 💰 This Week's Earnings
    entities:
      - entity: sensor.bella_weekly_allowance
        name: Total Allowance
        icon: mdi:cash-multiple
        secondary_info: last-changed

      - type: custom:bar-card
        entity: sensor.kc_bella_required_chores_completed_weekly
        name: Required Chores Progress
        max: >
          {{ state_attr('sensor.kc_bella_required_chores_completed_weekly', 'total_required') }}
        positions:
          indicator: inside
          name: inside
        color: >
          {% if state_attr('sensor.bella_weekly_allowance', 'base_unlocked') %}
            green
          {% else %}
            orange
          {% endif %}

      - type: attribute
        entity: sensor.bella_weekly_allowance
        attribute: base_unlocked
        name: "$10 Base Unlocked?"
        icon: mdi:lock-open-variant

      - type: attribute
        entity: sensor.bella_weekly_allowance
        attribute: bonus_earned
        name: Bonus Earned
        icon: mdi:star

  # Card 2: Pending Chores
  - type: custom:auto-entities
    card:
      type: entities
      title: 📋 Pending Chores
    filter:
      include:
        - entity_id: "sensor.kidschores_chore_*bella*"
          state: "pending"
      sort:
        method: attribute
        attribute: due_date

  # Card 3: Quick Actions
  - type: horizontal-stack
    cards:
      - type: button
        name: Claim Chore
        icon: mdi:hand-back-right
        tap_action:
          action: navigate
          navigation_path: /kc-claim-chore

      - type: button
        name: View Calendar
        icon: mdi:calendar
        tap_action:
          action: more-info
          entity: calendar.kc_bella
```

### Parent Dashboard (Oversight)

```yaml
title: Parent Chore Oversight
type: vertical-stack
cards:
  # Card 1: Allowance Overview
  - type: glance
    title: 💰 Allowance Status
    columns: 2
    entities:
      - entity: sensor.bella_weekly_allowance
        name: Bella
      - entity: sensor.lilly_weekly_allowance
        name: Lilly

  # Card 2: Required Chores Completion
  - type: entities
    title: 📊 Required Chores Progress
    entities:
      - type: custom:bar-card
        entity: sensor.kc_bella_required_chores_completed_weekly
        name: Bella
        max: >
          {{ state_attr('sensor.kc_bella_required_chores_completed_weekly', 'total_required') }}
        severity:
          - value: 100
            color: green
          - value: 90
            color: yellow
          - value: 0
            color: red

      - type: custom:bar-card
        entity: sensor.kc_lilly_required_chores_completed_weekly
        name: Lilly
        max: >
          {{ state_attr('sensor.kc_lilly_required_chores_completed_weekly', 'total_required') }}

  # Card 3: Pending Approvals
  - type: entities
    title: ⏳ Pending Approvals
    entities:
      - entity: sensor.kc_global_chore_pending_approvals
        name: Chores Waiting

      - type: custom:auto-entities
        card:
          type: entities
        filter:
          include:
            - entity_id: "sensor.kidschores_chore_*"
              state: "claimed"
          sort:
            method: last_changed

  # Card 4: Banking Balances
  - type: glance
    title: 🏦 Account Balances
    columns: 4
    entities:
      - entity: input_number.bella_checking
        name: Bella Checking
      - entity: input_number.bella_savings
        name: Bella Savings
      - entity: input_number.lilly_checking
        name: Lilly Checking
      - entity: input_number.lilly_savings
        name: Lilly Savings
```

### Pseudo Code: Dashboard Data Flow

```python
# Dashboard updates based on state changes

on sensor.bella_weekly_allowance change:
  update allowance_card.value
  update allowance_card.color (green if base_unlocked else orange)
  update progress_bar.percentage

  if base_unlocked changed from False to True:
    show_achievement_banner("$10 Base Unlocked!")

on sensor.kc_bella_required_chores_completed_weekly change:
  update progress_bar.current_value
  calculate completion_percentage

  if completion_percentage >= 100:
    change_color_to_green()
    show_celebration_animation()
  elif completion_percentage >= 90:
    change_color_to_yellow()
  else:
    change_color_to_red()

on sensor.kidschores_chore_* state change to "claimed":
  add to pending_approvals_list
  increment pending_count_badge
  send notification to parent
```

---

## 5. Banking System

### Purpose
Teach financial responsibility:
- Checking account (liquid, for spending)
- Savings account (locked, earns interest)
- Automatic savings transfer (20%)
- Monthly interest (3% on savings)

### Flow Chart: Auto-Transfer to Savings

```
┌─────────────────────────────────┐
│ Allowance Deposited to Checking │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│ Calculate 20% = savings_amount  │
│ Calculate 80% = checking_keep   │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│ Transfer savings_amount to      │
│ input_number.bella_savings      │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│ Update checking to checking_keep│
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│ Notify:                         │
│ "$3.50 auto-saved! (20%)"       │
└─────────────────────────────────┘
```

### Pseudo Code

```python
automation_auto_savings_transfer:
  # Triggers when checking account increases (allowance deposit)
  trigger:
    - platform: state
      entity_id: input_number.bella_checking

  condition:
    # Only when value INCREASES (not decreases from spending)
    - condition: template
      value_template: >
        {{ trigger.to_state.state | float > trigger.from_state.state | float }}

  variables:
    deposit_amount: >
      {{ trigger.to_state.state | float - trigger.from_state.state | float }}
    savings_transfer: "{{ deposit_amount * 0.20 }}"
    checking_keep: "{{ deposit_amount * 0.80 }}"
    current_checking: "{{ trigger.from_state.state | float }}"
    current_savings: "{{ states('input_number.bella_savings') | float }}"

  action:
    # Transfer 20% to savings
    - service: input_number.set_value
      target:
        entity_id: input_number.bella_savings
      data:
        value: "{{ current_savings + savings_transfer }}"

    # Keep 80% in checking
    - service: input_number.set_value
      target:
        entity_id: input_number.bella_checking
      data:
        value: "{{ current_checking + checking_keep }}"

    # Notify
    - service: notify.mobile_app_bellas_phone
      data:
        title: "💰 Auto-Saved!"
        message: >
          ${{ savings_transfer | round(2) }} transferred to savings (20%)
          Checking: ${{ current_checking + checking_keep | round(2) }}
          Savings: ${{ current_savings + savings_transfer | round(2) }}

automation_monthly_interest:
  trigger:
    - platform: time
      at: "00:00:01"
    - condition: template
      value_template: "{{ now().day == 1 }}"  # First of month

  variables:
    bella_savings: "{{ states('input_number.bella_savings') | float }}"
    interest_rate: 0.03  # 3% monthly
    interest_earned: "{{ bella_savings * interest_rate }}"

  action:
    - service: input_number.set_value
      target:
        entity_id: input_number.bella_savings
      data:
        value: "{{ bella_savings + interest_earned }}"

    - service: notify.mobile_app_bellas_phone
      data:
        title: "💸 Interest Earned!"
        message: >
          Your savings earned ${{ interest_earned | round(2) }} this month! (3%)
          New balance: ${{ (bella_savings + interest_earned) | round(2) }}

script_withdraw_from_savings:
  # Manual withdrawal (requires parent approval)
  fields:
    kid:
      description: "Which kid?"
      example: "bella"
    amount:
      description: "How much?"
      example: 25.00

  sequence:
    - variables:
        savings_entity: "input_number.{{ kid }}_savings"
        checking_entity: "input_number.{{ kid }}_checking"
        current_savings: "{{ states(savings_entity) | float }}"
        current_checking: "{{ states(checking_entity) | float }}"

    # Check sufficient funds
    - condition: template
      value_template: "{{ current_savings >= amount }}"

    # Deduct from savings
    - service: input_number.set_value
      target:
        entity_id: "{{ savings_entity }}"
      data:
        value: "{{ current_savings - amount }}"

    # Add to checking
    - service: input_number.set_value
      target:
        entity_id: "{{ checking_entity }}"
      data:
        value: "{{ current_checking + amount }}"

    # Log transaction
    - service: persistent_notification.create
      data:
        title: "Withdrawal: {{ kid | title }}"
        message: >
          Withdrew ${{ amount }} from savings → checking
          New balances:
            Savings: ${{ current_savings - amount }}
            Checking: ${{ current_checking + amount }}
```

### Mock Test

```yaml
# Test Case 1: Weekly Allowance Auto-Save
GIVEN:
  - Bella checking = $10.00
  - Bella savings = $50.00
  - Weekly allowance = $17.50

WHEN:
  - Payout automation deposits $17.50 to checking
  - Checking balance changes: $10.00 → $27.50

THEN:
  - Auto-save triggers
  - Savings transfer = $3.50 (20% of $17.50)
  - Checking keep = $14.00 (80% of $17.50)
  - Final balances:
    - Checking = $24.00 ($10 + $14)
    - Savings = $53.50 ($50 + $3.50)

VERIFY:
  assert states('input_number.bella_checking') == 24.00
  assert states('input_number.bella_savings') == 53.50
  assert notification.message contains "$3.50 transferred to savings"

# Test Case 2: Monthly Interest
GIVEN:
  - Bella savings = $100.00
  - First day of month

WHEN:
  - Automation triggers at midnight

THEN:
  - Interest = $3.00 (3% of $100)
  - New savings = $103.00

VERIFY:
  assert states('input_number.bella_savings') == 103.00
  assert notification.message contains "$3.00 this month"

# Test Case 3: Withdrawal
GIVEN:
  - Bella savings = $75.00
  - Bella checking = $10.00

WHEN:
  - Parent approves withdrawal of $25.00

THEN:
  - Savings = $50.00
  - Checking = $35.00
  - Transaction logged

VERIFY:
  assert states('input_number.bella_savings') == 50.00
  assert states('input_number.bella_checking') == 35.00
```

---

## 6. Goal Tracking

### Purpose
Visualize savings goals and progress.

### Entities Required

```yaml
input_number:
  bella_goal_amount:
    name: "Bella Savings Goal"
    min: 0
    max: 1000
    step: 5
    unit_of_measurement: "$"
    mode: box

input_text:
  bella_goal_name:
    name: "What Bella is Saving For"
    max: 100
    initial: "Not set"

input_datetime:
  bella_goal_deadline:
    name: "Goal Deadline"
    has_date: true
    has_time: false
```

### Pseudo Code

```python
template_sensor_goal_progress:
  - sensor:
      - name: "Bella Goal Progress"
        unique_id: bella_goal_progress
        unit_of_measurement: "%"
        icon: mdi:flag-checkered
        state: >
          {% set current = states('input_number.bella_savings') | float(0) %}
          {% set goal = states('input_number.bella_goal_amount') | float(1) %}
          {{ (current / goal * 100) | round(1) if goal > 0 else 0 }}
        attributes:
          current_saved: >
            {{ states('input_number.bella_savings') | float(0) }}
          goal_amount: >
            {{ states('input_number.bella_goal_amount') | float(0) }}
          goal_name: >
            {{ states('input_text.bella_goal_name') }}
          amount_needed: >
            {{ states('input_number.bella_goal_amount') | float(0) -
               states('input_number.bella_savings') | float(0) }}
          weeks_to_goal: >
            {% set needed = states('input_number.bella_goal_amount') | float(0) -
                            states('input_number.bella_savings') | float(0) %}
            {% set weekly = states('sensor.bella_weekly_allowance') | float(1) %}
            {{ (needed / weekly) | round(0) if weekly > 0 else 999 }}

automation_goal_achieved:
  trigger:
    - platform: template
      value_template: >
        {{ states('input_number.bella_savings') | float >=
           states('input_number.bella_goal_amount') | float }}

  condition:
    - condition: template
      value_template: >
        {{ states('input_number.bella_goal_amount') | float > 0 }}

  action:
    - service: notify.mobile_app_bellas_phone
      data:
        title: "🎉 GOAL ACHIEVED!"
        message: >
          You saved ${{ states('input_number.bella_goal_amount') }}
          for {{ states('input_text.bella_goal_name') }}!

          Congratulations on your hard work! 🏆
        data:
          tag: "goal_achieved"
          priority: high

    # Reset goal
    - service: input_number.set_value
      target:
        entity_id: input_number.bella_goal_amount
      data:
        value: 0
```

### Dashboard Card

```yaml
type: vertical-stack
cards:
  - type: entities
    title: 🎯 Savings Goal
    entities:
      - entity: input_text.bella_goal_name
        name: Goal

      - entity: input_number.bella_goal_amount
        name: Target Amount

      - type: custom:bar-card
        entity: sensor.bella_goal_progress
        name: Progress
        max: 100
        severity:
          - value: 100
            color: green
          - value: 75
            color: yellow
          - value: 0
            color: red

      - type: attribute
        entity: sensor.bella_goal_progress
        attribute: amount_needed
        name: Still Need
        icon: mdi:cash-minus

      - type: attribute
        entity: sensor.bella_goal_progress
        attribute: weeks_to_goal
        name: Weeks to Goal
        icon: mdi:calendar-clock
```

---

## 7. Bathroom Rotation

### Solution: Two Separate Chores

**Why not automation?** Simpler to manage, less error-prone.

```yaml
# In chores_bella_lilly.yaml

# Week A chores (Bella)
- name: "Bathroom Sink Clean - Week A"
  assigned_kids: ["Bella"]
  points: 3
  frequency: biweekly
  due_time: "18:00"
  applicable_days: ["thu"]
  description: "Clean faucet, handles, and sink bowl"
  icon: "mdi:faucet"
  labels: ["Required", "Bathroom"]

- name: "Toilet Scrub - Week A"
  assigned_kids: ["Bella"]
  points: 4
  frequency: biweekly
  due_time: "18:00"
  applicable_days: ["thu"]
  description: "Scrub toilet bowl and seat"
  icon: "mdi:toilet"
  labels: ["Required", "Bathroom"]

# Week B chores (Lilly)
- name: "Bathroom Sink Clean - Week B"
  assigned_kids: ["Lilly"]
  points: 3
  frequency: biweekly
  due_time: "18:00"
  applicable_days: ["thu"]
  description: "Clean faucet, handles, and sink bowl"
  icon: "mdi:faucet"
  labels: ["Required", "Bathroom"]

- name: "Toilet Scrub - Week B"
  assigned_kids: ["Lilly"]
  points: 4
  frequency: biweekly
  due_time: "18:00"
  applicable_days: ["thu"]
  description: "Scrub toilet bowl and seat"
  icon: "mdi:toilet"
  labels: ["Required", "Bathroom"]
```

**Offset setup:**
- Week A chores: Start on Thursday, Jan 16
- Week B chores: Start on Thursday, Jan 23
- They alternate automatically (biweekly = every 2 weeks)

**No automation needed!** KidsChores handles the biweekly recurrence.

---

## 8. Integration Test Suite

### Test Suite Structure

```python
# tests/test_allowance_system.py

class TestAllowanceCalculation:
    def test_base_unlocked_all_required_complete():
        """Test base $10 unlocks when all Required chores done"""
        # Setup
        create_kid("Bella")
        assign_chores(["Chore1", "Chore2"], labels=["Required"])

        # Execute
        approve_chore("Chore1", "Bella")
        approve_chore("Chore2", "Bella")

        # Verify
        assert sensor.kc_bella_required_chores_completed_weekly.state == 2
        assert sensor.kc_bella_required_chores_completed_weekly.attributes.all_required_done == True
        assert sensor.bella_weekly_allowance.attributes.base_allowance == 10

    def test_base_locked_missing_one_required():
        """Test base $10 locked if missing ANY Required chore"""
        # Setup
        create_kid("Bella")
        assign_chores(["Chore1", "Chore2"], labels=["Required"])

        # Execute - complete only 1 of 2
        approve_chore("Chore1", "Bella")

        # Verify
        assert sensor.kc_bella_required_chores_completed_weekly.state == 1
        assert sensor.kc_bella_required_chores_completed_weekly.attributes.all_required_done == False
        assert sensor.bella_weekly_allowance.attributes.base_allowance == 0

    def test_bonus_calculation():
        """Test bonus = points × $0.25"""
        # Setup
        create_kid("Bella")
        assign_chores([
            {"name": "Chore1", "points": 3, "labels": ["Required"]},
            {"name": "Chore2", "points": 5, "labels": ["Bonus"]}
        ])

        # Execute
        approve_chore("Chore1", "Bella")  # 3 points
        approve_chore("Chore2", "Bella")  # 5 points

        # Verify
        assert sensor.kc_bella_points.state == 8
        assert sensor.bella_weekly_allowance.attributes.bonus_earned == 2.00  # 8 × $0.25

class TestCustodyToggle:
    def test_disable_chores_when_away():
        """Test Lilly's chores disabled when toggle OFF"""
        # Setup
        create_kid("Lilly")
        chores = assign_chores(["Laundry", "Litter"], kid="Lilly")
        set_state("input_boolean.lilly_home_this_week", "on")

        # Execute
        set_state("input_boolean.lilly_home_this_week", "off")
        wait_for_automation()

        # Verify
        for chore in chores:
            assert chore.due_date == None

    def test_enable_chores_when_home():
        """Test Lilly's chores re-enabled when toggle ON"""
        # Setup
        create_kid("Lilly")
        chores = assign_chores(["Laundry"], kid="Lilly", frequency="weekly", day="tue")
        set_state("input_boolean.lilly_home_this_week", "off")
        wait_for_automation()

        # Execute
        set_state("input_boolean.lilly_home_this_week", "on")
        wait_for_automation()

        # Verify
        assert chores[0].due_date != None
        assert chores[0].due_date.weekday() == 1  # Tuesday

class TestWeeklyPayout:
    def test_allowance_deposited_to_checking():
        """Test weekly payout adds allowance to checking"""
        # Setup
        create_kid("Bella")
        set_allowance("Bella", 17.50)
        set_account("bella_checking", 10.00)

        # Execute
        trigger_time("sunday", "18:00")
        wait_for_automation()

        # Verify
        assert states("input_number.bella_checking") == 27.50

    def test_auto_save_20_percent():
        """Test 20% auto-transferred to savings"""
        # Setup
        create_kid("Bella")
        set_allowance("Bella", 20.00)
        set_account("bella_checking", 0)
        set_account("bella_savings", 0)

        # Execute
        trigger_payout()
        wait_for_automation()

        # Verify
        assert states("input_number.bella_checking") == 16.00  # 80%
        assert states("input_number.bella_savings") == 4.00    # 20%

class TestNotifications:
    def test_saturday_warning_if_base_locked():
        """Test warning notification on Saturday if base not unlocked"""
        # Setup
        create_kid("Bella")
        assign_required_chores(total=20)
        complete_chores(count=18)  # 18/20 = base locked

        # Execute
        trigger_time("saturday", "16:00")

        # Verify
        assert notification_sent("Bella", title="⚠️ Allowance Alert!")
        assert "2" in notification.message  # Missing 2 chores

    def test_no_warning_if_base_unlocked():
        """Test NO warning if base already unlocked"""
        # Setup
        create_kid("Bella")
        assign_required_chores(total=20)
        complete_chores(count=20)  # All done

        # Execute
        trigger_time("saturday", "16:00")

        # Verify
        assert not notification_sent("Bella")
```

### Mock Test Runner

```python
# Pseudo code for test execution

def run_integration_tests():
    results = {
        "passed": 0,
        "failed": 0,
        "errors": []
    }

    for test_class in [TestAllowanceCalculation, TestCustodyToggle, TestWeeklyPayout, TestNotifications]:
        for test_method in test_class.methods:
            try:
                # Setup test environment
                setup_test_environment()

                # Run test
                test_method()

                # Cleanup
                teardown_test_environment()

                results["passed"] += 1
                print(f"✅ {test_method.__name__}")

            except AssertionError as e:
                results["failed"] += 1
                results["errors"].append({
                    "test": test_method.__name__,
                    "error": str(e)
                })
                print(f"❌ {test_method.__name__}: {e}")

            except Exception as e:
                results["failed"] += 1
                results["errors"].append({
                    "test": test_method.__name__,
                    "error": f"Unexpected error: {e}"
                })
                print(f"💥 {test_method.__name__}: {e}")

    # Summary
    print(f"\n{'='*50}")
    print(f"Tests Passed: {results['passed']}")
    print(f"Tests Failed: {results['failed']}")
    print(f"Success Rate: {results['passed'] / (results['passed'] + results['failed']) * 100:.1f}%")

    return results
```

---

## Summary: Feature Dependencies

```
Foundation (DONE):
├── KidsChores Integration ✅
├── RequiredChoresCompletedWeeklySensor ✅
├── 36 Chores Imported ✅
└── Allowance Template Sensors ✅

Phase 1 (Week 1):
├── Custody Toggle ⬅️ Start here
│   └── Depends on: input_boolean helper
├── Basic Dashboards
│   └── Depends on: Lovelace YAML skills
└── Weekly Payout
    └── Depends on: input_number banking entities

Phase 2 (Week 2-3):
├── Notifications
│   └── Depends on: Mobile app setup
├── Auto-Savings (20% rule)
│   └── Depends on: Weekly payout working
└── Bathroom Rotation
    └── Depends on: Update YAML, re-import chores

Phase 3 (Month 2+):
├── Monthly Interest
│   └── Depends on: Savings accounts working
├── Goal Tracking
│   └── Depends on: Template sensors
└── Advanced Dashboards
    └── Depends on: All features working
```

---

## File Structure for Implementation

```
/config/
├── configuration.yaml
│   └── Includes: packages/kidschores.yaml
├── packages/
│   └── kidschores.yaml
│       ├── input_boolean (custody toggle)
│       ├── input_number (banking accounts)
│       ├── input_text (goal tracking)
│       ├── template sensors (allowance, goals)
│       ├── automations (custody, payout, notifications)
│       └── scripts (withdrawals, transfers)
├── ui-lovelace.yaml
│   ├── Kid dashboard (Bella)
│   ├── Kid dashboard (Lilly)
│   └── Parent dashboard
└── home-config/
    ├── config/
    │   └── chores_bella_lilly.yaml (updated with rotation chores)
    └── tests/
        ├── test_allowance.py
        ├── test_custody.py
        ├── test_payout.py
        └── test_notifications.py
```

---

**End of Feature Blueprints**

This document provides everything needed to implement the remaining features:
- Exact entity definitions
- Complete automation logic
- Dashboard structures
- Test cases for validation
- Integration points with existing system

Local Claude can use this as a reference to build each feature step by step! 🚀
