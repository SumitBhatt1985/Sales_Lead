# UI Specification: Sales Lead Management Platform

## Design System

### Color Palette

```
Primary Colors:
- Primary Blue: #2563EB (Buttons, Links, Active States)
- Primary Dark: #1E40AF (Hover States)
- Primary Light: #DBEAFE (Backgrounds, Highlights)

Secondary Colors:
- Success Green: #10B981 (Won deals, Completed actions)
- Warning Orange: #F59E0B (Overdue items, Medium priority)
- Error Red: #EF4444 (Lost deals, Critical items)
- Info Purple: #8B5CF6 (Defence/Aerospace indicators)

Neutral Colors:
- Gray 900: #111827 (Primary Text)
- Gray 700: #374151 (Secondary Text)
- Gray 500: #6B7280 (Tertiary Text, Icons)
- Gray 300: #D1D5DB (Borders)
- Gray 100: #F3F4F6 (Backgrounds)
- Gray 50: #F9FAFB (Page Background)
- White: #FFFFFF (Cards, Modals)
```

### Typography

```
Font Family: Inter, system-ui, sans-serif

Headings:
- H1: 32px / 600 weight / 40px line-height
- H2: 24px / 600 weight / 32px line-height
- H3: 20px / 600 weight / 28px line-height
- H4: 16px / 600 weight / 24px line-height

Body:
- Large: 16px / 400 weight / 24px line-height
- Regular: 14px / 400 weight / 20px line-height
- Small: 12px / 400 weight / 16px line-height

Labels:
- 14px / 500 weight / 20px line-height / Letter-spacing: 0.5px
```

### Spacing Scale

```
xs: 4px
sm: 8px
md: 16px
lg: 24px
xl: 32px
2xl: 48px
3xl: 64px
```

### Component Library

#### Buttons

```
Primary Button:
  Background: #2563EB | Hover: #1E40AF | Active: #1D4ED8
  Text: White | Padding: 10px 20px | Border-radius: 8px | Height: 40px
  Font: 14px / 500

Secondary Button:
  Background: White | Hover: #F9FAFB | Border: 1px solid #D1D5DB
  Text: #374151 | Padding: 10px 20px | Border-radius: 8px | Height: 40px

Danger Button:
  Background: #EF4444 | Hover: #DC2626
  Text: White | Padding: 10px 20px | Border-radius: 8px | Height: 40px

Ghost Button:
  Background: Transparent | Hover: #F3F4F6
  Text: #2563EB | Padding: 10px 20px | Border-radius: 8px | Height: 40px

Icon Button:
  Background: Transparent | Hover: #F3F4F6
  Size: 32x32px | Border-radius: 6px
```

#### Status Badges

```
New:         Background: #DBEAFE | Text: #1E40AF | Border-radius: 9999px | Padding: 2px 8px
Contacted:   Background: #F3F4F6 | Text: #374151
Discovery:   Background: #FEF3C7 | Text: #92400E
Qualified:   Background: #D1FAE5 | Text: #065F46
Demo/POC:    Background: #EDE9FE | Text: #5B21B6
Proposal:    Background: #DBEAFE | Text: #1E40AF
Negotiation: Background: #FEF3C7 | Text: #92400E
Won:         Background: #D1FAE5 | Text: #065F46 | Font: 600
Lost:        Background: #FEE2E2 | Text: #991B1B
On Hold:     Background: #F3F4F6 | Text: #374151

Priority Badges:
  High:   Background: #FEE2E2 | Text: #991B1B
  Medium: Background: #FEF3C7 | Text: #92400E
  Low:    Background: #F3F4F6 | Text: #374151
```

#### Input Fields

```
Default:
  Height: 40px | Border: 1px solid #D1D5DB | Border-radius: 8px
  Padding: 0 12px | Background: White | Font: 14px / #111827

Focus:
  Border: 2px solid #2563EB | Box-shadow: 0 0 0 3px #DBEAFE

Error:
  Border: 2px solid #EF4444 | Box-shadow: 0 0 0 3px #FEE2E2
  Error text below: 12px / #EF4444

Textarea:
  Min-height: 80px | Padding: 8px 12px | Resize: vertical

Select:
  Same as input + dropdown arrow icon
```

#### Cards

