# Feature Specification: Resizable Columns with Smart Defaults

## Overview
This document specifies the implementation requirements for adding drag-resizable columns with smart default widths to the JSONL-Lab VSCode extension table view.

## Requirements

### 1. Smart Default Column Widths
- **Timestamp Column**: Should auto-fit to content width (typically ~180-200px for ISO 8601 timestamps)
- **Level Column**: Should auto-fit to content width (typically ~80-100px for log levels like ERROR, WARN, INFO, DEBUG)
- **Message Column**: Should expand to fill remaining available space
- Custom columns should default to auto-fit unless configured otherwise

### 2. Drag-Resizable Columns
Users should be able to resize columns by:
- Hovering over column borders to see resize cursor
- Clicking and dragging column borders to adjust width
- Double-clicking column borders to auto-fit to content
- Maintaining resize state across sessions (persist in workspace/user settings)

### 3. Column Configuration UI
Add a "Column Settings" popup/panel that allows users to:
- Toggle which columns are visible
- Set column width behavior (auto-fit vs. expand vs. fixed)
- Reorder columns (drag-and-drop or up/down buttons)
- Reset columns to default configuration

## Implementation Approach

### Data Structure for Column Configuration

```typescript
interface ColumnConfig {
  id: string;
  label: string;
  width?: number;           // Fixed width in pixels (optional)
  minWidth?: number;        // Minimum width in pixels
  maxWidth?: number;        // Maximum width in pixels
  resizable: boolean;       // Whether column can be resized
  visible: boolean;         // Whether column is currently visible
  sizeMode: 'auto' | 'fixed' | 'flex';  // How width is determined
  flexGrow?: number;        // Flex-grow value when sizeMode is 'flex'
  order: number;            // Display order
}

const defaultColumns: ColumnConfig[] = [
  {
    id: 'timestamp',
    label: 'Timestamp',
    minWidth: 120,
    maxWidth: 300,
    resizable: true,
    visible: true,
    sizeMode: 'auto',
    order: 0
  },
  {
    id: 'level',
    label: 'Level',
    minWidth: 60,
    maxWidth: 150,
    resizable: true,
    visible: true,
    sizeMode: 'auto',
    order: 1
  },
  {
    id: 'message',
    label: 'Message',
    minWidth: 200,
    resizable: true,
    visible: true,
    sizeMode: 'flex',
    flexGrow: 1,
    order: 2
  }
];
```

### VSCode Webview Implementation

Since VSCode extensions use webviews for custom UI, the table should be implemented using HTML/CSS with JavaScript for interactions.

#### HTML Structure
```html
<div class="jsonl-table-container">
  <table class="jsonl-table">
    <thead>
      <tr class="header-row">
        <th class="column-header resizable" data-column="timestamp">
          <span class="column-label">Timestamp</span>
          <div class="resize-handle"></div>
        </th>
        <th class="column-header resizable" data-column="level">
          <span class="column-label">Level</span>
          <div class="resize-handle"></div>
        </th>
        <th class="column-header resizable flex-column" data-column="message">
          <span class="column-label">Message</span>
          <div class="resize-handle"></div>
        </th>
      </tr>
    </thead>
    <tbody>
      <!-- Table rows -->
    </tbody>
  </table>
</div>
```

#### CSS Implementation
```css
.jsonl-table-container {
  width: 100%;
  height: 100%;
  overflow: auto;
}

.jsonl-table {
  width: 100%;
  table-layout: fixed;
  border-collapse: collapse;
}

.column-header {
  position: relative;
  padding: 8px;
  border: 1px solid var(--vscode-panel-border);
  background: var(--vscode-editor-background);
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Auto-fit columns (Timestamp, Level) */
.column-header[data-column="timestamp"],
.column-header[data-column="level"] {
  width: auto;
}

/* Flex column (Message) */
.column-header.flex-column {
  width: 100%;
}

.resize-handle {
  position: absolute;
  top: 0;
  right: -3px;
  width: 6px;
  height: 100%;
  cursor: col-resize;
  z-index: 1;
}

.resize-handle:hover,
.resize-handle.resizing {
  background: var(--vscode-focusBorder);
}

.column-header.resizing {
  user-select: none;
}
```

