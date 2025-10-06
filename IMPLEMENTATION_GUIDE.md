# Implementation Guide: Column Resize Feature

This guide provides step-by-step instructions for implementing the column resize feature in the JSONL-Lab VSCode extension.

## Prerequisites

- VSCode Extension Development Environment
- TypeScript/JavaScript knowledge
- Understanding of VSCode Webview API
- Familiarity with HTML/CSS/JavaScript

## Implementation Steps

### Step 1: Update Package.json Configuration

Add configuration settings for column widths in `package.json`:

```json
{
  "contributes": {
    "configuration": {
      "title": "JSONL Lab",
      "properties": {
        "jsonlLab.columns.defaultWidths": {
          "type": "object",
          "default": {
            "timestamp": "auto",
            "level": "auto",
            "message": "flex"
          },
          "description": "Default width modes for columns (auto, flex, or pixel value)"
        },
        "jsonlLab.columns.minWidth": {
          "type": "number",
          "default": 50,
          "description": "Minimum width for resizable columns in pixels"
        },
        "jsonlLab.columns.persistWidths": {
          "type": "boolean",
          "default": true,
          "description": "Remember column widths across sessions"
        }
      }
    }
  }
}
```

### Step 2: Create Column Configuration Module

Create a new file `src/columnConfig.ts`:

```typescript
import * as vscode from 'vscode';

export type SizeMode = 'auto' | 'flex' | 'fixed';

export interface ColumnConfig {
  id: string;
  label: string;
  width?: number;
  minWidth: number;
  maxWidth?: number;
  resizable: boolean;
  visible: boolean;
  sizeMode: SizeMode;
  flexGrow?: number;
  order: number;
}

export const DEFAULT_COLUMNS: ColumnConfig[] = [
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

export class ColumnConfigManager {
  private static readonly STORAGE_KEY = 'jsonlLab.columnWidths';
  
  constructor(private context: vscode.ExtensionContext) {}
  
  public getColumnConfigs(): ColumnConfig[] {
    const stored = this.context.workspaceState.get<Record<string, Partial<ColumnConfig>>>(
      ColumnConfigManager.STORAGE_KEY,
      {}
    );
    
    return DEFAULT_COLUMNS.map(col => ({
      ...col,
      ...stored[col.id]
    }));
  }
  
  public async updateColumnConfig(columnId: string, updates: Partial<ColumnConfig>): Promise<void> {
    const stored = this.context.workspaceState.get<Record<string, Partial<ColumnConfig>>>(
      ColumnConfigManager.STORAGE_KEY,
      {}
    );
    
    stored[columnId] = {
      ...stored[columnId],
      ...updates
    };
    
    await this.context.workspaceState.update(ColumnConfigManager.STORAGE_KEY, stored);
  }
  
  public async resetColumnConfigs(): Promise<void> {
    await this.context.workspaceState.update(ColumnConfigManager.STORAGE_KEY, {});
  }
  
  public async setColumnWidth(columnId: string, width: number): Promise<void> {
    const config = vscode.workspace.getConfiguration('jsonlLab.columns');
    if (config.get('persistWidths', true)) {
      await this.updateColumnConfig(columnId, { width, sizeMode: 'fixed' });
    }
  }
}
```

### Step 3: Update Webview HTML

Modify your webview HTML to include resize handles and proper column structure:

```typescript
// In your webview provider file
function getWebviewContent(columns: ColumnConfig[]): string {
  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>JSONL Viewer</title>
  <style>${getStyles()}</style>
</head>
<body>
  <div class="toolbar">
    <button id="columnSettingsBtn" class="toolbar-btn">
      <span class="codicon codicon-settings-gear"></span>
      Column Settings
    </button>
  </div>
  
  <div class="jsonl-table-container">
    <table class="jsonl-table">
      <thead>
        <tr class="header-row">
          ${columns.map(col => `
            <th class="column-header ${col.sizeMode === 'flex' ? 'flex-column' : ''}" 
                data-column="${col.id}"
                style="${getColumnStyle(col)}">
              <span class="column-label">${col.label}</span>
              ${col.resizable ? '<div class="resize-handle"></div>' : ''}
            </th>
          `).join('')}
        </tr>
      </thead>
      <tbody id="tableBody">
        <!-- Rows will be inserted here -->
      </tbody>
    </table>
  </div>
  
  <div id="columnSettingsPanel" class="column-settings-panel" hidden>
    <!-- Settings panel content -->
  </div>
  
  <script>${getScript()}</script>
</body>
</html>`;
}

