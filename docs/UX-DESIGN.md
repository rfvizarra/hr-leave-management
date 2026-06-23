# HR Leave Management — UX/UI Design Specification  

> **Version**: 1.1  
> **Audience**: Frontend developers implementing Vue 3 + Tailwind CSS v4  
> **Reference**: [SPEC.md](./SPEC.md) for functional requirements · [API-DESIGN.md](./API-DESIGN.md) for API contracts  

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Color System](#2-color-system)
3. [Typography](#3-typography)
4. [Spacing & Layout Grid](#4-spacing--layout-grid)
5. [Icon System](#5-icon-system)
6. [Common Components](#6-common-components)
7. [Application Shell](#7-application-shell)
8. [Screen Designs](#8-screen-designs)
   - [8.1 Login](#81-login)
   - [8.2 Employee Dashboard](#82-employee-dashboard)
   - [8.3 New Leave Request](#83-new-leave-request)
   - [8.4 Leave Detail](#84-leave-detail)
   - [8.5 Team Calendar](#85-team-calendar)
   - [8.6 Approval Queue](#86-approval-queue)
   - [8.7 Employee Balances](#87-employee-balances)
   - [8.8 HR — Employee List](#88-hr--employee-list)
   - [8.9 HR — Policy Editor](#89-hr--policy-editor)
   - [8.10 HR — Reports Dashboard](#810-hr--reports-dashboard)
   - [8.11 Admin — User Management](#811-admin--user-management)
9. [Dark Mode](#9-dark-mode)
10. [Responsive Strategy](#10-responsive-strategy)
11. [Accessibility](#11-accessibility)
12. [Animation & Motion](#12-animation--motion)

---

## 1. Design Philosophy  

**Guiding metaphor**: *Clarity through calm order*.  

An HR leave system handles sensitive, sometimes emotional decisions (medical leave, family emergencies, vacation denied). The interface must feel **trustworthy, calm, and efficient** — never cold, never chaotic.  

| Principle | Expression |
|-----------|-----------|
| **Trust** | Predictable layouts, clear status indicators, no surprises |
| **Clarity** | Generous whitespace, one primary action per view, progressive disclosure |
| **Efficiency** | Keyboard-first for power users (HR processes dozens of requests daily), buttery navigation |
| **Warmth** | Rounded corners (12px on cards, 8px on buttons), warm accent tones, human-readable dates |
| **Accessibility** | WCAG 2.2 AA minimum, 16px+ body text, 4.5:1+ contrast everywhere |

**Role-aware design**: The interface adapts to who you are. An employee sees their world — balances, their requests, team calendar. A manager sees pending approvals and team coverage. An HR administrator sees company-wide tools. Never show what the user cannot act upon.

---

## 2. Color System  

Built on the **OKLCH** color space for perceptual uniformity. Every color resolves to a full 50–950 shade scale, usable as Tailwind utilities (`bg-primary-500`, `text-accent-600`).

### 2.1 Light Theme  

| Token | Role | Key Shades |
|-------|------|------------|
| **Primary** | Main brand, primary buttons, active states, selected rows | Blue with subtle teal undertone — feels professional but not corporate |
| **Accent** | Call-to-action highlights, new-request button, positive indicators | Warm amber-gold — draws attention without alarm |
| **Neutral** | Text, backgrounds, borders, subtle surfaces | Cool gray — avoids yellow/brown casts common in warm grays |
| **Success** | Approved status, sufficient balance, confirmation toasts | Green — fresh, affirmative |
| **Warning** | Pending status, low balance, reminders | Amber — noticeable, not alarming |
| **Error** | Rejected status, validation errors, destructive actions | Red — urgent but not harsh (slightly desaturated) |
| **Info** | Informational banners, tips, help text | Soft blue |

#### Full Palette  

```
Primary                          Accent                           Neutral
 50  oklch(0.97 0.012 240)        50  oklch(0.97 0.025 85)        50  oklch(0.98 0.002 260)
100  oklch(0.93 0.030 240)       100  oklch(0.93 0.050 85)       100  oklch(0.94 0.004 260)
200  oklch(0.87 0.060 240)       200  oklch(0.87 0.075 82)       200  oklch(0.87 0.006 260)
300  oklch(0.80 0.090 240)       300  oklch(0.80 0.100 80)       300  oklch(0.78 0.008 260)
400  oklch(0.70 0.130 240)       400  oklch(0.72 0.130 78)       400  oklch(0.65 0.010 260)
500  oklch(0.57 0.170 245)  ←    500  oklch(0.65 0.165 75)  ←    500  oklch(0.52 0.012 260)
600  oklch(0.50 0.155 245)       600  oklch(0.57 0.155 72)       600  oklch(0.42 0.010 260)
700  oklch(0.42 0.130 245)       700  oklch(0.48 0.135 70)       700  oklch(0.34 0.008 260)
800  oklch(0.34 0.100 245)       800  oklch(0.39 0.110 68)       800  oklch(0.26 0.006 260)
900  oklch(0.26 0.070 245)       900  oklch(0.30 0.080 66)       900  oklch(0.18 0.004 260)
950  oklch(0.15 0.040 245)       950  oklch(0.18 0.045 66)       950  oklch(0.10 0.002 260)

Success          Warning          Error             Info
500  oklch(0.60 0.185 150)   500  oklch(0.72 0.160 80)   500  oklch(0.52 0.195 20)   500  oklch(0.62 0.090 250)
```

### 2.2 Semantic Color Usage  

| Context | Light | Dark |
|---------|-------|------|
| Page background | `neutral-50` | `neutral-950` |
| Card / surface | `white` | `neutral-900` |
| Primary button | `primary-600` bg, white text | `primary-500` bg, `neutral-950` text |
| Accent button (new request) | `accent-500` bg, `neutral-950` text | `accent-400` bg, `neutral-950` text |
| Body text | `neutral-800` | `neutral-200` |
| Muted text | `neutral-500` | `neutral-400` |
| Border | `neutral-200` | `neutral-700` |
| Input background | `white` | `neutral-800` |
| Hover state (row, card) | `primary-50` bg | `neutral-800` bg → `neutral-700` |
| Selected state | `primary-100` bg, `primary-700` left border | `primary-900/50` bg, `primary-400` border |
| Status: Approved | `success-100` bg, `success-700` text | `success-900/50` bg, `success-400` text |
| Status: Pending | `warning-100` bg, `warning-700` text | `warning-900/40` bg, `warning-400` text |
| Status: Rejected | `error-100` bg, `error-700` text | `error-900/50` bg, `error-400` text |
| Status: Draft | `neutral-200` bg, `neutral-600` text | `neutral-700` bg, `neutral-400` text |

### 2.3 Leave Type Color Coding (Calendar)  

Used in calendar views to visually distinguish leave types at a glance:  

| Leave Type | Calendar Color | Meaning |
|-----------|---------------|---------|
| Vacation | `oklch(0.65 0.180 235)` Blue | Planned time off |
| Sick Leave | `oklch(0.58 0.170 15)` Red-magenta | Unplanned absence |
| Birth / Childcare | `oklch(0.60 0.170 300)` Purple | Life event |
| Paid Leave (marriage, etc.) | `oklch(0.68 0.170 130)` Green | Entitled leave |
| Unpaid Leave | `oklch(0.55 0.040 260)` Gray-blue | Voluntary absence |
| Parental / Family | `oklch(0.70 0.160 55)` Amber-gold | Care leave |
| Excedencia | `oklch(0.55 0.030 260)` Slate | Long absence |
| Public Holiday | `oklch(0.60 0.100 40)` Warm gray | Non-working day |
| Bridge Day | `oklch(0.60 0.100 40)` with dashed border | Company-granted |

---

## 3. Typography  

### 3.1 Font Stack  

| Role | Font | Weight Range | Usage |
|------|------|-------------|-------|
| **Headings** | DM Sans Variable | 500–700 | Page titles, card headers, section labels |
| **Body** | Inter Variable | 400–600 | All body text, form labels, table cells, buttons |
| **Monospace** | JetBrains Mono (optional) | 400 | Employee codes, audit log IDs |

Self-hosted via `@fontsource`. No Google Fonts CDN.

### 3.2 Type Scale  

```
text-xs    0.75rem / 1rem       — Badges, fine print, table metadata
text-sm    0.875rem / 1.25rem   — Secondary text, form hints, sidebar items
text-base  1rem / 1.5rem        — Body text, form labels, table cells (MINIMUM for readability)
text-lg    1.125rem / 1.75rem   — Card titles, emphasized body
text-xl    1.25rem / 1.75rem    — Section headers, modal titles
text-2xl   1.5rem / 2rem        — Page titles (h1)
text-3xl   1.875rem / 2.25rem   — Dashboard hero numbers
text-4xl   2.25rem / 2.5rem     — Login page heading
```

### 3.3 Typography Rules  

- Body text never smaller than `text-base` (16px).  
- Headings use `font-heading` with `tracking-tight` (-0.02em).  
- All interactive text (buttons, links) has a visible focus ring (2px `primary-400`, offset 2px).  
- Line-height: 1.5 for body, 1.25 for headings, 1.0 for hero numbers.  
- Paragraph max-width: 65ch for readability.  

---

## 4. Spacing & Layout Grid  

### 4.1 Global Layout  

```
┌──────────────────────────────────────────────────────────┐
│  Sidebar (w-64 → w-16 collapsed)  │  Content Area        │
│  ┌──────────────────────────────┐  │  ┌───────────────┐  │
│  │ Logo + Tenant                │  │  │ Header (h-16) │  │
│  │ ─────────────────────────── │  │  │               │  │
│  │ Nav Group: Main              │  │  ├───────────────┤  │
│  │   Dashboard                  │  │  │               │  │
│  │   My Leaves                  │  │  │               │  │
│  │   Calendar                   │  │  │   Page        │  │
│  │   Balances                   │  │  │   Content     │  │
│  │ ─────────────────────────── │  │  │               │  │
│  │ Nav Group: Management        │  │  │               │  │
│  │   Team (if manager)          │  │  │               │  │
│  │   HR (if HR role)            │  │  │               │  │
│  │   Admin (if sys admin)       │  │  │               │  │
│  │ ─────────────────────────── │  │  │               │  │
│  │ User info + Collapse toggle  │  │  │               │  │
│  └──────────────────────────────┘  │  └───────────────┘  │
└──────────────────────────────────────────────────────────┘
```

- **Sidebar**: 256px expanded, 64px collapsed. Smooth CSS transition (200ms ease-out).  
- **Header**: 64px tall, sticky at top of content area.  
- **Content**: max-width 1280px for forms/detail views; full-width for tables/calendars. Centered with `mx-auto`.  
- **Page padding**: `p-6` (24px) on desktop, `p-4` on tablet, `p-3` on mobile.  

### 4.2 Card & Section Spacing  

```
+-- Section (mb-8) ------------------------------------+
|  Section Title (mb-4)                                 |
|  +-- Card (rounded-xl, p-6, shadow-sm, mb-4) -----+  |
|  |  Card content with internal p-4 gaps            |  |
|  +-------------------------------------------------+  |
|  +-- Card -----------------------------------------+  |
|  |  ...                                            |  |
|  +-------------------------------------------------+  |
+-------------------------------------------------------+
```

- Cards use `rounded-xl` (12px), `p-6`, a subtle `shadow-sm` (`0 1px 2px oklch(0 0 0 / 0.04)`), and `border border-neutral-200`.  
- Section titles are `text-xl font-heading font-semibold text-neutral-800`.  
- Gap between cards: `gap-4` (16px) in card grids.  
- Gap between sections: `mb-8` (32px).  

---

## 5. Icon System  

All icons from **Lucide Vue** (`lucide-vue-next`). Tree-shakeable — only shipped icons count toward bundle size.

### 5.1 Icon Map by Context  

| Context | Icon | Lucide Name |
|---------|------|-------------|
| Dashboard | Grid | `LayoutDashboard` |
| My Leaves | File text | `FileText` |
| New Request | Plus circle | `PlusCircle` |
| Calendar | Calendar | `Calendar` |
| Balances | Scale | `Scale` |
| Team | Users | `Users` |
| Approvals | Check check | `CheckCheck` |
| Employees | User round | `UserRound` |
| Policies | Shield | `Shield` |
| Reports | Bar chart | `BarChart3` |
| Settings | Settings | `Settings` |
| Notifications | Bell | `Bell` |
| Profile | User circle | `UserCircle` |
| Logout | Log out | `LogOut` |
| Search | Search | `Search` |
| Filter | Sliders | `SlidersHorizontal` |
| Download | Download | `Download` |
| Upload | Upload | `Upload` |
| Delete | Trash | `Trash2` |
| Edit | Pencil | `Pencil` |
| Approve | Check | `Check` |
| Reject | X | `X` |
| Cancel request | X circle | `XCircle` |
| Calendar sync | Refresh | `RefreshCw` |
| Document | File | `File` |
| Medical | Heart pulse | `HeartPulse` |
| Vacation | Sun | `Sun` |
| Sick leave | Stethoscope | `Stethoscope` |
| Warning / Conflict | Alert triangle | `AlertTriangle` |
| Theme toggle | Sun moon | `SunMoon` |
| Language | Languages | `Languages` |
| Expand sidebar | Panel left open | `PanelLeftOpen` |
| Collapse sidebar | Panel left close | `PanelLeftClose` |

### 5.2 Icon Sizing  

- **Navigation**: 20px (`size-5`)  
- **Buttons**: 18px (`size-[18px]`)  
- **Status badges**: 14px (`size-3.5`)  
- **Empty states**: 48px (`size-12`), muted color  
- **Feature hero icons**: 32px (`size-8`), in colored circle background  

---

## 6. Common Components  

### 6.1 Button  

Four variants, all with consistent height and padding.  

```
┌──────────────────────────────────────────────────────────────┐
│  Primary                              Accent                  │
│  ┌──────────────────┐                ┌──────────────────┐    │
│  │  Save Changes    │                │  + New Request   │    │
│  └──────────────────┘                └──────────────────┘    │
│  bg-primary-600 text-white           bg-accent-500 text-neutral-950
│  hover:bg-primary-700                hover:bg-accent-600     │
│                                                               │
│  Secondary                            Danger                  │
│  ┌──────────────────┐                ┌──────────────────┐    │
│  │  Cancel          │                │  Delete Request  │    │
│  └──────────────────┘                └──────────────────┘    │
│  bg-white border-neutral-300         bg-error-50 text-error-700
│  text-neutral-700                    border-error-200         │
│  hover:bg-neutral-50                 hover:bg-error-100      │
│                                                               │
│  Ghost                                Icon-only               │
│  ┌──────────────────┐                ┌──┐                    │
│  │  View Details    │                │⚙│                    │
│  └──────────────────┘                └──┘                    │
│  text-primary-600                     size-10 rounded-full    │
│  hover:bg-primary-50                  aria-label required     │
└──────────────────────────────────────────────────────────────┘
```

**Specs**:  
- Height: 40px (`h-10`) for standard, 36px (`h-9`) for compact (table rows), 48px (`h-12`) for hero CTAs.  
- Padding: `px-5` standard, `px-3` compact.  
- Border radius: `rounded-lg` (8px).  
- Font: `text-sm font-medium`.  
- Focus ring: `focus-visible:ring-2 focus-visible:ring-primary-400 focus-visible:ring-offset-2`.  
- Disabled: `opacity-50 cursor-not-allowed`.  
- Loading state: spinner icon replaces text, button width preserved via `min-width`.  

### 6.2 Status Badge  

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Approved │  │ Pending  │  │ Rejected │  │  Draft   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
  success       warning        error         neutral
```

**Specs**:  
- `inline-flex items-center gap-1.5 px-2.5 py-0.5 rounded-full text-xs font-medium`.  
- Optional dot indicator: 6px circle before text.  
- Map `LeaveRequest.status` → badge as per §2.2 Semantic Color Usage.  

### 6.3 Data Table  

```
┌─────────────────────────────────────────────────────────────┐
│  [Search input]              [Filters]  [Columns]  [Export] │
├─────────────────────────────────────────────────────────────┤
│  ☐ │ Employee      │ Type       │ Dates          │ Status  │
├────┼───────────────┼────────────┼────────────────┼─────────┤
│  ☐ │ John Smith    │ Vacation   │ Aug 1 – Aug 15  │ Approved│
│  ☐ │ María García  │ Sick Leave │ Jun 20 – Jun 25 │ Approved│
│  ☐ │ Ana López     │ Marriage   │ Sep 10 – Sep 25 │ Pending │
├────┴───────────────┴────────────┴────────────────┴─────────┤
│  Rows per page: 25 ▼     ← 1 2 3 ... 8 →     156 total     │
└─────────────────────────────────────────────────────────────┘
```

**Specs**:  
- Header row: `bg-neutral-100`, `text-sm font-semibold text-neutral-600 uppercase tracking-wider`.  
- Body rows: `border-b border-neutral-100`, `hover:bg-primary-50/50` transition.  
- Row height: 48px (`h-12`) for comfortable tap targets.  
- Checkbox column for bulk actions (Manager/HR).  
- Last column: actions (icon buttons).  
- Pagination: compact at bottom, `Rows per page` selector + page numbers + total count.  
- Empty state: centered illustration + "No results" message + clear filters CTA.  
- Loading state: skeleton rows (6 shimmering placeholders matching column widths).  

### 6.4 Card  

```
┌──────────────────────────────────┐
│  ┌──────────────────────────┐    │
│  │  Icon  │  Title          │    │
│  │        │  Subtitle       │    │
│  └──────────────────────────┘    │
│                                  │
│  Content area                    │
│  • Item one                     │
│  • Item two                     │
│                                  │
│  ┌──────────────────────────┐    │
│  │  Action link / button    │    │
│  └──────────────────────────┘    │
└──────────────────────────────────┘
```

**Specs**:  
- `bg-white rounded-xl border border-neutral-200 shadow-sm`.  
- Padding: `p-6`.  
- Hover (if interactive): `shadow-md border-primary-200` transition.  
- Accepts `class` prop for Tailwind overrides.  

### 6.5 Modal / Dialog  

**Specs**:  
- `<Teleport to="body">`.  
- Backdrop: `bg-neutral-950/50 backdrop-blur-sm`.  
- Panel: `bg-white rounded-2xl shadow-xl max-w-lg w-full mx-4`.  
- Header: title + close button, `border-b border-neutral-100 p-6`.  
- Body: `p-6`.  
- Footer: actions, `border-t border-neutral-100 p-4 flex justify-end gap-3`.  
- Focus trapped inside; Escape closes; click-outside closes (with confirmation if form is dirty).  
- Enter/leave transition: `transition opacity-0 scale-95` → `opacity-100 scale-100` (200ms ease-out).  

### 6.6 Toast Notification  

**Specs**:  
- Fixed bottom-right (`fixed bottom-6 right-6`).  
- Max 3 visible; stack with `gap-3`.  
- Variants: success (green), error (red), warning (amber), info (blue).  
- Auto-dismiss: 5s for success/info, 10s for warning, persistent for error (manual dismiss).  
- Manual dismiss: X button.  
- Enter animation: slide up + fade in.  
- Exit animation: slide right + fade out.  

### 6.7 Skeleton Loader  

```
┌──────────────────────────────────────┐
│  ┌──────────┐                        │
│  │ ████████ │  ┌──────────────────┐  │
│  │ ████████ │  │ ██████████████   │  │
│  │ ████████ │  └──────────────────┘  │
│  └──────────┘                        │
│  ┌──────────────────────────────┐    │
│  │ ██████████████████████████   │    │
│  │ ████████████████             │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

**Specs**:  
- `bg-neutral-200 animate-pulse rounded`.  
- Match the shape and size of the content they replace.  
- No text — pure shape placeholders.  

### 6.8 Empty State  

```
┌──────────────────────────────────────────┐
│                                          │
│              [Icon 48px]                 │
│                                          │
│         No leave requests yet            │
│    Plan your time off and submit your    │
│          first leave request.            │
│                                          │
│        ┌──────────────────┐             │
│        │  + New Request   │             │
│        └──────────────────┘             │
│                                          │
└──────────────────────────────────────────┘
```

**Specs**:  
- Centered in the content area, `py-16`.  
- Icon: 48px, `text-neutral-300`.  
- Title: `text-lg font-heading font-semibold text-neutral-600`.  
- Description: `text-sm text-neutral-400 max-w-md mx-auto`.  
- CTA button when applicable (e.g., "New Request").  

### 6.9 Error State  

Same layout as Empty State but with:  
- Icon: `AlertTriangle` in `text-error-400`.  
- Title: "Something went wrong".  
- Description: human-readable error or "We couldn't load this data."  
- CTA: "Try Again" button that re-fires the failed request.  

### 6.10 File Upload  

```
┌──────────────────────────────────────────┐
│                                          │
│           ┌────────────────┐            │
│           │  Upload icon   │            │
│           │  (CloudUpload) │            │
│           └────────────────┘            │
│                                          │
│     Drag & drop your file here or       │
│        ┌──────────────────┐            │
│        │  Browse files    │            │
│        └──────────────────┘            │
│                                          │
│   PDF, JPG, PNG, WebP up to 10 MB       │
│                                          │
│   ── Uploaded file ──────────────────   │
│   📄 marriage_cert.pdf   245 KB  [✕]    │
└──────────────────────────────────────────┘
```

**Specs**:  
- Drop zone: `border-2 border-dashed border-neutral-300 rounded-xl p-8 text-center`, hover: `border-primary-400 bg-primary-50/30`.  
- Accepts click or drag-and-drop.  
- After upload: shows file name, size, and remove button.  
- Error states: file too large (>10 MB), wrong type (not PDF/JPG/PNG/WebP).  

---

## 7. Application Shell  

### 7.1 Sidebar  

```
┌────────────────────┐
│  🏢  Company Name  │  ← Tenant name + logo placeholder
│                    │
│  ══════════════════│
│                    │
│  MAIN              │  ← Nav group label (text-xs uppercase tracking-wider)
│  ▣ Dashboard       │  ← Active: bg-primary-50 text-primary-700, left border primary-500
│  📄 My Leaves      │  ← Inactive: text-neutral-600 hover:bg-neutral-100
│  📅 Calendar       │
│  ⚖ Balances       │
│                    │
│  MANAGEMENT        │  ← Only visible if user has relevant roles
│  👥 Team           │    (Manager role)
│  ✅ Approvals      │    (Manager role) + badge count
│  🏥 HR             │    (HR roles) — expands to sub-items
│    ├ Employees     │
│    ├ Policies      │
│    ├ Calendars     │
│    ├ Reports       │
│    └ Documents     │
│  ⚙ Admin          │    (Sys Admin) — expands to sub-items
│    ├ Users         │
│    ├ Roles         │
│    └ Settings      │
│                    │
│  ══════════════════│
│                    │
│  👤 John Doe       │  ← User avatar + name + role badge
│     Employee       │
│                    │
│  ◀ Collapse        │  ← Collapse sidebar to icons-only
└────────────────────┘
```

**Specs**:  
- Width: `w-64` expanded, `w-16` collapsed. Transition: `transition-[width] duration-200 ease-out`.  
- Background: `bg-white border-r border-neutral-200`. Dark: `bg-neutral-900 border-neutral-800`.  
- Logo area: `h-16 px-4 flex items-center gap-3`, logo placeholder: 32px circle with primary bg + white initial.  
- Nav group label: `px-4 pt-6 pb-2 text-xs font-semibold text-neutral-400 uppercase tracking-wider`.  
- Nav item: `mx-2 px-3 py-2 rounded-lg text-sm font-medium flex items-center gap-3`.  
- Active nav item: `bg-primary-50 text-primary-700` with `border-l-2 border-primary-500`. Dark: `bg-primary-900/30 text-primary-300`.  
- Badge (approval count): `ml-auto bg-accent-500 text-neutral-950 text-xs font-bold rounded-full min-w-[20px] h-5 flex items-center justify-center px-1.5`.  
- Collapse button: at bottom, `border-t border-neutral-100 p-3`.  
- Collapsed state: icons only (20px), tooltip on hover showing full label. Nav groups hidden.  

### 7.2 Header  

```
┌──────────────────────────────────────────────────────────────┐
│  ☰ (mobile)   │  🔍 Search...        │  🌐 ES  │  🔔  │  👤 │
│               │                      │         │  3   │     │
└──────────────────────────────────────────────────────────────┘
```

**Specs**:  
- Height: `h-16`, `bg-white border-b border-neutral-200`, `sticky top-0 z-30`. Dark: `bg-neutral-900 border-neutral-800`.  
- Hamburger: visible only on mobile (`md:hidden`), toggles sidebar overlay.  
- Search: `max-w-md`, keyboard shortcut `⌘K` / `Ctrl+K`. Opens command palette with fuzzy search across employees, leave requests, policies.  
- Locale switcher: `ES` / `EN` toggle, flag icons.  
- Notifications: Bell icon + red badge (count). Click opens dropdown with recent notifications.  
- Profile: Avatar (initials fallback) + chevron. Click opens dropdown: profile, settings, logout.  

---

## 8. Screen Designs  

### 8.1 Login  

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│                                                              │
│                    ┌────────────────────┐                    │
│                    │    🏢  Logo         │                    │
│                    │  HR Leave Manager   │                    │
│                    └────────────────────┘                    │
│                                                              │
│              Welcome back                                    │
│        Sign in to manage your leave                          │
│                                                              │
│     ┌──────────────────────────────────────────┐            │
│     │  ┌──────────────────────────────────┐    │            │
│     │  │  G   Sign in with Google         │    │            │
│     │  └──────────────────────────────────┘    │            │
│     │                                          │            │
│     │  ┌──────────────────────────────────┐    │            │
│     │  │  Z   Sign in with Zoho           │    │            │
│     │  └──────────────────────────────────┘    │            │
│     │                                          │            │
│     │  ────────── or ──────────               │            │
│     │                                          │            │
│     │  ┌ Email ──────────────────────────┐    │            │
│     │  │ user@company.com                 │    │            │
│     │  └──────────────────────────────────┘    │            │
│     │                                          │            │
│     │  ┌ Password ───────────────────────┐    │            │
│     │  │ ••••••••••                  👁   │    │            │
│     │  └──────────────────────────────────┘    │            │
│     │                                          │            │
│     │  ┌──────────────────────────────────┐    │            │
│     │  │         Sign in                  │    │            │
│     │  └──────────────────────────────────┘    │            │
│     └──────────────────────────────────────────┘            │
│                                                              │
│         🇪🇸 Español  |  🇬🇧 English                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- SSO buttons: full-width, 48px tall, white bg with gray border, brand logo left-aligned.  
- Divider: `─ or ─` with `text-neutral-400 text-sm`.  
- Local login: appears only if local auth is enabled for the tenant.  
- Password field: show/hide toggle (eye icon).  
- Form validation with vee-validate + zod displayed inline.  
- "Sign in" button disabled until email + password are non-empty.  
- Error state: red banner above form ("Invalid credentials" / "Account inactive").  
- On SSO click: redirect to backend, backend handles OAuth flow, sets cookies, redirects to `/login/callback`, which calls `/auth/me` and redirects to `/dashboard`.  
- No tokens in URL.  

### 8.2 Employee Dashboard  

```
┌──────────────────────────────────────────────────────────────┐
│  Dashboard                                    June 23, 2026  │
│                                                              │
│  ┌─────────────────┐ ┌─────────────────┐ ┌────────────────┐ │
│  │ Vacation        │ │ Pending         │ │ Upcoming       │ │
│  │ Balance         │ │ Approvals       │ │ Leave          │ │
│  │                 │ │                 │ │                │ │
│  │   22 days       │ │      2          │ │  Aug 1–15     │ │
│  │   remaining     │ │  awaiting       │ │  Vacation      │ │
│  │                 │ │                 │ │                │ │
│  │  ████████░░ 73% │ │  → Review       │ │  39 days away  │ │
│  └─────────────────┘ └─────────────────┘ └────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────┐ ┌─────────────────┐ │
│  │  Team Calendar — June 2026         │ │ Quick Actions   │ │
│  │                                    │ │                 │ │
│  │  Mo Tu We Th Fr Sa Su             │ │ + New Request   │ │
│  │              1  2  3  4           │ │                 │ │
│  │   5  6  7  8  9 10 11            │ │ 📅 Team Calendar│ │
│  │  12 13 14 15 16 17 18            │ │ ⚖ My Balances  │ │
│  │  19 20 21 22 23 24 25            │ │ 📄 My Documents │ │
│  │  26 27 28 29 30                  │ │                 │ │
│  │                                    │ │                 │ │
│  │  ● Vacation  ● Sick  ◆ Holiday    │ │                 │ │
│  └────────────────────────────────────┘ └─────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │  Recent Leave Requests                                   ││
│  │                                                          ││
│  │  Vacation          Aug 1–15, 2026    Approved    →      ││
│  │  Medical appt.     Jul 5, 2026       Pending     →      ││
│  │  Sick Leave        Mar 10–14, 2026   Approved    →      ││
│  │                                                          ││
│  │                    View all requests →                   ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Balance card**: Progress bar showing used vs remaining. Color shifts: green (>50%), amber (20–50%), red (<20%).  
- **Pending approvals card**: Only shows if user is a manager/approver. Displays count + link to approval queue. Hidden if count is 0.  
- **Upcoming leave card**: Shows next approved leave. If none, shows "No upcoming leave" with CTA to request.  
- **Team calendar mini**: Small month grid. Colored dots for team absences. Legend at bottom. "View full calendar" link.  
- **Quick Actions**: 2-column grid of large icon-buttons. "New Request" uses accent color to draw attention.  
- **Recent requests**: Condensed list (max 5), each with type, dates, status badge, and `→` link.  
- All cards use four-state coverage: skeleton on load, empty messages when no data, error with retry.  

### 8.3 New Leave Request  

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to My Leaves                                         │
│                                                              │
│  New Leave Request                                           │
│                                                              │
│  Step 1 of 3: Select Leave Type                              │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ ☀        │ │ 🩺       │ │ 💜       │ │ 💚       │       │
│  │ Vacation │ │Sick Leave│ │ Birth &  │ │ Marriage │       │
│  │          │ │          │ │Childcare │ │  15 days │       │
│  │22 remain │ │          │ │          │ │          │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ ⏸        │ │ 👶       │ │ 🏠       │ │ 📋       │       │
│  │ Unpaid   │ │ Parental │ │Family Care│ │ More...  │       │
│  │ Leave    │ │  Leave   │ │          │ │          │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                              │
│  ──────────────────────────────────────────────────────      │
│                                                              │
│  Step 2 of 3: Choose Dates                                   │
│                                                              │
│  ┌ Start Date ────────┐  ┌ End Date ──────────┐            │
│  │ August 1, 2026  📅 │  │ August 15, 2026 📅 │            │
│  └────────────────────┘  └────────────────────┘            │
│                                                              │
│  ┌ Full day ────────────────────────────────────────┐       │
│  │  ● Full day    ○ Partial day (e.g., 2 hours)     │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  Duration: 15 calendar days / 11 working days                │
│  Balance after request: 11 days remaining                    │
│                                                              │
│  ⚠ 2 team members are off during this period:                │
│     • Jane Smith — Aug 1–7                                  │
│     • Carlos Ruiz — Aug 10–12                                │
│  You may still submit — your manager will review.            │
│                                                              │
│  ──────────────────────────────────────────────────────      │
│                                                              │
│  Step 3 of 3: Review & Submit                                │
│                                                              │
│  ┌ Notes (optional) ────────────────────────────────┐       │
│  │ Family trip to the coast — will be reachable     │       │
│  │ by phone if urgent.                              │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  ┌ Required Documents ──────────────────────────────┐       │
│  │ No documents required for this leave type.       │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  ┌──────────────┐  ┌──────────────────────────────┐         │
│  │ Save Draft   │  │       Submit Request         │         │
│  └──────────────┘  └──────────────────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Three-step flow** presented as a single scrollable page (not a wizard with Next buttons). Sections become visible as previous selections are made.  
- **Leave type cards**: Each shows icon, name, and relevant info (remaining balance / fixed days). Selected card has `ring-2 ring-primary-500 bg-primary-50`. Cards for leave types the employee is ineligible for are grayed out with tooltip explaining why.  
- **Date pickers**: Custom date range picker. Weekend days visually distinct. Public holidays marked with a flag icon. Team absences shown as colored dots on calendar days.  
- **Partial day toggle**: When enabled, shows start/end time selects.  
- **Conflict warning**: Amber banner with `AlertTriangle` icon. Lists conflicting team members. Does NOT block submission — just informs.  
- **Balance preview**: Live-updated as dates change. Shows `remaining - requested = after`. Goes red if balance goes negative.  
- **Document requirement**: If the leave type requires documents, shows upload zone (§6.10). Shows which document types are accepted (from policy).  
- **Save Draft**: saves in DRAFT status, returns to list.  
- **Submit**: calls backend, which resolves policy, runs DMN eligibility, creates approval steps, starts BPMN workflow. On success: toast + redirect to leave detail. On 409 conflict: inline error.  

### 8.4 Leave Detail  

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back    Leave Request #LV-2026-0042                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Vacation                             [ Approved ]    │    │
│  │                                                       │    │
│  │  August 1 – 15, 2026  (15 calendar / 11 working)     │    │
│  │                                                       │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │    │
│  │  │Submitted │→ │ Manager  │→ │HR Approved│           │    │
│  │  │Jun 23    │  │Approved  │  │Jun 24    │           │    │
│  │  │          │  │Jun 24    │  │          │           │    │
│  │  └──────────┘  └──────────┘  └──────────┘           │    │
│  │                                                       │    │
│  │  Manager comment: "Team coverage confirmed."          │    │
│  │  HR comment: "Approved per policy."                   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Details                                              │    │
│  │                                                       │    │
│  │  Policy applied:  Spain General — Vacation 2026      │    │
│  │  Balance impact:  30 → 19 days remaining             │    │
│  │  Documents:        None required                      │    │
│  │  Notes:            Family trip to the coast           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────┐                                        │
│  │  Cancel Request  │   ← Only if status = APPROVED          │
│  └──────────────────┘                                        │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Status badge**: prominent, at top of card.  
- **Approval timeline**: Horizontal stepper showing each step, approver name, decision date, and status. Completed steps in `success-500`. Current step in `primary-500` with pulse animation. Rejected step in `error-500`.  
- **Comments**: Each approval step's comment shown below the timeline.  
- **Policy info**: Shows which policy was applied and at which version.  
- **Cancel button**: Only visible if status is APPROVED and leave hasn't started yet. Opens confirmation dialog → creates cancellation workflow.  
- **Manager view extras**: "Team Calendar" link showing this leave in context.  
- **HR view extras**: "View audit trail" link.  

### 8.5 Team Calendar  

```
┌──────────────────────────────────────────────────────────────┐
│  Team Calendar                               June 2026  ◀ ▶ │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │  Mon    Tue    Wed    Thu    Fri    Sat    Sun           ││
│  │        1      2      3      4      5      6              ││
│  │  ██████       ██████████████████                         ││
│  │  Carlos V     Ana L. (Marriage)                          ││
│  │                                                          ││
│  │  7      8      9      10     11     12     13            ││
│  │                                                          ││
│  │  14     15     16     17     18     19     20            ││
│  │  ████████████████████████████████                        ││
│  │  María G. (Vacation)                                     ││
│  │                                                          ││
│  │  21     22     23     24     25     26     27            ││
│  │                 ░░░░░░                                   ││
│  │                 Holiday                                  ││
│  │                                                          ││
│  │  28     29     30                                        ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌ Legend ─────────────────────────────────────────┐        │
│  │ ████ Vacation   ████ Sick   ████ Paid Leave     │        │
│  │ ████ Parental   ░░░░ Holiday ░░░░ Bridge Day    │        │
│  │                                                  │        │
│  │  ☐ Show weekends   ☐ Show my leaves only        │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  ┌ Team Members ────────────────────────────────────┐        │
│  │  ☑ All  ☑ Carlos V.  ☑ Ana L.  ☑ María G.     │        │
│  │  ☑ John D. (me)  ☑ Pedro S.                     │        │
│  └──────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Month grid**: 7-column CSS Grid. Each day cell shows date + colored bars for leaves.  
- **Leave bars**: colored background with employee name + leave type. Hover shows tooltip with dates and status. Click navigates to leave detail (if user has permission).  
- **Public holidays**: distinct pattern (dotted/dashed) with holiday name.  
- **Today**: highlighted cell with `ring-2 ring-primary-400`.  
- **Navigation**: `◀ ▶` arrows for month. "Today" button to jump back.  
- **Team member filter**: Checkboxes to show/hide individuals. "All" toggle.  
- **Week view toggle** (mobile): Switches to a 7-day horizontal scroll for small screens.  

### 8.6 Approval Queue  

```
┌──────────────────────────────────────────────────────────────┐
│  Pending Approvals                          You have 3       │
│                                                              │
│  ┌ Tabs ───────────────────────────────────────────────────┐ │
│  │ [All (3)]  [Vacation (1)]  [Sick Leave (1)]  [...]     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🟡 Pending — Your action needed                     │    │
│  │                                                       │    │
│  │  Ana López                   Marriage Leave           │    │
│  │  Sales Department            Sep 10 – Sep 25, 2026    │    │
│  │                              15 days                  │    │
│  │                                                       │    │
│  │  ⚠ No team conflicts                                 │    │
│  │                                                       │    │
│  │  [View Calendar]  [Approve]  [Reject]                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🟡 Pending — Your action needed                     │    │
│  │                                                       │    │
│  │  Carlos Vega                 Vacation                 │    │
│  │  Engineering                 Jul 20 – Jul 30, 2026    │    │
│  │                              10 days                  │    │
│  │                                                       │    │
│  │  ⚠ 2 team members already off during this period     │    │
│  │     • John D. — Jul 20–25                            │    │
│  │     • Pedro S. — Jul 24–28                           │    │
│  │                                                       │    │
│  │  [View Calendar]  [Approve]  [Reject]                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🟢 All caught up — no more pending approvals        │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Tabs**: Filter by leave type. Count badges on each tab.  
- **Approval cards**: Each request as a card. Employee name (bold), department, leave type, dates, duration.  
- **Conflict indicator**: If team members overlap, show amber warning with names.  
- **Actions**: "View Calendar" opens team calendar scoped to the request dates. "Approve" button in primary green. "Reject" opens dialog with required comment.  
- **Bulk approve**: Checkbox on each card + "Approve selected (2)" button at top.  
- **Empty state**: Green checkmark + "All caught up — no more pending approvals".  
- **SLA indicator**: If a request has been pending >3 business days, show subtle clock icon + "3 days pending". After 5 days, show escalation warning.  

### 8.7 Employee Balances  

```
┌──────────────────────────────────────────────────────────────┐
│  My Leave Balances                           Year 2026  ▼   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ☀  Vacation                                        │    │
│  │  ████████████████░░░░░░░░  22 / 30 days remaining   │    │
│  │                                                       │    │
│  │  Entitlement:  30 days    Consumed: 5 days           │    │
│  │  Carried over:  3 days   Expires: Mar 31, 2026      │    │
│  │  (2 days from 2025)                                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  🩺  Sick Leave (Common illness)                     │    │
│  │  No fixed balance — as needed per medical certificate │    │
│  │                                                       │    │
│  │  Used this year: 5 days across 2 periods             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  💚  Paid Leave (Misc.)                              │    │
│  │                                                       │    │
│  │  Marriage:      15 days available / 0 used           │    │
│  │  Death relative:  3 days available / 0 used          │    │
│  │  Moving house:    1 day  available / 1 used          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ⏸  Unpaid Leave                                    │    │
│  │  No entitlement — subject to manager/HR approval     │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  👶  Parental Leave                                  │    │
│  │  8 weeks available / 0 used                          │    │
│  │  Per child under 8 years old                         │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Year selector**: Dropdown to view historical balances.  
- **Balance cards**: One per leave type with an entitlement.  
- **Progress bar**: Green (>50% remaining), amber (20–50%), red (<20%).  
- **Carry-over**: Shown as distinct segment on the progress bar (lighter shade). Expiry date prominent.  
- **Unlimited types**: For sick leave, parental leave — show "as needed" instead of progress bar, with usage counter.  
- **One-time types**: Marriage, death relative — show "X days available / Y used".  

### 8.8 HR — Employee List  

```
┌──────────────────────────────────────────────────────────────┐
│  Employees                                                   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 🔍 Search employees...     │ + Add Employee │ Export │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌ Filters ────────────────────────────────────────────┐    │
│  │ Department: [All ▾]  Status: [Active ▾]            │    │
│  │ Convenio: [All ▾]    Contract: [All ▾]             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ Name          │ Dept      │ Contract  │ Convenio  │ Act. ││
│  ├───────────────┼───────────┼───────────┼───────────┼──────┤│
│  │ John Smith    │ Sales     │ Full-time │ General   │  ··· ││
│  │ María García  │ Engineer  │ Full-time │ Metal     │  ··· ││
│  │ Ana López     │ Sales     │ Part-time │ Consulting│  ··· ││
│  │ Carlos Vega   │ Engineer  │ Shift     │ Metal     │  ··· ││
│  │ Pedro Sánchez │ HR        │ Full-time │ General   │  ··· ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  Rows per page: 25 ▼     ← 1 2 3 ... 12 →     287 total     │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Data table** (§6.3) with sortable columns.  
- **Search**: Debounced (300ms), searches across name, employee code, email.  
- **Filters**: Collapsible row of dropdowns. Applied filters show as removable chips.  
- **Row click**: Navigates to employee detail.  
- **Row actions** (··· menu): Edit, View Leaves, View Balances, Deactivate.  
- **Bulk actions** (via checkboxes): Assign convenio, Export selected.  
- **Export**: Downloads CSV of current filtered/sorted view.  

### 8.9 HR — Policy Editor  

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to Policies                                          │
│                                                              │
│  Edit Policy: Spain General — Marriage Leave 2026            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  General                                             │    │
│  │                                                       │    │
│  │  Name:    [Spain General — Marriage Leave 2026    ]  │    │
│  │  Type:    [Marriage Leave ▾]                         │    │
│  │  Convenio:[General (default) ▾]                      │    │
│  │  Effective:[2026-01-01]  to  [____] (indefinite)    │    │
│  │  Priority:[10]                                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Eligibility Rules                                    │    │
│  │                                                       │    │
│  │  ┌──────────────────────────────────────────────┐    │    │
│  │  │ Rule 1: EMPLOYMENT_STATUS                    │    │    │
│  │  │ Allowed: [FULL_TIME] [PART_TIME]             │    │    │
│  │  │                                     [Remove] │    │    │
│  │  └──────────────────────────────────────────────┘    │    │
│  │                                                       │    │
│  │  ┌──────────────────────────────────────────────┐    │    │
│  │  │ Rule 2: MIN_SENIORITY                        │    │    │
│  │  │ Minimum months: [12]                         │    │    │
│  │  │                                     [Remove] │    │    │
│  │  └──────────────────────────────────────────────┘    │    │
│  │                                                       │    │
│  │  + Add Eligibility Rule                               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Duration Rules                                       │    │
│  │  Rule: FIXED_DAYS  Days: [15]                  [Edit]│    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Documentation Rules                                  │    │
│  │  Rule: REQUIRES_DOCUMENT                            │    │
│  │  Accepted types: [MARRIAGE_CERTIFICATE]       [Edit]│    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Workflow                                             │    │
│  │  BPMN Process: [approval-manager-only ▾]             │    │
│  │  DMN Eligibility: [leave-eligibility ▾]              │    │
│  │  DMN Duration: [leave-duration ▾]                    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐       │
│  │ Save Draft   │  │ Preview  │  │ Publish Version 3 │       │
│  └──────────────┘  └──────────┘  └──────────────────┘       │
│                                                              │
│  ┌ Version History ────────────────────────────────────┐     │
│  │ v2 · Jan 2025 – Dec 2025 · 15 days · Published      │     │
│  │ v1 · Jan 2024 – Dec 2024 · 15 days · Superseded     │     │
│  └──────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Structured form** with collapsible sections.  
- **Rules**: Each rule is a card with type, config, and remove button. "Add Rule" dropdown with all rule types.  
- **Rule config**: Dynamic form based on rule type (MIN_SENIORITY shows number input, DOCUMENT_TYPE shows multi-select, etc.).  
- **Preview**: Opens a side panel or modal showing how a sample employee would be evaluated against this policy.  
- **Publish**: Creates new version, activates it (deactivates previous version if overlapping dates).  
- **Version history**: Timeline at bottom showing all versions with dates and status.  

### 8.10 HR — Reports Dashboard  

```
┌──────────────────────────────────────────────────────────────┐
│  Reports                                                     │
│                                                              │
│  ┌ Date Range ───────────────────────────────────────┐      │
│  │ From: [2026-01-01]  To: [2026-12-31]  [Apply]     │      │
│  └────────────────────────────────────────────────────┘      │
│                                                              │
│  ┌──────────────────────┐ ┌──────────────────────┐          │
│  │ ▴ Vacation Balances  │ │ 📊 Absence Rate      │          │
│  │                      │ │                      │          │
│  │  Avg remaining: 18d  │ │  Overall: 3.2%       │          │
│  │  Below 10d: 12 empl. │ │  Engineering: 2.1%   │          │
│  │                      │ │  Sales: 4.8% ⚠       │          │
│  │      [View →]        │ │      [View →]        │          │
│  └──────────────────────┘ └──────────────────────┘          │
│                                                              │
│  ┌──────────────────────┐ ┌──────────────────────┐          │
│  │ 🩺 Sick Leave Stats  │ │ 📋 Leave by Type     │          │
│  │                      │ │                      │          │
│  │  Total: 145 days     │ │  ████████ Vacation   │          │
│  │  Avg duration: 4.2d  │ │  ██ Sick Leave       │          │
│  │  Most common: Q1     │ │  █ Paid Leave        │          │
│  │                      │ │  ▌ Parental          │          │
│  │      [View →]        │ │      [View →]        │          │
│  └──────────────────────┘ └──────────────────────┘          │
│                                                              │
│  ┌──────────────────────┐ ┌──────────────────────┐          │
│  │ ⏸ Excedencia Track.  │ │ 📄 Pending Documents │          │
│  │                      │ │                      │          │
│  │  Active: 3           │ │  Missing docs: 7     │          │
│  │  Returning soon: 1   │ │  Overdue: 2 ⚠        │          │
│  │                      │ │                      │          │
│  │      [View →]        │ │      [View →]        │          │
│  └──────────────────────┘ └──────────────────────┘          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  💰 Payroll Impact                           [Export] │    │
│  │  Unpaid leave days: 34    Reduced hours: 5 employees │    │
│  │  Contract suspensions: 2                              │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Date range**: Defaults to current year.  
- **Report cards**: Each card is a mini-report summary. Click navigates to detailed report view.  
- **Data visualization**: Use CSS-only charts where possible (progress bars, simple bar charts via flexbox). Heavy charts (line charts, pie charts) dynamically import a charting library only when the detailed report view is opened.  
- **Alerts**: Values exceeding thresholds get warning styling (amber/red).  
- **Export**: Each detailed report supports CSV export.  

### 8.11 Admin — User Management  

```
┌──────────────────────────────────────────────────────────────┐
│  Users                                                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 🔍 Search users...              │ + Invite User      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ User           │ Email              │ Roles       │ Stat.││
│  ├────────────────┼────────────────────┼─────────────┼──────┤│
│  │ John Smith     │ john@company.com   │ Emp, Mgr    │  🟢  ││
│  │ María García   │ maria@company.com  │ Emp, HR Adm │  🟢  ││
│  │ Ana López      │ ana@company.com    │ Emp         │  🟢  ││
│  │ Pedro Sánchez  │ pedro@company.com  │ Emp, SysAdm │  🟡  ││
│  │ Carlos Vega    │ carlos@company.com │ Emp         │  🔴  ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  Rows per page: 25 ▼     ← 1 2 3 ... 5 →     124 total      │
└──────────────────────────────────────────────────────────────┘
```

**Key behaviors**:  
- **Status indicator**: 🟢 Active, 🟡 Inactive, 🔴 Terminated.  
- **Roles column**: Shows role badges (max 3, then "+2 more" chip with tooltip).  
- **Invite user**: Opens modal with email, first name, last name, and role assignment. Sends invitation email with SSO link.  
- **Row click**: Opens user detail side panel showing full profile, role assignment (checkboxes per role), and activity log.  
- **Role assignment**: Checkbox grid. Must always have at least 1 role. Warning if removing "System Administrator" from the last sys admin.  

---

## 9. Dark Mode  

### 9.1 Strategy  

Tailwind `class` strategy: `.dark` class on `<html>`. Toggle managed by `useTheme` composable. Preference persisted to `localStorage`. First visit respects `prefers-color-scheme`.  

### 9.2 Color Mapping  

| Element | Light | Dark |
|---------|-------|------|
| Page background | `bg-neutral-50` | `bg-neutral-950` |
| Sidebar background | `bg-white` | `bg-neutral-900` |
| Card background | `bg-white` | `bg-neutral-900` |
| Card border | `border-neutral-200` | `border-neutral-800` |
| Input background | `bg-white` | `bg-neutral-800` |
| Input border | `border-neutral-300` | `border-neutral-700` |
| Body text | `text-neutral-800` | `text-neutral-200` |
| Muted text | `text-neutral-500` | `text-neutral-400` |
| Table header | `bg-neutral-100` | `bg-neutral-800` |
| Table row hover | `bg-primary-50/50` | `bg-neutral-800/50` |
| Selected nav item | `bg-primary-50 text-primary-700` | `bg-primary-900/30 text-primary-300` |
| Skeleton loader | `bg-neutral-200` | `bg-neutral-800` |

### 9.3 Dark Mode Images & Charts  

- Charts: use CSS variable references so colors automatically invert.  
- Icons: Lucide icons inherit `currentColor` — no changes needed.  
- Logos: provide a white variant for dark mode (`<img src="logo.svg" class="dark:hidden">` + `<img src="logo-white.svg" class="hidden dark:block">`).  

---

## 10. Responsive Strategy  

### 10.1 Breakpoints (Tailwind defaults)  

| Breakpoint | Width | Behavior |
|-----------|-------|----------|
| Default (mobile) | < 640px | Single column, sidebar hidden (hamburger overlay), simplified tables |
| `sm` | ≥ 640px | Sidebar overlay, cards 1-col |
| `md` | ≥ 768px | Sidebar visible (collapsed), cards 2-col |
| `lg` | ≥ 1024px | Sidebar expanded, full layout |
| `xl` | ≥ 1280px | Content max-width 1280px centered |
| `2xl` | ≥ 1536px | Wider data tables, more calendar columns |

### 10.2 Mobile Adaptations  

| Component | Desktop | Mobile |
|-----------|---------|--------|
| **Sidebar** | Persistent (w-64) | Hidden; hamburger opens overlay |
| **Data Table** | Full columns | Card list (each row becomes a card) |
| **Calendar** | Month grid | Week scroll or agenda list |
| **Dashboard** | 3-column card grid | Single column stack |
| **New Request** | 3 sections visible | Accordion (one section at a time) |
| **Approval Queue** | Cards side by side | Full-width stacked cards |
| **Header** | Search visible | Search behind search icon tap |
| **Modals** | Centered panel | Full-screen bottom sheet |

### 10.3 Card-List Pattern (Mobile Data Table)  

```
┌──────────────────────────────┐
│  John Smith                  │
│  Sales Department            │
│  Vacation · Aug 1–15         │
│  [Approved]              [→] │
├──────────────────────────────┤
│  María García                │
│  Engineering                 │
│  Sick Leave · Jun 20–25      │
│  [Approved]              [→] │
└──────────────────────────────┘
```

Each table row becomes a card with key info displayed in a vertical stack. Action buttons become icons at the bottom-right.

---

## 11. Accessibility  

### 11.1 WCAG 2.2 AA Targets  

| Requirement | Implementation |
|-------------|---------------|
| **Contrast ratio ≥ 4.5:1** (text), ≥ 3:1 (large text) | OKLCH palette pre-verified. All text-on-background combos meet this. |
| **Keyboard navigation** | All interactive elements in tab order. Visible focus ring (`ring-2 ring-primary-400 ring-offset-2`). Modals trap focus. |
| **Screen readers** | Semantic HTML. `aria-label` on icon-only buttons. `aria-live="polite"` for dynamic content (toasts, status changes). `role` attributes on custom components. |
| **Form labels** | Every input has a visible `<label>` associated via `for`/`id`. Error messages linked via `aria-describedby`. |
| **Status messages** | Approval status changes, form submission results announced via `aria-live` regions. |
| **Skip link** | "Skip to main content" link as first focusable element. |
| **Color independence** | Status never conveyed by color alone. Status badges include text + icon. Calendar dots have tooltips. |
| **Motion** | `prefers-reduced-motion` respected throughout. |
| **Touch targets** | Minimum 44×44px for all interactive elements (buttons, nav items, table row actions). |

### 11.2 Focus Management  

- **Page navigation**: Focus moves to `<h1>` after route change (via router `afterEach` + `focus()`).  
- **Modal open**: Focus moves to first focusable element inside modal.  
- **Modal close**: Focus returns to the element that opened the modal.  
- **Toast**: Does not steal focus.  

---

## 12. Animation & Motion  

### 12.1 Principles  

- CSS-only. No JavaScript animation libraries.  
- Subtle and functional — never decorative.  
- All animations respect `prefers-reduced-motion`.  

### 12.2 Animation Catalog  

| Element | Animation | Duration | Easing |
|---------|-----------|----------|--------|
| Sidebar expand/collapse | `transition-[width]` | 200ms | `ease-out` |
| Page enter (route change) | Fade in + slide up 8px | 200ms | `ease-out` |
| Modal open | Opacity 0→1 + scale 0.95→1 | 200ms | `ease-out` |
| Modal close | Opacity 1→0 + scale 1→0.95 | 150ms | `ease-in` |
| Toast enter | Slide up + fade in | 300ms | `ease-out` |
| Toast exit | Slide right + fade out | 200ms | `ease-in` |
| Card hover | Shadow increase + slight lift (`-translate-y-0.5`) | 150ms | `ease-out` |
| Button hover | Background color transition | 150ms | `ease-out` |
| Skeleton loader | Pulse opacity 0.5↔1 | 1.5s | `ease-in-out` (infinite) |
| Dropdown open | Opacity + scaleY 0→1 | 150ms | `ease-out` |
| Approval step current | Subtle pulse ring | 2s | `ease-in-out` (infinite) |

### 12.3 Reduced Motion  

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

Or per-element via Tailwind: `class="animate-fade-in motion-reduce:animate-none"`

---

> **End of UX/UI Design Specification**  
> All screens, components, and behaviors described here are implementable with Vue 3 + Tailwind CSS v4 + Lucide Vue, following the patterns and constraints defined in [SPEC.md](./SPEC.md).
