# Implementation Plan: Custom/Flexible Shift Type

## Executive Summary

Add a new **"Custom/Flexible"** shift type to the PHAC Overtime Calculator that allows users to define shifts with arbitrary durations (1-168 hours). The system will automatically generate the appropriate number of 8-hour blocks, with the final block being partial if needed.

**Key Requirement:** All existing OT/SB calculation logic must remain unchanged. The new shift type integrates seamlessly with the existing block-based architecture.

---

## Current Architecture Analysis

### Existing Shift Types

| Shift Type | Duration | Blocks | Time Slots | Block Size |
|------------|----------|--------|------------|------------|
| Weekday | 16 hours | 2 | 16 | 8 hours each |
| Weekend | 64 hours | 8 | 64 | 8 hours each |
| Stat Holiday | 24 hours | 3 | 24 | 8 hours each |

### Current Flow

```
User selects shift type (fixed) → System generates fixed time slots
→ System creates fixed blocks → User enters email data
→ System calculates OT/SB per block → Save shift
```

### Proposed Flow for Custom Shifts

```
User selects "Custom" → User enters duration & start time
→ System generates dynamic time slots → System creates dynamic blocks
→ User enters email data → System calculates OT/SB per block
→ Save shift (with custom metadata)
```

---

## Design Decisions

### Decision 1: Partial Blocks
**Question:** What happens if hours don't divide evenly by 8?

**Options:**
- **A) Allow partial final block** (e.g., 12 hours = 8hr block + 4hr block)
- **B) Round up to full blocks** (e.g., 12 hours → 16 hours = 2 full blocks)

**Recommendation:** **Option A** - Allow partial blocks
- **Reason:** Provides accurate compensation without inflating hours
- **Implementation:** Final block with `blockHours < 8` where `OT + SB = blockHours`

**Example:**
```
Custom shift: 12 hours starting at 08:00

Block 1: 08:00-16:00 (8 hours) → Normal calculation (OT + SB = 8.0)
Block 2: 16:00-20:00 (4 hours) → Proportional (OT + SB = 4.0)
```

---

### Decision 2: Multi-Day Custom Shifts
**Question:** How to handle overtime codes for shifts spanning multiple days?

**Example:** 36-hour custom shift starting Monday 08:00
- Hours 0-8: Monday morning → Code 260
- Hours 8-16: Monday afternoon → Code 260
- Hours 16-24: Monday evening → Code 260
- Hours 24-32: Tuesday morning → Code 260
- Hours 32-36: Tuesday morning → Code 260

**Recommendation:** Each block gets the OT code for the day/time it occurs
- Calculate the actual date/time for each block
- Apply existing `getOvertimeCode()` logic

---

### Decision 3: Start Time Handling
**Question:** Should custom shifts allow any start time, or only on the hour?

**Recommendation:** **On-the-hour only** (e.g., 08:00, 14:00, 23:00)
- **Reason:** Simplifies time slot generation and block alignment
- **User Impact:** Minimal - most shifts start on the hour anyway

---

### Decision 4: Maximum Duration
**Question:** What's the maximum allowed custom shift duration?

**Recommendation:** **168 hours (1 week)**
- **Reason:** Prevents extreme cases, aligns with government on-call limits
- **Validation:** Show error if user enters > 168

---

## Implementation Plan

### Phase 1: Core Data Structure Updates

#### 1.1 Add Custom Shift State Variables

**Location:** `MonthlyOTTracker` component, state declarations

```javascript
// Add after existing state variables
const [customShiftHours, setCustomShiftHours] = useState(8); // Default 8 hours
const [customShiftStart, setCustomShiftStart] = useState('00:00'); // Default midnight
```

**Why:** Stores user-defined shift parameters

---

#### 1.2 Update Shift Type Constants

**Location:** Helper functions section (top of script)

```javascript
// Add new constant for shift type metadata
const SHIFT_TYPE_CONFIG = {
  weekday: {
    label: 'Weekday (16hr: Mon-Thu)',
    fixedHours: 16,
    fixedBlocks: 2,
    fixedSlots: 16,
    defaultStart: '16:00',
    defaultEnd: '00:00'
  },
  weekend: {
    label: 'Weekend (64hr: Fri-Mon)',
    fixedHours: 64,
    fixedBlocks: 8,
    fixedSlots: 64,
    defaultStart: '16:00',
    defaultEnd: '08:00'
  },
  stat_holiday: {
    label: 'Stat Holiday (24hr: 00:00-00:00)',
    fixedHours: 24,
    fixedBlocks: 3,
    fixedSlots: 24,
    defaultStart: '00:00',
    defaultEnd: '00:00'
  },
  custom: {
    label: 'Custom/Flexible (specify hours)',
    fixedHours: null, // Dynamic
    fixedBlocks: null, // Dynamic
    fixedSlots: null, // Dynamic
    defaultStart: '00:00',
    defaultEnd: null // Calculated
  }
};
```

**Why:** Centralizes shift type configuration, makes it easier to add new types

---

### Phase 2: Time Slot Generation

#### 2.1 Update `generateTimeSlots()` Function

**Location:** Helper functions section

**Current function:**
```javascript
const generateTimeSlots = (shiftType) => {
  if (shiftType === 'weekday') {
    return [ /* 16 slots */ ];
  } else if (shiftType === 'stat_holiday') {
    return [ /* 24 slots */ ];
  } else {
    return [ /* 64 slots */ ];
  }
};
```