```
Standard Card:
  Background: White | Border: 1px solid #E5E7EB
  Border-radius: 12px | Padding: 20px | Box-shadow: 0 1px 3px rgba(0,0,0,0.1)

Hover Card (Kanban):
  Box-shadow: 0 4px 12px rgba(0,0,0,0.15) | Transform: translateY(-1px)

Stat Card:
  Background: White | Border-radius: 12px | Padding: 24px
  Box-shadow: 0 1px 3px rgba(0,0,0,0.1)
```

---

## Layout Structure

### App Shell

```
┌─────────────────────────────────────────────────────────────────┐
│  Top Navigation Bar (height: 64px, bg: #FFFFFF, border-bottom)  │
│  [Logo]  [Search]                    [Notif Bell] [User Avatar]  │
├──────────────────┬──────────────────────────────────────────────┤
│                  │                                               │
│   Sidebar        │   Main Content Area                          │
│   (width: 256px) │   (flex: 1, bg: #F9FAFB, padding: 24px)    │
│   bg: #111827    │                                               │
│   text: white    │                                               │
│                  │                                               │
│                  │                                               │
│                  │                                               │
└──────────────────┴──────────────────────────────────────────────┘
```

### Sidebar Navigation

```
Sidebar (256px wide, bg: #111827, full height)
├── Logo Area (height: 64px, padding: 16px 20px)
│   [Griffyn Logo] "SalesIQ"
│
├── Navigation Items (padding: 8px 12px)
│   ┌─────────────────────────────┐
│   │ 📊  Dashboard               │  ← Active: bg #1E40AF, rounded 8px
│   │ 👥  Leads                   │
│   │ 🏢  Accounts                │
│   │ 📅  Meetings                │
│   │ 🎯  Pipeline                │
│   │ ✅  Action Items            │
│   │ 🔍  AI Search               │
│   │ 📄  Reports                 │
│   └─────────────────────────────┘
│
├── Divider
│
├── Settings Items
│   │ ⚙️  Settings                │
│   │ 👤  My Profile              │
│
└── User Card (bottom, padding: 16px)
    [Avatar] Name | Role
    [Logout icon]
```


---

## Screen 1: Dashboard

### Layout

```
┌───────────────────────────────────────────────────────────────────┐
│  Dashboard                                         [Date Filter ▾] │
├───────────────────────────────────────────────────────────────────┤
│  Stat Cards Row (grid: 4 columns, gap: 20px)                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────┐ │
│  │ Total Leads  │ │ Pipeline Val │ │ Won This Qtr │ │ Overdue  │ │
│  │    247       │ │   $2.4M      │ │    $450K     │ │    12    │ │
│  │ +12% ↑       │ │ +5.2% ↑      │ │ +18% ↑       │ │ -3 ↓     │ │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────┘ │
├───────────────────────────────────────────────────────────────────┤
│  Charts Row (grid: 2 columns, gap: 20px)                         │
│  ┌──────────────────────────────┐ ┌──────────────────────────┐   │
│  │ Pipeline by Stage            │ │ Leads by Status          │   │
│  │                              │ │                          │   │
│  │ [Funnel/Bar Chart]           │ │ [Donut Chart]            │   │
│  │                              │ │                          │   │
│  └──────────────────────────────┘ └──────────────────────────┘   │
├───────────────────────────────────────────────────────────────────┤
│  Recent Activity Table                                            │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Lead Name        │ Status    │ Owner      │ Last Updated    │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ Automotive Co    │ Qualified │ John Smith │ 2 hours ago     │ │
│  │ Defence Project  │ Discovery │ Jane Doe   │ 5 hours ago     │ │
│  │ ...              │           │            │                 │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

### Components

**Stat Card:**
- Icon (top-left, 40x40px, bg: primary-light, rounded)
- Value (H1, gray-900)
- Label (Regular, gray-500)
- Trend indicator (Small, green/red with arrow icon)

**Chart Card:**
- Card header with title (H3) and dropdown filter
- Chart area using Recharts
- Legend below chart

**Activity Table:**
- Zebra striping (even rows: gray-50)
- Row hover: gray-100
- Clickable rows → navigate to lead detail

---

## Screen 2: Leads List

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Leads                                              [+ New Lead]     │
├─────────────────────────────────────────────────────────────────────┤
│  Filters Bar (height: 56px, bg: white, border-radius: 8px)         │
│  [🔍 Search...]  [Status ▾]  [Owner ▾]  [Priority ▾]  [Clear]      │
├─────────────────────────────────────────────────────────────────────┤
│  View Toggle:  [📋 Table] [📊 Kanban]                               │
├─────────────────────────────────────────────────────────────────────┤
│  TABLE VIEW                                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ ☐ │ Lead Name        │ Account     │ Status   │ Value │ Owner │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │ ☐ │ Automotive Deal  │ TechCorp    │ Qualified│ $120K │ John  │  │
│  │ ☐ │ Defence Contract │ AeroDefence │ Discovery│ $350K │ Jane  │  │
│  │ ☐ │ Industrial Auto  │ ManufCo     │ Proposal │ $85K  │ Mike  │  │
│  │ ...                                                             │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  Pagination: ← 1 2 3 ... 12 →                    Showing 1-50 of 247 │
└─────────────────────────────────────────────────────────────────────┘
```

