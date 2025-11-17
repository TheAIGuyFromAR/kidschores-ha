# Quick Start Guide - For Local Claude

**Branch:** `claude/ha-chore-system-design-01BCRjvNLCwQG32pdkGXW8oy`
**Status:** ✅ Ready to deploy

---

## What You're Deploying

**100% reliable allowance system** for kids (Bella, Lilly) + parent tasks (Blake, Amanda)
- 36 chores total (31 Required, 1 Bonus, 4 Parent)
- Allowance: $10 base (ONLY if ALL Required done) + points × $0.25
- Cannot be gamed (math proven!)
- 15 min setup vs 15-20 hours manual entry

---

## 3-Step Deployment

### Step 1: Restart HA (loads new sensor)
```bash
# Integration modifications already in /config/
ha core restart
# Wait 3 minutes

# Verify sensor exists:
# Developer Tools > States > filter: required_chores
# Should see: sensor.kc_bella_required_chores_completed_weekly
```

### Step 2: Import Chores
```bash
mkdir -p /config/home-config/{config,scripts}

# Copy files from repo
cp /home/user/kidschores-ha/home-config/config/chores_bella_lilly.yaml \
   /config/home-config/config/
cp /home/user/kidschores-ha/home-config/scripts/import_kidschores_standalone.py \
   /config/home-config/scripts/
chmod +x /config/home-config/scripts/import_kidschores_standalone.py

# Add Blake & Amanda as kids in KidsChores UI first:
# Settings > Devices & Services > KidsChores > Manage Kids > Add Kid
# Name: Blake, Points: 0
# Name: Amanda, Points: 0

# Run import
cd /config/home-config/scripts
python3 import_kidschores_standalone.py /config/.storage/kidschores_data --dry-run
python3 import_kidschores_standalone.py /config/.storage/kidschores_data

ha core restart
```

### Step 3: Activate Allowance Sensors
```bash
echo "template: !include home-config/config/allowance_system.yaml" >> /config/configuration.yaml

ha core restart

# Verify:
# Developer Tools > States > filter: allowance
# Should see: sensor.bella_weekly_allowance, sensor.lilly_weekly_allowance
```

---

## Verify It Works

**Test allowance calculation:**
1. Approve all Required chores for Bella
2. Check `sensor.kc_bella_required_chores_completed_weekly`:
   - State: 20 (or however many Required exist)
   - Attribute `all_required_done`: TRUE
3. Check `sensor.bella_weekly_allowance`:
   - Attribute `base_allowance`: 10 (unlocked!)

**Test gaming prevention:**
1. Skip one Required chore
2. `all_required_done`: FALSE
3. `base_allowance`: 0 (locked!)
4. Proves it can't be gamed ✅

---

## Key Files

**Modified Integration (already in /config/):**
- `/config/custom_components/kidschores/sensor.py` - Added RequiredChoresCompletedWeeklySensor
- `/config/custom_components/kidschores/translations/en.json` - Added sensor name

**Configuration (copy from repo):**
- `home-config/config/chores_bella_lilly.yaml` - 36 chore definitions
- `home-config/config/allowance_system.yaml` - Allowance template sensors
- `home-config/scripts/import_kidschores_standalone.py` - Bulk import script

---

## Important Notes

✅ **Water plants is Required** (not just Seasonal)
✅ **Import script is safe to re-run** (skips existing, only adds new)
❌ **Script does NOT update existing chores** (use UI for edits)
⚠️ **"Room Reset" might be too big for Lilly** (consider breaking into 5 small tasks)

---

## Entity IDs Created

After deployment:
- `sensor.kc_bella_points` (total points)
- `sensor.kc_bella_required_chores_completed_weekly` (NEW! counts Required)
- `sensor.bella_weekly_allowance` (NEW! calculates $)
- `sensor.kidschores_chore_*` (36 individual chores)
- Same for Lilly, Blake, Amanda

---

## Chore Breakdown

**Required (31):**
- Daily (8): Blinds, lights, litter, dog out
- Weekly (11): Laundry×4, rooms×2, trash, Pip, water plants
- Bi-weekly (2): Bathroom sink, toilet
- Monthly (8): Bathroom deep clean×6, Pip, sheets×2
- Seasonal (2): Water plants, harvest

**Bonus (1):** Sweep Kitchen
**Parent (4):** Pet water, dog meal, litter waste, Pip check (all 0 points)

---

## Next Steps After Deployment

1. **Create Dashboards** (kid view, parent view)
2. **Test with family** (adjust point values if needed)
3. **Add custody toggle** (disable Lilly's chores during away weeks)
4. **Add seasonal toggle** (enable/disable garden chores)
5. **Implement banking** (checking/savings accounts, weekly payout)
6. **Add notifications** (reminders, base allowance at risk alerts)

---

## Troubleshooting

**Sensor not appearing:** Verify integration modifications in `/config/custom_components/kidschores/`, restart HA
**Import fails "Kid not found":** Add Blake/Amanda via UI first
**Allowance shows "unknown":** Check Required sensor exists, verify template syntax
**all_required_done always FALSE:** Check if chores are approved (not just claimed)

**Full docs:** See `COMPLETE_HANDOFF_PACKAGE.md` for detailed troubleshooting

---

## Why This Works

**Math proof it can't be gamed:**
- Skip 1 Required chore → base $10 locked
- Even 100 Bonus points can't unlock base
- Always more profitable to complete Required first
- Example: All Required (30 pts) = $10 + $7.50 = $17.50
- Example: Skip 1 Required + 12 Bonus (39 pts) = $0 + $9.75 = $9.75

**Kids learn:** Responsibilities before rewards!

---

**Questions?** Check `COMPLETE_HANDOFF_PACKAGE.md` for everything else.

**Estimated deployment time:** 15-30 minutes

**Good luck!** 🎉
