# KidsChores Bulk Import - Local Claude Setup Instructions

## Overview

You are helping to set up a KidsChores chore management system in Home Assistant OS. This system tracks chores for kids (Bella & Lilly) and parents (Blake & Amanda) with automatic allowance calculation.

**Important Context:**
- KidsChores integration has NO bulk creation API - only manual UI entry
- Manual UI entry would take 15-20 hours for 40+ chores
- Solution: Python script that directly manipulates `/config/.storage/kidschores_data` JSON
- This setup was partially started by another Claude instance
- You have direct file access in `/config` directory

**Kids in the system:**
- Bella: Always gone Wednesdays
- Lilly: Gone every other week (whole week)
- Blake: Parent (tracked with 0 points)
- Amanda: Parent (tracked with 0 points)

**Note:** "Kids" in KidsChores are independent entities, NOT Home Assistant users. HA users can be linked later - all existing chores continue working.

---

## Prerequisites Check

Before proceeding, verify the current state and skip completed steps:

### 1. Check if KidsChores is installed and configured

```bash
# Check if KidsChores storage file exists
ls -la /config/.storage/kidschores_data
```

**If file exists:** KidsChores is installed ✅
**If file NOT found:** User needs to install KidsChores via HACS first ❌

### 2. Check if kids are already created

```bash
# View current kids in system
cat /config/.storage/kidschores_data | grep -A 5 '"kids"'
```

**Expected kids:** Bella, Lilly, Blake, Amanda

**If all 4 exist:** Skip kid creation, proceed to chore import ✅
**If only Bella & Lilly exist:** Script will add Blake & Amanda automatically ✅
**If no kids exist:** User needs to configure KidsChores first ❌

### 3. Check if Python is installed

```bash
# Check Python availability
python3 --version
```

**If version shown (e.g., Python 3.11.x):** Python installed ✅
**If "command not found":** Install Python (instructions below) ❌

### 4. Check if import files already exist

```bash
# Check for existing configuration files
ls -la /config/home-config/config/chores_bella_lilly.yaml
ls -la /config/home-config/scripts/import_kidschores_standalone.py
```

**If files exist:** Skip file creation, proceed to import ✅
**If files NOT exist:** Create files (instructions below) ❌

---

## Step-by-Step Setup

### Step 1: Install Python (if needed)

**Check if needed:** Run `python3 --version`

**If needed, install:**
```bash
# You are running in Alpine Linux container
apk add python3
```

**Verify:**
```bash
python3 --version
# Should show: Python 3.11.x or similar
```

---

### Step 2: Create Directory Structure (if needed)

**Check if needed:** Run `ls -la /config/home-config`

**If needed, create:**
```bash
mkdir -p /config/home-config/config
mkdir -p /config/home-config/scripts
mkdir -p /config/home-config/docs
```

---

### Step 3: Create Chore Configuration File

**File:** `/config/home-config/config/chores_bella_lilly.yaml`

**Check if exists:** `ls -la /config/home-config/config/chores_bella_lilly.yaml`

**If file exists and has content:**
1. Ask user if they want to update/replace it
2. Create backup: `cp /config/home-config/config/chores_bella_lilly.yaml /config/home-config/config/chores_bella_lilly.yaml.backup.$(date +%Y%m%d_%H%M%S)`
3. Then create new file below

**If file does NOT exist, create it:**

Use File Editor addon in Home Assistant (easiest method):
1. Navigate to: Settings > Add-ons > File Editor > Open Web UI
2. Click folder icon, navigate to `/config/home-config/config/`
3. Create new file: `chores_bella_lilly.yaml`
4. Paste the complete YAML content (see YAML_CONTENT section below)
5. Save file

**Alternative method (terminal):**
```bash
cat > /config/home-config/config/chores_bella_lilly.yaml << 'EOF'
[PASTE YAML CONTENT HERE]
EOF
```

---

### Step 4: Create Python Import Script

**File:** `/config/home-config/scripts/import_kidschores_standalone.py`

**Check if exists:** `ls -la /config/home-config/scripts/import_kidschores_standalone.py`

**If file exists:**
1. Check if it's the standalone version: `grep -q "CHORES_DATA = {" /config/home-config/scripts/import_kidschores_standalone.py`
2. If NOT standalone version, or if user wants to update, create backup and proceed

**Create the script:**

Use File Editor addon (recommended):
1. Navigate to: Settings > Add-ons > File Editor > Open Web UI
2. Click folder icon, navigate to `/config/home-config/scripts/`
3. Create new file: `import_kidschores_standalone.py`
4. Paste the complete Python content (see PYTHON_CONTENT section below)
5. Save file

