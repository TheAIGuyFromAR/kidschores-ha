# KidsChores Integration - Capabilities Audit

**Audit Date:** 2025-01-17
**Integration Version:** Latest from HACS
**Purpose:** Determine what's already available vs what needs to be added

---

## ✅ What Already Exists

### **Services Available** (`services.yaml`)

**Chore Management:**
- `claim_chore` - Kid claims a chore
- `approve_chore` - Parent approves with points
- `disapprove_chore` - Parent rejects claim
- `set_chore_due_date` - Set or clear due date
- `skip_chore_due_date` - Skip to next occurrence

**Reward System:**
- `redeem_reward`
- `approve_reward`
- `disapprove_reward`

**Points Adjustments:**
- `apply_penalty` - Deduct points
- `apply_bonus` - Award extra points

**Reset Operations:**
- `reset_all_data`
- `reset_all_chores`
- `reset_overdue_chores`
- `reset_penalties`
- `reset_bonuses`
- `reset_rewards`

### **Chore Sensor Attributes** (`sensor.py:410-456`)

Each chore sensor exposes:
```python
{
    "kid_name": "Bella",
    "chore_name": "Open Blinds (Morning)",
    "description": "Open all blinds in common areas",
    "chore_claims_count": 5,
    "chore_approvals_count": 5,
    "recurring_frequency": "daily",
    "applicable_days": ["mon", "tue", "thu", "fri"],
    "due_date": "2025-01-18T09:00:00",
    "default_points": 1,
    "partial_allowed": false,
    "allow_multiple_claims_per_day": false,
    "assigned_kids": ["Bella", "Lilly"],
    "labels": ["Required", "Daily"]  # ✅ LABELS ALREADY EXPOSED!
}
```

**Key Finding:** ✅ **Labels are already exposed as `labels` attribute!**

### **Global Chore Sensor Attributes** (`sensor.py:1078-1113`)

Per-chore global sensors also expose labels:
```python
sensor.kidschores_chore_open_blinds_morning:
  attributes:
    labels: ["Required", "Daily"]
    assigned_kids: ["Bella", "Lilly"]
    recurring_frequency: "daily"
    applicable_days: ["mon", "tue", "thu", "fri"]
```

---

## ❌ What's Missing

### **1. Disable/Enable Chore Service**

**Not Available:**
- No `disable_chore` service
- No `enable_chore` service
- No `is_enabled` field in storage

**Workarounds:**
- Use `set_chore_due_date` with null to clear due date (prevents recurrence)
- Use `skip_chore_due_date` to defer

**Problem:** These workarounds don't truly "disable" - they manipulate due dates. For Lilly's custody weeks, we'd need to clear/restore due dates for 20+ chores every week.

### **2. Alternating Weekly Frequency**

**Not Available:**
- Frequencies: daily, weekly, biweekly, monthly, none
- No `alternating_weekly` or `every_other_week`

**Workaround:** Create separate chores or use automation to manipulate assignments

### **3. Query Chores by Label**

**Not Available:**
- No `get_chores_by_label` service
- Can't easily query "all Required chores for Bella"

**Workaround:** Can be done in templates by iterating entities

### **4. Count Completed Chores by Label**

**Not Available:**
- Can't query "how many 'Required' chores did Bella complete this week?"
- Only have total points

**Workaround:** Would need custom template to iterate all chore sensors

---

## 🔧 What We Need to Add

Based on our use cases (allowance system, custody toggle), we need:

### **Priority 1: Allowance Calculation Support**

**Challenge:** Calculate "$10 base if ALL Required chores completed"

**Options:**

**Option A: Use Point Threshold (No code changes needed)**
```yaml
# Set "Required" chores to total exactly 40 points
# Base unlocked if >= 40 points earned
template:
  - sensor:
      - name: Bella Allowance
        state: >
          {% set pts = states('sensor.kidschores_bella_points')|int(0) %}
          {% set base = 10 if pts >= 40 else 0 %}
          {% set bonus = pts * 0.25 %}
          {{ base + bonus }}
```

**Option B: Add Custom Sensor (Code change required)**

Add to `sensor.py`:
```python
class RequiredChoresCompletedSensor(CoordinatorEntity, SensorEntity):
    """Count completed Required chores for a kid this week"""

    @property
    def state(self):
        completed_count = 0
        total_required = 0

        for chore_id, chore in self.coordinator.chores_data.items():
            # If assigned to this kid and has "Required" label
            if self._kid_id in chore.get("assigned_kids", []):
                if "Required" in chore.get("chore_labels", []):
                    total_required += 1
                    if chore.get("state") == "approved":
                        completed_count += 1

        return completed_count

    @property
    def extra_state_attributes(self):
        return {
            "total_required": self._count_total(),
            "completion_rate": f"{self.state}/{self._count_total()}"
        }
```

