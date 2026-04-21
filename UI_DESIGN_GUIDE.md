# Invenzo ITAM — UI Design Guide

**Purpose**: A single source of truth for page layouts, headers, tables, buttons, and controls. Every new page and every refactor of an existing page MUST follow these patterns exactly. Do not invent new styles.

---

## 1. Page Layout — Three Patterns Only

There are **three** approved page layout patterns. Pick one based on the page type.

### Pattern A — **Full-Width Table Page** (most common)

For any page that is dominated by a table/list (Assets, Software, Agents, Tasks, Changes, Compliance, Relationships, Software, Patch Compliance, Users, Webhooks, etc.).

```tsx
export default function MyPage() {
  return (
    <div className="-m-3 md:-m-4 lg:-m-6 flex flex-col flex-1 min-h-0 overflow-hidden">
      {/* Sticky header bar */}
      <div className="shrink-0 bg-white">
        <div className="px-4 md:px-6 pt-5 md:pt-6 pb-4 md:pb-5">
          <PageHeader
            title="My Page"
            description="Short one-liner description"
            icon={<Icon className="h-5 w-5" />}
            actions={<Button size="sm">Primary Action</Button>}
          />
        </div>

        {/* Optional: Tabs or stat cards strip go here inside the sticky area */}
        <Tabs tabs={TABS} active={tab} onChange={setTab} className="px-4 md:px-6" />
      </div>

      {/* Optional: Controls strip (filters, search, per-page) */}
      <div className="shrink-0 flex items-center justify-between gap-2 px-4 md:px-6 py-2.5 border-b border-slate-100">
        <div>{/* filters */}</div>
        <div>{/* search + view toggles */}</div>
      </div>

      {/* Full-bleed table area — edge to edge */}
      <div className="flex-1 min-h-0 overflow-hidden">
        <DataTable
          data={data}
          columns={columns}
          scrollable
          getRowId={(r) => r.id}
        />
      </div>
    </div>
  );
}
```

**Key rules:**
- Root: `-m-3 md:-m-4 lg:-m-6 flex flex-col flex-1 min-h-0 overflow-hidden`
- Header block: `shrink-0 bg-white` — header padding is `px-4 md:px-6 pt-5 md:pt-6 pb-4 md:pb-5`
- Tabs, if present, go **inside** the sticky header block with `px-4 md:px-6`
- Controls strip: `px-4 md:px-6 py-2.5 border-b border-slate-100`
- Table gets `scrollable` prop (handles sticky header + pagination footer)
- **NO rounded card wrapper around the table.** The table is edge-to-edge.

### Pattern B — **Dashboard / Multi-Card Page**

For pages with multiple cards/charts (Dashboard, Patch Compliance overview, Finance overview).

```tsx
export default function MyDashboard() {
  return (
    <div className="-m-3 md:-m-4 lg:-m-6 flex flex-col flex-1 min-h-0 overflow-hidden">
      {/* Sticky header */}
      <div className="shrink-0 bg-white border-b border-slate-200/60">
        <div className="px-4 md:px-6 pt-5 md:pt-6 pb-4 md:pb-5">
          <PageHeader title="..." description="..." icon={<Icon />} actions={...} />
        </div>
      </div>

      {/* Scrollable content area */}
      <div className="flex-1 overflow-auto px-4 md:px-6 py-5 space-y-5">
        {/* KPI strip */}
        <div className="grid grid-cols-2 md:grid-cols-4 gap-3">
          <Stat ... />
          <Stat ... />
        </div>

        {/* Content cards */}
        <div className="bg-white rounded-xl border border-border">
          <div className="px-4 py-3 border-b border-border">
            <h3 className="text-sm font-semibold text-heading">Section Title</h3>
            <p className="text-[11px] text-slate-400 mt-0.5">Description</p>
          </div>
          <div className="p-4">...</div>
        </div>
      </div>
    </div>
  );
}
```

**Key rules:**
- Same root pattern as Pattern A
- Header is sticky white bar
- Scrollable content uses `px-4 md:px-6 py-5` with `space-y-5` between sections
- Cards inside content area: `bg-white rounded-xl border border-border`

### Pattern C — **Detail / Form Page**

For single-record detail views (AssetDetail, TaskDetail) or wizard-like pages.

- Uses own PageHeader at top
- Content uses card grids below
- No sticky header requirements beyond the page
- Padding: `p-3 md:p-4 lg:p-6`

---

## 2. PageHeader — Standard Title Bar

**Always use the shared `<PageHeader>` component**, never a raw `<h1>`.

