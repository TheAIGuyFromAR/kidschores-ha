# KidsChores Setup - Updated with Parent Chores

## What's New

The KidsChores configuration has been updated to include parent chores for Blake and Amanda. All files are now ready for your local Claude instance to deploy.

---

## Summary of Changes

### 1. Updated Kids List
**Before:** Bella, Lilly
**Now:** Bella, Lilly, Blake, Amanda

Blake and Amanda are added as "kids" in the system with 0 points - this is the workaround to track parent tasks since KidsChores doesn't have a separate "parent" entity type.

### 2. New Parent Chores Added

**Daily Parent Chores (2):**
- Pet Water Refresh (Blake & Amanda, 0 points)
- Dog Evening Meal (Blake & Amanda, 0 points)

**Weekly Parent Chores (1):**
- Litter Waste Bin (Blake only, Sundays, 0 points)

**Monthly Parent Chores (1):**
- Pip Environment Check (Blake & Amanda, 0 points)

All parent chores are labeled with "Parent" tag for easy filtering in dashboards.

### 3. Files Updated

✅ **chores_bella_lilly.yaml**
- Added Blake and Amanda to kids list
- Added 4 parent chores (daily, weekly, monthly)
- Now contains 40+ total chores

✅ **import_kidschores_standalone.py**
- Updated embedded CHORES_DATA with Blake and Amanda
- Added all 4 parent chores to embedded data
- No external dependencies needed

✅ **README.md**
- Updated chore count (40+ chores)
- Updated system overview with parent information
- Updated chore categories breakdown

✅ **CLAUDE_HANDOFF_INSTRUCTIONS.md** (NEW!)
- Comprehensive step-by-step guide for local Claude
- Handles partially completed setup
- Includes patching/update scenarios
- Complete troubleshooting section
- Quick command reference

---

## What Your Local Claude Needs to Do

Your local Claude instance has access to `/config` directory and can execute these steps:

### Prerequisites to Verify First

1. **Check if KidsChores is installed:**
   ```bash
   ls -la /config/.storage/kidschores_data
   ```
   If not found → User needs to install KidsChores via HACS first

2. **Check if Python is installed:**
   ```bash
   python3 --version
   ```
   If not found → Install with `apk add python3`

3. **Check current kids:**
   ```bash
   cat /config/.storage/kidschores_data | grep -A 5 '"kids"'
   ```
   Expected: Should see Bella and Lilly at minimum

### Main Tasks for Local Claude

**Task 1: Copy Configuration Files**
```bash
# Create directory structure
mkdir -p /config/home-config/config
mkdir -p /config/home-config/scripts

# Copy YAML configuration
cp /home/user/kidschores-ha/home-config/config/chores_bella_lilly.yaml \
   /config/home-config/config/chores_bella_lilly.yaml

# Copy Python import script
cp /home/user/kidschores-ha/home-config/scripts/import_kidschores_standalone.py \
   /config/home-config/scripts/import_kidschores_standalone.py

# Make script executable
chmod +x /config/home-config/scripts/import_kidschores_standalone.py
```

**Task 2: Add Blake & Amanda as Kids (if not already done)**

If Blake and Amanda don't exist yet, user needs to add them via UI first:
1. Settings > Devices & Services > KidsChores > Configure
2. Manage Kids > Add Kid
3. Name: Blake, Points: 0
4. Repeat for Amanda

Alternative: Script will warn if kids don't exist - user can add them after and re-run import.

**Task 3: Run Import Script**

```bash
cd /config/home-config/scripts

# Dry run first (test mode)
python3 import_kidschores_standalone.py /config/.storage/kidschores_data --dry-run

# If dry run looks good, run actual import
python3 import_kidschores_standalone.py /config/.storage/kidschores_data
```

**Task 4: Restart Home Assistant**
```bash
ha core restart
```

**Task 5: Verify Import**

Check that chores appear in:
- Developer Tools > States (filter: `sensor.kidschores`)
- Settings > Devices & Services > KidsChores > Manage Chores

---

## What to Expect

### If Everything Goes Smoothly