### Table Specifications

```
Header Row:
  Height: 48px | Background: #F9FAFB | Border-bottom: 2px solid #E5E7EB
  Text: 12px / 600 / uppercase / gray-500 / letter-spacing: 0.5px
  Sortable columns have arrow icon (up/down based on sort direction)

Data Rows:
  Height: 64px | Border-bottom: 1px solid #E5E7EB
  Hover: background #F9FAFB | Cursor: pointer
  
Columns:
  1. Checkbox (40px) - bulk actions
  2. Lead Name (flex: 2) - bold, gray-900
  3. Account (flex: 1.5) - regular, gray-700, truncate with ellipsis
  4. Status (flex: 1) - Badge component
  5. Value (flex: 1) - right-aligned, formatted currency
  6. Owner (flex: 1) - avatar + name
  7. Actions (60px) - kebab menu icon
```

### Kanban View

```
┌─────────────────────────────────────────────────────────────────────┐
│  [New] →  [Contacted] →  [Discovery] →  [Qualified] →  [Proposal]  │
│                                                                      │
│  ┌──────┐   ┌──────┐      ┌──────┐       ┌──────┐      ┌──────┐   │
│  │ Card │   │ Card │      │ Card │       │ Card │      │ Card │   │
│  │ $50K │   │ $30K │      │ $120K│       │ $85K │      │ $200K│   │
│  └──────┘   └──────┘      └──────┘       └──────┘      └──────┘   │
│                                                                      │
│  ┌──────┐   ┌──────┐      ┌──────┐                                 │
│  │ Card │   │ Card │      │ Card │                                 │
│  │ $80K │   │ $45K │      │ $350K│                                 │
│  └──────┘   └──────┘      └──────┘                                 │
└─────────────────────────────────────────────────────────────────────┘
```

**Kanban Card:**
```
┌────────────────────────────────┐
│ Lead Name                  [⋮] │ ← gray-900, 14px/600
│ Account Name                   │ ← gray-500, 12px
│ ────────────────────────────── │
│ 💰 $120,000                    │ ← 16px/600, primary
│ 👤 John Smith                  │ ← gray-700, 12px
│ 📅 Close: Dec 2026             │ ← gray-500, 12px
└────────────────────────────────┘
Width: 280px | Padding: 16px | Drag cursor on hover
```


---

## Screen 3: Lead Detail Page

### Layout (Tabbed)

```
┌─────────────────────────────────────────────────────────────────────┐
│  ← Back   Automotive Robotics Vision System          [Edit] [Delete]│
│  Priority: High 🔴  |  Owner: John Smith  |  Created: Nov 2025      │
├─────────────────────────────────────────────────────────────────────┤
│  Status Pipeline (visual breadcrumb)                                │
│  New → Contacted → [Discovery] → Qualified → Proposal → ...        │
│                       ↑ current                                     │
│                     [Move to Qualified ▸]                          │
├─────────────────────────────────────────────────────────────────────┤
│  Info Row: [Account: TechCorp] [Value: $120,000] [Close: Dec 2026] │
├─────────────────────────────────────────────────────────────────────┤
│  Tabs:                                                               │
│  [Overview] [Meetings] [Contacts] [Opportunities] [Competitors]     │
│  [Action Items] [Documents] [AI Insights] [Notes]                   │
├─────────────────────────────────────────────────────────────────────┤
│  Tab Content Area                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Tab: Overview
```
Two-column layout (60% / 40%)