**Updated function:**
```javascript
const generateTimeSlots = (shiftType, customHours = null, customStart = '00:00') => {
  // Handle custom shifts
  if (shiftType === 'custom' && customHours) {
    const slots = [];
    const [startHour, startMin] = customStart.split(':').map(Number);

    for (let i = 0; i < customHours; i++) {
      const currentHour = (startHour + i) % 24;
      const nextHour = (startHour + i + 1) % 24;
      const start = `${String(currentHour).padStart(2, '0')}:00`;
      const end = `${String(nextHour).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }
    return slots;
  }

  // Existing logic for other shift types
  if (shiftType === 'weekday') {
    return [
      '16:00-17:00', '17:00-18:00', '18:00-19:00', '19:00-20:00',
      '20:00-21:00', '21:00-22:00', '22:00-23:00', '23:00-00:00',
      '00:00-01:00', '01:00-02:00', '02:00-03:00', '03:00-04:00',
      '04:00-05:00', '05:00-06:00', '06:00-07:00', '07:00-08:00'
    ];
  } else if (shiftType === 'stat_holiday') {
    const slots = [];
    for (let i = 0; i < 24; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = i === 23 ? '00:00' : `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }
    return slots;
  } else {
    // Weekend shift (64 hours) - existing logic
    const slots = [];
    // ... existing weekend slot generation
    return slots;
  }
};
```

**Why:** Generates hourly time slots starting from user-specified time

**Example Output:**
```javascript
generateTimeSlots('custom', 12, '08:00')
// Returns:
[
  '08:00-09:00', '09:00-10:00', '10:00-11:00', '11:00-12:00',
  '12:00-13:00', '13:00-14:00', '14:00-15:00', '15:00-16:00',
  '16:00-17:00', '17:00-18:00', '18:00-19:00', '19:00-20:00'
]
```

---

### Phase 3: Block Structure Generation

#### 3.1 Create `calculateCustomBlocks()` Helper Function

**Location:** Helper functions section (add new function)

```javascript
/**
 * Generates block structure for custom shifts
 * @param {number} totalHours - Total shift duration (1-168)
 * @param {string} startTime - Shift start time (HH:MM format)
 * @param {string} shiftDate - Shift date (YYYY-MM-DD)
 * @returns {Array} Array of block metadata objects
 */
const calculateCustomBlocks = (totalHours, startTime = '00:00', shiftDate = '') => {
  const fullBlocks = Math.floor(totalHours / 8);
  const remainingHours = totalHours % 8;
  const blocks = [];

  const [startHour, startMin] = startTime.split(':').map(Number);

  // Generate full 8-hour blocks
  for (let i = 0; i < fullBlocks; i++) {
    const hoursIntoShift = i * 8;
    const blockStartHour = startHour + hoursIntoShift;
    const blockEndHour = blockStartHour + 8;

    // Calculate which day this block falls on
    const daysOffset = Math.floor(blockStartHour / 24);
    const blockStartHourMod = blockStartHour % 24;
    const blockEndHourMod = blockEndHour % 24;

    blocks.push({
      name: `Block ${i + 1}`,
      timeRange: `${String(blockStartHourMod).padStart(2, '0')}:00-${String(blockEndHourMod).padStart(2, '0')}:00`,
      hours: 8,
      startSlotIndex: i * 8,
      endSlotIndex: (i + 1) * 8,
      isPartial: false,
      daysOffset: daysOffset,
      blockDate: calculateBlockDate(shiftDate, daysOffset)
    });
  }

  // Add partial final block if needed
  if (remainingHours > 0) {
    const hoursIntoShift = fullBlocks * 8;
    const blockStartHour = startHour + hoursIntoShift;
    const blockEndHour = blockStartHour + remainingHours;

    const daysOffset = Math.floor(blockStartHour / 24);
    const blockStartHourMod = blockStartHour % 24;
    const blockEndHourMod = blockEndHour % 24;

    blocks.push({
      name: `Block ${fullBlocks + 1}`,
      timeRange: `${String(blockStartHourMod).padStart(2, '0')}:00-${String(blockEndHourMod).padStart(2, '0')}:00`,
      hours: remainingHours,
      startSlotIndex: fullBlocks * 8,
      endSlotIndex: fullBlocks * 8 + remainingHours,
      isPartial: true,
      daysOffset: daysOffset,
      blockDate: calculateBlockDate(shiftDate, daysOffset)
    });
  }

  return blocks;
};

/**
 * Helper: Calculate block date based on shift start date and days offset
 */
const calculateBlockDate = (shiftDate, daysOffset) => {
  if (!shiftDate) return '';
  const date = new Date(shiftDate + 'T12:00:00');
  date.setDate(date.getDate() + daysOffset);
  return date.toISOString().split('T')[0];
};
```

**Why:** Creates block metadata for dynamic rendering and OT code calculation

**Example Output:**
```javascript
calculateCustomBlocks(12, '08:00', '2025-10-20')
// Returns:
[
  {
    name: 'Block 1',
    timeRange: '08:00-16:00',
    hours: 8,
    startSlotIndex: 0,
    endSlotIndex: 8,
    isPartial: false,
    daysOffset: 0,
    blockDate: '2025-10-20'
  },
  {
    name: 'Block 2',
    timeRange: '16:00-20:00',
    hours: 4,
    startSlotIndex: 8,
    endSlotIndex: 12,
    isPartial: true,
    daysOffset: 0,
    blockDate: '2025-10-20'
  }
]
```

---

### Phase 4: Block Calculation Updates

#### 4.1 Modify `calculateBlockTotals()` Function

**Current signature:**
```javascript
const calculateBlockTotals = (emailData, startIdx, endIdx, blockOverride = { manualOT: 0, reason: '' }) => {
  // ...
  const totalBlockOT = Math.min(workloadOT + manualOverrideOT, 8.0);
  const blockSB = 8.0 - totalBlockOT;
  // ...
}
```

**Updated signature:**
```javascript
const calculateBlockTotals = (
  emailData,
  startIdx,
  endIdx,
  blockOverride = { manualOT: 0, reason: '' },
  blockHours = 8 // NEW PARAMETER: defaults to 8 for existing shifts
) => {
  let totalActionable = 0;
  let totalEmails = 0;
  let totalCalls = 0;

  for (let i = startIdx; i < endIdx; i++) {
    totalActionable += emailData[i]?.actionable || 0;
    totalEmails += emailData[i]?.total || 0;
    totalCalls += emailData[i]?.calls || 0;
  }

  const emailBasedOT = totalActionable * 0.25;
  const callBasedOT = totalCalls * 0.25;
  const workloadOT = Math.min(emailBasedOT + callBasedOT, blockHours); // Use blockHours cap
  const manualOverrideOT = blockOverride.manualOT || 0;
  const totalBlockOT = Math.min(workloadOT + manualOverrideOT, blockHours); // Use blockHours cap
  const blockSB = blockHours - totalBlockOT; // Use blockHours for SB calculation

  return {
    totalActionable,
    totalEmails,
    totalCalls,
    emailBasedOT: Math.round(emailBasedOT * 100) / 100,
    callBasedOT: Math.round(callBasedOT * 100) / 100,
    workloadOT: Math.round(workloadOT * 100) / 100,
    manualOverrideOT: Math.round(manualOverrideOT * 100) / 100,
    totalBlockOT: Math.round(totalBlockOT * 100) / 100,
    blockSB: Math.round(blockSB * 100) / 100,
    blockHours: blockHours, // NEW: Track block size
    overrideReason: blockOverride.reason || '',
    hasOverride: manualOverrideOT > 0
  };
};
```

**Key Changes:**
1. Add `blockHours` parameter (defaults to 8 for backward compatibility)
2. Use `blockHours` instead of hardcoded `8.0` for caps
3. Return `blockHours` in result object

**Why:** Allows partial blocks to calculate OT/SB correctly

**Example:**
```javascript
// Full 8-hour block
calculateBlockTotals(emailData, 0, 8, {}, 8)
// 10 actionable emails → 2.5h OT, 5.5h SB (total = 8.0)

// Partial 4-hour block
calculateBlockTotals(emailData, 8, 12, {}, 4)
// 10 actionable emails → 2.5h OT, 1.5h SB (total = 4.0)
```

---

#### 4.2 Update `calculateShiftBlocks()` Function

**Location:** Inside `MonthlyOTTracker` component

**Current logic:**
```javascript
const calculateShiftBlocks = () => {
  const blocks = [];

  if (shiftType === 'weekday') {
    blocks.push({
      name: 'Evening',
      timeRange: '16:00-23:59',
      ...calculateBlockTotals(emailData, 0, 8, blockOverrides[0]),
      code: getOvertimeCode(shiftDate, userSettings.statHolidays)
    });
    // ... more blocks
  }
  // ... other shift types

  return blocks;
};
```

**Updated logic:**
```javascript
const calculateShiftBlocks = () => {
  const blocks = [];

  if (shiftType === 'weekday') {
    // Existing weekday logic (unchanged)
    blocks.push({
      name: 'Evening',
      timeRange: '16:00-23:59',
      ...calculateBlockTotals(emailData, 0, 8, blockOverrides[0], 8), // Explicit 8
      code: getOvertimeCode(shiftDate, userSettings.statHolidays)
    });
    blocks.push({
      name: 'Morning',
      timeRange: '00:00-08:00',
      ...calculateBlockTotals(emailData, 8, 16, blockOverrides[1], 8), // Explicit 8
      code: getOvertimeCode(shiftDate, userSettings.statHolidays)
    });
  } else if (shiftType === 'stat_holiday') {
    // Existing stat holiday logic (unchanged)
    blocks.push({
      name: 'Block 1',
      timeRange: '00:00-08:00',
      ...calculateBlockTotals(emailData, 0, 8, blockOverrides[0], 8),
      code: '263'
    });
    blocks.push({
      name: 'Block 2',
      timeRange: '08:00-16:00',
      ...calculateBlockTotals(emailData, 8, 16, blockOverrides[1], 8),
      code: '263'
    });
    blocks.push({
      name: 'Block 3',
      timeRange: '16:00-00:00',
      ...calculateBlockTotals(emailData, 16, 24, blockOverrides[2], 8),
      code: '263'
    });
  } else if (shiftType === 'weekend') {
    // Existing weekend logic (unchanged) - 8 blocks
    const blockNames = [
      { name: 'Friday Evening', range: '16:00-23:59', start: 0 },
      // ... rest of blocks
    ];

    blockNames.forEach((block, idx) => {
      blocks.push({
        name: block.name,
        timeRange: block.range,
        ...calculateBlockTotals(emailData, block.start, block.start + 8, blockOverrides[idx], 8),
        code: getWeekendBlockCode(idx)
      });
    });
  } else if (shiftType === 'custom') {
    // NEW: Custom shift logic
    const customBlocks = calculateCustomBlocks(customShiftHours, customShiftStart, shiftDate);

    customBlocks.forEach((blockMeta, idx) => {
      blocks.push({
        name: blockMeta.name,
        timeRange: blockMeta.timeRange,
        ...calculateBlockTotals(
          emailData,
          blockMeta.startSlotIndex,
          blockMeta.endSlotIndex,
          blockOverrides[idx] || { manualOT: 0, reason: '' },
          blockMeta.hours // Pass actual block hours (8 or less)
        ),
        code: getOvertimeCode(blockMeta.blockDate, userSettings.statHolidays),
        isPartial: blockMeta.isPartial,
        blockDate: blockMeta.blockDate
      });
    });
  }

  return blocks;
};
```

**Why:** Integrates custom shift block generation with existing calculation logic

---

### Phase 5: Overtime Code Logic

#### 5.1 Update `getOvertimeCode()` Function (No Changes Needed!)

**Current function:**
```javascript
const getOvertimeCode = (shiftDate, statHolidays = []) => {
  if (!shiftDate) return '260';

  // Check if this date is a stat holiday
  if (statHolidays.includes(shiftDate)) {
    return '263';
  }

  const date = new Date(shiftDate + 'T12:00:00');
  const dayOfWeek = date.getDay();

  if (dayOfWeek >= 1 && dayOfWeek <= 4) return '260';
  if (dayOfWeek === 5) return '261';
  if (dayOfWeek === 0 || dayOfWeek === 6) return '262';
  return '260';
};
```

**Why it works for custom shifts:**
- Each block in `calculateShiftBlocks()` passes its own `blockDate`
- Multi-day custom shifts automatically get correct OT codes per block
- No modification needed! ✅

**Example:**
```javascript
// 36-hour custom shift starting Monday Oct 20, 08:00
Block 1 (Mon 08:00-16:00): blockDate='2025-10-20' → Code 260
Block 2 (Mon 16:00-00:00): blockDate='2025-10-20' → Code 260
Block 3 (Tue 00:00-08:00): blockDate='2025-10-21' → Code 260
Block 4 (Tue 08:00-16:00): blockDate='2025-10-21' → Code 260
Block 5 (Tue 16:00-20:00): blockDate='2025-10-21' → Code 260 (4hr partial)
```

---

### Phase 6: UI Updates

#### 6.1 Update Shift Type Selector

**Location:** Shift Information section in JSX

**Current code:**
```javascript
<select
  value={shiftType}
  onChange={(e) => setShiftType(e.target.value)}
  className="w-full border border-gray-300 rounded px-3 py-2"
>
  <option value="weekday">Weekday (16hr: Mon-Thu)</option>
  <option value="weekend">Weekend (64hr: Fri-Mon)</option>
  <option value="stat_holiday">Stat Holiday (24hr: 00:00-00:00)</option>
</select>
```

**Updated code:**
```javascript
<select
  value={shiftType}
  onChange={(e) => setShiftType(e.target.value)}
  className="w-full border border-gray-300 rounded px-3 py-2"
>
  <option value="weekday">Weekday (16hr: Mon-Thu)</option>
  <option value="weekend">Weekend (64hr: Fri-Mon)</option>
  <option value="stat_holiday">Stat Holiday (24hr: 00:00-00:00)</option>
  <option value="custom">Custom/Flexible (specify hours)</option>
</select>
```

---

#### 6.2 Add Custom Shift Configuration Panel

**Location:** After shift type selector, before hourly tracking section

```javascript
{/* NEW: Custom Shift Configuration Panel */}
{shiftType === 'custom' && (
  <div className="bg-gradient-to-r from-indigo-50 to-purple-50 border-2 border-indigo-300 p-4 rounded-lg mb-4">
    <div className="flex items-center gap-2 mb-3">
      <Clock />
      <h3 className="font-bold text-indigo-800 text-lg">Custom Shift Configuration</h3>
    </div>

    <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
      {/* Start Time Input */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Shift Start Time <span className="text-red-500">*</span>
        </label>
        <input
          type="time"
          value={customShiftStart}
          onChange={(e) => setCustomShiftStart(e.target.value)}
          className="w-full border-2 border-indigo-300 rounded px-3 py-2 focus:border-indigo-500 focus:ring-2 focus:ring-indigo-200"
        />
        <p className="text-xs text-gray-600 mt-1">
          When does the shift start?
        </p>
      </div>

      {/* Total Hours Input */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Total Hours <span className="text-red-500">*</span>
        </label>
        <input
          type="number"
          min="1"
          max="168"
          step="1"
          value={customShiftHours}
          onChange={(e) => {
            const hours = parseInt(e.target.value) || 8;
            if (hours < 1) {
              alert('⚠️ Minimum 1 hour');
              setCustomShiftHours(1);
            } else if (hours > 168) {
              alert('⚠️ Maximum 168 hours (1 week)');
              setCustomShiftHours(168);
            } else {
              setCustomShiftHours(hours);
            }
          }}
          className="w-full border-2 border-indigo-300 rounded px-3 py-2 text-lg font-bold focus:border-indigo-500 focus:ring-2 focus:ring-indigo-200"
        />
        <p className="text-xs text-gray-600 mt-1">
          1-168 hours (1 hour to 1 week)
        </p>
      </div>

      {/* Generated Structure Preview */}
      <div className="flex flex-col justify-end">
        <div className="bg-white border-2 border-indigo-300 rounded-lg p-3">
          <div className="text-xs font-medium text-gray-600 mb-1">Generated Structure:</div>
          <div className="text-lg font-bold text-indigo-700">
            {Math.floor(customShiftHours / 8) === 0 ? (
              <>1 partial block ({customShiftHours}hr)</>
            ) : (
              <>
                {Math.floor(customShiftHours / 8)} full block{Math.floor(customShiftHours / 8) > 1 ? 's' : ''}
                {customShiftHours % 8 > 0 && (
                  <> + 1 partial ({customShiftHours % 8}hr)</>
                )}
              </>
            )}
          </div>
          <div className="text-xs text-gray-600 mt-1">
            Total: {customShiftHours} hourly slots
          </div>
        </div>
      </div>
    </div>

    {/* Shift End Time Display */}
    <div className="mt-3 bg-white border border-indigo-200 rounded p-3">
      <div className="text-sm text-gray-700">
        <strong>Shift Timeline:</strong> {customShiftStart} →
        {(() => {
          const [startHour] = customShiftStart.split(':').map(Number);
          const endHour = (startHour + customShiftHours) % 24;
          const daysSpan = Math.floor((startHour + customShiftHours) / 24);
          return ` ${String(endHour).padStart(2, '0')}:00${daysSpan > 0 ? ` (+${daysSpan} day${daysSpan > 1 ? 's' : ''})` : ''}`;
        })()}
      </div>
    </div>
  </div>
)}
```

**Why:** Provides clear UI for users to configure custom shifts with real-time feedback

**Visual Design:**
```
┌─────────────────────────────────────────────────────────────┐
│ 🕐 Custom Shift Configuration                               │
├─────────────────────────────────────────────────────────────┤
│ Start Time: [08:00 ▼]  Total Hours: [12  ]  Structure:     │
│                                              ┌──────────────┐│
│                                              │ 1 full block ││
│                                              │ + 1 partial  ││
│                                              │ (4hr)        ││
│                                              │ Total: 12    ││
│                                              │ hourly slots ││
│                                              └──────────────┘│
│ Shift Timeline: 08:00 → 20:00                               │
└─────────────────────────────────────────────────────────────┘
```

---

#### 6.3 Update Block Rendering Logic

**Location:** Hourly tracking section JSX

**Current logic:** Hardcoded block names and colors for each shift type

**Updated logic:** Dynamic rendering based on `calculateCustomBlocks()` output

```javascript
{/* Hourly Email Tracking Section */}
<div className="mb-6">
  <h3 className="text-lg font-semibold mb-4 flex items-center gap-2">
    <Mail />
    Hourly Email Tracking ({
      shiftType === 'weekday' ? '16:00 - 08:00' :
      shiftType === 'stat_holiday' ? '00:00 - 00:00 (24 hours)' :
      shiftType === 'custom' ? `${customShiftStart} - ${(() => {
        const [startHour] = customShiftStart.split(':').map(Number);
        const endHour = (startHour + customShiftHours) % 24;
        return `${String(endHour).padStart(2, '0')}:00`;
      })()} (${customShiftHours} hours)` :
      'Fri 16:00 - Mon 08:00'
    })
  </h3>

  {shiftType === 'weekday' ? (
    // Existing weekday block rendering
    <>
      <div className="mb-6">
        <button onClick={() => toggleBlockCollapse('evening')} /* ... */>
          Evening Block: 16:00-23:59
        </button>
        {!collapsedBlocks['evening'] && (
          <>
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-3">
              {currentTimeSlots.slice(0, 8).map((slot, idx) => renderHourlyBlock(slot, idx))}
            </div>
            <ExceptionOverrideCard /* ... */ />
          </>
        )}
      </div>
      {/* Morning block ... */}
    </>
  ) : shiftType === 'stat_holiday' ? (
    // Existing stat holiday block rendering
    // ...
  ) : shiftType === 'weekend' ? (
    // Existing weekend block rendering
    // ...
  ) : shiftType === 'custom' ? (
    // NEW: Custom shift block rendering
    <>
      {calculateCustomBlocks(customShiftHours, customShiftStart, shiftDate).map((blockMeta, blockIdx) => {
        const blockKey = `custom-block-${blockIdx}`;
        const colors = ['indigo', 'purple', 'pink', 'blue', 'green', 'yellow', 'orange', 'red', 'teal', 'cyan'];
        const color = colors[blockIdx % colors.length];

        return (
          <div key={blockIdx} className="mb-4">
            <button
              onClick={() => toggleBlockCollapse(blockKey)}
              className={`w-full bg-${color}-50 text-${color}-700 px-4 py-2 rounded flex items-center justify-between mb-3 hover:bg-${color}-100 border-2 border-${color}-300`}
            >
              <span className="font-semibold flex items-center gap-2">
                {blockMeta.name}: {blockMeta.timeRange}
                <span className="text-sm font-normal">
                  ({blockMeta.hours} hour{blockMeta.hours > 1 ? 's' : ''})
                </span>
                {blockMeta.isPartial && (
                  <span className="bg-yellow-200 text-yellow-800 px-2 py-1 rounded text-xs font-bold">
                    PARTIAL
                  </span>
                )}
              </span>
              {collapsedBlocks[blockKey] ? <ChevronDown /> : <ChevronUp />}
            </button>

            {!collapsedBlocks[blockKey] && (
              <>
                <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-3">
                  {currentTimeSlots.slice(blockMeta.startSlotIndex, blockMeta.endSlotIndex).map((slot, idx) =>
                    renderHourlyBlock(slot, blockMeta.startSlotIndex + idx)
                  )}
                </div>

                <ExceptionOverrideCard
                  blockIndex={blockIdx}
                  blockName={blockMeta.name}
                  emailBasedOT={(() => {
                    const blocks = calculateShiftBlocks();
                    return blocks[blockIdx]?.emailBasedOT || 0;
                  })()}
                  overrideHours={blockOverrides[blockIdx]?.manualOT || 0}
                  overrideReason={blockOverrides[blockIdx]?.reason || ''}
                  isOpen={openOverrideCards[blockIdx]}
                  onToggle={toggleOverrideCard}
                  onUpdateHours={(idx, val) => updateBlockOverride(idx, 'manualOT', val)}
                  onUpdateReason={(idx, val) => updateBlockOverride(idx, 'reason', val)}
                />
              </>
            )}
          </div>
        );
      })}
    </>
  ) : null}
</div>
```

**Why:** Dynamically renders blocks based on user-defined shift parameters

---

### Phase 7: State Management Updates

#### 7.1 Update `useEffect` for Shift Type Changes

**Location:** Inside `MonthlyOTTracker` component

**Current code:**
```javascript
useEffect(() => {
  let newSize, numBlocks;

  if (shiftType === 'weekday') {
    newSize = 16;
    numBlocks = 2;
  } else if (shiftType === 'stat_holiday') {
    newSize = 24;
    numBlocks = 3;
  } else { // weekend
    newSize = 64;
    numBlocks = 8;
  }

  setEmailData(Array(newSize).fill().map(() => ({
    total: 0,
    actionable: 0,
    calls: 0,
    comment: ''
  })));

  const newOverrides = {};
  for (let i = 0; i < numBlocks; i++) {
    newOverrides[i] = { manualOT: 0, reason: '' };
  }
  setBlockOverrides(newOverrides);
  setOpenOverrideCards({});

  // Update shift times for stat holidays
  if (shiftType === 'stat_holiday') {
    setRegularShiftStart('00:00');
    setRegularShiftEnd('00:00');
  }
}, [shiftType]);
```

**Updated code:**
```javascript
useEffect(() => {
  let newSize, numBlocks;

  if (shiftType === 'weekday') {
    newSize = 16;
    numBlocks = 2;
  } else if (shiftType === 'stat_holiday') {
    newSize = 24;
    numBlocks = 3;
    setRegularShiftStart('00:00');
    setRegularShiftEnd('00:00');
  } else if (shiftType === 'weekend') {
    newSize = 64;
    numBlocks = 8;
  } else if (shiftType === 'custom') {
    // NEW: Custom shift sizing
    newSize = customShiftHours;
    numBlocks = Math.ceil(customShiftHours / 8);
    // No fixed start/end times - user controls
  }

  setEmailData(Array(newSize).fill().map(() => ({
    total: 0,
    actionable: 0,
    calls: 0,
    comment: ''
  })));

  const newOverrides = {};
  for (let i = 0; i < numBlocks; i++) {
    newOverrides[i] = { manualOT: 0, reason: '' };
  }
  setBlockOverrides(newOverrides);
  setOpenOverrideCards({});

}, [shiftType, customShiftHours]); // IMPORTANT: Add customShiftHours as dependency
```

**Why:** Dynamically resizes email data array and block overrides when user changes custom hours

**Critical Note:** Adding `customShiftHours` as a dependency means the form will reset when user adjusts hours. This is acceptable since it prevents data corruption.

---

### Phase 8: Save/Load Logic Updates

#### 8.1 Update `handleSaveShift` Function

**Location:** Inside `MonthlyOTTracker` component

**Current code:**
```javascript
const handleSaveShift = () => {
  if (!employeeName || employeeName.trim().length === 0) {
    alert('Please enter your employee name first');
    return;
  }

  if (!shiftDate) {
    alert('Please select a shift date');
    return;
  }

  const shiftTotals = calculateShiftTotals();
  const hasAnyOverride = Object.values(blockOverrides).some(o => o.manualOT > 0);

  if (hasAnyOverride) {
    if (!confirm('⚠️ This shift contains manual OT overrides...')) {
      return;
    }
  }

  const shiftMonth = shiftDate.substring(0, 7);
  const isDifferentMonth = shiftMonth !== currentMonth;

  if (isDifferentMonth) {
    if (!confirm('📅 Cross-Month Shift Detected...')) {
      return;
    }
  }

  const newShift = {
    id: Date.now(),
    date: shiftDate,
    shiftType,
    regularShiftStart,
    regularShiftEnd,
    emailData: [...emailData],
    totalOT: shiftTotals.totalOT,
    totalSB: shiftTotals.totalSB,
    blocks: shiftTotals.blocks,
    notes: shiftNotes,
    timeSlots: generateTimeSlots(shiftType),
    overtimeCode: getOvertimeCode(shiftDate, userSettings.statHolidays),
    blockOverrides: { ...blockOverrides }
  };

  // ... save logic
};
```

**Updated code:**
```javascript
const handleSaveShift = () => {
  // ... existing validation

  // NEW: Custom shift validation
  if (shiftType === 'custom') {
    if (!customShiftHours || customShiftHours < 1) {
      alert('⚠️ Please enter a valid number of hours for custom shift (minimum 1)');
      return;
    }
    if (customShiftHours > 168) {
      alert('⚠️ Custom shift cannot exceed 168 hours (1 week)');
      return;
    }
    if (!customShiftStart) {
      alert('⚠️ Please select a start time for custom shift');
      return;
    }
  }

  const shiftTotals = calculateShiftTotals();

  // ... existing override and cross-month checks

  const newShift = {
    id: Date.now(),
    date: shiftDate,
    shiftType,

    // NEW: Store custom shift parameters
    customHours: shiftType === 'custom' ? customShiftHours : null,
    customStartTime: shiftType === 'custom' ? customShiftStart : null,

    regularShiftStart: shiftType === 'custom' ? customShiftStart : regularShiftStart,
    regularShiftEnd,
    emailData: [...emailData],
    totalOT: shiftTotals.totalOT,
    totalSB: shiftTotals.totalSB,
    blocks: shiftTotals.blocks,
    notes: shiftNotes,
    timeSlots: generateTimeSlots(
      shiftType,
      shiftType === 'custom' ? customShiftHours : null,
      shiftType === 'custom' ? customShiftStart : null
    ),
    overtimeCode: getOvertimeCode(shiftDate, userSettings.statHolidays),
    blockOverrides: { ...blockOverrides }
  };

  // ... save to localStorage

  // Clear form
  setShiftDate('');
  setShiftType('weekday');
  setCustomShiftHours(8); // NEW: Reset custom hours
  setCustomShiftStart('00:00'); // NEW: Reset custom start
  // ... rest of clear logic
};
```

**Why:** Persists custom shift parameters for accurate display and recalculation

---

#### 8.2 Update Saved Shift Display

**Location:** Saved Shifts section JSX

**Current code:**
```javascript
<div className="font-semibold text-lg text-gray-800 flex items-center gap-2">
  {shift.date}
  {hasOverrides && <span className="text-orange-600 text-sm">⚠️ Has Overrides</span>}
</div>
<div className="text-sm text-gray-600">
  {shift.shiftType === 'weekday' ? 'Weekday (16 hours)' : 'Weekend (64 hours)'} - OT Code: {shift.overtimeCode}
</div>
```

**Updated code:**
```javascript
<div className="font-semibold text-lg text-gray-800 flex items-center gap-2">
  {shift.date}
  {hasOverrides && <span className="text-orange-600 text-sm">⚠️ Has Overrides</span>}
</div>
<div className="text-sm text-gray-600">
  {(() => {
    if (shift.shiftType === 'weekday') return 'Weekday (16 hours)';
    if (shift.shiftType === 'stat_holiday') return 'Stat Holiday (24 hours)';
    if (shift.shiftType === 'weekend') return 'Weekend (64 hours)';
    if (shift.shiftType === 'custom') return `Custom (${shift.customHours} hours)`;
    return shift.shiftType;
  })()} - OT Code: {shift.overtimeCode}
</div>
```

**Why:** Displays custom shift hours in saved shifts list

---

### Phase 9: Excel Export Updates

#### 9.1 Update Monthly Summary Sheet

**Location:** `exportToExcel` function, Monthly Summary sheet generation

**Current code:**
```javascript
const getShiftTypeLabel = (type) => {
  if (type === 'weekday') return 'Weekday (16hr)';
  if (type === 'stat_holiday') return 'Stat Holiday (24hr)';
  return 'Weekend (64hr)';
};
```

**Updated code:**
```javascript
const getShiftTypeLabel = (shift) => {
  if (shift.shiftType === 'weekday') return 'Weekday (16hr)';
  if (shift.shiftType === 'stat_holiday') return 'Stat Holiday (24hr)';
  if (shift.shiftType === 'weekend') return 'Weekend (64hr)';
  if (shift.shiftType === 'custom') return `Custom (${shift.customHours}hr)`;
  return shift.shiftType;
};

// Usage in sheet generation:
savedShifts.forEach((shift, index) => {
  // ...
  const rowData = [
    shift.date,
    getShiftTypeLabel(shift), // Pass entire shift object
    shift.totalOT.toFixed(2),
    // ...
  ];
});
```

**Why:** Shows custom shift duration in Excel report

---

#### 9.2 Update Individual Shift Sheets

**Location:** `exportToExcel` function, individual shift detail sheets

**Current code:**
```javascript
ws.getCell('A3').value = 'Shift Type:';
const shiftTypeDisplay = shift.shiftType === 'weekday' ? 'Weekday (16 hours)'
  : shift.shiftType === 'stat_holiday' ? 'Stat Holiday (24 hours)'
  : 'Weekend (64 hours)';
ws.getCell('B3').value = shiftTypeDisplay;
```

**Updated code:**
```javascript
ws.getCell('A3').value = 'Shift Type:';
let shiftTypeDisplay;
if (shift.shiftType === 'weekday') {
  shiftTypeDisplay = 'Weekday (16 hours)';
} else if (shift.shiftType === 'stat_holiday') {
  shiftTypeDisplay = 'Stat Holiday (24 hours)';
} else if (shift.shiftType === 'weekend') {
  shiftTypeDisplay = 'Weekend (64 hours)';
} else if (shift.shiftType === 'custom') {
  shiftTypeDisplay = `Custom (${shift.customHours} hours)`;

  // Add custom shift details
  ws.getCell('A5').value = 'Custom Start Time:';
  ws.getCell('B5').value = shift.customStartTime || 'N/A';
  ws.getCell('A5').font = { bold: true };
  ws.getCell('A5').fill = { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FFE7E6E6' } };
}
ws.getCell('B3').value = shiftTypeDisplay;
```

**Why:** Includes custom shift metadata in detailed Excel sheets

---

#### 9.3 Update Block Table Generation

**Location:** Individual shift sheets, block calculations table

**Current block header row:**
```javascript
const blockHeaders = ['Block', 'StartTime', 'EndTime', 'Actionable', 'TotalEmails', 'Email OT', 'Override OT', 'Total OT', 'SB', 'Code'];
```

**Updated block header row:**
```javascript
const blockHeaders = ['Block', 'StartTime', 'EndTime', 'Hours', 'Actionable', 'TotalEmails', 'Email OT', 'Override OT', 'Total OT', 'SB', 'Code'];
//                                                        ^^^^^^ NEW COLUMN
```

**Updated block row generation:**
```javascript
shift.blocks.forEach((block, idx) => {
  const blockRow = ws.getRow(11 + idx);
  blockRow.values = [
    block.name || '',
    (block.timeRange || '').split('-')[0],
    (block.timeRange || '').split('-')[1],
    block.blockHours || 8, // NEW: Show actual block hours
    block.totalActionable || 0,
    block.totalEmails || 0,
    (block.emailBasedOT || 0).toFixed(2),
    (block.manualOverrideOT || 0).toFixed(2),
    (block.totalBlockOT || 0).toFixed(2),
    (block.blockSB || 0).toFixed(2),
    block.code || ''
  ];

  // Highlight partial blocks
  if (block.blockHours && block.blockHours < 8) {
    blockRow.eachCell((cell) => {
      cell.fill = { type: 'pattern', pattern: 'solid', fgColor: { argb: 'FFFFE4B5' } }; // Light orange
      cell.font = { bold: true };
    });
  }

  // ... rest of styling
});
```

**Why:** Makes partial blocks visually distinct in Excel export

---

### Phase 10: Validation & Error Handling

#### 10.1 Add Input Validation Functions

**Location:** Helper functions section

```javascript
/**
 * Validates custom shift parameters
 * @returns {Object} { valid: boolean, error: string }
 */
const validateCustomShift = (hours, startTime, date) => {
  if (!hours || isNaN(hours)) {
    return { valid: false, error: 'Hours must be a valid number' };
  }

  if (hours < 1) {
    return { valid: false, error: 'Minimum shift duration is 1 hour' };
  }

  if (hours > 168) {
    return { valid: false, error: 'Maximum shift duration is 168 hours (1 week)' };
  }

  if (!startTime || !/^\d{2}:\d{2}$/.test(startTime)) {
    return { valid: false, error: 'Start time must be in HH:MM format' };
  }

  if (!date) {
    return { valid: false, error: 'Shift date is required' };
  }

  return { valid: true, error: '' };
};
```

**Usage in `handleSaveShift`:**
```javascript
if (shiftType === 'custom') {
  const validation = validateCustomShift(customShiftHours, customShiftStart, shiftDate);
  if (!validation.valid) {
    alert(`⚠️ ${validation.error}`);
    return;
  }
}
```

---

#### 10.2 Add User Warnings

```javascript
// Warn about very long shifts
if (shiftType === 'custom' && customShiftHours > 48) {
  if (!confirm(`⚠️ Long Custom Shift\n\nYou've entered a ${customShiftHours}-hour shift. This will generate ${Math.ceil(customShiftHours / 8)} blocks.\n\nAre you sure this is correct?`)) {
    return;
  }
}