**Option C: Helper Entity + Template (No code changes)**
```yaml
# Manually maintain list of Required chore names
input_text:
  bella_required_chores:
    initial: "Open Blinds,Close Blinds,Scoop Litter,..."

# Template counts completed
template:
  - sensor:
      - name: Bella Required Chores Completed
        state: >
          {% set required = states('input_text.bella_required_chores').split(',') %}
          {% set completed = namespace(count=0) %}
          {% for chore in required %}
            {% set entity = 'sensor.kidschores_bella_chore_' ~ chore|slugify %}
            {% if states(entity) == 'approved' %}
              {% set completed.count = completed.count + 1 %}
            {% endif %}
          {% endfor %}
          {{ completed.count }}
```

### **Priority 2: Custody Toggle (Lilly's Alternating Weeks)**

**Challenge:** Disable all Lilly chores during away weeks

**Options:**

**Option A: Add disable_chore Service (Code change)**

Add to `coordinator.py`:
```python
async def async_disable_chore(self, chore_name: str):
    """Disable a chore temporarily"""
    chore = self._find_chore_by_name(chore_name)
    if chore:
        chore["is_enabled"] = False
        await self._storage.async_save()

async def async_enable_chore(self, chore_name: str):
    """Re-enable a chore"""
    chore = self._find_chore_by_name(chore_name)
    if chore:
        chore["is_enabled"] = True
        await self._storage.async_save()
```

Add to `services.yaml`:
```yaml
disable_chore:
  name: "Disable Chore"
  description: "Temporarily disable a chore (stops recurrence)"
  fields:
    chore_name:
      name: "Chore Name"
      required: true
      selector:
        text:

enable_chore:
  name: "Enable Chore"
  description: "Re-enable a previously disabled chore"
  fields:
    chore_name:
      name: "Chore Name"
      required: true
      selector:
        text:
```

**Option B: Create Lilly-Away Versions (No code changes)**

Duplicate all Lilly chores with "Away" suffix, manually enable/disable:
- "Laundry - Lilly (Home weeks)" - enabled weeks 1,3,5,7
- "Laundry - Lilly (Away weeks)" - disabled weeks 1,3,5,7

**Option C: Automation with set_chore_due_date (Hacky workaround)**
```yaml
automation:
  - alias: Disable Lilly Chores When Away
    trigger:
      - platform: state
        entity_id: input_boolean.lilly_home_this_week
        to: 'off'
    action:
      # Clear due dates for all Lilly chores
      - repeat:
          for_each:
            - "Laundry - Lilly"
            - "Scoop Litter - Lilly"
            # ... 20+ more chores
          sequence:
            - service: kidschores.set_chore_due_date
              data:
                chore_name: "{{ repeat.item }}"
                due_date: null  # Clear due date
```

---

## 💡 Recommendation

### **For Allowance System:**
Use **Option A (Point Threshold)** - No code changes needed
- Adjust point values so "Required" chores = exactly 40 points
- Simple template sensor calculates allowance
- Easy to maintain

### **For Custody Toggle:**
Use **Option C (Automation with set_chore_due_date)** initially
- Test with automation clearing/setting due dates
- If too clunky, then add disable_chore service (Option A)

### **Code Changes Priority:**
1. **Low Priority:** Allowance works fine with point threshold
2. **Medium Priority:** Add `disable_chore`/`enable_chore` services if custody toggle is painful
3. **Nice to Have:** Add `RequiredChoresCompletedSensor` for stricter allowance tracking

---

## 🎯 Next Steps

**Immediate (No code changes):**
1. ✅ Verify labels work in templates
2. Create allowance template sensors using point threshold
3. Test custody toggle automation with `set_chore_due_date`

**If Needed (Code changes):**
4. Add `disable_chore`/`enable_chore` services
5. Add `is_enabled` field to chore storage schema
6. Add `RequiredChoresCompletedSensor` class

**Benefits of Editing Integration:**
- Have Claude to maintain it (no update dependency)
- Can add exactly what we need
- Could contribute back to project

**Risks:**
- Must test thoroughly (could break KidsChores)
- Need to document changes
- Potential merge conflicts if upstream changes

---

## Summary

**Good News:** Labels already exposed, most of what we need exists!

**Can do without code changes:**
- Allowance calculation (point threshold method)
- Custody toggle (automation + set_chore_due_date workaround)
- Dashboard filtering (template conditionals on labels)

**Would benefit from code changes:**
- Cleaner custody toggle (disable_chore service)
- Stricter allowance tracking (count completed Required chores)
- Alternating weeks (new frequency type)

**Recommended:** Try template/automation approach first, add code changes if too clunky.