Left Column:
  ┌─────────────────────────────────────┐
  │ Lead Details Card                   │
  │ Lead Source:    Referral            │
  │ Industry:       Automotive          │
  │ Region:         South India         │
  │ Contact:        Rajesh Kumar        │
  │ Email:          r.kumar@techcorp.in │
  │ Mobile:         +91 9876543210      │
  └─────────────────────────────────────┘

  ┌─────────────────────────────────────┐
  │ Recent Activity Timeline            │
  │  ○ Status → Discovery  [2 days ago] │
  │  ○ Meeting Scheduled   [3 days ago] │
  │  ○ Lead Created        [1 week ago] │
  └─────────────────────────────────────┘

Right Column:
  ┌─────────────────────────────────────┐
  │ Deal Summary                        │
  │ Estimated Value: $120,000           │
  │ Expected Close:  Dec 31, 2026       │
  │ Budget Confirmed: ✗ No              │
  │ Decision Maker:  ✗ Not Met          │
  └─────────────────────────────────────┘

  ┌─────────────────────────────────────┐
  │ Open Action Items (3)               │
  │ • Send proposal draft  [Overdue 🔴] │
  │ • Technical demo setup [Due Fri]    │
  │ • Follow up on pricing [In Progress]│
  │ [View All →]                        │
  └─────────────────────────────────────┘
```

### Tab: AI Insights

```
┌─────────────────────────────────────────────────────────────────────┐
│  🤖 AI Insights                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 💡 Recommended Next Action                [Refresh]          │   │
│  │                                                              │   │
│  │ "Schedule a technical demo to address the client's          │   │
│  │  requirement for real-time defect detection. They mentioned  │   │
│  │  a Q1 budget cycle — send proposal before Dec 15."          │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ⚠️  Risk Signals                                              │   │
│  │  • Decision maker not met                                    │   │
│  │  • 2 overdue action items                                    │   │
│  │  • No meeting in 14 days                                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 🔗 Similar Leads (top 5)                                     │   │
│  │  • Automotive Vision System - MumbaiCo  (92% similar)       │   │
│  │  • Robotic Inspection - PuneCorp        (87% similar)       │   │
│  │  • Vision Inspection - AutoPlant        (84% similar)       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Screen 4: Create/Edit Lead Form

### Layout (Modal or Full Page)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Create New Lead                                           [✕ Close] │
├─────────────────────────────────────────────────────────────────────┤
│  SECTION 1: Basic Information                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Lead Name *                                                  │   │
│  │ [                                              ]             │   │
│  │                                                              │   │
│  │ Account *                     Lead Source                    │   │
│  │ [Search account...      ▾]    [Select source...      ▾]     │   │
│  │                                                              │   │
│  │ Owner *                       Priority                       │   │
│  │ [Search user...         ▾]    [● High  ○ Medium  ○ Low]     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  SECTION 2: Deal Information                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Estimated Value               Expected Close Date            │   │
│  │ [₹ ________________]          [📅 dd/mm/yyyy     ]           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ⚠️  Duplicate Warning (shown if similarity > 0.92):               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ ⚠️ Similar leads found:                                     │   │
│  │  • Automotive Vision Deal (TechCorp) — 94% similar          │   │
│  │  • [View Lead]  or  [Continue Creating]                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                              [Cancel]  [Create Lead]                 │
└─────────────────────────────────────────────────────────────────────┘
```


---

## Screen 5: Meeting Report Form

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Create Meeting Report                                    [✕ Close]  │
├─────────────────────────────────────────────────────────────────────┤
│  STEP INDICATOR                                                      │
│  [1. Meeting Info] → [2. Attendees] → [3. Discussion] → [4. Actions]│
│     ━━━━━━━━━━                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  STEP 1: Meeting Information                                         │
│                                                                      │
│  Meeting Type *                                                      │
│  ○ Commercial / Industrial    ○ Defence / Aerospace                  │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Lead *                        Meeting Date *                 │   │
│  │ [Search lead...        ▾]     [📅 dd/mm/yyyy     ]           │   │
│  │                                                              │   │
│  │ Client Company *              Meeting Time *                 │   │
│  │ [                        ]    [🕒 HH:MM          ]           │   │
│  │                                                              │   │
│  │ Location / Mode *                                            │   │
│  │ [○ On-site  ○ Virtual  ○ Phone]                             │   │
│  │                                                              │   │
│  │ --- Commercial Fields ---                                    │   │
│  │ Industry / Vertical           Plant / Facility Location      │   │
│  │ [                        ]    [                        ]     │   │
│  │                                                              │   │
│  │ --- Defence Fields (if Defence selected) ---                 │   │
│  │ Department / Directorate *    Security Classification *      │   │
│  │ [                        ]    [Confidential       ▾]         │   │
│  │                                                              │   │
│  │ Programme / Project Reference                                │   │
│  │ [                                                   ]        │   │
│  │                                                              │   │
│  │ Meeting Objective                                            │   │
│  │ [                                                   ]        │   │
│  │ [                                                   ]        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                         [Back]  [Next: Attendees →]                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 2: Attendees

```
┌─────────────────────────────────────────────────────────────────────┐
│  Attendees *                                         [+ Add Attendee]│
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Attendee 1:                                          [✕]     │   │
│  │ Name *              Designation                              │   │
│  │ [                ] [                                ]        │   │
│  │                                                              │   │
│  │ Role in Decision *                                           │   │
│  │ [Decision Maker         ▾]                                   │   │
│  │ (options: Decision Maker, Influencer, User, Gatekeeper)      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Attendee 2:                                          [✕]     │   │
│  │ ...                                                          │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                     [← Back]  [Next: Discussion →]                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 3: Discussion

