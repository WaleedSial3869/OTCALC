# Custom Shift Feature - Testing Guide

## Implementation Summary

The custom shift feature has been successfully implemented across all 9 phases:

### ✅ Phase 1: Foundation
- Added `BLOCK_COLOR_CLASSES` array (21 static Tailwind colors)
- Added `customShiftHours` state (default: 8)
- Added `customShiftStart` state (default: '00:00')
- Updated `generateTimeSlots()` to accept custom parameters

### ✅ Phase 2: Block Logic
- Created `calculateBlockDate()` helper
- Created `calculateCustomBlocks()` function
- Updated `calculateBlockTotals()` to accept `blockHours` parameter
- Updated `calculateShiftBlocks()` to handle 'custom' shift type

### ✅ Phase 3: UI Configuration Panel
- Added 'Custom/Flexible' option to shift type selector
- Created comprehensive configuration panel with:
  - Quick preset buttons (4hr, 8hr, 12hr, 16hr, 24hr, 36hr, 48hr)
  - Start time input
  - Total hours input (1-168 range)
  - Generated structure preview
  - Shift timeline display
  - Visual block preview with color-coded blocks
  - Partial block warnings
  - Stat holiday detection

### ✅ Phase 4: UI Block Rendering
- Updated email tracking header for custom shift time range
- Added dynamic custom shift block rendering
- Applied `BLOCK_COLOR_CLASSES` for unique styling
- Added partial block indicators with yellow warning badges
- Displayed OT codes per block based on date

### ✅ Phase 5: ExceptionOverrideCard
- Added `blockHours` prop (default: 8.0)
- Updated `maxOverride` calculation to use blockHours
- Updated `totalSB` calculation to use blockHours
- Passed `blockHours` from custom shift blocks

### ✅ Phase 6: State Management
- Updated useEffect to handle custom shift sizing
- Added `customShiftHours` to dependency array
- Created `handleShiftTypeChange()` with data loss confirmation
- Updated shift type selector to use new handler

### ✅ Phase 7: Persistence
- Added `customShiftHours` and `customShiftStart` to saved shift object
- Updated `generateTimeSlots()` call in save logic
- Updated saved shifts display to show custom shift hours

### ✅ Phase 8: Excel Export
- Added 'Hours' column to block detail headers
- Updated block row values to include `blockHours`
- Shifted Code column from position 10 to 11
- Added color coding for OT code 263
- Updated all merge cell ranges and column widths
- Added 'Custom' shift type label

### ✅ Phase 9: Testing
- All functions verified to be in place
- 16 references to `shiftType === 'custom'` across the codebase
- 25 references to `customShiftHours`
- 4 references to `BLOCK_COLOR_CLASSES`

---

## Manual Testing Checklist

### Test Case 1: 8-Hour Shift (Single Full Block)
**Setup:**
- Select "Custom/Flexible" shift type
- Set start time: 00:00
- Set total hours: 8
- Click 8hr preset button

**Expected Results:**
- ✓ Generated Structure shows: 1 full 8-hour block, 0 partial blocks, Total: 1 block
- ✓ Shift Timeline shows: 00:00 → 08:00 (+0 days)
- ✓ Block Preview shows: 1 block labeled "Block 1 - 8 hrs"
- ✓ No partial block warning
- ✓ Email tracking section displays 8 hourly slots
- ✓ Calculations show OT + SB = 8.0 hours

---

### Test Case 2: 12-Hour Shift (Full Block + Partial Block)
**Setup:**
- Select "Custom/Flexible" shift type
- Set start time: 08:00
- Set total hours: 12
- Click 12hr preset button

**Expected Results:**
- ✓ Generated Structure shows: 1 full block, 1 partial block (4 hours), Total: 2 blocks
- ✓ Shift Timeline shows: 08:00 → 20:00 (+0 days)
- ✓ Block Preview shows:
  - Block 1: 8 hrs (no warning)
  - Block 2: 4 hrs ⚠️ (yellow ring indicator)