function getColumnStyle(col: ColumnConfig): string {
  if (col.sizeMode === 'fixed' && col.width) {
    return `width: ${col.width}px;`;
  } else if (col.sizeMode === 'auto') {
    return `width: auto;`;
  } else if (col.sizeMode === 'flex') {
    return `width: ${col.flexGrow || 1}fr;`;
  }
  return '';
}
```

### Step 4: Add CSS Styles

Create the styles function:

```typescript
function getStyles(): string {
  return `
    :root {
      --min-column-width: 50px;
      --resize-handle-width: 6px;
    }
    
    body {
      margin: 0;
      padding: 0;
      overflow: hidden;
      font-family: var(--vscode-font-family);
      color: var(--vscode-foreground);
      background: var(--vscode-editor-background);
    }
    
    .toolbar {
      padding: 8px;
      border-bottom: 1px solid var(--vscode-panel-border);
      display: flex;
      gap: 8px;
    }
    
    .toolbar-btn {
      background: var(--vscode-button-background);
      color: var(--vscode-button-foreground);
      border: none;
      padding: 6px 12px;
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 4px;
      border-radius: 2px;
    }
    
    .toolbar-btn:hover {
      background: var(--vscode-button-hoverBackground);
    }
    
    .jsonl-table-container {
      width: 100%;
      height: calc(100vh - 45px);
      overflow: auto;
    }
    
    .jsonl-table {
      width: 100%;
      border-collapse: collapse;
      table-layout: fixed;
    }
    
    .column-header {
      position: relative;
      padding: 8px;
      border: 1px solid var(--vscode-panel-border);
      background: var(--vscode-editorGroupHeader-tabsBackground);
      text-align: left;
      font-weight: 600;
      white-space: nowrap;
      user-select: none;
    }
    
    .column-label {
      display: inline-block;
      overflow: hidden;
      text-overflow: ellipsis;
      max-width: calc(100% - var(--resize-handle-width));
    }
    
    .resize-handle {
      position: absolute;
      top: 0;
      right: -3px;
      width: var(--resize-handle-width);
      height: 100%;
      cursor: col-resize;
      z-index: 10;
    }
    
    .resize-handle:hover {
      background: var(--vscode-focusBorder);
      opacity: 0.5;
    }
    
    .resize-handle.resizing {
      background: var(--vscode-focusBorder);
      opacity: 1;
    }
    
    .column-header.resizing {
      user-select: none;
      cursor: col-resize;
    }
    
    .jsonl-table tbody tr {
      border-bottom: 1px solid var(--vscode-panel-border);
    }
    
    .jsonl-table tbody tr:hover {
      background: var(--vscode-list-hoverBackground);
    }
    
    .jsonl-table tbody td {
      padding: 6px 8px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    
    .column-settings-panel {
      position: fixed;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      background: var(--vscode-editor-background);
      border: 1px solid var(--vscode-panel-border);
      box-shadow: 0 4px 16px rgba(0, 0, 0, 0.3);
      padding: 16px;
      min-width: 400px;
      max-width: 600px;
      z-index: 1000;
    }
    
    .settings-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 16px;
      padding-bottom: 8px;
      border-bottom: 1px solid var(--vscode-panel-border);
    }
    
    .settings-header h3 {
      margin: 0;
    }
    
    .close-btn {
      background: transparent;
      border: none;
      font-size: 24px;
      cursor: pointer;
      color: var(--vscode-foreground);
      padding: 0;
      width: 24px;
      height: 24px;
      line-height: 24px;
    }
    
    .close-btn:hover {
      background: var(--vscode-toolbar-hoverBackground);
    }
    
    .column-list {
      max-height: 400px;
      overflow-y: auto;
    }
    
    .column-item {
      display: flex;
      align-items: center;
      gap: 12px;
      padding: 8px;
      border-bottom: 1px solid var(--vscode-panel-border);
    }
    
    .column-item:last-child {
      border-bottom: none;
    }
    
    .column-visible {
      margin: 0;
    }
    
    .column-item label {
      flex: 1;
      min-width: 100px;
    }
    
    .size-mode {
      padding: 4px 8px;
      background: var(--vscode-input-background);
      color: var(--vscode-input-foreground);
      border: 1px solid var(--vscode-input-border);
    }
    
    .reorder-buttons {
      display: flex;
      gap: 4px;
    }
    
    .reorder-buttons button {
      padding: 2px 6px;
      background: var(--vscode-button-secondaryBackground);
      color: var(--vscode-button-secondaryForeground);
      border: none;
      cursor: pointer;
    }
    
    .reorder-buttons button:hover {
      background: var(--vscode-button-secondaryHoverBackground);
    }
    
    .settings-footer {
      margin-top: 16px;
      padding-top: 16px;
      border-top: 1px solid var(--vscode-panel-border);
      display: flex;
      justify-content: space-between;
    }
    
    .reset-btn, .apply-btn {
      padding: 6px 16px;
      border: none;
      cursor: pointer;
      border-radius: 2px;
    }
    
    .reset-btn {
      background: var(--vscode-button-secondaryBackground);
      color: var(--vscode-button-secondaryForeground);
    }
    
    .reset-btn:hover {
      background: var(--vscode-button-secondaryHoverBackground);
    }
    
    .apply-btn {
      background: var(--vscode-button-background);
      color: var(--vscode-button-foreground);
    }
    
    .apply-btn:hover {
      background: var(--vscode-button-hoverBackground);
    }
  `;
}
```

### Step 5: Add JavaScript for Resize Functionality

Create the script function:

```typescript
function getScript(): string {
  return `
    (function() {
      const vscode = acquireVsCodeApi();
      
      class ColumnResizer {
        constructor() {
          this.table = document.querySelector('.jsonl-table');
          this.resizing = false;
          this.currentColumn = null;
          this.startX = 0;
          this.startWidth = 0;
          
          this.init();
        }
        
        init() {
          this.initResizeHandles();
          this.initColumnSettings();
        }
        
        initResizeHandles() {
          const handles = document.querySelectorAll('.resize-handle');
          
          handles.forEach(handle => {
            handle.addEventListener('mousedown', (e) => this.startResize(e));
            handle.addEventListener('dblclick', (e) => this.autoFit(e));
          });
          
          document.addEventListener('mousemove', (e) => this.doResize(e));
          document.addEventListener('mouseup', (e) => this.stopResize(e));
        }
        
        startResize(e) {
          e.preventDefault();
          e.stopPropagation();
          
          this.resizing = true;
          this.currentColumn = e.target.closest('.column-header');
          this.startX = e.pageX;
          this.startWidth = this.currentColumn.offsetWidth;
          
          this.currentColumn.classList.add('resizing');
          e.target.classList.add('resizing');
          document.body.style.cursor = 'col-resize';
        }
        
        doResize(e) {
          if (!this.resizing || !this.currentColumn) return;
          
          const diff = e.pageX - this.startX;
          const newWidth = Math.max(50, this.startWidth + diff);
          
          this.currentColumn.style.width = newWidth + 'px';
          
          // Update all cells in this column
          const columnId = this.currentColumn.dataset.column;
          const cells = document.querySelectorAll(\`td[data-column="\${columnId}"]\`);
          cells.forEach(cell => {
            cell.style.width = newWidth + 'px';
          });
        }
        
        stopResize(e) {
          if (!this.resizing) return;
          
          this.resizing = false;
          document.body.style.cursor = '';
          
          if (this.currentColumn) {
            const columnId = this.currentColumn.dataset.column;
            const width = this.currentColumn.offsetWidth;
            
            this.currentColumn.classList.remove('resizing');
            const handle = this.currentColumn.querySelector('.resize-handle');
            if (handle) handle.classList.remove('resizing');
            
            // Save to extension
            vscode.postMessage({
              command: 'updateColumnWidth',
              columnId: columnId,
              width: width
            });
            
            this.currentColumn = null;
          }
        }
        
        autoFit(e) {
          e.preventDefault();
          e.stopPropagation();
          
          const column = e.target.closest('.column-header');
          const columnId = column.dataset.column;
          
          const width = this.calculateAutoFitWidth(columnId);
          column.style.width = width + 'px';
          
          // Update all cells in this column
          const cells = document.querySelectorAll(\`td[data-column="\${columnId}"]\`);
          cells.forEach(cell => {
            cell.style.width = width + 'px';
          });
          
          vscode.postMessage({
            command: 'updateColumnWidth',
            columnId: columnId,
            width: width
          });
        }
        
        calculateAutoFitWidth(columnId) {
          const cells = document.querySelectorAll(\`td[data-column="\${columnId}"]\`);
          let maxWidth = 0;
          
          const tempSpan = document.createElement('span');
          tempSpan.style.visibility = 'hidden';
          tempSpan.style.position = 'absolute';
          tempSpan.style.whiteSpace = 'nowrap';
          document.body.appendChild(tempSpan);
          
          cells.forEach(cell => {
            tempSpan.textContent = cell.textContent;
            tempSpan.style.font = window.getComputedStyle(cell).font;
            maxWidth = Math.max(maxWidth, tempSpan.offsetWidth);
          });
          
          document.body.removeChild(tempSpan);
          
          // Add padding
          return Math.max(maxWidth + 32, 50);
        }
        
        initColumnSettings() {
          const settingsBtn = document.getElementById('columnSettingsBtn');
          const settingsPanel = document.getElementById('columnSettingsPanel');
          const closeBtn = settingsPanel?.querySelector('.close-btn');
          
          if (settingsBtn && settingsPanel) {
            settingsBtn.addEventListener('click', () => {
              settingsPanel.hidden = !settingsPanel.hidden;
            });
            
            if (closeBtn) {
              closeBtn.addEventListener('click', () => {
                settingsPanel.hidden = true;
              });
            }
          }
        }
      }
      
      // Initialize when DOM is ready
      if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', () => {
          new ColumnResizer();
        });
      } else {
        new ColumnResizer();
      }
    })();
  `;
}
```

### Step 6: Handle Messages in Extension

Update your extension's message handler:

```typescript
// In your webview provider
panel.webview.onDidReceiveMessage(
  async message => {
    switch (message.command) {
      case 'updateColumnWidth':
        await columnConfigManager.setColumnWidth(
          message.columnId,
          message.width
        );
        break;
        
      case 'resetColumns':
        await columnConfigManager.resetColumnConfigs();
        // Refresh webview
        break;
        
      case 'updateColumnConfig':
        await columnConfigManager.updateColumnConfig(
          message.columnId,
          message.updates
        );
        break;
    }
  },
  undefined,
  context.subscriptions
);
```

### Step 7: Testing

Create test scenarios:

1. **Basic Resize Test**
   ```typescript
   // Test that columns can be resized
   // Test that resize persists across sessions
   ```

2. **Auto-fit Test**
   ```typescript
   // Test double-click auto-fits column
   // Test with various content lengths
   ```

3. **Smart Defaults Test**
   ```typescript
   // Verify Timestamp/Level use auto-fit
   // Verify Message uses flex
   ```

## Deployment Checklist

- [ ] Update package.json with new configuration options
- [ ] Implement ColumnConfigManager class
- [ ] Update webview HTML with resize handles
- [ ] Add CSS for resize functionality
- [ ] Implement JavaScript resize logic
- [ ] Add message handlers in extension
- [ ] Test resize functionality
- [ ] Test persistence across sessions
- [ ] Test with different VSCode themes
- [ ] Update extension README with new feature
- [ ] Create demo GIF/video showing resize in action
- [ ] Bump extension version
- [ ] Publish to marketplace

## Troubleshooting

### Columns not resizing smoothly
- Check for CSS transitions that might interfere
- Ensure table-layout is set to 'fixed'
- Verify event listeners are properly attached

### Widths not persisting
- Verify workspace state is being updated
- Check configuration setting for persistWidths
- Ensure message passing is working correctly

### Auto-fit not working correctly
- Check text measurement calculation
- Verify canvas context is available
- Ensure all cells are being measured

## Performance Optimization

1. **Debounce resize events** - Only save to storage after resize completes
2. **Throttle resize updates** - Limit DOM updates during drag
3. **Lazy calculate auto-fit** - Only measure visible cells
4. **Cache measurements** - Store calculated widths temporarily

## Accessibility Improvements

Add keyboard support:
```javascript
// Allow keyboard resize
column.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowLeft') {
    // Decrease width
  } else if (e.key === 'ArrowRight') {
    // Increase width
  }
});
```

Add ARIA attributes:
```html
<div class="resize-handle" 
     role="separator" 
     aria-orientation="vertical"
     aria-label="Resize column">
</div>
```
