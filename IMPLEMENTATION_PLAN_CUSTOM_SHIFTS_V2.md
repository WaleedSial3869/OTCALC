# Implementation Plan: Custom/Flexible Shift Type (Version 2)

**Last Updated:** Based on comprehensive review feedback
**Status:** Ready for implementation
**Estimated Time:** 18 hours (16 original + 2 for fixes/polish)

---

## 📋 Changes from Version 1

### Critical Fixes Applied:
1. ✅ **Fixed Tailwind dynamic class issue** - Using static class mapping
2. ✅ **Added currentTimeSlots update logic** - Passes custom parameters
3. ✅ **Updated ExceptionOverrideCard** - Added blockHours prop
4. ✅ **Included complete weekend slot generation** - No missing code
5. ✅ **Added form data loss confirmation** - Prevents accidental resets

### Improvements Added:
6. ✅ **Quick preset buttons** - 4hr, 8hr, 12hr, 24hr shortcuts
7. ✅ **End date display** - Shows actual end date for multi-day shifts
8. ✅ **Visual block preview** - Shows block structure before data entry
9. ✅ **Stat holiday warning** - Alerts user when custom shift on stat holiday

### Questions Answered:
- **Q1: Stat holiday auto-detection?** → Don't auto-switch; show warning instead
- **Q2: Custom shifts on stat holidays?** → Yes, allow but warn and use code 263
- **Q3: Non-hour start times?** → Keep on-the-hour for simplicity

---

## CRITICAL FIX #1: Tailwind Dynamic Classes

### ❌ Original (Won't Work):
```javascript
const color = colors[blockIdx % colors.length];
className={`w-full bg-${color}-50 text-${color}-700 ...`}
// ↑ Tailwind purges these at build time!
```

### ✅ Fixed Version:
```javascript
// Add to helper functions section
const BLOCK_COLOR_CLASSES = [
  'bg-indigo-50 text-indigo-700 border-2 border-indigo-300 hover:bg-indigo-100',
  'bg-purple-50 text-purple-700 border-2 border-purple-300 hover:bg-purple-100',
  'bg-pink-50 text-pink-700 border-2 border-pink-300 hover:bg-pink-100',
  'bg-blue-50 text-blue-700 border-2 border-blue-300 hover:bg-blue-100',
  'bg-green-50 text-green-700 border-2 border-green-300 hover:bg-green-100',
  'bg-yellow-50 text-yellow-700 border-2 border-yellow-300 hover:bg-yellow-100',
  'bg-orange-50 text-orange-700 border-2 border-orange-300 hover:bg-orange-100',
  'bg-red-50 text-red-700 border-2 border-red-300 hover:bg-red-100',
  'bg-teal-50 text-teal-700 border-2 border-teal-300 hover:bg-teal-100',
  'bg-cyan-50 text-cyan-700 border-2 border-cyan-300 hover:bg-cyan-100',
  'bg-lime-50 text-lime-700 border-2 border-lime-300 hover:bg-lime-100',
  'bg-emerald-50 text-emerald-700 border-2 border-emerald-300 hover:bg-emerald-100',
  'bg-sky-50 text-sky-700 border-2 border-sky-300 hover:bg-sky-100',
  'bg-violet-50 text-violet-700 border-2 border-violet-300 hover:bg-violet-100',
  'bg-fuchsia-50 text-fuchsia-700 border-2 border-fuchsia-300 hover:bg-fuchsia-100',
  'bg-rose-50 text-rose-700 border-2 border-rose-300 hover:bg-rose-100',
  'bg-amber-50 text-amber-700 border-2 border-amber-300 hover:bg-amber-100',
  'bg-slate-50 text-slate-700 border-2 border-slate-300 hover:bg-slate-100',
  'bg-gray-50 text-gray-700 border-2 border-gray-300 hover:bg-gray-100',
  'bg-zinc-50 text-zinc-700 border-2 border-zinc-300 hover:bg-zinc-100',
  'bg-stone-50 text-stone-700 border-2 border-stone-300 hover:bg-stone-100'
  // Supports up to 21 blocks (168 hours / 8 = 21)
];

// Usage in custom shift block rendering:
className={`w-full ${BLOCK_COLOR_CLASSES[blockIdx % BLOCK_COLOR_CLASSES.length]} px-4 py-2 rounded flex items-center justify-between mb-3`}
```

**Why this works:** All classes are static strings, so Tailwind includes them in the build.

---

## CRITICAL FIX #2: currentTimeSlots Update

### Location: Inside `MonthlyOTTracker` component, before return statement

