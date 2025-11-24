# PHAC Overtime Calculator - Complete Business Logic Analysis

## Executive Summary

This application tracks overtime and standby hours for **Public Health Agency of Canada (PHAC)** duty officers who monitor communications during off-hours. The system calculates compensation based on email workload, phone calls, and exceptional circumstances, adhering to government compensation regulations (GC 179 form).

---

## 1. SHIFT STRUCTURE

### 1.1 Shift Types

| Shift Type | Duration | Time Range | Use Case |
|------------|----------|------------|----------|
| **Weekday** | 16 hours | 16:00 → 08:00 (next day) | Monday-Thursday on-call |
| **Weekend** | 64 hours | Friday 16:00 → Monday 08:00 | Friday evening through Monday morning |
| **Stat Holiday** | 24 hours | 00:00 → 00:00 (next day) | Statutory holidays |

### 1.2 Time Block Organization

#### Weekday (16 hours → 2 blocks)
- **Evening Block:** 16:00-23:59 (8 hours)
- **Morning Block:** 00:00-08:00 (8 hours)

#### Weekend (64 hours → 8 blocks)
- Friday Evening: 16:00-23:59 (8 hours)
- Saturday Morning: 00:00-08:00 (8 hours)
- Saturday Afternoon: 08:00-16:00 (8 hours)
- Saturday Evening: 16:00-23:59 (8 hours)
- Sunday Morning: 00:00-08:00 (8 hours)
- Sunday Afternoon: 08:00-16:00 (8 hours)
- Sunday Evening: 16:00-23:59 (8 hours)
- Monday Morning: 00:00-08:00 (8 hours)

#### Stat Holiday (24 hours → 3 blocks)
- Block 1: 00:00-08:00 (8 hours)
- Block 2: 08:00-16:00 (8 hours)
- Block 3: 16:00-00:00 (8 hours)

---

## 2. OVERTIME CODES (Government Compensation Codes)

The system automatically assigns overtime codes based on the day of the week:

| Code | Description | Days |
|------|-------------|------|
| **260** | Regular weekday overtime | Monday-Thursday |
| **261** | Friday overtime | Friday |
| **262** | Weekend overtime | Saturday-Sunday |
| **263** | Statutory holiday overtime | Configured stat holidays |

### Weekend Block Code Logic
For 64-hour weekend shifts, each 8-hour block gets a specific code:
- Block 0 (Friday Evening): **261**
- Blocks 1-6 (Saturday-Sunday): **262**
- Block 7 (Monday Morning): **260**

---

## 3. OVERTIME CALCULATION RULES

### 3.1 Hourly Calculation (Per Time Slot)

**Formula:**
```
Email OT = Actionable Emails × 0.25 hours (15 minutes)
Call OT = Calls × 0.25 hours (15 minutes)
Total Hourly OT = min(Email OT + Call OT, 1.0 hour)
Hourly SB = 1.0 - Total Hourly OT
```

**Business Rule:** Each actionable email or call requires approximately 15 minutes of work. Maximum 1 hour of OT per hour (the rest is standby).

**Example:**
- 3 actionable emails + 1 call = (3 × 0.25) + (1 × 0.25) = 1.0 hour OT, 0.0 hour SB
- 2 actionable emails + 0 calls = 0.5 hour OT, 0.5 hour SB
- 0 actionable emails + 0 calls = 0.0 hour OT, 1.0 hour SB (full standby)

### 3.2 Block Calculation (Per 8-Hour Block)

**Email-Based OT Formula:**
```
Email-Based OT = Total Actionable Emails in Block × 0.25 hours
Capped at 8.0 hours
```

**Call-Based OT Formula:**
```
Call-Based OT = Total Calls in Block × 0.25 hours
```

**Workload OT:**
```
Workload OT = min(Email-Based OT + Call-Based OT, 8.0 hours)
```

**Manual Override OT:**
```
Override OT = User-entered hours for non-email work
Range: 0.00 to (8.0 - Email-Based OT)
```

**Final Block Calculation:**
```
Total Block OT = min(Workload OT + Override OT, 8.0 hours)
Block SB = 8.0 - Total Block OT
```

**Validation Rules:**
- Total Block OT cannot exceed 8.0 hours
- Override OT cannot be negative
- Override OT + Email-Based OT cannot exceed 8.0 hours