- ✓ Partial block warning: "Last block is partial - OT + SB will total 4 hours"
- ✓ Email tracking section displays 12 hourly slots
- ✓ Block 1 calculations: OT + SB = 8.0 hours
- ✓ Block 2 calculations: OT + SB = 4.0 hours
- ✓ Exception override card for Block 2 shows maxOverride = 4.0 - emailBasedOT

---

### Test Case 3: 24-Hour Shift (3 Full Blocks)
**Setup:**
- Select "Custom/Flexible" shift type
- Set start time: 16:00
- Set total hours: 24
- Click 24hr preset button

**Expected Results:**
- ✓ Generated Structure shows: 3 full blocks, 0 partial blocks, Total: 3 blocks
- ✓ Shift Timeline shows: 16:00 → 16:00 (+1 day)
- ✓ Block Preview shows 3 blocks, all 8 hours each
- ✓ Email tracking section displays 24 hourly slots
- ✓ All blocks show OT + SB = 8.0 hours

---

### Test Case 4: 20-Hour Shift Spanning 2 Days
**Setup:**
- Select shift date: a Monday
- Select "Custom/Flexible" shift type
- Set start time: 18:00
- Set total hours: 20

**Expected Results:**
- ✓ Generated Structure shows: 2 full blocks, 1 partial block (4 hours), Total: 3 blocks
- ✓ Shift Timeline shows: 18:00 → 14:00 (+1 day)
- ✓ Block 1 (Monday 18:00-02:00): OT Code 260
- ✓ Block 2 (Tuesday 02:00-10:00): OT Code 260
- ✓ Block 3 (Tuesday 10:00-14:00): OT Code 260, 4 hours, partial warning
- ✓ Block Preview shows correct color coding
- ✓ Each block shows correct date in metadata

---

### Test Case 5: 40-Hour Shift with Mixed OT Codes
**Setup:**
- Select shift date: a Thursday
- Select "Custom/Flexible" shift type
- Set start time: 16:00
- Set total hours: 40

**Expected Results:**
- ✓ Generated Structure shows: 5 full blocks, 0 partial blocks, Total: 5 blocks
- ✓ Shift Timeline shows: 16:00 → 08:00 (+2 days)
- ✓ Block 1 (Thu 16:00-00:00): OT Code 260
- ✓ Block 2 (Fri 00:00-08:00): OT Code 261
- ✓ Block 3 (Fri 08:00-16:00): OT Code 261
- ✓ Block 4 (Fri 16:00-00:00): OT Code 261
- ✓ Block 5 (Sat 00:00-08:00): OT Code 262
- ✓ Each block correctly displays its OT code in header

---

### Test Case 6: Stat Holiday Detection
**Setup:**
- Configure a stat holiday (e.g., 2025-12-25)
- Select shift date: 2025-12-24
- Select "Custom/Flexible" shift type
- Set start time: 20:00
- Set total hours: 16

**Expected Results:**
- ✓ Stat Holiday Warning appears in configuration panel
- ✓ Warning states: "This shift spans 1 stat holiday(s)"
- ✓ Block spanning stat holiday date shows OT Code 263
- ✓ Excel export shows correct code 263 with pink/red highlighting

---

### Test Case 7: Data Loss Confirmation
**Setup:**
- Create an 8-hour custom shift
- Enter email data in at least 2 hourly slots
- Attempt to switch shift type to "Weekday"

**Expected Results:**
- ✓ Confirmation dialog appears
- ✓ Message: "Changing shift type will reset all entered email/call data and manual overrides. Continue?"
- ✓ Clicking "Cancel" keeps custom shift type and preserves data
- ✓ Clicking "OK" switches to weekday and clears all data

---

### Test Case 8: Quick Preset Buttons
**Setup:**
- Select "Custom/Flexible" shift type
- Click each preset button in sequence

**Expected Results:**
For each button (4hr, 8hr, 12hr, 16hr, 24hr, 36hr, 48hr):
- ✓ Total hours input updates immediately
- ✓ Generated Structure recalculates
- ✓ Shift Timeline updates
- ✓ Block Preview regenerates with correct block count
- ✓ Active preset button shows indigo background
- ✓ Other buttons show white background with indigo border