```
┌─────────────────────────────────────────────────────────────────────┐
│  Requirement / Technical Discussion *                                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │  (Rich text area, min 3 rows)                               │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Competitive Landscape                                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  (Optional)                                                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Client Feedback / Concerns                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  (Optional)                                                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Deal Qualification Notes                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  (Optional)                                                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Internal Notes                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  (Optional, not visible to client)                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                     [← Back]  [Next: Actions →]                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: Action Items

```
┌─────────────────────────────────────────────────────────────────────┐
│  Action Items                                        [+ Add Action]  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Action 1:                                            [✕]     │   │
│  │ Action Item *                                                │   │
│  │ [                                                   ]        │   │
│  │                                                              │   │
│  │ Owner *                 Due Date *       Priority            │   │
│  │ [Search user... ▾]      [📅 date  ]      [Medium      ▾]    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  (Repeat for multiple action items)                                  │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                   [← Back]  [Save Meeting Report]                    │
└─────────────────────────────────────────────────────────────────────┘
```


---

## Screen 6: Meeting Detail View

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  ← Back   Commercial Meeting - TechCorp              [Edit] [Export]│
│  Meeting Date: Nov 15, 2026  |  Type: Commercial/Industrial        │
│  Security: None  |  Prepared by: John Smith                         │
├─────────────────────────────────────────────────────────────────────┤
│  SECTION: Meeting Information                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Client Company:    TechCorp Industries                       │   │
│  │ Industry/Vertical: Automotive Manufacturing                  │   │
│  │ Plant Location:    Pune, Maharashtra                         │   │
│  │ Location/Mode:     On-site                                   │   │
│  │ Objective:         Discuss robotic vision inspection system  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  SECTION: Attendees (3)                                              │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 👤 Rajesh Kumar           Plant Manager     Decision Maker   │   │
│  │ 👤 Priya Sharma           Quality Head      Influencer       │   │
│  │ 👤 Anil Verma             Operations        User             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  SECTION: Requirement Discussion                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Client requires automated vision inspection for detecting    │   │
│  │ paint defects on automotive body panels. Current manual      │   │
│  │ inspection causes 15% throughput loss...                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  SECTION: Competitive Landscape                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Competitor: VisionTech Systems                               │   │
│  │ Offering: Standard 2D inspection, slower cycle time          │   │
│  │ Our Edge: 3D vision + AI defect classification              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  SECTION: Action Items (2)                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ □ Send technical proposal with ROI analysis                  │   │
│  │   Owner: John Smith  |  Due: Nov 20, 2026  |  Priority: High│   │
│  │                                                              │   │
│  │ □ Schedule demo at Pune plant                                │   │
│  │   Owner: Jane Doe    |  Due: Nov 25, 2026  |  Priority: Med │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

[Export PDF] button: Downloads formatted PDF with all sections
```