### ✅ Add This Line:
```javascript
const MonthlyOTTracker = () => {
  // ... all state declarations

  // ... all functions

  // IMPORTANT: Update this line (currently generates fixed slots)
  const currentTimeSlots = generateTimeSlots(
    shiftType,
    shiftType === 'custom' ? customShiftHours : null,
    shiftType === 'custom' ? customShiftStart : null
  );

  const shiftTotals = calculateShiftTotals();
  const monthlyTotals = calculateMonthlyTotals();

  // ... render logic
};
```

**Critical:** Without this, custom shifts will generate wrong time slots.

---

## CRITICAL FIX #3: ExceptionOverrideCard blockHours Prop

### Current ExceptionOverrideCard Implementation:
```javascript
const ExceptionOverrideCard = React.memo(({
  blockIndex,
  blockName,
  emailBasedOT,
  overrideHours,
  overrideReason,
  isOpen,
  onToggle,
  onUpdateHours,
  onUpdateReason
}) => {
  // ... existing code

  const maxOverride = 8.0 - emailBasedOT; // ❌ HARDCODED 8.0!

  // ...
});
```

### ✅ Fixed Version:
```javascript
const ExceptionOverrideCard = React.memo(({
  blockIndex,
  blockName,
  emailBasedOT,
  overrideHours,
  overrideReason,
  isOpen,
  onToggle,
  onUpdateHours,
  onUpdateReason,
  blockHours = 8 // NEW PROP with default
}) => {
  const [localReason, setLocalReason] = useState(overrideReason);
  const textareaRef = useRef(null);

  useEffect(() => {
    setLocalReason(overrideReason);
  }, [overrideReason]);

  const handleReasonChange = (e) => {
    setLocalReason(e.target.value);
  };

  const handleReasonBlur = () => {
    if (localReason !== overrideReason) {
      onUpdateReason(blockIndex, localReason);
    }
  };

  // ✅ USE blockHours PROP INSTEAD OF HARDCODED 8.0
  const maxOverride = blockHours - emailBasedOT;
  const totalOT = emailBasedOT + overrideHours;
  const totalSB = blockHours - totalOT;

  return (
    <div className="mt-4">
      <button
        onClick={() => onToggle(blockIndex)}
        className={`w-full flex items-center justify-between px-4 py-3 rounded-lg font-semibold transition-all ${
          overrideHours > 0
            ? 'bg-orange-100 border-2 border-orange-400 text-orange-800 hover:bg-orange-200'
            : 'bg-gray-100 border-2 border-gray-300 text-gray-700 hover:bg-gray-200'
        }`}
      >
        <div className="flex items-center gap-2">
          {overrideHours > 0 ? <AlertTriangle /> : <Plus />}
          <span>
            {overrideHours > 0
              ? `⚠️ Override Active: +${overrideHours.toFixed(2)}h`
              : '➕ Add Exception Override'}
          </span>
        </div>
        {isOpen ? <ChevronUp /> : <ChevronDown />}
      </button>

      {isOpen && (
        <div className="bg-orange-50 border-2 border-orange-300 rounded-lg p-4 mt-2">
          <div className="flex items-center gap-2 mb-3">
            <AlertTriangle />
            <h4 className="font-bold text-orange-800 text-lg">
              Exception Override - {blockName} Block
              {blockHours < 8 && (
                <span className="ml-2 text-sm bg-yellow-200 text-yellow-800 px-2 py-1 rounded">
                  PARTIAL BLOCK ({blockHours}hr)
                </span>
              )}
            </h4>
          </div>

          <div className="bg-yellow-50 border border-yellow-300 rounded p-3 mb-3">
            <p className="text-sm text-yellow-900">
              <strong>⚠️ Use only for non-email work:</strong> System maintenance, training, reports, emergency tasks, etc.
            </p>
            {blockHours < 8 && (
              <p className="text-sm text-yellow-900 mt-2">
                <strong>Note:</strong> This is a {blockHours}-hour block, so max override is {maxOverride.toFixed(2)} hours (not 8.0).
              </p>
            )}
          </div>

          <div className="bg-white rounded p-3 mb-3">
            <label className="block text-sm font-medium text-gray-700 mb-2">
              Manual OT Hours <span className="text-gray-500">(0 to {maxOverride.toFixed(2)})</span>
            </label>
            <input
              type="number"
              min="0"
              max={maxOverride}
              step="0.25"
              value={overrideHours}
              onChange={(e) => onUpdateHours(blockIndex, e.target.value)}
              className="w-full border-2 border-orange-300 rounded px-3 py-2 text-lg font-bold focus:border-orange-500 focus:ring-2 focus:ring-orange-200"
              placeholder="0.00"
            />
          </div>

          {overrideHours > 0 && (
            <div className="bg-white rounded p-3 mb-3">
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Reason for Override <span className="text-gray-500">(Optional)</span>
              </label>
              <textarea
                ref={textareaRef}
                value={localReason}
                onChange={handleReasonChange}
                onBlur={handleReasonBlur}
                className="w-full border-2 border-orange-300 rounded px-3 py-2 h-24 focus:border-orange-500 focus:ring-2 focus:ring-orange-200"
                placeholder="Optional: Add notes about this override (e.g., 'System maintenance')"
              />
              <p className="text-xs text-gray-600 mt-1">
                Add any notes about this override (optional)
              </p>
            </div>
          )}

          <div className="bg-blue-50 rounded p-4 border border-blue-200">
            <div className="text-sm font-semibold text-gray-800 mb-3 flex items-center gap-2">
              <Calculator />
              Block Calculation Preview:
            </div>
            <div className="grid grid-cols-2 gap-3 text-sm">
              <div className="text-gray-700">Email-based OT:</div>
              <div className="font-bold text-right">{emailBasedOT.toFixed(2)}h</div>

              <div className="text-gray-700">Manual Override OT:</div>
              <div className="font-bold text-right text-orange-600">
                +{overrideHours.toFixed(2)}h
              </div>

              <div className="border-t pt-2 font-bold text-gray-800">Final Block OT:</div>
              <div className="border-t pt-2 font-bold text-right text-green-600">
                {totalOT.toFixed(2)}h
              </div>

              <div className="text-gray-700">Final Block SB:</div>
              <div className="font-bold text-right">{totalSB.toFixed(2)}h</div>

              <div className="col-span-2 border-t pt-2 text-center font-bold text-lg">
                Total: {(totalOT + totalSB).toFixed(2)}h / {blockHours.toFixed(2)}h ✓
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}, (prevProps, nextProps) => {
  return (
    prevProps.emailBasedOT === nextProps.emailBasedOT &&
    prevProps.overrideHours === nextProps.overrideHours &&
    prevProps.overrideReason === nextProps.overrideReason &&
    prevProps.isOpen === nextProps.isOpen &&
    prevProps.blockHours === nextProps.blockHours // ADD THIS
  );
});
```