```
============================================================
KidsChores Bulk Import Script (Standalone)
============================================================

📁 Creating backup...
✅ Backup created: /config/.storage/backups/kidschores_data.backup.20250116_143052

📖 Loading existing storage...
📖 Loading chore definitions (embedded)...
   Found 4 existing kids: Bella, Lilly, Blake, Amanda

🔨 Processing chores...
   ✅ Created 'Open Blinds (Morning)' (ID: abc12345...)
   ✅ Created 'Pet Water Refresh' (ID: def67890...)
   [... 40+ more chores ...]

💾 Saving 43 new chores...
✅ Storage updated: /config/.storage/kidschores_data

============================================================
✅ SUCCESS! Added 43 chores
============================================================

📋 Next steps:
   1. Restart Home Assistant: ha core restart
   2. Check Developer Tools > States for new entities
   3. If issues, restore backup:
      cp /config/.storage/backups/kidschores_data.backup.TIMESTAMP /config/.storage/kidschores_data
```

### If Blake & Amanda Don't Exist Yet

```
⚠️  Warning: Kid 'Blake' not found for chore 'Pet Water Refresh'
⚠️  Warning: Kid 'Amanda' not found for chore 'Pet Water Refresh'
```

**Solution:** Add Blake and Amanda via UI, then re-run script. Script will skip existing chores and only add the parent ones.

### If Some Chores Already Exist

```
🔨 Processing chores...
   ⏭️  Skipping 'Open Blinds (Morning)' (already exists)
   ⏭️  Skipping 'Close Blinds (Evening)' (already exists)
   ✅ Created 'Pet Water Refresh' (ID: abc12345...)
   ✅ Created 'Litter Waste Bin' (ID: def67890...)

============================================================
✅ SUCCESS! Added 4 chores
   (Skipped 39 existing chores)
============================================================
```

**This is NORMAL** if re-running the script. It safely skips duplicates.

---

## Chore Breakdown

### Total: 43 Chores

**Daily Chores (10):**
- Open Blinds (Morning) - Bella & Lilly - 1 point
- Close Blinds (Evening) - Bella & Lilly - 1 point
- Plant Lights On - Bella & Lilly - 1 point
- Plant Lights Off - Bella & Lilly - 1 point
- Scoop Litter - Bella (Mon/Wed/Fri/Sun) - 3 points
- Scoop Litter - Lilly (Tue/Thu/Sat) - 3 points
- Dog Out After School - Bella - 2 points
- Dog Out After School - Lilly - 2 points
- **Pet Water Refresh - Blake & Amanda - 0 points** ⭐ NEW
- **Dog Evening Meal - Blake & Amanda - 0 points** ⭐ NEW

**Weekly Chores (12):**
- Laundry - Bella (Sunday) - 4 points
- Laundry - Lilly (Tuesday) - 4 points
- Towels - Lilly (Wednesday) - 3 points
- Delicates - Bella (Thursday) - 3 points
- Room Reset - Bella (Wednesday) - 5 points
- Room Reset - Lilly (Wednesday) - 5 points
- Bathroom Trash (Thursday) - 2 points
- Sweep Kitchen (Mon/Wed/Fri) - 3 points
- Pip Feeding - Lilly (Wednesday) - 3 points
- Room Trash Check - Bella (Sun/Thu) - 2 points
- Room Trash Check - Lilly (Sun/Thu) - 2 points
- **Litter Waste Bin - Blake (Sunday) - 0 points** ⭐ NEW

**Bi-weekly Chores (2):**
- Bathroom Sink Clean - 3 points
- Toilet Scrub - 4 points

**Monthly Chores (9):**
- Bathroom Floor Mop - 4 points
- Bathroom Tub/Shower Scrub - 5 points
- Bathroom Mirror Clean - 2 points
- Bathroom Ceiling Dust - 3 points
- Bathroom Baseboards - 3 points
- Bathroom Countertop Deep Clean - 3 points
- Pip Habitat Spot Clean - 4 points
- Bed Sheets Change - Bella - 4 points
- Bed Sheets Change - Lilly - 4 points
- **Pip Environment Check - Blake & Amanda - 0 points** ⭐ NEW

**Seasonal Chores (2):**
- Water Outdoor Plants (Tue/Fri) - 3 points
- Harvest Vegetables (Saturday) - 3 points

---

## Dashboard Filtering

Since Blake and Amanda are tracked with 0 points, you can filter them out of kids' dashboards:

**Example Lovelace Card (Kids View):**
```yaml
type: entities
title: Pending Chores
entities:
  - entity: sensor.kidschores_bella_points
  - entity: sensor.kidschores_lilly_points
# Blake and Amanda excluded from kids view
```