### 3.3 Shift Total Calculation

```
Shift Total OT = Sum of all Block OTs
Shift Total SB = Sum of all Block SBs
```

### 3.4 Monthly Aggregation

```
Monthly Total OT = Sum of all saved shifts' Total OT
Monthly Total SB = Sum of all saved shifts' Total SB
Monthly Total Hours = Monthly Total OT + Monthly Total SB
```

---

## 4. DATA MODEL

### 4.1 Hourly Email Data Structure
```javascript
{
  total: Number,          // Total emails received
  actionable: Number,     // Emails requiring action (subset of total)
  calls: Number,          // Phone calls received
  comment: String         // Optional notes for this hour
}
```

**Validation:**
- `actionable ≤ total` (enforced by system)
- If user increases `actionable` beyond `total`, system alerts and blocks
- If user decreases `total` below `actionable`, system auto-adjusts `actionable` down

### 4.2 Block Override Structure
```javascript
{
  manualOT: Number,       // 0.00 to 8.00 hours
  reason: String          // Optional explanation (e.g., "System maintenance")
}
```

### 4.3 Saved Shift Structure
```javascript
{
  id: Timestamp,                    // Unique identifier
  date: "YYYY-MM-DD",              // Shift date
  shiftType: "weekday|weekend|stat_holiday",
  regularShiftStart: "HH:MM",      // e.g., "16:00"
  regularShiftEnd: "HH:MM",        // e.g., "08:00"
  emailData: [HourlyData],         // Array of 16/24/64 hourly records
  totalOT: Number,                 // Calculated shift OT
  totalSB: Number,                 // Calculated shift SB
  blocks: [BlockCalculation],      // Array of block summaries
  notes: String,                   // General shift notes
  timeSlots: [String],             // e.g., ["16:00-17:00", ...]
  overtimeCode: String,            // "260", "261", "262", or "263"
  blockOverrides: {                // Override data per block
    0: {manualOT, reason},
    1: {manualOT, reason},
    ...
  }
}
```

---

## 5. EXCEPTION OVERRIDE SYSTEM

### 5.1 Purpose
Manual overrides allow duty officers to claim OT for work **not captured by email/call tracking**, such as:
- System maintenance
- Emergency response coordination
- Training or briefings
- Report writing
- Infrastructure issues

### 5.2 User Workflow
1. User clicks **"➕ Add Exception Override"** button for a block
2. Card expands showing:
   - Current email-based OT
   - Available override hours (max: 8.0 - email OT)
   - Reason text field (optional but recommended)
   - Real-time calculation preview
3. User enters override hours (validated: 0 to max available)
4. User optionally enters reason
5. System shows warning ⚠️ indicator on blocks with overrides
6. On save, user confirms understanding of overrides

### 5.3 Visual Indicators
- **Orange highlight:** Blocks with active overrides
- **⚠️ Symbol:** Displayed next to overridden blocks
- **Override summary:** Shown in shift details and Excel export

### 5.4 Collapsible Interface (Latest Feature)
- Override cards start collapsed to reduce visual clutter
- Toggle button shows current override status:
  - "➕ Add Exception Override" (no override)
  - "⚠️ Override Active: +X.XXh" (override present)
- Prevents performance issues when typing reasons (uses local state + onBlur sync)

---

## 6. STATUTORY HOLIDAY MANAGEMENT

### 6.1 Configuration
- Users configure stat holidays in **Settings** modal
- Holidays stored as dates: `["2025-01-01", "2025-07-01", ...]`
- Dates displayed with full formatting: "Monday, January 1, 2025"

### 6.2 Auto-Detection Logic
When user selects a shift date:

```javascript
if (date is in statHolidays array) {
  → Auto-switch to "Stat Holiday" shift type
  → Fix shift time to 00:00 - 00:00
  → Show red banner with holiday details
  → Set overtime code to 263
}
else if (date is NOT in statHolidays but shift type is still stat_holiday) {
  → Revert to appropriate type:
     - Monday-Thursday → Weekday
     - Friday-Sunday → Weekend
}
```

### 6.3 User Experience
- **Red gradient banner:** Shown when stat holiday detected
- **Disabled time fields:** Start/end times locked to 00:00 for stat holidays
- **24-hour tracking:** Three 8-hour blocks instead of two

---