**Key Changes:**
1. Added `blockHours` prop with default value 8
2. Calculate `maxOverride` using `blockHours` instead of hardcoded 8.0
3. Show warning label for partial blocks
4. Display actual block hours in total calculation

---

## CRITICAL FIX #4: Complete Weekend Slot Generation

### ✅ Full generateTimeSlots Implementation:
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

  // Weekday shift: 16 hours (16:00 to 08:00 next day)
  if (shiftType === 'weekday') {
    return [
      '16:00-17:00', '17:00-18:00', '18:00-19:00', '19:00-20:00',
      '20:00-21:00', '21:00-22:00', '22:00-23:00', '23:00-00:00',
      '00:00-01:00', '01:00-02:00', '02:00-03:00', '03:00-04:00',
      '04:00-05:00', '05:00-06:00', '06:00-07:00', '07:00-08:00'
    ];
  }

  // Stat holiday: 24 hours (00:00 to 00:00 next day)
  if (shiftType === 'stat_holiday') {
    const slots = [];
    for (let i = 0; i < 24; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = i === 23 ? '00:00' : `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }
    return slots;
  }

  // Weekend shift: 64 hours (Friday 16:00 to Monday 08:00)
  if (shiftType === 'weekend') {
    const slots = [];

    // Friday evening: 16:00-00:00 (8 hours)
    for (let i = 16; i < 24; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = i === 23 ? '00:00' : `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    // Saturday morning: 00:00-08:00 (8 hours)
    for (let i = 0; i < 8; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    // Saturday afternoon: 08:00-16:00 (8 hours)
    for (let i = 8; i < 16; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    // Saturday evening: 16:00-00:00 (8 hours)
    for (let i = 16; i < 24; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = i === 23 ? '00:00' : `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    // Sunday morning: 00:00-08:00 (8 hours)
    for (let i = 0; i < 8; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    // Sunday afternoon: 08:00-16:00 (8 hours)
    for (let i = 8; i < 16; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    // Sunday evening: 16:00-00:00 (8 hours)
    for (let i = 16; i < 24; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = i === 23 ? '00:00' : `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    // Monday morning: 00:00-08:00 (8 hours)
    for (let i = 0; i < 8; i++) {
      const start = `${String(i).padStart(2, '0')}:00`;
      const end = `${String(i + 1).padStart(2, '0')}:00`;
      slots.push(`${start}-${end}`);
    }

    return slots; // Total: 64 slots
  }

  // Fallback (should never reach here)
  return [];
};
```

---

## CRITICAL FIX #5: Form Data Loss Confirmation

### ✅ Add Confirmation Handler:
```javascript
// Add this function in MonthlyOTTracker component
const handleCustomHoursChange = (newHours) => {
  // Validate range
  if (newHours < 1) {
    alert('⚠️ Minimum 1 hour');
    setCustomShiftHours(1);
    return;
  }

  if (newHours > 168) {
    alert('⚠️ Maximum 168 hours (1 week)');
    setCustomShiftHours(168);
    return;
  }

  // Check if user has entered any data
  const hasData = emailData.some(slot =>
    slot.total > 0 || slot.actionable > 0 || slot.calls > 0 || slot.comment
  );

  const hasOverrides = Object.values(blockOverrides).some(o => o.manualOT > 0 || o.reason);

  if ((hasData || hasOverrides) && newHours !== customShiftHours) {
    if (!confirm(
      '⚠️ Changing Shift Hours\n\n' +
      'This will reset all entered data (emails, calls, overrides).\n\n' +
      `Current: ${customShiftHours} hours → New: ${newHours} hours\n\n` +
      'Continue?'
    )) {
      // User cancelled - don't change hours
      return;
    }
  }

  setCustomShiftHours(newHours);
};

// Update the input onChange:
<input
  type="number"
  min="1"
  max="168"
  step="1"
  value={customShiftHours}
  onChange={(e) => handleCustomHoursChange(parseInt(e.target.value) || 1)}
  // ... rest of props
/>
```

---

## IMPROVEMENT #1: Quick Preset Buttons

### ✅ Add Above Custom Hours Input:
```javascript
{shiftType === 'custom' && (
  <div className="bg-gradient-to-r from-indigo-50 to-purple-50 border-2 border-indigo-300 p-4 rounded-lg mb-4">
    <div className="flex items-center gap-2 mb-3">
      <Clock />
      <h3 className="font-bold text-indigo-800 text-lg">Custom Shift Configuration</h3>
    </div>

    {/* NEW: Quick Presets */}
    <div className="mb-4 bg-white border-2 border-indigo-200 rounded-lg p-3">
      <div className="text-sm font-medium text-gray-700 mb-2">Quick Presets:</div>
      <div className="flex gap-2 flex-wrap">
        {[4, 8, 12, 16, 24, 36, 48].map(hours => (
          <button
            key={hours}
            onClick={() => handleCustomHoursChange(hours)}
            className={`px-4 py-2 rounded-lg font-semibold transition-all ${
              customShiftHours === hours
                ? 'bg-indigo-600 text-white border-2 border-indigo-700'
                : 'bg-indigo-100 text-indigo-700 border-2 border-indigo-300 hover:bg-indigo-200'
            }`}
          >
            {hours}hr
            {hours === 8 && <span className="ml-1 text-xs">(1 block)</span>}
            {hours === 16 && <span className="ml-1 text-xs">(2 blocks)</span>}
            {hours === 24 && <span className="ml-1 text-xs">(3 blocks)</span>}
          </button>
        ))}
      </div>
      <p className="text-xs text-gray-500 mt-2">
        Or enter a custom value below (1-168 hours)
      </p>
    </div>

    <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
      {/* Rest of custom shift config... */}
    </div>
  </div>
)}
```

---

## IMPROVEMENT #2: End Date Display for Multi-Day Shifts

### ✅ Add Below Shift Timeline:
```javascript
{/* Shift Timeline Display */}
<div className="mt-3 bg-white border border-indigo-200 rounded p-3">
  <div className="text-sm text-gray-700">
    <strong>Shift Timeline:</strong> {customShiftStart} →
    {(() => {
      const [startHour] = customShiftStart.split(':').map(Number);
      const endHour = (startHour + customShiftHours) % 24;
      const daysSpan = Math.floor((startHour + customShiftHours) / 24);

      return (
        <>
          {` ${String(endHour).padStart(2, '0')}:00`}
          {daysSpan > 0 && ` (+${daysSpan} day${daysSpan > 1 ? 's' : ''})`}
        </>
      );
    })()}
  </div>

  {/* NEW: End Date Display */}
  {shiftDate && (() => {
    const [startHour] = customShiftStart.split(':').map(Number);
    const daysSpan = Math.floor((startHour + customShiftHours) / 24);

    if (daysSpan > 0) {
      const endDate = new Date(shiftDate + 'T12:00:00');
      endDate.setDate(endDate.getDate() + daysSpan);
      const endDateStr = endDate.toLocaleDateString('en-US', {
        weekday: 'short',
        month: 'short',
        day: 'numeric',
        year: 'numeric'
      });

      return (
        <div className="mt-2 text-sm">
          <span className="text-orange-600 font-semibold">
            ⚠️ Multi-Day Shift:
          </span>
          {` Ends on ${endDateStr} (${daysSpan + 1} day${daysSpan > 0 ? 's' : ''} total)`}
        </div>
      );
    }
    return null;
  })()}
</div>
```

---

## IMPROVEMENT #3: Visual Block Preview

### ✅ Add After Shift Timeline:
```javascript
{/* NEW: Visual Block Preview */}
{shiftDate && customShiftHours > 0 && (
  <div className="mt-3 bg-gray-50 border border-gray-300 rounded-lg p-3">
    <div className="text-sm font-semibold text-gray-700 mb-2 flex items-center gap-2">
      <Calendar />
      Block Structure Preview:
    </div>
    <div className="flex flex-wrap gap-2">
      {calculateCustomBlocks(customShiftHours, customShiftStart, shiftDate).map((block, idx) => (
        <div
          key={idx}
          className={`inline-block px-3 py-2 rounded-lg text-xs font-semibold border-2 ${
            block.isPartial
              ? 'bg-yellow-100 border-yellow-400 text-yellow-800'
              : 'bg-blue-100 border-blue-400 text-blue-800'
          }`}
        >
          <div className="font-bold">{block.name}</div>
          <div className="text-xs opacity-80">{block.timeRange}</div>
          <div className="text-xs mt-1">
            {block.hours} hour{block.hours > 1 ? 's' : ''}
            {block.isPartial && ' (partial)'}
          </div>
        </div>
      ))}
    </div>
    <p className="text-xs text-gray-600 mt-2">
      Each block allows independent email tracking and override management
    </p>
  </div>
)}
```

---

## IMPROVEMENT #4: Stat Holiday Warning for Custom Shifts

### ✅ Add After Custom Configuration Panel:
```javascript
{/* NEW: Stat Holiday Warning */}
{shiftType === 'custom' && shiftDate && isStatHoliday(shiftDate, userSettings.statHolidays) && (
  <div className="bg-gradient-to-r from-red-50 to-orange-50 border-2 border-red-400 p-4 rounded-lg mb-4">
    <div className="flex items-center gap-3">
      <div className="text-3xl">🎉</div>
      <div>
        <h3 className="font-bold text-red-800 text-lg">Statutory Holiday Detected</h3>
        <p className="text-sm text-red-700 mt-1">
          {new Date(shiftDate + 'T12:00:00').toLocaleDateString('en-US', {
            weekday: 'long',
            year: 'numeric',
            month: 'long',
            day: 'numeric'
          })} is configured as a statutory holiday.
        </p>
        <p className="text-sm text-red-700 mt-1">
          <strong>⚠️ All blocks in this custom shift will use Overtime Code 263.</strong>
        </p>
      </div>
    </div>
  </div>
)}
```

---

## Updated Custom Shift Block Rendering (With All Fixes)

### ✅ Complete Implementation:
```javascript
{shiftType === 'custom' ? (
  <>
    {calculateCustomBlocks(customShiftHours, customShiftStart, shiftDate).map((blockMeta, blockIdx) => {
      const blockKey = `custom-block-${blockIdx}`;

      return (
        <div key={blockIdx} className="mb-4">
          <button
            onClick={() => toggleBlockCollapse(blockKey)}
            className={`w-full ${BLOCK_COLOR_CLASSES[blockIdx % BLOCK_COLOR_CLASSES.length]} px-4 py-2 rounded flex items-center justify-between mb-3`}
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
              {blockMeta.blockDate && isStatHoliday(blockMeta.blockDate, userSettings.statHolidays) && (
                <span className="bg-red-200 text-red-800 px-2 py-1 rounded text-xs font-bold">
                  STAT HOLIDAY
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
                blockHours={blockMeta.hours} // CRITICAL: Pass actual block hours
              />
            </>
          )}
        </div>
      );
    })}
  </>
) : null}
```

---

## Updated handleSaveShift with Enhanced Warnings

### ✅ Complete Validation:
```javascript
const handleSaveShift = () => {
  // Existing employee name validation
  if (!employeeName || employeeName.trim().length === 0) {
    alert('Please enter your employee name first');
    return;
  }

  // Existing shift date validation
  if (!shiftDate) {
    alert('Please select a shift date');
    return;
  }

  // NEW: Custom shift validation
  if (shiftType === 'custom') {
    const validation = validateCustomShift(customShiftHours, customShiftStart, shiftDate);
    if (!validation.valid) {
      alert(`⚠️ ${validation.error}`);
      return;
    }

    // Warn about very long shifts
    if (customShiftHours > 48) {
      if (!confirm(
        `⚠️ Long Custom Shift\n\n` +
        `You've entered a ${customShiftHours}-hour shift.\n` +
        `This will generate ${Math.ceil(customShiftHours / 8)} block${Math.ceil(customShiftHours / 8) > 1 ? 's' : ''}.\n\n` +
        `Are you sure this is correct?`
      )) {
        return;
      }
    }

    // Warn about multi-day shifts
    const [startHour] = customShiftStart.split(':').map(Number);
    const daysSpan = Math.floor((startHour + customShiftHours) / 24);

    if (daysSpan > 0) {
      const endDate = new Date(shiftDate + 'T12:00:00');
      endDate.setDate(endDate.getDate() + daysSpan);
      const endDateStr = endDate.toISOString().split('T')[0];

      if (!confirm(
        `📅 Multi-Day Custom Shift\n\n` +
        `This shift spans from ${shiftDate} to ${endDateStr} (${daysSpan + 1} day${daysSpan > 0 ? 's' : ''}).\n\n` +
        `OT codes will be assigned per block based on the day of the week.\n\n` +
        `Continue?`
      )) {
        return;
      }
    }

    // Warn if custom shift on stat holiday
    if (isStatHoliday(shiftDate, userSettings.statHolidays)) {
      if (!confirm(
        `🎉 Statutory Holiday Custom Shift\n\n` +
        `${shiftDate} is configured as a statutory holiday.\n\n` +
        `All blocks will use Overtime Code 263.\n\n` +
        `Continue?`
      )) {
        return;
      }
    }
  }

  const shiftTotals = calculateShiftTotals();

  // Existing override validation
  const hasAnyOverride = Object.values(blockOverrides).some(o => o.manualOT > 0);
  if (hasAnyOverride) {
    if (!confirm('⚠️ This shift contains manual OT overrides.\n\nPlease verify:\n• Override hours are correct\n• Reasons are properly documented\n\nContinue saving?')) {
      return;
    }
  }

  // Existing cross-month validation
  const shiftMonth = shiftDate.substring(0, 7);
  const isDifferentMonth = shiftMonth !== currentMonth;

  if (isDifferentMonth) {
    const shiftMonthDate = new Date(shiftDate + 'T12:00:00');
    const shiftMonthName = shiftMonthDate.toLocaleDateString('en-US', { month: 'long', year: 'numeric' });
    const currentMonthDate = new Date(currentMonth + '-01T12:00:00');
    const currentMonthName = currentMonthDate.toLocaleDateString('en-US', { month: 'long', year: 'numeric' });

    if (!confirm(`📅 Cross-Month Shift Detected\n\nYou are currently viewing: ${currentMonthName}\nThis shift date is: ${shiftMonthName}\n\nThe shift will be saved to ${shiftMonthName} (based on the shift start date).\n\nContinue saving?`)) {
      return;
    }
  }

  const newShift = {
    id: Date.now(),
    date: shiftDate,
    shiftType,
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

  const existingShifts = loadFromLocalStorage(employeeName, shiftMonth);
  const updatedShifts = [...existingShifts, newShift];

  saveToLocalStorage(employeeName, shiftMonth, updatedShifts);

  if (!isDifferentMonth) {
    setSavedShifts(updatedShifts);
  }

  // Clear form
  setShiftDate('');
  setShiftType('weekday');
  setCustomShiftHours(8); // Reset custom hours
  setCustomShiftStart('00:00'); // Reset custom start
  setRegularShiftStart('16:00');
  setRegularShiftEnd('00:00');
  setEmailData(Array(16).fill().map(() => ({
    total: 0,
    actionable: 0,
    calls: 0,
    comment: ''
  })));
  setShiftNotes('');
  setCollapsedBlocks({});
  setBlockOverrides({
    0: { manualOT: 0, reason: '' },
    1: { manualOT: 0, reason: '' }
  });
  setOpenOverrideCards({});

  if (isDifferentMonth) {
    const savedMonthDate = new Date(shiftDate + 'T12:00:00');
    const savedMonthName = savedMonthDate.toLocaleDateString('en-US', { month: 'long', year: 'numeric' });
    alert(`✅ Shift saved successfully to ${savedMonthName}!\n\nTo view this shift, change the month selector to ${savedMonthName}.`);
  } else {
    alert('✅ Shift saved successfully! Your data is automatically saved to your browser.');
  }
};
```

---

## Updated validateCustomShift Helper

### ✅ Complete Validation Function:
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

---

## Updated Implementation Checklist

### Phase 1: Foundation ✓
- [ ] Add state variables (`customShiftHours`, `customShiftStart`)
- [ ] Create `BLOCK_COLOR_CLASSES` constant (fix Tailwind issue)
- [ ] Update `generateTimeSlots()` function with complete weekend logic
- [ ] Update `currentTimeSlots` generation call

### Phase 2: Block Logic ✓
- [ ] Create `calculateCustomBlocks()` function
- [ ] Create `calculateBlockDate()` helper
- [ ] Update `calculateBlockTotals()` signature (add `blockHours` param)
- [ ] Update `calculateShiftBlocks()` to handle custom type

### Phase 3: UI ✓
- [ ] Add "Custom" option to shift type selector
- [ ] Create custom shift configuration panel UI
- [ ] Add quick preset buttons (4hr, 8hr, 12hr, 16hr, 24hr, 36hr, 48hr)
- [ ] Add end date display for multi-day shifts
- [ ] Add visual block preview
- [ ] Add stat holiday warning for custom shifts
- [ ] Update block rendering logic for dynamic blocks
- [ ] Add partial block visual indicators
- [ ] Fix block color classes (use static mapping)

### Phase 4: ExceptionOverrideCard ✓
- [ ] Add `blockHours` prop to component
- [ ] Update max override calculation
- [ ] Add partial block warning label
- [ ] Update memoization to include `blockHours`
- [ ] Update all call sites to pass `blockHours`

### Phase 5: State Management ✓
- [ ] Update `useEffect` for shift type changes
- [ ] Add `customShiftHours` to dependency array
- [ ] Create `handleCustomHoursChange()` with confirmation
- [ ] Add form data loss protection

### Phase 6: Persistence ✓
- [ ] Update `handleSaveShift()` validation
- [ ] Add custom shift-specific warnings (long shifts, multi-day, stat holidays)
- [ ] Store custom shift parameters in shift object
- [ ] Update saved shift display to show custom hours
- [ ] Update `handleClearForm()` to reset custom fields

### Phase 7: Excel Export ✓
- [ ] Update `getShiftTypeLabel()` function
- [ ] Add "Hours" column to block table
- [ ] Highlight partial blocks in Excel
- [ ] Add custom start time to individual sheets

### Phase 8: Validation ✓
- [ ] Create `validateCustomShift()` function
- [ ] Add long shift warning (> 48 hours)
- [ ] Add multi-day shift confirmation dialog
- [ ] Add stat holiday custom shift warning
- [ ] Add input constraints (min=1, max=168)

### Phase 9: Polish ✓
- [ ] Create `migrateShiftFormat()` for backward compatibility
- [ ] Add shift timeline display in UI
- [ ] Update all string labels and messages
- [ ] Test all existing shift types still work

### Phase 10: Testing ✓
- [ ] Run all 8 test cases
- [ ] Verify Excel export formatting
- [ ] Test edge cases (1 hour, 168 hours, midnight crossings)
- [ ] Verify overtime code accuracy for multi-day shifts
- [ ] Test partial block override limits
- [ ] Test stat holiday custom shifts
- [ ] Test form data loss confirmation

---

## Updated Test Cases

### Test Case 1: Single Partial Block (4 hours)
**Setup:**
- Custom shift: 4 hours, start 14:00
- Date: Friday, Oct 24, 2025
- 8 actionable emails, 0 calls
- Add 1.5h override

**Expected:**
- 1 block: "Block 1" (14:00-18:00, 4 hours, PARTIAL tag)
- Email OT: 2.0h
- Max override: 2.0h (not 8.0h!)
- User tries 1.5h override: ✓ Accepted
- Total: 3.5h OT, 0.5h SB (totals to 4.0)
- OT Code: 261 (Friday)
- Excel: Block highlighted in light orange

---

### Test Case 2: Custom Shift on Stat Holiday
**Setup:**
- Custom shift: 12 hours, start 08:00
- Date: July 1, 2025 (Canada Day, in config)
- Minimal emails

**Expected:**
- Red stat holiday warning banner shows
- User confirms "All blocks will use Code 263"
- 2 blocks: 8hr + 4hr partial
- Both blocks: OT Code 263 (stat holiday)
- Block buttons show "STAT HOLIDAY" badge

---

### Test Case 3: Multi-Day with Different OT Codes
**Setup:**
- Custom shift: 36 hours, start Friday 16:00
- Date: Friday, Oct 24, 2025

**Expected:**
- End date display: "Ends on Sun, Oct 26, 2025 (3 days total)"
- Confirmation: "This shift spans from 2025-10-24 to 2025-10-26"
- 5 blocks:
  - Block 1 (Fri 16:00-00:00, 8hr): Code 261
  - Block 2 (Sat 00:00-08:00, 8hr): Code 262
  - Block 3 (Sat 08:00-16:00, 8hr): Code 262
  - Block 4 (Sat 16:00-00:00, 8hr): Code 262
  - Block 5 (Sun 00:00-04:00, 4hr, PARTIAL): Code 262

---

### Test Case 4: Form Data Loss Protection
**Setup:**
- Custom shift: 8 hours, enter email data in 3 time slots
- Change hours to 12

**Expected:**
- Confirmation dialog: "This will reset all entered data... Continue?"
- If user clicks Cancel: Hours stay at 8, data preserved
- If user clicks OK: Hours change to 12, form resets

---

### Test Case 5: Quick Presets
**Setup:**
- Select "Custom" shift type
- Click "12hr" preset button

**Expected:**
- `customShiftHours` instantly becomes 12
- Preview shows: "1 full block + 1 partial (4hr)"
- 12 time slots generated
- 12hr button highlighted in blue

---

## Estimated Time (Updated)

| Phase | Estimated Time | Notes |
|-------|----------------|-------|
| Phase 1: Foundation | 2 hours | +30min for Tailwind fix |
| Phase 2: Block Logic | 3 hours | Unchanged |
| Phase 3: UI | 5 hours | +1hr for presets/preview/warnings |
| Phase 4: ExceptionOverrideCard | 1 hour | New separate phase |
| Phase 5: State Management | 1 hour | +30min for confirmation handler |
| Phase 6: Persistence | 2 hours | +30min for additional warnings |
| Phase 7: Excel Export | 2 hours | Unchanged |
| Phase 8: Validation | 1 hour | Unchanged |
| Phase 9: Testing & Polish | 3 hours | +30min for new test cases |
| **Total** | **20 hours** | Was 16, now 18 + 2 buffer |

---

## Critical Success Criteria (Updated)

- [✓] User can create custom shifts with any duration (1-168 hours)
- [✓] System generates correct number of blocks (full + partial if needed)
- [✓] Partial blocks calculate OT/SB correctly (totaling to < 8 hours)
- [✓] Partial blocks show correct override limits (< 8.0 max)
- [✓] Multi-day shifts assign correct OT codes per block
- [✓] Stat holidays on custom shifts show warning and use code 263
- [✓] All existing shift types still work without changes
- [✓] Excel export includes custom shift data with proper formatting
- [✓] Backward compatible with old saved shifts
- [✓] No performance degradation with custom shifts
- [✓] **NEW:** Tailwind classes work correctly (no dynamic classes)
- [✓] **NEW:** Form data loss protection works
- [✓] **NEW:** Quick presets provide shortcuts
- [✓] **NEW:** Visual block preview helps users understand structure

---

## Final Implementation Order

1. **Phase 1** (Foundation) - 2.5 hours
   - Add state variables
   - Create `BLOCK_COLOR_CLASSES` constant
   - Update `generateTimeSlots()` with full weekend logic
   - Update `currentTimeSlots` generation

2. **Phase 2** (Block Logic) - 3 hours
   - Create `calculateCustomBlocks()`
   - Update `calculateBlockTotals()` with `blockHours` param
   - Update `calculateShiftBlocks()` for custom type

3. **Phase 3** (UI - Configuration Panel) - 2.5 hours
   - Add "Custom" to selector
   - Create configuration panel
   - Add quick presets
   - Add end date display
   - Add visual block preview
   - Add stat holiday warning

4. **Phase 4** (UI - Block Rendering) - 2.5 hours
   - Update block rendering for custom shifts
   - Fix block color classes (static mapping)
   - Add PARTIAL and STAT HOLIDAY badges

5. **Phase 5** (ExceptionOverrideCard) - 1 hour
   - Add `blockHours` prop
   - Update max override calculation
   - Add partial block warning

6. **Phase 6** (State Management) - 1.5 hours
   - Update `useEffect` dependencies
   - Create `handleCustomHoursChange()` with confirmation

7. **Phase 7** (Persistence) - 2 hours
   - Update `handleSaveShift()` with all validations
   - Add warning dialogs

8. **Phase 8** (Excel Export) - 2 hours
   - Update labels and headers
   - Add "Hours" column
   - Highlight partial blocks

9. **Phase 9** (Testing) - 3 hours
   - Run all test cases
   - Edge case testing
   - Regression testing

**Total: 20 hours**

---

## Ready to Implement? 🚀

All critical issues from the review have been addressed:
- ✅ Tailwind dynamic classes → Static mapping
- ✅ currentTimeSlots → Updated with custom params
- ✅ ExceptionOverrideCard → Added blockHours prop
- ✅ Weekend slot generation → Complete code included
- ✅ Form data loss → Confirmation dialog added

All improvements incorporated:
- ✅ Quick presets
- ✅ End date display
- ✅ Visual block preview
- ✅ Stat holiday warnings

All questions answered:
- ✅ Stat holiday handling: Show warning, don't auto-switch
- ✅ Custom on stat holidays: Yes, use code 263
- ✅ Non-hour start times: Keep on-the-hour

**Next step:** Start implementing Phase 1 (Foundation)?