// Warn about multi-day shifts
if (shiftType === 'custom') {
  const [startHour] = customShiftStart.split(':').map(Number);
  const daysSpan = Math.floor((startHour + customShiftHours) / 24);

  if (daysSpan > 0) {
    const endDate = new Date(shiftDate + 'T12:00:00');
    endDate.setDate(endDate.getDate() + daysSpan);
    const endDateStr = endDate.toISOString().split('T')[0];

    if (!confirm(`📅 Multi-Day Custom Shift\n\nThis shift spans from ${shiftDate} to ${endDateStr} (${daysSpan} day${daysSpan > 1 ? 's' : ''}).\n\nOT codes will be assigned per block based on the day of the week.\n\nContinue?`)) {
      return;
    }
  }
}
```

---

### Phase 11: Backward Compatibility

#### 11.1 Handle Legacy Shifts

**Location:** Load functions and Excel export

```javascript
/**
 * Migrates old shift format to include custom shift fields
 */
const migrateShiftFormat = (shift) => {
  // If customHours is missing but shiftType is custom, infer from time slots
  if (shift.shiftType === 'custom' && !shift.customHours) {
    shift.customHours = shift.timeSlots?.length || shift.emailData?.length || 8;
    shift.customStartTime = shift.regularShiftStart || '00:00';
  }

  // If blockHours is missing from blocks, add default 8
  if (shift.blocks) {
    shift.blocks = shift.blocks.map(block => ({
      ...block,
      blockHours: block.blockHours || 8,
      isPartial: block.blockHours && block.blockHours < 8
    }));
  }

  return shift;
};