## 7. CROSS-MONTH SHIFT HANDLING

### 7.1 Problem Scenario
Weekend shifts (Friday 16:00 → Monday 08:00) can span month boundaries:
- Example: Shift starts **October 31** (Friday) and ends **November 3** (Monday)

### 7.2 Business Rule
**Shifts are saved to the month of the shift START date**, not the current viewing month.

### 7.3 User Experience
```javascript
if (shift date month ≠ current viewing month) {
  → Show confirmation dialog:
     "You are viewing: October 2025
      This shift date is: November 2025
      Shift will be saved to November 2025.
      Continue?"
  → On save success:
     "✅ Shift saved to November 2025!
      To view, change month selector to November 2025."
}
```

### 7.4 Implementation
```javascript
const shiftMonth = shiftDate.substring(0, 7); // "2025-11"
const existingShifts = loadFromLocalStorage(employeeName, shiftMonth);
saveToLocalStorage(employeeName, shiftMonth, [...existingShifts, newShift]);
```

---

## 8. DATA PERSISTENCE

### 8.1 Storage Strategy
- **Technology:** Browser LocalStorage (client-side)
- **Key Format:** `ot_tracker_[employee_name_lowercase]`
- **Value Structure:**
```javascript
{
  "2025-10": [shift1, shift2, ...],
  "2025-11": [shift3, shift4, ...],
  ...
}
```

### 8.2 Auto-Save Behavior
- Data loads automatically when employee name entered
- Data saves automatically on every shift save
- Uses React `useEffect` hooks to trigger saves on state changes

### 8.3 Storage Key Generation
```javascript
employeeName = "John Doe";
storageKey = "ot_tracker_john_doe";
```

### 8.4 Settings Storage
- Separate key: `ot_tracker_settings`
- Stores: Statutory holiday dates

---

## 9. EXCEL EXPORT SYSTEM

### 9.1 Export Sheets Generated

#### Sheet 1: Monthly Summary
- Employee name and month header
- Key metrics cards:
  - Total Shifts, Total OT Hours, Total SB Hours, Total Hours
  - Total Emails, Total Actionable, Total Calls
- Shift breakdown table:
  - Date, Type, OT Hours, SB Hours, Total Emails, Actionable, Calls, OT Code
- Color-coded OT codes:
  - 260 = Green (E2EFDA)
  - 261 = Orange (FCE4D6)
  - 262 = Blue (DDEBF7)
  - 263 = Red (for stat holidays)

#### Sheet 2-N: Individual Shift Details
- One sheet per saved shift
- RegularShiftStart/End fields
- **Block Calculations Table:**
  - Block Name, StartTime, EndTime, Actionable, TotalEmails, Email OT, Override OT, Total OT, SB, Code
  - Rows with overrides highlighted in orange (FFF4E6)
  - ⚠️ prefix on block names with overrides
- **Override Section (if applicable):**
  - Separate table listing override reasons
  - Orange header and background
- **Hourly Detail Table:**
  - Hour, Time, TotalEmails, ActionableEmails, OT, SB, Calls, Comments
  - Alternating row colors for readability

#### Sheet N+1: All Shifts Detail
- Combined hourly data from all shifts
- Columns: Date, Type, Hour, Time, Total, Actionable, Calls, OT, SB, Comments
- Useful for bulk analysis

#### Sheet N+2: GC 179 Summary
- Government form-ready format
- Columns: Date, Total OT Hours, Total SB Hours, OT Code, Total Emails, Actionable Emails, Notes
- Summary section with totals
- Color-coded by OT code

### 9.2 Excel Formatting Features
- Professional styling with borders and colors
- Bold headers with white text on colored backgrounds
- Alternating row colors (white/light gray)
- Auto-sized columns for readability
- Merged cells for section headers
- Number formatting (2 decimal places for hours)

### 9.3 Export Filename
```
OT_Tracker_[EmployeeName]_[YYYY-MM].xlsx
Example: OT_Tracker_John_Doe_2025-10.xlsx
```

---

## 10. VALIDATION & BUSINESS RULES

### 10.1 Form Validation

| Field | Rule | Error Handling |
|-------|------|----------------|
| Employee Name | Required | Alert on save attempt |
| Shift Date | Required | Alert on save attempt |
| Actionable Emails | ≤ Total Emails | Alert + block save |
| Override Hours | 0 ≤ x ≤ (8.0 - email OT) | Alert + block save |
| Total Block OT | ≤ 8.0 hours | Automatic capping |