```tsx
<PageHeader
  title="All Assets"                          // 20px/semibold
  description="2 assets · Hardware..."        // 12px/slate-500
  icon={<Monitor className="h-5 w-5" />}      // optional, rendered in accent color
  helpId="assets"                              // optional help icon
  actions={                                    // optional right-side actions
    <div className="flex items-center gap-2">
      <Button size="sm" variant="ghost">CSV</Button>
      <Button size="sm">+ Add Asset</Button>
    </div>
  }
/>
```

**Do NOT:**
- Write custom `<h1>` + `<p>` markup for page titles
- Use `text-lg` or `text-2xl` — `PageHeader` already uses `text-xl`
- Wrap the header in a card or border — it's rendered on the white sticky bar

---

## 3. Tabs — Two Styles

### Main tabs (`<Tabs>` component) — underline style

Used at the top of multi-tab pages (Software's 8 tabs, Compliance's 4 tabs, Settings' 9 tabs, Relationships' 3 tabs).

```tsx
<Tabs
  tabs={[
    { id: 'overview', label: 'Overview', icon: <Icon className="h-3.5 w-3.5" />, count: 5 },
    { id: 'history',  label: 'History' },
  ]}
  active={tab}
  onChange={setTab}
  className="px-4 md:px-6"   /* match parent horizontal padding */
/>
```

Renders as: `text-[12.5px] font-medium` tabs with a `h-[2px] bg-accent` underline under the active one.

### Sub-tabs — pill/segment style

Used inside embedded sections (Settings > Notifications sub-tabs, severity filter pills).

```tsx
<div className="inline-flex items-center gap-1 p-1 rounded-lg bg-slate-100/80">
  {FILTERS.map((f) => (
    <button
      key={f.key}
      onClick={() => setFilter(f.key)}
      className={cn(
        'px-3 py-1 text-xs font-medium rounded-md transition',
        filter === f.key
          ? 'bg-white text-slate-800 shadow-sm'
          : 'text-slate-500 hover:text-slate-700',
      )}
    >
      {f.label}
    </button>
  ))}
</div>
```

**Rule**: Main page tabs MUST use `<Tabs>` (underline). Sub-tabs INSIDE a tab panel MUST use pill/segment style. Never mix — this creates visual confusion.

---

## 4. Tables — Always Full-Width, Never Carded

### Correct: full-bleed DataTable

```tsx
<DataTable
  data={data}
  columns={columns}
  scrollable                   /* sticky header + pagination footer */
  getRowId={(r) => r.id}
  highlightId={highlightId}    /* optional, for deep-linking */
  rowClassName={(row) => ...}  /* optional, conditional row styling */
  onRowClick={(row) => ...}    /* optional */
/>
```

The DataTable component already handles:
- Sticky column header
- Full horizontal scroll when content overflows
- Pagination footer ("Showing 1-25 of 100")
- Per-column sort indicators
- Empty state (with `<EmptyState>` fallback)

### Incorrect patterns to remove

```tsx
{/* ❌ DO NOT wrap the table in a rounded card */}
<div className="bg-white rounded-xl border border-border overflow-hidden">
  <DataTable data={data} columns={columns} />
</div>

{/* ❌ DO NOT use pageSize prop without scrollable */}
<DataTable data={data} columns={columns} pageSize={25} />

{/* ❌ DO NOT create custom table markup */}
<table className="min-w-full">...</table>
```

### Row styling rules

- Row height is handled by DataTable — do not override
- Row hover: automatic via DataTable
- Conditional row backgrounds (e.g., unprotected assets highlighted red): use `rowClassName` prop
- Selected/highlighted rows: use `highlightId` prop

---

## 5. Buttons — Always Use `<Button>` Component

```tsx
import { Button } from '../components/ui';

<Button size="sm">Default primary — accent plum</Button>
<Button size="sm" variant="secondary">Secondary — bordered</Button>
<Button size="sm" variant="ghost">Ghost — minimal, no border</Button>
<Button size="sm" variant="danger">Danger — red for destructive</Button>

<Button size="sm" icon={<Icon className="h-3.5 w-3.5" />}>With icon</Button>
<Button size="sm" disabled>Disabled</Button>
```

**Rules:**
- **Always** `size="sm"` for in-page buttons (slim/compact look)
- **Never** use raw `btn-primary`/`btn-secondary` CSS classes
- **Never** use raw `<button>` tags for action buttons — they miss focus/disabled styling
- Icons: always `h-3.5 w-3.5` inside buttons, with `mr-1` or `mr-1.5` gap before label
- Group multiple buttons with `flex items-center gap-2`

---

## 6. Stat Cards — KPI Strip