// Use in loadFromLocalStorage
const loadFromLocalStorage = (employeeName, month) => {
  // ... existing load logic
  if (data) {
    const parsedData = JSON.parse(data);
    const shifts = parsedData[month] || [];
    return shifts.map(migrateShiftFormat); // Apply migration
  }
  return [];
};
```

**Why:** Ensures old saved shifts still work after update

---

## Testing Plan

### Test Case 1: Single Full Block (8 hours)
**Setup:**
- Custom shift: 8 hours, start 08:00
- Date: Monday, Oct 20, 2025
- 10 actionable emails, 2 calls

**Expected:**
- 1 block: "Block 1" (08:00-16:00, 8 hours)
- Email OT: 2.5h, Call OT: 0.5h, Total: 3.0h OT, 5.0h SB
- OT Code: 260 (Monday)

---

### Test Case 2: Partial Block (4 hours)
**Setup:**
- Custom shift: 4 hours, start 14:00
- Date: Friday, Oct 24, 2025
- 8 actionable emails, 0 calls

**Expected:**
- 1 block: "Block 1" (14:00-18:00, 4 hours)
- Email OT: 2.0h, Total: 2.0h OT, 2.0h SB (totals to 4.0)
- OT Code: 261 (Friday)

---

### Test Case 3: Full + Partial (12 hours)
**Setup:**
- Custom shift: 12 hours, start 08:00
- Date: Saturday, Oct 25, 2025
- Block 1: 16 actionable emails → 4.0h OT
- Block 2: 8 actionable emails → 2.0h OT

**Expected:**
- Block 1: "Block 1" (08:00-16:00, 8 hours) → 4.0h OT, 4.0h SB
- Block 2: "Block 2" (16:00-20:00, 4 hours) → 2.0h OT, 2.0h SB
- Total: 6.0h OT, 6.0h SB (totals to 12.0)
- Both blocks OT Code: 262 (Saturday)

---

### Test Case 4: Multi-Day Shift (36 hours)
**Setup:**
- Custom shift: 36 hours, start Monday 08:00
- Date: Monday, Oct 20, 2025
- Minimal email activity

**Expected:**
- Block 1: Mon 08:00-16:00 → Code 260
- Block 2: Mon 16:00-00:00 → Code 260
- Block 3: Tue 00:00-08:00 → Code 260
- Block 4: Tue 08:00-16:00 → Code 260
- Block 5: Tue 16:00-20:00 (4hr partial) → Code 260
- Total: 5 blocks, all code 260

---

### Test Case 5: Override on Partial Block
**Setup:**
- Custom shift: 4 hours, start 10:00
- 0 actionable emails
- Override: +3.0 hours (system maintenance)

**Expected:**
- Block 1: 4 hours total
- Email OT: 0h, Override: 3.0h → Total: 3.0h OT, 1.0h SB
- Override max: 4.0 hours (not 8.0)

---

### Test Case 6: Weekend Spanning Custom Shift
**Setup:**
- Custom shift: 48 hours, start Saturday 08:00
- Date: Saturday, Oct 25, 2025

**Expected:**
- Blocks 1-3: Saturday → Code 262
- Blocks 4-6: Sunday → Code 262
- Total: 6 blocks, all code 262

---

### Test Case 7: Stat Holiday Custom Shift
**Setup:**
- Custom shift: 16 hours, start 00:00
- Date: July 1, 2025 (Canada Day, in stat holidays config)

**Expected:**
- Block 1: 00:00-08:00 → Code 263
- Block 2: 08:00-16:00 → Code 263
- Both blocks use stat holiday code

---

### Test Case 8: Excel Export with Custom Shift
**Setup:**
- Save 1 weekday, 1 weekend, 1 custom (12hr) shift
- Export to Excel

**Expected:**
- Monthly Summary: Custom shows "Custom (12hr)"
- Individual Sheet: Shows custom start time, 2 blocks
- Block table: "Hours" column shows 8 and 4
- Partial block highlighted in light orange

---

## Implementation Checklist

### Phase 1: Foundation ✓
- [ ] Add state variables (`customShiftHours`, `customShiftStart`)
- [ ] Create `SHIFT_TYPE_CONFIG` constant
- [ ] Update `generateTimeSlots()` function

### Phase 2: Block Logic ✓
- [ ] Create `calculateCustomBlocks()` function
- [ ] Create `calculateBlockDate()` helper
- [ ] Update `calculateBlockTotals()` signature (add `blockHours` param)
- [ ] Update `calculateShiftBlocks()` to handle custom type

### Phase 3: UI ✓
- [ ] Add "Custom" option to shift type selector
- [ ] Create custom shift configuration panel UI
- [ ] Update block rendering logic for dynamic blocks
- [ ] Add partial block visual indicators

### Phase 4: State Management ✓
- [ ] Update `useEffect` for shift type changes
- [ ] Add `customShiftHours` to dependency array

### Phase 5: Persistence ✓
- [ ] Update `handleSaveShift()` validation
- [ ] Store custom shift parameters in shift object
- [ ] Update saved shift display to show custom hours
- [ ] Update `handleClearForm()` to reset custom fields

### Phase 6: Excel Export ✓
- [ ] Update `getShiftTypeLabel()` function
- [ ] Add "Hours" column to block table
- [ ] Highlight partial blocks in Excel
- [ ] Add custom start time to individual sheets

### Phase 7: Validation ✓
- [ ] Create `validateCustomShift()` function
- [ ] Add long shift warning (> 48 hours)
- [ ] Add multi-day shift confirmation dialog
- [ ] Add input constraints (min=1, max=168)

### Phase 8: Polish ✓
- [ ] Create `migrateShiftFormat()` for backward compatibility
- [ ] Add shift timeline display in UI
- [ ] Update all string labels and messages
- [ ] Test all existing shift types still work

### Phase 9: Testing ✓
- [ ] Run all 8 test cases
- [ ] Verify Excel export formatting
- [ ] Test edge cases (1 hour, 168 hours, midnight crossings)
- [ ] Verify overtime code accuracy for multi-day shifts

### Phase 10: Documentation ✓
- [ ] Update user instructions in UI
- [ ] Add custom shift examples
- [ ] Update `BUSINESS_LOGIC_ANALYSIS.md`
- [ ] Update `CALCULATION_FLOW_DIAGRAMS.md`

---

## Potential Issues & Solutions

### Issue 1: Form Reset on Hours Change
**Problem:** When user changes `customShiftHours`, the form resets (due to `useEffect` dependency)

**Solution:** Add confirmation dialog:
```javascript
const handleCustomHoursChange = (newHours) => {
  const hasData = emailData.some(slot => slot.total > 0 || slot.actionable > 0);

  if (hasData) {
    if (!confirm('⚠️ Changing shift hours will reset all entered data. Continue?')) {
      return;
    }
  }

  setCustomShiftHours(newHours);
};
```

---

### Issue 2: Overtime Code Confusion for Multi-Day
**Problem:** Users might not understand why a 36-hour shift has multiple OT codes

**Solution:** Add info tooltip:
```jsx
<div className="bg-blue-50 border border-blue-200 p-3 rounded mt-2">
  <p className="text-sm text-blue-800">
    ℹ️ <strong>Multi-Day Shifts:</strong> Each 8-hour block receives the overtime code for the day it occurs.
    For example, a shift starting Monday and extending to Tuesday will have blocks with code 260 (weekday) for both days.
  </p>