---

## Screen 7: Pipeline / Opportunity View (Kanban)

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Pipeline                     [Forecast View] [Settings]             │
├─────────────────────────────────────────────────────────────────────┤
│  Filters: [Owner ▾] [Date Range ▾] [Product ▾]             [Clear]  │
├─────────────────────────────────────────────────────────────────────┤
│  Stage Summary Row                                                   │
│  Discovery: 8 ($450K) | Qualified: 12 ($1.2M) | Proposal: 5 ($800K) │
├─────────────────────────────────────────────────────────────────────┤
│  Kanban Board (horizontal scroll if needed)                          │
│                                                                      │
│  Discovery        Qualified       Demo/POC        Proposal           │
│  ──────────       ──────────      ──────────      ──────────         │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐       │
│  │ Opp Card│     │ Opp Card│     │ Opp Card│     │ Opp Card│       │
│  │ $120K   │     │ $350K   │     │ $85K    │     │ $200K   │       │
│  │ 30%     │     │ 50%     │     │ 60%     │     │ 75%     │       │
│  │ TechCorp│     │ AeroDef │     │ ManufCo │     │ AutoPlnt│       │
│  │ 👤 John │     │ 👤 Jane │     │ 👤 Mike │     │ 👤 Sara │       │
│  └─────────┘     └─────────┘     └─────────┘     └─────────┘       │
│                                                                      │
│  ┌─────────┐     ┌─────────┐                     ┌─────────┐       │
│  │ Opp Card│     │ Opp Card│                     │ Opp Card│       │
│  │ $80K    │     │ $95K    │                     │ $150K   │       │
│  └─────────┘     └─────────┘                     └─────────┘       │
│                                                                      │
│  + Add Opportunity                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Opportunity Card (Detailed)

```
┌──────────────────────────────────┐
│ Automotive Vision System     [⋮] │ ← 14px/600, gray-900
│ TechCorp Industries              │ ← 12px, gray-500
│ ──────────────────────────────── │
│ 💰 $120,000  |  📊 30%           │ ← Value & Probability
│ 👤 John Smith                    │ ← Owner avatar + name
│ 📅 Close: Dec 2026               │ ← Expected close
│ ──────────────────────────────── │
│ ✓ Budget Confirmed               │ ← Qualification indicators
│ ✗ Decision Maker Not Met         │
│ ⚠️ POC Required                  │
└──────────────────────────────────┘
Width: 300px | Min-height: 180px | Padding: 16px
Drag-and-drop enabled with visual feedback
```

### Forecast View (Toggle)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Weighted Pipeline Forecast                       [Export] [Share]   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │  [Funnel Chart with Weighted Values]                        │   │
│  │                                                              │   │
│  │  Discovery:     $450K × 30% = $135K                         │   │
│  │  Qualified:     $1.2M × 50% = $600K                         │   │
│  │  Demo/POC:      $400K × 60% = $240K                         │   │
│  │  Proposal:      $800K × 75% = $600K                         │   │
│  │  Negotiation:   $500K × 85% = $425K                         │   │
│  │                                                              │   │
│  │  Total Weighted Pipeline: $2.0M                             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Breakdown Table                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Stage       │ # Opps │ Total Value │ Avg Prob │ Weighted Val │   │
│  ├──────────────────────────────────────────────────────────────┤   │
│  │ Discovery   │   8    │   $450K     │   30%    │   $135K      │   │
│  │ Qualified   │  12    │  $1.2M      │   50%    │   $600K      │   │
│  │ Demo/POC    │   5    │   $400K     │   60%    │   $240K      │   │
│  │ ...         │        │             │          │              │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```


---

## Screen 8: Action Items

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  Action Items                                   [+ New Action Item]  │
├─────────────────────────────────────────────────────────────────────┤
│  Filters: [Status ▾] [Owner ▾] [Priority ▾] [Lead ▾]    [Clear]    │
│  View: [⊞ My Items] [All Items] [Overdue 🔴 12]                     │
├─────────────────────────────────────────────────────────────────────┤
│  OVERDUE SECTION (expanded by default)                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 🔴 OVERDUE (3)                                     [collapse]│   │
│  │ ────────────────────────────────────────────────────────     │   │
│  │ ✕ Send proposal to TechCorp           🔴 Was: Nov 10         │   │
│  │   John Smith  |  TechCorp Lead  |  High Priority             │   │
│  │                                                              │   │
│  │ ✕ Technical spec review               🔴 Was: Nov 12         │   │
│  │   Jane Doe    |  AeroDefence     |  High Priority            │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  DUE TODAY (2)                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ □ Follow up on demo scheduling        📅 Today               │   │
│  │ □ Send updated pricing sheet         📅 Today               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  UPCOMING (8)                                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ □ Prepare ROI analysis               📅 Nov 20               │   │
│  │ □ Internal POC review                📅 Nov 22               │   │
│  │ ...                                                          │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Action Item Row States

```
Normal Row:
  ┌────────────────────────────────────────────────────────────────┐
  │ [□] Send proposal to client           📅 Nov 20   [In Progress]│
  │     👤 John Smith  |  🔗 TechCorp Lead  |  🔴 High Priority    │
  └────────────────────────────────────────────────────────────────┘