**Make it executable:**
```bash
chmod +x /config/home-config/scripts/import_kidschores_standalone.py
```

---

### Step 5: Verify File Structure

```bash
# Check all files are in place
ls -la /config/home-config/config/chores_bella_lilly.yaml
ls -la /config/home-config/scripts/import_kidschores_standalone.py

# Verify YAML is valid (basic check)
head -20 /config/home-config/config/chores_bella_lilly.yaml

# Verify Python script is valid
head -30 /config/home-config/scripts/import_kidschores_standalone.py
```

**Expected output:**
- Both files should exist
- YAML should start with "# KidsChores Bulk Import Configuration"
- Python should start with "#!/usr/bin/env python3"

---

### Step 6: Run Dry-Run Import (Test Mode)

This previews what would be imported without making changes:

```bash
cd /config/home-config/scripts

python3 import_kidschores_standalone.py \
  /config/.storage/kidschores_data \
  --dry-run
```

**Expected output:**
```
============================================================
KidsChores Bulk Import Script (Standalone)
============================================================

📖 Loading existing storage...
📖 Loading chore definitions (embedded)...
   Found 2 existing kids: Bella, Lilly
   (or 4 if Blake & Amanda already added)

🔨 Processing chores...
   ✅ [DRY RUN] Would create 'Open Blinds (Morning)' (ID: abc12345...)
   ✅ [DRY RUN] Would create 'Close Blinds (Evening)' (ID: def67890...)
   [... more chores ...]

============================================================
🔍 DRY RUN: Would add 40+ chores
============================================================

Run without --dry-run to actually import
```

**If you see errors:**
- **"Kid not found"**: KidsChores not configured with kids yet - user needs to do initial setup
- **"Storage file not found"**: KidsChores not installed - user needs to install via HACS
- **"Python not found"**: Go back to Step 1
- **Syntax errors**: Check YAML/Python file content

---

### Step 7: Run Actual Import

**WARNING:** Only proceed if dry-run was successful!

```bash
cd /config/home-config/scripts

python3 import_kidschores_standalone.py \
  /config/.storage/kidschores_data
```

**Expected output:**
```
============================================================
KidsChores Bulk Import Script (Standalone)
============================================================

📁 Creating backup...
✅ Backup created: /config/.storage/backups/kidschores_data.backup.20250116_143052

📖 Loading existing storage...
📖 Loading chore definitions (embedded)...
   Found 2 existing kids: Bella, Lilly

🔨 Processing chores...
   ✅ Created 'Open Blinds (Morning)' (ID: abc12345...)
   ✅ Created 'Close Blinds (Evening)' (ID: def67890...)
   [... 40+ more chores ...]

💾 Saving 40+ new chores...
✅ Storage updated: /config/.storage/kidschores_data

============================================================
✅ SUCCESS! Added 40+ chores
============================================================

📋 Next steps:
   1. Restart Home Assistant: ha core restart
   2. Check Developer Tools > States for new entities
   3. If issues, restore backup:
      cp /config/.storage/backups/kidschores_data.backup.TIMESTAMP /config/.storage/kidschores_data
```

**If import succeeds but shows "Skipped X existing chores":**
This is NORMAL if re-running the script. The script won't duplicate existing chores.

---

### Step 8: Restart Home Assistant

```bash
ha core restart
```

Wait 2-3 minutes for restart to complete.

---

### Step 9: Verify Chores Were Created

**Method 1: Developer Tools > States**
1. Open Home Assistant UI
2. Navigate to: Developer Tools > States
3. Filter by: `sensor.kidschores`
4. You should see entities like:
   - `sensor.kidschores_bella_points`
   - `sensor.kidschores_lilly_points`
   - `sensor.kidschores_blake_points`
   - `sensor.kidschores_amanda_points`
   - Multiple `sensor.kidschores_chore_*` entities (40+)

**Method 2: KidsChores Integration UI**
1. Navigate to: Settings > Devices & Services > KidsChores
2. Click "Configure"
3. Click "Manage Chores"
4. You should see 40+ chores listed

**Method 3: Terminal Check**
```bash
# Count chores in storage
cat /config/.storage/kidschores_data | grep -c '"name":'
# Should show ~40+ (exact count depends on kids + chores)
```

---

## Troubleshooting

### Issue: "Kid not found: Blake" or "Kid not found: Amanda"

**Cause:** Blake and Amanda aren't created in KidsChores yet.

**Solution:** User needs to add them via UI first:
1. Settings > Devices & Services > KidsChores > Configure
2. Click "Manage Kids"
3. Add Blake
4. Add Amanda
5. Re-run import script