**Example Lovelace Card (Parent View):**
```yaml
type: entities
title: Parent Tasks
entities:
  - entity: sensor.kidschores_blake_points
  - entity: sensor.kidschores_amanda_points
```

**Using Labels for Filtering:**

All parent chores have "Parent" label. You can create automations or dashboard filters using:
```yaml
condition: template
value_template: "{{ 'Parent' in state_attr(entity_id, 'labels') }}"
```

---

## Rollback Procedure

If something goes wrong, the script creates automatic backups:

```bash
# List backups
ls -lah /config/.storage/backups/kidschores_data.backup.*

# Restore backup (replace TIMESTAMP with actual timestamp)
cp /config/.storage/backups/kidschores_data.backup.20250116_143052 \
   /config/.storage/kidschores_data

# Restart Home Assistant
ha core restart
```

---

## Next Steps After Import

1. **Verify all chores imported:**
   - Check Developer Tools > States
   - Count should show 43+ entities

2. **Set up allowance template sensors** (kids only):
   - See `/home/user/kidschores-ha/home-config/docs/MVP_IMPLEMENTATION_PLAN.md`
   - Blake and Amanda won't affect allowance (0 points)

3. **Create dashboards:**
   - Kids dashboard (exclude parent chores)
   - Parent dashboard (only parent chores)
   - Family overview (all chores)

4. **Test the system:**
   - Claim a chore as Bella
   - Approve a chore
   - Verify points increase
   - Check parent chores appear correctly

5. **Adjust as needed:**
   - Fine-tune point values
   - Adjust due times
   - Add/remove labels

---

## Important Notes

### Kids vs HA Users

**Remember:** Blake, Amanda, Bella, and Lilly in KidsChores are NOT Home Assistant users.

- They are independent "kid" entities
- HA users can be linked later (optional)
- All chores continue working whether linked or not
- Optional field: `ha_user_id` can remain None

### Parent Points are Zero

Blake and Amanda chores all have 0 points because:
- Parents don't earn allowance
- Keeps point totals clean for kids
- Prevents accidental allowance payment to parents
- Dashboard filtering easier

### Quarterly/Semi-Annual Chores

For infrequent maintenance (HVAC, solar panels, etc.), use Google Calendar instead:
- Too infrequent for weekly tracking
- Better suited for calendar reminders
- Prevents clutter in chore system

---

## File Locations

**On Your Development Machine:**
- `/home/user/kidschores-ha/home-config/config/chores_bella_lilly.yaml`
- `/home/user/kidschores-ha/home-config/scripts/import_kidschores_standalone.py`
- `/home/user/kidschores-ha/home-config/CLAUDE_HANDOFF_INSTRUCTIONS.md`

**On Home Assistant (after copying):**
- `/config/home-config/config/chores_bella_lilly.yaml`
- `/config/home-config/scripts/import_kidschores_standalone.py`
- `/config/.storage/kidschores_data` (storage file)
- `/config/.storage/backups/` (automatic backups)

---

## Support Resources

**KidsChores Integration:**
- GitHub: https://github.com/ad-ha/kidschores-ha
- Community Forum: https://community.home-assistant.io/t/kidschores-family-chore-management-integration

**Documentation in this project:**
- `docs/IMPLEMENTATION_REVIEW.md` - Full design review
- `docs/MVP_IMPLEMENTATION_PLAN.md` - 2-week implementation guide
- `docs/BULK_CHORE_CREATION_GUIDE.md` - Detailed import guide
- `CLAUDE_HANDOFF_INSTRUCTIONS.md` - Step-by-step for local Claude

---

## Estimated Time

**Total setup time:** 15-30 minutes (vs 15-20 hours manual UI entry!)

**Breakdown:**
- Copy files: 5 minutes
- Add Blake & Amanda (if needed): 5 minutes
- Run import script: 2 minutes
- Restart HA: 3 minutes
- Verify: 5-10 minutes

---

## Success!

When complete, you'll have:

✅ 43 chores configured and tracked
✅ Kids (Bella & Lilly) earning points toward allowance
✅ Parents (Blake & Amanda) with task visibility
✅ Automatic backups of all data
✅ Foundation for future enhancements (banking, automations, etc.)

Good luck! 🎉