For dashboards and summary rows, use the shared `<Stat>` component.

```tsx
<div className="grid grid-cols-2 md:grid-cols-4 gap-3">
  <Stat
    label="Critical Missing"
    value={kpi.critical_missing}
    icon={<ShieldAlert className="h-5 w-5" />}
    iconBg="bg-red-50 text-red-600"
  />
  <Stat label="Compliance" value={`${pct}%`} icon={<ShieldCheck className="h-5 w-5" />} iconBg="bg-emerald-50 text-emerald-600" />
</div>
```

**Rules:**
- Always 4 columns on desktop (`md:grid-cols-4`), 2 on mobile (`grid-cols-2`)
- Gap: `gap-3`
- Place in scrollable content area (Pattern B) OR in the sticky header area (Pattern A with summary strip)

---

## 7. Form Controls

### Text inputs

```tsx
<input
  type="text"
  className="input-base"           /* shared class with consistent border/focus */
  placeholder="Search..."
/>
```

Never use `w-full` without wrapping container — let parent flex decide width.

### Selects

```tsx
<select className="input-base text-xs px-2 py-1">
  <option>Option 1</option>
</select>
```

### SlideOver for create/edit forms

For **all** create/edit flows, use `<SlideOver>`, **not** `<Modal>`:

```tsx
<SlideOver
  open={open}
  onClose={() => setOpen(false)}
  title="Create Asset"
  width="w-[520px]"                /* always 520px unless complex form */
>
  <div className="p-5 space-y-4">
    <div>
      <label className="block text-xs font-medium text-slate-600 mb-1">Name</label>
      <input className="input-base w-full" />
    </div>
    {/* ... more fields ... */}
  </div>
  <div className="border-t border-border px-5 py-3 flex justify-end gap-2 bg-slate-50">
    <Button size="sm" variant="ghost" onClick={onClose}>Cancel</Button>
    <Button size="sm" onClick={onSave}>Save</Button>
  </div>
</SlideOver>
```

### Modal for confirmations only

`<Modal>` is for destructive confirmations (delete, reject, etc.), **never** for edit forms:

```tsx
<Modal open={open} onClose={close} title="Delete asset?">
  <div className="space-y-3 p-1">
    <p className="text-sm text-slate-600">This action cannot be undone.</p>
    <div className="flex justify-end gap-2 pt-2">
      <Button size="sm" variant="ghost" onClick={close}>Cancel</Button>
      <Button size="sm" variant="danger" onClick={confirm}>Delete</Button>
    </div>
  </div>
</Modal>
```

### Toggles

**Always** use `<MiniToggle>` (filled track + white knob, `h-4 w-7`) — **never** use checkboxes or Eye icons for on/off state.

---

## 8. Badges

For status/severity/type labels, use `<Badge>`:

```tsx
<Badge variant="success">Active</Badge>
<Badge variant="warning">Pending</Badge>
<Badge variant="danger">Critical</Badge>
<Badge variant="info">In Progress</Badge>
<Badge variant="default">Unknown</Badge>

{/* With dot indicator */}
<Badge variant="success" dot>Healthy</Badge>
```

**Rule**: Never hand-roll `<span className="bg-green-100 text-green-700 px-2 py-0.5 rounded">`. Always use `<Badge>`.

---

## 9. Empty States

Use `<EmptyState>` for zero-data states:

```tsx
<EmptyState
  icon={<Inbox className="h-8 w-8" />}
  title="No Requests Yet"
  description="No pending requests — submit a new request to get started."
  action={
    <Button size="sm" onClick={onNew}>
      <Plus className="h-4 w-4 mr-1" />
      New Request
    </Button>
  }
/>
```

**Rule**: Never write "No data" text directly. Always wrap in `<EmptyState>` with an icon + title + description + optional action.

---

## 10. Colors & Spacing Reference

### Colors (Tailwind)
- **Accent (primary)**: `text-accent` (plum `#6B21A8`) — buttons, active tabs, highlights. Full scale: `accent.DEFAULT` = plum-700, `accent-hover` = plum-800, `accent-dark` = plum-900, `accent-light` = plum-500, `accent-muted` = plum-100. Semantic greens (`emerald-*` classes, success states, online indicators, compliance pass) stay green — they are NOT accent uses.
- **Headings**: `text-heading` (slate-900)
- **Body**: `text-slate-700`
- **Muted**: `text-slate-500` (descriptions)
- **Very muted**: `text-slate-400` (placeholder, disabled)
- **Border**: `border-border` (slate-200)
- **Subtle border**: `border-slate-200/60`
- **Background**: `bg-white` (sections), `bg-surface` (page), `bg-slate-50` (subtle fills)