### 10.2 Confirmation Dialogs

**Override Warning (on save):**
```
⚠️ This shift contains manual OT overrides.

Please verify:
• Override hours are correct
• Reasons are properly documented

Continue saving?
```

**Cross-Month Warning:**
```
📅 Cross-Month Shift Detected

You are currently viewing: October 2025
This shift date is: November 2025

The shift will be saved to November 2025 (based on shift start date).

Continue saving?
```

### 10.3 Data Integrity Rules
1. **Hourly totals:** Each hour contributes max 1.0 hour (OT + SB = 1.0)
2. **Block totals:** Each block contributes max 8.0 hours (OT + SB = 8.0)
3. **No negative values:** All time values ≥ 0
4. **Automatic capping:** System prevents over-allocation
5. **Reason optional:** Override reasons encouraged but not required (changed from required)

---

## 11. USER INTERFACE FEATURES

### 11.1 Collapsible Sections
- **Add New Shift:** Toggle to show/hide entire form
- **Time Blocks:** Individual blocks collapse to reduce scrolling
- **Override Cards:** Toggle to show/hide override input (latest feature)

### 11.2 Visual Indicators
- **Color-coded blocks:** Different colors for each 8-hour period
- **Orange highlights:** Blocks with manual overrides
- **Red banners:** Statutory holiday shifts
- **Green success messages:** Data save confirmations
- **Warning symbols:** ⚠️ for overrides, 📅 for cross-month, 🎉 for stat holidays

### 11.3 Responsive Design
- **Grid layouts:** Adapt to screen size (1/2/3/4 columns)
- **Mobile-friendly:** Touch-friendly buttons and inputs
- **Tailwind CSS:** Utility-first styling for consistency

### 11.4 Real-Time Calculations
- **Hourly preview:** Shows OT/SB as user types
- **Block preview:** Updates immediately with override changes
- **Shift totals:** Recalculated on every data change
- **Monthly summary:** Updates when shifts added/removed

---

## 12. PERFORMANCE OPTIMIZATIONS

### 12.1 React Memoization
```javascript
const ExceptionOverrideCard = React.memo(({ ... }), (prevProps, nextProps) => {
  // Only re-render if specific values change
  return (
    prevProps.emailBasedOT === nextProps.emailBasedOT &&
    prevProps.overrideHours === nextProps.overrideHours &&
    prevProps.overrideReason === nextProps.overrideReason &&
    prevProps.isOpen === nextProps.isOpen
  );
});
```

**Why:** Prevents entire form re-rendering while user types in override reason field.

### 12.2 Local State Pattern
```javascript
const [localReason, setLocalReason] = useState(overrideReason);

const handleReasonChange = (e) => {
  setLocalReason(e.target.value); // Update local only
};

const handleReasonBlur = () => {
  onUpdateReason(blockIndex, localReason); // Sync to parent on blur
};
```

**Why:** Typing in textarea updates local state only; parent state updates on blur, preventing expensive re-renders.

### 12.3 Efficient Data Loading
- Load data only when employee name provided
- Use `useEffect` with dependencies to minimize recalculations
- Memoize calculation functions where possible

---

## 13. PWA (Progressive Web App) FEATURES

### 13.1 Installation
- Service worker registration for offline capability
- Install prompt appears after 3 seconds
- Installable on mobile devices (Add to Home Screen)

### 13.2 Splash Screen
- Shows "OTCALC3" branding on load
- Auto-hides after 1 second
- Professional loading animation

### 13.3 App Manifest
```json
{
  "name": "OTCALC3 - PHAC Overtime Tracker",
  "short_name": "OTCALC3",
  "theme_color": "#1F4E78",
  "icons": [...]
}
```

---

## 14. BUSINESS RULES SUMMARY

### Core Principles:
1. **15-Minute Granularity:** Each actionable item = 0.25 hours
2. **8-Hour Blocks:** All compensation calculated in 8-hour segments
3. **Email-First:** Primary tracking via email workload
4. **Exception Handling:** Manual overrides for non-email work
5. **Government Compliance:** Overtime codes align with GC forms
6. **Transparency:** All calculations visible and auditable
7. **Month-Based Tracking:** Shifts belong to their start month
8. **Auto-Detection:** Stat holidays automatically adjust shift parameters