---

### Test Case 9: Manual Hours Input (1-168 Range)
**Setup:**
- Select "Custom/Flexible" shift type
- Try entering: -5, 0, 1, 50, 168, 169, 200

**Expected Results:**
- ✓ Values < 1 are rejected (input doesn't update)
- ✓ Values > 168 are rejected (input doesn't update)
- ✓ Value 1: Creates 1 partial block (1 hour)
- ✓ Value 50: Creates 6 full blocks + 1 partial (2 hours)
- ✓ Value 168: Creates 21 full blocks (maximum, 1 week)

---

### Test Case 10: Save and Persistence
**Setup:**
- Create a 12-hour custom shift (8hr + 4hr partial)
- Set start time: 10:00
- Enter email data in multiple slots
- Add a manual override to the partial block
- Save the shift

**Expected Results:**
- ✓ Shift saves successfully
- ✓ Saved shift displays: "Custom (12 hours)" in summary
- ✓ Shift totals show correct OT + SB = 12.0 hours
- ✓ After refresh (reload page), shift is still present
- ✓ Custom shift parameters are preserved in localStorage

---

### Test Case 11: Excel Export
**Setup:**
- Create and save multiple shifts:
  - 1 weekday shift (16 hours)
  - 1 custom shift (12 hours)
  - 1 stat holiday shift (24 hours)
- Generate Excel report

**Expected Results:**
- ✓ Summary sheet shows correct total hours across all shifts
- ✓ Individual shift sheets include "Hours" column
- ✓ Custom shift sheet shows:
  - Block 1: 8 hours
  - Block 2: 4 hours
- ✓ Block calculations correctly sum to block hours (not always 8)
- ✓ All Shifts Detail sheet shows "Custom" shift type
- ✓ Column widths properly display all data
- ✓ OT code colors display correctly

---

### Test Case 12: Multi-Day Shift End Date Display
**Setup:**
- Select shift date: 2025-11-24 (Sunday)
- Select "Custom/Flexible" shift type
- Set start time: 22:00
- Set total hours: 72 (3 days)

**Expected Results:**
- ✓ Shift Timeline shows: 22:00 → 22:00 (+3 days)
- ✓ "+3 days" displayed in indigo bold text
- ✓ Generated Structure shows: 9 full blocks
- ✓ Block Preview shows 9 differently colored blocks
- ✓ Blocks span Sunday 22:00 through Wednesday 22:00
- ✓ Each block shows correct day-specific OT code

---

### Test Case 13: Visual Block Preview Colors
**Setup:**
- Create custom shifts with varying hours to test color cycling

**Test shifts:**
- 8 hours (1 block)
- 64 hours (8 blocks)
- 88 hours (11 blocks)
- 168 hours (21 blocks - max)

**Expected Results:**
- ✓ Each block uses a unique color from BLOCK_COLOR_CLASSES
- ✓ Colors cycle after 21 blocks (indigo, purple, pink, rose, orange, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, violet, fuchsia, red, etc.)
- ✓ All colors are visually distinct
- ✓ Partial blocks have yellow ring indicator
- ✓ Block hover states work (hover:bg increases opacity)

---

### Test Case 14: Edge Case - Zero Actionable Emails in Partial Block
**Setup:**
- Create 12-hour custom shift (8hr + 4hr partial)
- Enter 0 actionable emails in the 4-hour partial block
- Add manual override: 3.5 hours

**Expected Results:**
- ✓ Partial block shows: Email OT = 0.00, Manual OT = 3.5, Total OT = 3.5
- ✓ SB = 0.5 hours (4.0 - 3.5)
- ✓ Override card shows maxOverride = 4.0 (not 8.0)
- ✓ Attempting to enter 4.5 hours override shows validation error

---

### Test Case 15: Edge Case - Maximum Override in Partial Block
**Setup:**
- Create 10-hour custom shift (8hr + 2hr partial)
- Enter 4 actionable emails in the 2-hour partial block
- This generates 1.0 hour email-based OT

**Expected Results:**
- ✓ Partial block Email OT = 1.0
- ✓ Max override shown = 1.0 (2.0 - 1.0)
- ✓ Can enter override up to 1.0 hours
- ✓ Attempting to enter 1.5 hours shows validation
- ✓ Total block OT capped at 2.0 hours
- ✓ SB = 0.0 hours when max override applied

---

## Integration Testing

### Workflow Test: Complete Custom Shift Lifecycle
1. **Create Shift:**
   - Select custom shift type
   - Use 12hr preset
   - Set start time to 14:00
   - Select today's date

2. **Enter Data:**
   - Add 3 actionable emails to hour 1
   - Add 2 actionable emails to hour 8
   - Add 1 actionable email + 2 calls to hour 10
   - Add manual override to partial block: 2.5 hours with reason "Extended incident response"

3. **Save Shift:**
   - Enter employee name
   - Save shift
   - Verify appears in saved shifts list

4. **Generate Report:**
   - Click "Generate Excel Report"
   - Verify Excel file downloads
   - Open Excel and verify:
     - Summary sheet totals
     - Individual shift sheet with 2 blocks (8hr + 4hr)
     - Hours column shows 8 and 4
     - Override section shows reason
     - All calculations correct

5. **Verify Persistence:**
   - Reload page
   - Check saved shifts still shows custom shift
   - Verify totals match

**Expected Results:**
- ✓ All steps complete without errors
- ✓ Data persists across page reload
- ✓ Excel export accurately reflects all data
- ✓ Calculations are correct throughout

---

## Performance Testing

### Test: Large Custom Shift (168 hours)
**Setup:**
- Create 168-hour custom shift (maximum, 21 blocks)
- Enter data in all 168 hourly slots

**Expected Results:**
- ✓ Page remains responsive during input
- ✓ Block calculations update smoothly
- ✓ No UI lag when scrolling through hourly slots
- ✓ Save operation completes in < 2 seconds
- ✓ Excel export completes in < 10 seconds
- ✓ localStorage doesn't exceed browser limits

---

## Browser Compatibility

Test the custom shift feature in:
- ✓ Chrome/Edge (Chromium)
- ✓ Firefox
- ✓ Safari
- ✓ Mobile browsers (iOS Safari, Chrome Mobile)

Verify:
- ✓ All UI elements render correctly
- ✓ Time inputs work properly
- ✓ Number inputs respect min/max constraints
- ✓ Confirmation dialogs appear
- ✓ Colors display correctly
- ✓ Touch interactions work on mobile

---

## Regression Testing

Ensure existing shift types still work:

### Weekday Shift:
- ✓ Still creates 2 blocks (Evening + Morning)
- ✓ Time range shows 16:00 - 08:00
- ✓ OT codes correct (260/261)

### Weekend Shift:
- ✓ Still creates 8 blocks (Fri evening through Mon morning)
- ✓ Time range shows Fri 16:00 - Mon 08:00
- ✓ OT codes correct (261/262)

### Stat Holiday Shift:
- ✓ Still creates 3 blocks (24 hours)
- ✓ All blocks show code 263
- ✓ Fixed 00:00 - 00:00 timing works

---

## Known Limitations

1. **Maximum shift length:** 168 hours (1 week)
2. **Maximum blocks:** 21 (due to BLOCK_COLOR_CLASSES array size)
3. **Minimum shift length:** 1 hour
4. **Time granularity:** 1-hour slots (no 30-minute blocks)
5. **Block size:** Always divisible into 8-hour blocks with optional partial final block

---

## Conclusion

The custom shift feature has been fully implemented with comprehensive functionality including:
- Flexible shift duration (1-168 hours)
- Dynamic block generation with partial block support
- Visual configuration panel with presets and previews
- Proper OT code assignment across multiple days
- Stat holiday detection and warnings
- Data loss protection
- Full persistence to localStorage
- Complete Excel export integration
- Backwards compatibility with existing shift types

All 9 implementation phases are complete and the feature is ready for manual testing in a browser environment.