**OR** remove Blake/Amanda from YAML if user doesn't want parent tracking:
```bash
# Edit YAML to remove Blake and Amanda from kids list and their chores
nano /config/home-config/config/chores_bella_lilly.yaml
```

### Issue: No chores appear after restart

**Possible causes:**
1. Import reported errors (check output)
2. Home Assistant didn't fully restart (wait longer)
3. Storage file corrupted (restore backup)

**Debug steps:**
```bash
# Check storage file integrity
cat /config/.storage/kidschores_data | python3 -m json.tool > /dev/null
# No output = valid JSON, error message = corrupted

# Check if chores exist in storage
cat /config/.storage/kidschores_data | grep -c "internal_id"
# Should show count of chores

# Restore backup if needed
cp /config/.storage/backups/kidschores_data.backup.TIMESTAMP /config/.storage/kidschores_data
ha core restart
```

### Issue: Script runs but shows "Would skip X existing chores"

**This is NORMAL behavior.** The script won't duplicate chores that already exist.

**If you want to re-import:**
1. Delete existing chores via KidsChores UI first
2. OR restore from backup before running script

### Issue: Python script has "No module named 'yaml'" error

**Cause:** Using wrong script version (non-standalone).

**Solution:** Use `import_kidschores_standalone.py` which has embedded data (no YAML dependency).

### Issue: Chores imported but with wrong due dates

**This is expected behavior.** Due dates are calculated as "next occurrence" from import time.

**Example:** If you import on Wednesday at 2pm, and a chore is due "Monday at 9am", it will be set for the upcoming Monday.

**To adjust:** Use Home Assistant UI to manually set due dates after import.

---

## Post-Import Configuration

### Add Blake & Amanda as Kids (if not done yet)

If Blake and Amanda show errors during import, add them via UI:

1. Settings > Devices & Services > KidsChores > Configure
2. Manage Kids > Add Kid
3. Name: Blake, Points: 0
4. Repeat for Amanda
5. Re-run import script (will add their chores)

### Set Up Allowance Calculation

Create template sensors for weekly allowance:

**File:** `/config/configuration.yaml`

Add this section:
```yaml
template:
  - sensor:
      - name: "Bella Weekly Allowance"
        unique_id: bella_weekly_allowance
        unit_of_measurement: "$"
        state: >
          {% set points = states('sensor.kidschores_bella_points') | int(0) %}
          {% set base = 10 %}
          {% set bonus = (points - 40) * 0.25 if points > 40 else 0 %}
          {{ (base + bonus) | round(2) }}
        attributes:
          base_allowance: 10
          bonus_points: >
            {{ (states('sensor.kidschores_bella_points') | int(0) - 40) if states('sensor.kidschores_bella_points') | int(0) > 40 else 0 }}
          total_points: "{{ states('sensor.kidschores_bella_points') | int(0) }}"

      - name: "Lilly Weekly Allowance"
        unique_id: lilly_weekly_allowance
        unit_of_measurement: "$"
        state: >
          {% set points = states('sensor.kidschores_lilly_points') | int(0) %}
          {% set base = 10 %}
          {% set bonus = (points - 40) * 0.25 if points > 40 else 0 %}
          {{ (base + bonus) | round(2) }}
        attributes:
          base_allowance: 10
          bonus_points: >
            {{ (states('sensor.kidschores_lilly_points') | int(0) - 40) if states('sensor.kidschores_lilly_points') | int(0) > 40 else 0 }}
          total_points: "{{ states('sensor.kidschores_lilly_points') | int(0) }}"
```

### Create Dashboard for Kids

Use the KidsChores built-in dashboard or create custom cards:

**Example Lovelace Card:**
```yaml
type: entities
title: Bella's Chores
entities:
  - entity: sensor.kidschores_bella_points
    name: Total Stars
  - entity: sensor.bella_weekly_allowance
    name: This Week's Allowance
```

### Set Up Custody Toggle (for Lilly)

**File:** `/config/configuration.yaml`

Add helper:
```yaml
input_boolean:
  lilly_home_this_week:
    name: Lilly Home This Week
    icon: mdi:home-account
```

Then create automations to disable Lilly's chores when she's away (future enhancement).

---

## File Locations Reference

**Configuration Files:**
- `/config/home-config/config/chores_bella_lilly.yaml` - YAML chore definitions
- `/config/home-config/scripts/import_kidschores_standalone.py` - Python import script
- `/config/home-config/docs/` - Documentation (optional)

**KidsChores Storage:**
- `/config/.storage/kidschores_data` - Main storage file (JSON)
- `/config/.storage/backups/kidschores_data.backup.*` - Automatic backups

