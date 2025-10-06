# UI Mockups: Column Resize Feature

This document provides visual mockups and descriptions of the UI changes for the column resize feature.

## Current State vs. Proposed State

### Current State (Issue)
```
Problem: Columns are equally sized, wasting space

┌─────────────────────────────────────┬─────────────────────────────────────┬─────────────────────────────────────┐
│ Timestamp                           │ Level                               │ Message                             │
│ (too wide for content)              │ (too wide for content)              │ (too narrow, gets truncated)        │
├─────────────────────────────────────┼─────────────────────────────────────┼─────────────────────────────────────┤
│ 2024-01-15T10:30:45.123Z           │ INFO                                │ User logged in successfully from... │
│ 2024-01-15T10:31:12.456Z           │ ERROR                               │ Database connection failed: timeo...│
│ 2024-01-15T10:31:45.789Z           │ WARN                                │ High memory usage detected: 85% o...│
└─────────────────────────────────────┴─────────────────────────────────────┴─────────────────────────────────────┘
```

### Proposed State (Smart Defaults)
```
Solution: Timestamp/Level auto-fit, Message expands

┌──────────────────────────┬──────────┬─────────────────────────────────────────────────────────────────────┐
│ Timestamp                │ Level    │ Message                                                             │
│ (fits content)           │ (fits)   │ (expands to use available space)                                    │
├──────────────────────────┼──────────┼─────────────────────────────────────────────────────────────────────┤
│ 2024-01-15T10:30:45.123Z │ INFO     │ User logged in successfully from IP 192.168.1.100                   │
│ 2024-01-15T10:31:12.456Z │ ERROR    │ Database connection failed: timeout after 30s                       │
│ 2024-01-15T10:31:45.789Z │ WARN     │ High memory usage detected: 85% of available RAM in use             │
└──────────────────────────┴──────────┴─────────────────────────────────────────────────────────────────────┘
```

## Interactive Elements

### Resize Handles

```
Column header with resize handle on hover:

┌──────────────────────────┐│┌──────────┐
│ Timestamp                ││││ Level    │
│                          ││││          │
└──────────────────────────┘│└──────────┘
                            ↑
                  Resize handle (visible on hover)
                  Cursor changes to ↔ (col-resize)
```

Visual states:
1. **Normal**: Resize handle invisible
2. **Hover**: Resize handle shows with highlight color
3. **Dragging**: Handle and column header show active state
4. **Double-click**: Triggers auto-fit animation

### Resize Handle States

```css
/* Normal - invisible */
.resize-handle {
  opacity: 0;
}

/* Hover - semi-transparent blue */
.resize-handle:hover {
  opacity: 0.5;
  background: var(--vscode-focusBorder);
}

/* Dragging - fully visible blue */
.resize-handle.resizing {
  opacity: 1;
  background: var(--vscode-focusBorder);
}
```

## Column Settings Panel

### Button Location
```
Toolbar at top of table view:

┌─────────────────────────────────────────────────────────────────────┐
│ [⚙ Column Settings]  [↻ Refresh]  [⬇ Export]                       │
└─────────────────────────────────────────────────────────────────────┘
┌──────────────────────────┬──────────┬────────────────────────────────┐
│ Timestamp                │ Level    │ Message                        │
├──────────────────────────┼──────────┼────────────────────────────────┤
```

### Settings Panel (Modal)
```
┌──────────────────────────────────────────────────────┐
│  Column Settings                                  [×] │
├──────────────────────────────────────────────────────┤
│                                                       │
│  ☑ Timestamp        [Auto-fit ▾]    [↑] [↓]         │
│  ☑ Level            [Auto-fit ▾]    [↑] [↓]         │
│  ☑ Message          [Expand   ▾]    [↑] [↓]         │
│  ☐ Source           [Auto-fit ▾]    [↑] [↓]         │
│  ☐ Thread ID        [Fixed    ▾]    [↑] [↓]         │
│                                                       │
├──────────────────────────────────────────────────────┤
│  [Reset to Defaults]                      [Apply]    │
└──────────────────────────────────────────────────────┘

Legend:
☑ = Visible column
☐ = Hidden column
[Auto-fit ▾] = Dropdown for size mode
[↑] [↓] = Reorder buttons
```

### Size Mode Dropdown Options
```
When clicking the dropdown:

[Auto-fit ▾]  →  ┌─────────────┐
                 │ Auto-fit    │ ← Fit to content width
                 │ Expand      │ ← Grow to fill space
                 │ Fixed       │ ← Manual width (current)
                 └─────────────┘
```

## Responsive Behavior

### Narrow Viewport
```
When window is narrow, Message column shrinks but maintains minimum width:

┌─────────────────────────┬────────┬──────────────────────────┐
│ Timestamp               │ Level  │ Message                  │
├─────────────────────────┼────────┼──────────────────────────┤
│ 2024-01-15T10:30:45.123Z│ INFO   │ User logged in success...│
│ 2024-01-15T10:31:12.456Z│ ERROR  │ Database connection fa...│
└─────────────────────────┴────────┴──────────────────────────┘
                                      ↑
                                Text truncates with ellipsis
                                (tooltip shows full text)
```