#### JavaScript Resize Logic
```javascript
class ColumnResizer {
  constructor(tableElement) {
    this.table = tableElement;
    this.currentHandle = null;
    this.currentColumn = null;
    this.startX = 0;
    this.startWidth = 0;
    
    this.initResizeHandles();
  }
  
  initResizeHandles() {
    const handles = this.table.querySelectorAll('.resize-handle');
    handles.forEach(handle => {
      handle.addEventListener('mousedown', this.onResizeStart.bind(this));
      handle.addEventListener('dblclick', this.onAutoFit.bind(this));
    });
  }
  
  onResizeStart(e) {
    e.preventDefault();
    this.currentHandle = e.target;
    this.currentColumn = e.target.closest('.column-header');
    this.startX = e.pageX;
    this.startWidth = this.currentColumn.offsetWidth;
    
    document.addEventListener('mousemove', this.onResize.bind(this));
    document.addEventListener('mouseup', this.onResizeEnd.bind(this));
    
    this.currentColumn.classList.add('resizing');
  }
  
  onResize(e) {
    if (!this.currentColumn) return;
    
    const diff = e.pageX - this.startX;
    const newWidth = Math.max(50, this.startWidth + diff); // Min 50px
    
    this.currentColumn.style.width = newWidth + 'px';
    
    // Notify extension of width change
    this.saveColumnWidth(this.currentColumn.dataset.column, newWidth);
  }
  
  onResizeEnd(e) {
    document.removeEventListener('mousemove', this.onResize.bind(this));
    document.removeEventListener('mouseup', this.onResizeEnd.bind(this));
    
    if (this.currentColumn) {
      this.currentColumn.classList.remove('resizing');
    }
    
    this.currentColumn = null;
    this.currentHandle = null;
  }
  
  onAutoFit(e) {
    const column = e.target.closest('.column-header');
    const columnId = column.dataset.column;
    
    // Calculate width based on content
    const width = this.calculateAutoFitWidth(columnId);
    column.style.width = width + 'px';
    
    this.saveColumnWidth(columnId, width);
  }
  
  calculateAutoFitWidth(columnId) {
    // Get max content width for this column across visible rows
    const cells = this.table.querySelectorAll(`td[data-column="${columnId}"]`);
    let maxWidth = 0;
    
    cells.forEach(cell => {
      const textWidth = this.getTextWidth(cell.textContent, 
        window.getComputedStyle(cell).font);
      maxWidth = Math.max(maxWidth, textWidth);
    });
    
    // Add padding
    return maxWidth + 16;
  }
  
  getTextWidth(text, font) {
    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    context.font = font;
    return context.measureText(text).width;
  }
  
  saveColumnWidth(columnId, width) {
    // Send message to extension to persist setting
    vscode.postMessage({
      command: 'updateColumnWidth',
      columnId: columnId,
      width: width
    });
  }
}

// Initialize on load
const resizer = new ColumnResizer(document.querySelector('.jsonl-table'));
```

### Extension Settings Storage

```typescript
// In extension.ts
import * as vscode from 'vscode';

interface ColumnWidths {
  [columnId: string]: number;
}

class ColumnConfigManager {
  private context: vscode.ExtensionContext;
  
  constructor(context: vscode.ExtensionContext) {
    this.context = context;
  }
  
  getColumnWidths(): ColumnWidths {
    return this.context.workspaceState.get('jsonlLab.columnWidths', {});
  }
  
  async setColumnWidth(columnId: string, width: number): Promise<void> {
    const widths = this.getColumnWidths();
    widths[columnId] = width;
    await this.context.workspaceState.update('jsonlLab.columnWidths', widths);
  }
  
  async resetColumnWidths(): Promise<void> {
    await this.context.workspaceState.update('jsonlLab.columnWidths', {});
  }
}

// Handle messages from webview
panel.webview.onDidReceiveMessage(
  message => {
    switch (message.command) {
      case 'updateColumnWidth':
        configManager.setColumnWidth(message.columnId, message.width);
        break;
    }
  },
  undefined,
  context.subscriptions
);
```

### Column Settings UI

Create a settings panel accessible via button in the table header:

```html
<div class="column-settings-panel" hidden>
  <div class="settings-header">
    <h3>Column Settings</h3>
    <button class="close-btn">×</button>
  </div>
  
  <div class="settings-body">
    <div class="column-list">
      <div class="column-item" data-column="timestamp">
        <input type="checkbox" checked class="column-visible">
        <label>Timestamp</label>
        <select class="size-mode">
          <option value="auto" selected>Auto-fit</option>
          <option value="fixed">Fixed</option>
          <option value="flex">Expand</option>
        </select>
        <div class="reorder-buttons">
          <button class="move-up">↑</button>
          <button class="move-down">↓</button>
        </div>
      </div>
      <!-- More column items -->
    </div>
  </div>
  
  <div class="settings-footer">
    <button class="reset-btn">Reset to Defaults</button>
    <button class="apply-btn">Apply</button>
  </div>
</div>
```

## Testing Considerations

1. **Resize Behavior**
   - Test resizing columns to various widths
   - Test minimum width constraints
   - Test maximum width constraints
   - Test double-click auto-fit
   
2. **Persistence**
   - Verify column widths persist across sessions
   - Verify settings sync across workspace vs user settings
   
3. **Flex Layout**
   - Test with different viewport sizes
   - Verify Message column expands/contracts appropriately
   - Test with different numbers of columns
   
4. **Edge Cases**
   - Very long content in columns
   - Very short content in columns
   - Resizing to extreme values
   - Multiple columns with flex-grow

## Accessibility

- Ensure resize handles are keyboard accessible (Tab to focus, Arrow keys to resize)
- Provide ARIA labels for resize handles
- Ensure column settings panel is screen reader friendly
- Support high contrast mode

## Performance

- Debounce resize events to avoid excessive updates
- Use CSS transforms where possible for smooth animations
- Lazy calculate auto-fit widths only for visible rows
- Cache calculated widths

## Migration

For existing users, provide a one-time migration that:
1. Applies default column widths if none are stored
2. Preserves any custom column visibility settings
3. Shows a notification about the new resize feature

## Future Enhancements

- Column reordering via drag-and-drop in table header
- Save/load column presets
- Export column configuration
- Column grouping/nesting
- Frozen columns (e.g., keep timestamp column always visible while scrolling)