Overdue Row:
  Background: #FFF5F5 | Left border: 4px solid #EF4444

Completed Row:
  Background: #F9FAFB | Text: line-through gray-400

Hover Row:
  Background: #F9FAFB | Show quick-action buttons [Complete] [Edit]
```

---

## Screen 9: AI Search Page

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  AI Search                                                           │
├─────────────────────────────────────────────────────────────────────┤
│  Search Input Area                                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 🔍 Ask anything about your leads, meetings, or clients...    │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Suggested Queries:                                                  │
│  [Show overdue leads]  [Pricing concerns]  [Unmet decision makers]  │
│  [High value stalled deals]  [Recent defence meetings]              │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│  RESULTS (shown after search)                                        │
│                                                                      │
│  12 results found for "pricing concerns and POC request"            │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 📋 Meeting Report                            Score: 0.9423   │   │
│  │ TechCorp — Nov 10, 2026                                      │   │
│  │ "...client expressed concern about pricing compared to       │   │
│  │  VisionTech. Requested POC at plant to validate ROI..."      │   │
│  │ [View Meeting →]                                             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 👥 Lead Note                                 Score: 0.8891   │   │
│  │ AeroDefence Contract — Discovery Stage                       │   │
│  │ "...pricing was flagged as a concern in internal notes.      │   │
│  │  Budget not yet confirmed, POC approval pending..."          │   │
│  │ [View Lead →]                                                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Filter By: [All] [Meeting Reports] [Lead Notes] [Action Items]     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Screen 10: Account Detail Page

### Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  ← Back   TechCorp Industries                  [Edit Account]        │
│  Industry: Automotive | Status: Active | Location: Pune, India      │
├─────────────────────────────────────────────────────────────────────┤
│  Tabs: [Overview] [Contacts] [Leads] [Meetings] [Documents]         │
├─────────────────────────────────────────────────────────────────────┤
│  Overview                                                            │
│  ┌─────────────────────────────┐  ┌──────────────────────────────┐  │
│  │ Account Info                │  │ Relationship Summary         │  │
│  │ Website: www.techcorp.in    │  │ Total Leads: 4               │  │
│  │ Status: Existing Customer   │  │ Active Opportunities: 2      │  │
│  │ Account Owner: John Smith   │  │ Total Pipeline: $320K        │  │
│  │                             │  │ Last Meeting: Nov 15, 2026   │  │
│  └─────────────────────────────┘  └──────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│  Contacts (3)                                        [+ Add Contact] │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 👤 Rajesh Kumar  │  Plant Manager  │ Decision Maker  │ [Edit]│   │
│  │ 👤 Priya Sharma  │  Quality Head   │ Influencer      │ [Edit]│   │
│  │ 👤 Anil Verma    │  Operations     │ User            │ [Edit]│   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Screen 11: Notification Center

### Notification Bell Dropdown

```
Top-right corner bell icon with badge (unread count)

┌────────────────────────────────────┐
│ Notifications            [Mark all]│
├────────────────────────────────────┤
│ 🔴 Overdue: "Send proposal"        │
│    TechCorp Lead · 2 hours ago     │
├────────────────────────────────────┤
│ 👤 Lead Assigned: Defence Proj     │
│    Assigned by: Manager · 4h ago   │
├────────────────────────────────────┤
│ ✅ Action Completed: Demo setup    │
│    Jane Doe · 6 hours ago          │
├────────────────────────────────────┤
│      [View All Notifications]      │
└────────────────────────────────────┘