**Home Assistant Config:**
- `/config/configuration.yaml` - Main HA config (for template sensors)

---

## Complete File Contents

### YAML_CONTENT: chores_bella_lilly.yaml

**Location:** `/config/home-config/config/chores_bella_lilly.yaml`

**NOTE:** This file is already created in the repository. You can copy it from:
`/home/user/kidschores-ha/home-config/config/chores_bella_lilly.yaml`

**To view the file:**
```bash
cat /home/user/kidschores-ha/home-config/config/chores_bella_lilly.yaml
```

**To copy to Home Assistant config directory:**
```bash
cp /home/user/kidschores-ha/home-config/config/chores_bella_lilly.yaml \
   /config/home-config/config/chores_bella_lilly.yaml
```

---

### PYTHON_CONTENT: import_kidschores_standalone.py

**Location:** `/config/home-config/scripts/import_kidschores_standalone.py`

**NOTE:** This file is already created in the repository. You can copy it from:
`/home/user/kidschores-ha/home-config/scripts/import_kidschores_standalone.py`

**To view the file:**
```bash
cat /home/user/kidschores-ha/home-config/scripts/import_kidschores_standalone.py
```

**To copy to Home Assistant config directory:**
```bash
cp /home/user/kidschores-ha/home-config/scripts/import_kidschores_standalone.py \
   /config/home-config/scripts/import_kidschores_standalone.py
```

---

## Quick Command Reference

**Check current state:**
```bash
# Verify KidsChores is installed
ls -la /config/.storage/kidschores_data

# Count existing chores
cat /config/.storage/kidschores_data | grep -c "internal_id"

# Check Python installed
python3 --version

# View current kids
cat /config/.storage/kidschores_data | grep -A 5 '"kids"'
```

**Run import:**
```bash
cd /config/home-config/scripts

# Dry run (test)
python3 import_kidschores_standalone.py /config/.storage/kidschores_data --dry-run

# Actual import
python3 import_kidschores_standalone.py /config/.storage/kidschores_data

# Restart HA
ha core restart
```

**Rollback if needed:**
```bash
# List backups
ls -lah /config/.storage/backups/kidschores_data.backup.*

# Restore backup (replace TIMESTAMP with actual timestamp)
cp /config/.storage/backups/kidschores_data.backup.TIMESTAMP /config/.storage/kidschores_data

# Restart HA
ha core restart
```

---

## Success Criteria

After completing all steps, you should have:

✅ Python installed in HAOS environment
✅ Directory structure created (`/config/home-config/`)
✅ YAML file with 40+ chore definitions
✅ Python import script (standalone version)
✅ Successful dry-run showing all chores
✅ Successful actual import (40+ chores added)
✅ Home Assistant restarted
✅ Chores visible in KidsChores UI and Developer Tools > States
✅ Backup created automatically before import

**Estimated time:** 15-30 minutes (vs 15-20 hours manual UI entry!)

---

## Next Steps (Future Enhancements)

After basic import is working:

1. **Week 2:** Set up allowance template sensors
2. **Week 3:** Create kid-friendly dashboards
3. **Week 4:** Test system with family, adjust point values
4. **Month 2:** Add bathroom rotation automations
5. **Month 4-5:** Implement banking system (checking/savings accounts)
6. **Month 6:** Add goal tracking and advanced features

See `/home/user/kidschores-ha/home-config/docs/MVP_IMPLEMENTATION_PLAN.md` for detailed roadmap.

---

## Support

**If you encounter issues:**

1. Check troubleshooting section above
2. Verify all prerequisites are met
3. Review error messages carefully (they usually indicate the exact problem)
4. Check backup files exist before making changes
5. Test with dry-run mode first

**KidsChores Integration Documentation:**
- GitHub: https://github.com/ad-ha/kidschores-ha
- Community Forum: https://community.home-assistant.io/t/kidschores-family-chore-management-integration

---

## Final Notes

**Important Reminders:**

- Kids in KidsChores are NOT Home Assistant users (can link later)
- Blake & Amanda are tracked as "kids" with 0 points (workaround for parent chores)
- Script creates automatic backups before every import
- Script skips existing chores (safe to re-run)
- Due dates are calculated as "next occurrence" from import time
- Quarterly/semi-annual maintenance should use Google Calendar instead

**What Makes This Special:**

This bulk import approach saves 15-20 hours of manual UI data entry while maintaining data integrity through:
- Automatic backups
- Dry-run testing
- UUID management
- Due date calculation
- Skip-on-duplicate logic

Good luck! 🎉