</div>
```

---

### Issue 3: Partial Block Override Limits
**Problem:** Users might not realize override limit is lower for partial blocks

**Solution:** Update `ExceptionOverrideCard` max calculation:
```javascript
const maxOverride = blockHours - emailBasedOT; // Uses actual block hours, not 8

<label>
  Manual OT Hours <span className="text-gray-500">(0 to {maxOverride.toFixed(2)})</span>
</label>
```

---

### Issue 4: Excel Column Width for "Hours"
**Problem:** New "Hours" column might be too narrow

**Solution:** Set explicit width:
```javascript
ws.getColumn(4).width = 8; // "Hours" column
```

---

## Rollback Plan

If critical issues arise:

1. **Revert UI Changes:**
   - Remove "Custom" option from selector
   - Hide custom configuration panel

2. **Add Migration Check:**
```javascript
if (shift.shiftType === 'custom') {
  console.warn('Custom shift type not supported in this version');
  return null; // Skip rendering
}
```

3. **Export Fallback:**
```javascript
const getShiftTypeLabel = (shift) => {
  if (shift.shiftType === 'custom') {
    return `Custom (${shift.customHours || 'unknown'}hr) - LEGACY`;
  }
  // ... existing logic
};
```

---

## Estimated Implementation Time

| Phase | Estimated Time |
|-------|----------------|
| Phase 1-2: Data structures & time slots | 2 hours |
| Phase 3: Block calculation logic | 3 hours |
| Phase 4: Overtime code (already works!) | 0 hours |
| Phase 5-6: UI updates | 4 hours |
| Phase 7: State management | 1 hour |
| Phase 8: Excel export | 2 hours |
| Phase 9: Validation | 1 hour |
| Phase 10: Testing & polish | 3 hours |
| **Total** | **16 hours** |

---

## Success Criteria

- [✓] User can create custom shifts with any duration (1-168 hours)
- [✓] System generates correct number of blocks (full + partial if needed)
- [✓] Partial blocks calculate OT/SB correctly (totaling to < 8 hours)
- [✓] Multi-day shifts assign correct OT codes per block
- [✓] All existing shift types still work without changes
- [✓] Excel export includes custom shift data with proper formatting
- [✓] Backward compatible with old saved shifts
- [✓] No performance degradation with custom shifts

---

## Next Steps

1. **Review this plan** with stakeholders
2. **Create backup** of current working code
3. **Implement Phase 1-2** (foundation)
4. **Test basic functionality** (single block custom shift)
5. **Continue with Phase 3-4** (calculations)
6. **Test advanced scenarios** (partial blocks, multi-day)
7. **Complete Phase 5-8** (UI and persistence)
8. **Full regression testing** (all shift types)
9. **User acceptance testing**
10. **Deploy to production**

---

## Conclusion

This implementation plan adds a flexible custom shift type while maintaining all existing functionality. The design leverages the existing block-based architecture, requiring minimal changes to core calculation logic. The most significant work is in UI rendering and state management for dynamic block generation.

**Key Advantages:**
- ✅ Seamless integration with existing overtime calculation rules
- ✅ No changes to existing shift types
- ✅ Backward compatible with saved data
- ✅ Accurate OT code assignment for multi-day shifts
- ✅ Proper handling of partial blocks

**Ready to implement!** 🚀