Width: 360px | Max-height: 400px (scrollable)
```

---

## Screen 12: Mobile PWA Views

### Mobile Navigation (Bottom Tab Bar)

```
┌────────────────────────────────────────┐
│   Mobile Header (56px)                 │
│   [☰] SalesIQ              [🔔] [👤]  │
├────────────────────────────────────────┤
│                                        │
│   Content Area                         │
│   (full width, padding: 16px)          │
│                                        │
│                                        │
├────────────────────────────────────────┤
│  Bottom Tab Bar (56px, bg: white)      │
│  [📊] [👥] [📅] [✅] [🔍]            │
│  Dash Leads Meet Tasks Search          │
└────────────────────────────────────────┘
```

### Mobile: Quick Lead Create (FAB)

```
Floating Action Button (FAB):
  Position: fixed bottom-right (margin: 24px)
  Size: 56x56px | Background: #2563EB | Shadow: deep
  Icon: + | Color: white | Border-radius: 50%

On Tap → Bottom Sheet slides up:
┌────────────────────────────────────────┐
│ ╌╌╌╌╌╌╌╌ (drag handle)               │
│ Quick Create Lead                      │
├────────────────────────────────────────┤
│ Lead Name *                            │
│ [                                  ]   │
│                                        │
│ Account *                              │
│ [Search account...              ▾]     │
│                                        │
│ Priority                               │
│ [🔴 High] [🟡 Medium] [⚪ Low]        │
│                                        │
│     [Cancel]    [Create Lead]          │
└────────────────────────────────────────┘
```

### Mobile: My Action Items

```
┌────────────────────────────────────────┐
│ My Actions              [Today] [All]  │
├────────────────────────────────────────┤
│ 🔴 OVERDUE                             │
│ ┌──────────────────────────────────┐   │
│ │ Send proposal - TechCorp        │   │
│ │ Due: Nov 10    [Done] [Details] │   │
│ └──────────────────────────────────┘   │
├────────────────────────────────────────┤
│ 📅 TODAY                               │
│ ┌──────────────────────────────────┐   │
│ │ Follow up demo scheduling       │   │
│ │ Due: Today     [Done] [Details] │   │
│ └──────────────────────────────────┘   │
└────────────────────────────────────────┘
```

### Offline Banner

```
When offline:
┌────────────────────────────────────────┐
│ 🔴 You're offline. Drafts saved        │
│    locally. Will sync when online.     │
└────────────────────────────────────────┘
Height: 40px | Background: #FEF3C7 | Text: #92400E
Position: top of content area, below header
```

---

## Interaction States

### Loading States

```
Page Load Skeleton:
  Use shimmer animation on card placeholders
  Color: linear-gradient(90deg, #F3F4F6, #E5E7EB, #F3F4F6)
  Animation: 1.5s ease-in-out infinite

Button Loading:
  Replace text with spinner icon
  Disable button, opacity: 0.7

Table Loading:
  Show 5 skeleton rows with column-matching widths
```

### Error States

```
Field Validation Error:
  Red border + red helper text below field
  Icon: ⚠️ before error message

Form Submit Error (banner):
  Background: #FEE2E2 | Border-left: 4px solid #EF4444
  Padding: 12px 16px | Icon: ✕ | Dismissible

Empty State:
  Centered illustration (120x120px)
  H3 title + body text + CTA button
  Example: "No leads yet. Create your first lead to get started."
```

### Transition Animations

```
Page transitions: fade-in 150ms ease
Modal open: scale 0.95→1 + opacity 0 → 1, 200ms
Dropdown open: translateY(-4px)→0 + opacity 0→1, 150ms
Toast notifications: slide in from right, 300ms
Kanban card drag: scale 1.02, rotate 1.5deg, shadow increase
```

---

## Responsive Breakpoints

```
Mobile:  < 768px  (single column, bottom tabs, condensed cards)
Tablet:  768–1024px (2-column layouts, collapsible sidebar)
Desktop: > 1024px (full sidebar, multi-column, data tables)

Sidebar behavior:
  Desktop: Always visible (256px)
  Tablet:  Collapsible to icon-only (64px)
  Mobile:  Hidden, accessible via hamburger → overlay drawer
```