### Calculation Hierarchy:
```
Monthly Totals
  ↓
Shift Totals
  ↓
Block Totals (8 hours each)
  ↓
Hourly Slots (1 hour each)
  ↓
Individual Emails/Calls (0.25 hours each)
```

---

## 15. USE CASE SCENARIOS

### Scenario A: Quiet Weekday Shift
- **Setup:** Monday 16:00-08:00, 5 total emails, 2 actionable, 1 call
- **Calculation:**
  - Email OT: 2 × 0.25 = 0.5 hours
  - Call OT: 1 × 0.25 = 0.25 hours
  - Total OT: 0.75 hours
  - Total SB: 15.25 hours (16 - 0.75)

### Scenario B: Busy Weekend Block
- **Setup:** Saturday morning (8 hours), 20 actionable emails, 3 calls
- **Calculation:**
  - Email OT: 20 × 0.25 = 5.0 hours
  - Call OT: 3 × 0.25 = 0.75 hours
  - Workload OT: 5.75 hours
  - Total OT: 5.75 hours (under 8.0 cap)
  - SB: 2.25 hours

### Scenario C: Override for System Maintenance
- **Setup:** Friday evening, 12 actionable emails (3.0 hours OT), 2 hours maintenance work
- **Calculation:**
  - Email OT: 3.0 hours
  - Override OT: 2.0 hours (manually entered with reason "Emergency server restart")
  - Total Block OT: 5.0 hours
  - Block SB: 3.0 hours
- **Excel Export:** Shows ⚠️ warning and orange highlight

### Scenario D: Stat Holiday
- **Setup:** Canada Day (July 1), 24-hour shift, 8 actionable emails in evening block
- **Calculation:**
  - Shift type: Auto-detected as "Stat Holiday"
  - Times: Fixed to 00:00-00:00
  - OT Code: 263 (all blocks)
  - Block 3 (16:00-00:00): 8 emails × 0.25 = 2.0 hours OT, 6.0 hours SB
  - Blocks 1-2: 0 OT, 8.0 SB each (if no emails)

---

## 16. FUTURE ENHANCEMENT CONSIDERATIONS

### Potential Improvements:
1. **Cloud sync:** Multi-device access via backend database
2. **Team dashboards:** Manager view of all team members
3. **Approval workflow:** Submit shifts for supervisor review
4. **Automated reminders:** Prompt to log shifts after on-call period
5. **Historical analytics:** Trend analysis, busy period identification
6. **Integration:** Connect to email systems for auto-counting
7. **Mobile app:** Native iOS/Android apps
8. **Audit trail:** Track all edits to shift data

---

## 17. COMPLIANCE & DOCUMENTATION

### Government Form Alignment:
- **GC 179 Form:** Overtime compensation request form
- Excel export includes "GC 179 Summary" sheet ready for submission
- Overtime codes match government classification system
- Detailed hourly breakdown supports audit requirements

### Data Retention:
- Data persists in browser until manually cleared
- No server-side storage (privacy-friendly)
- Employee controls their own data
- Export to Excel for permanent records

---

## TECHNICAL STACK SUMMARY

- **Frontend:** React 18 (via CDN)
- **Styling:** Tailwind CSS (via CDN)
- **Excel Generation:** ExcelJS 4.3.0
- **Data Storage:** Browser LocalStorage
- **Build:** None (single HTML file, runs directly in browser)
- **PWA:** Service Worker, Web App Manifest

---

## CONCLUSION

This application implements a sophisticated yet user-friendly system for tracking government duty officer overtime. The core business logic centers on **email-based workload measurement** (0.25 hours per actionable item) organized into **8-hour blocks** with **manual override capabilities** for exceptional circumstances. The system ensures compliance with government compensation rules while providing detailed audit trails through comprehensive Excel exports.

**Key Strengths:**
✅ Clear, auditable calculations
✅ Automatic overtime code assignment
✅ Cross-month shift handling
✅ Statutory holiday auto-detection
✅ Professional reporting
✅ No backend required (privacy + simplicity)

**Key Constraints:**
⚠️ LocalStorage only (no multi-device sync)
⚠️ Manual data entry required
⚠️ No approval workflow
⚠️ Single-user per browser