### Wide Viewport
```
When window is wide, Message column expands to use space:

┌─────────────────────────┬────────┬─────────────────────────────────────────────────────────────────────────────────┐
│ Timestamp               │ Level  │ Message                                                                         │
├─────────────────────────┼────────┼─────────────────────────────────────────────────────────────────────────────────┤
│ 2024-01-15T10:30:45.123Z│ INFO   │ User logged in successfully from IP 192.168.1.100 using Chrome 120.0           │
│ 2024-01-15T10:31:12.456Z│ ERROR  │ Database connection failed: timeout after 30s while attempting to connect to... │
└─────────────────────────┴────────┴─────────────────────────────────────────────────────────────────────────────────┘
```

## Visual Feedback During Resize

### Step 1: Hover over border
```
Cursor changes, resize handle appears:

┌──────────────────────────┐│
│ Timestamp                ││  ← Highlighted resize handle
└──────────────────────────┘│
```

### Step 2: Click and drag
```
Column width changes in real-time:

Dragging right (wider):
┌────────────────────────────────┐│
│ Timestamp                      ││
└────────────────────────────────┘│

Dragging left (narrower):
┌──────────────────┐│
│ Timestamp        ││
└──────────────────┘│
```

### Step 3: Release
```
Column maintains new width, setting is saved:

┌────────────────────────────────┬────────┬──────────────────────────┐
│ Timestamp (resized)            │ Level  │ Message                  │
└────────────────────────────────┴────────┴──────────────────────────┘
```

## Animation and Transitions

### Smooth resize (CSS)
```css
.column-header {
  transition: width 0.1s ease-out;
}

/* Disable transition during drag for better performance */
.column-header.resizing {
  transition: none;
}
```

### Auto-fit animation (Double-click)
```css
.column-header.auto-fitting {
  transition: width 0.2s ease-in-out;
}
```

Visual effect:
```
Before double-click:          After double-click:
┌──────────────────┐          ┌────────────────────────┐
│ Timestamp        │    →     │ Timestamp              │
└──────────────────┘          └────────────────────────┘
    (200px)                        (180px - fits content)
```

## Accessibility Features

### Visual Indicators
1. **Focus ring** on resize handle when tabbed to
2. **High contrast mode** support for resize handle
3. **Cursor change** to col-resize on hover
4. **Visual feedback** during resize (highlighted border)

### Keyboard Support (Future Enhancement)
```
When column header is focused:
- Left Arrow: Decrease width by 10px
- Right Arrow: Increase width by 10px
- Shift + Left/Right: Decrease/increase by 50px
- Enter: Open column settings
- Space: Toggle auto-fit
```

### Screen Reader Announcements
```
"Column Timestamp, resizable. Current width 200 pixels."
"Resizing Timestamp column to 250 pixels."
"Timestamp column auto-fitted to 180 pixels."
```

## Color Scheme Support

### Light Theme
```
Background: #ffffff
Border: #e0e0e0
Resize Handle (hover): #007acc (blue)
Resize Handle (active): #005a9e (darker blue)
```

### Dark Theme
```
Background: #1e1e1e
Border: #3e3e3e
Resize Handle (hover): #0e639c (blue)
Resize Handle (active): #1177bb (lighter blue)
```

### High Contrast
```
Background: #000000
Border: #ffffff
Resize Handle: #ffff00 (yellow)
Focus indicator: #00ff00 (green)
```

## Edge Cases

### Very Narrow Column
```
Minimum width enforced (50px):

┌─────┐┌─────┬──────────────────────────────────────────┐
│Time ││Level│ Message                                  │
└─────┘└─────┴──────────────────────────────────────────┘
  ↑
Prevents column from becoming unusable
```

### Very Wide Column
```
Maximum width enforced (if configured):

┌──────────────────────────────────────────────────┐
│ Timestamp (capped at max-width)                  │
└──────────────────────────────────────────────────┘
```

### Long Content Overflow
```
Content longer than column shows ellipsis:

┌──────────────────────────┐
│ Message                  │
├──────────────────────────┤
│ This is a very long me...│ ← Tooltip shows full text on hover
└──────────────────────────┘
```

## Mobile/Touch Support (Future)

While VSCode is primarily desktop, touch support could include:
- **Touch and drag** on resize handles
- **Double-tap** for auto-fit
- **Long press** to open column settings

```
Touch target (minimum 44x44px):

┌─────┐┌──────────┐
│     ││          │
│  ◄► ││  Level   │ ← Enlarged touch area
│     ││          │
└─────┘└──────────┘
```

## Performance Considerations

### Virtual Scrolling
```
Only render visible rows to maintain performance:

┌──────────────────────────┬────────┬──────────────────────────┐
│ Timestamp                │ Level  │ Message                  │
├──────────────────────────┼────────┼──────────────────────────┤
│ [Visible Row 1]                                              │
│ [Visible Row 2]                                              │
│ [Visible Row 3]                                              │
└──────────────────────────────────────────────────────────────┘
  ↓ [1000 more rows not rendered]
```

### Resize Performance
- Throttle resize events to max 60fps
- Use CSS transforms where possible
- Debounce save-to-storage to after resize completes

## Implementation Priority

### Phase 1 (MVP) - Core Functionality
✓ Smart default widths (auto-fit vs. expand)
✓ Drag-to-resize with mouse
✓ Persist widths in workspace state
✓ Visual feedback (cursor, highlight)

### Phase 2 - Enhanced UX
□ Double-click auto-fit
□ Column settings panel
□ Column visibility toggle
□ Reset to defaults button

### Phase 3 - Advanced Features
□ Keyboard resize support
□ Column reordering
□ Column presets/profiles
□ Touch/mobile support

This phased approach allows incremental delivery while maintaining quality.