### Standard Spacing
- Page horizontal padding: `px-4 md:px-6`
- Section vertical padding: `pt-5 md:pt-6 pb-4 md:pb-5`
- Inner content padding: `p-4` or `p-5`
- Gap between actions in a row: `gap-2` (tight) or `gap-3` (normal)
- Gap between cards in a grid: `gap-3` (tight) or `gap-4` (spacious)
- Vertical stack spacing: `space-y-4` or `space-y-5`

### Icons
- lucide-react **only**
- Size inside buttons: `h-3.5 w-3.5`
- Size in card titles / page headers: `h-5 w-5`
- Size in inline badges / row actions: `h-3 w-3` or `h-4 w-4`

---

## 11. Icons & lucide-react

- Always import from `lucide-react`
- Never use emoji or other icon libraries (FontAwesome, Heroicons, etc.)
- Icons in section titles: pair with accent background: `<span className="text-accent">{icon}</span>`

---

## 12. Responsive / Mobile

- Sidebar auto-collapses below 1024px, hides (hamburger) below 768px
- Use responsive breakpoints: `md:` (768px+), `lg:` (1024px+), `xl:` (1280px+)
- Tables should be `scrollable` — DataTable handles horizontal overflow on mobile
- Cards: `grid-cols-1 md:grid-cols-2 xl:grid-cols-3` for 3-up on desktop, 1-up on mobile

---

## 13. Checklist — Before Committing

When adding or refactoring any page:

- [ ] Uses one of the 3 approved layout patterns (A/B/C)
- [ ] Root container: `-m-3 md:-m-4 lg:-m-6 flex flex-col flex-1 min-h-0 overflow-hidden`
- [ ] Uses shared `<PageHeader>` component (no raw `<h1>`)
- [ ] Tables use `<DataTable scrollable>` (no rounded card wrapper)
- [ ] Multi-tab page uses shared `<Tabs>` component (no custom tabs)
- [ ] All buttons use `<Button size="sm">` (no raw `<button>` for actions)
- [ ] Create/edit forms use `<SlideOver>` (not `<Modal>`)
- [ ] Confirmations use `<Modal>` (not alert/confirm)
- [ ] Badges use `<Badge>` (no hand-rolled span/color)
- [ ] Toggles use `<MiniToggle>` (no raw checkboxes)
- [ ] Empty states use `<EmptyState>` (no "No data" text)
- [ ] Icons from lucide-react only, sized per above rules
- [ ] Horizontal padding: `px-4 md:px-6` (consistent across all strips)
- [ ] `cd ui && npx tsc --noEmit` passes

---

## 14. Shared Components Inventory

Located in `ui/src/components/ui/`:

| Component | Purpose |
|---|---|
| `PageHeader` | Title + description + icon + actions |
| `Tabs` | Main page tabs (underline style) |
| `DataTable` | TanStack Table wrapper with all table features |
| `Button` | All action buttons (primary / secondary / ghost / danger) |
| `SlideOver` | Right-side drawer for create/edit forms |
| `Modal` | Center-screen dialog for confirmations |
| `Badge` | Status / severity / type labels |
| `MiniToggle` | Boolean on/off switch |
| `EmptyState` | Zero-data placeholder |
| `Stat` | KPI card for dashboards |
| `Card` / `CardBody` / `CardHeader` | Content card wrapper |
| `Toast` | Transient success/error messages |
| `Combobox` | Searchable dropdown |
| `AssetTypeIcon` | Icon for asset types (computer, printer, etc.) |
| `TableSkeleton` | Loading placeholder for tables |

---

## 15. What NOT to Do

- ❌ Write custom CSS for page layouts — use Tailwind utility classes
- ❌ Wrap tables in rounded cards — they must be edge-to-edge
- ❌ Use `Modal` for long create/edit forms — use `SlideOver`
- ❌ Create new tab styles — use `<Tabs>` for main, pill/segment for sub
- ❌ Hand-roll status colors — use `<Badge>`
- ❌ Use emojis for icons — use lucide-react
- ❌ Use `size="md"` or larger on buttons — always `size="sm"`
- ❌ Use `h-screen` — breaks scroll inside AppLayout. Use flex column with `flex-1 min-h-0`
- ❌ Skip the sticky header — every page must have `shrink-0 bg-white` header bar
- ❌ Invent new spacing — stick to the `px-4 md:px-6` + `pt-5 md:pt-6 pb-4 md:pb-5` standard

---

*If you find yourself about to write something that's not covered here, stop and ask. Consistency is the whole point.*
