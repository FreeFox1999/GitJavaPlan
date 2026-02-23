# GitHub Copilot Prompt — Fix Button Alignment in MainFrame.java (Java Swing / GridBagLayout)

## Context
This is a Java Swing desktop application (`MainFrame.java`) that uses `GridBagLayout` for its panels.
It has two key panels: **Jira Settings** and **File Selection**. Each panel has rows that consist of:
- A label (column 0)
- A text field / combo box (columns 1–2)
- An action button (column 3)

## Root Cause of the Bug
The helper method `addRow(panel, c, row, label, field, extra)` places the field at `gridx=1`
with `gridwidth = 1 + extra`. When `extra=2` is passed the field spans columns 1, 2, **and 3**.
But the action button is then placed at `gridx=3`, causing it to **overlap** the field.

## What Needs to Be Fixed

### 1. `buildJiraSettingsPanel()`
- The URL row calls `addRow(..., tfUrl, 2)` → change to `addRow(..., tfUrl, 1)` so the field only
  spans cols 1–2, leaving col 3 free for the **"Test Connection"** button.
- The PAT row has no button so it can keep `extra=2` (field spans cols 1–3).
- The checkbox and card panel rows must use `gridwidth=4` and `weightx=1`.

### 2. `buildFileSelectionPanel()`
- All three rows (CSV Folder, CSV File, Config File) call `addRow(..., field, 2)` → change to
  `addRow(..., field, 1)` so each field spans cols 1–2 and buttons land cleanly at col 3.

### 3. `addRow()` helper — add `weightx`
- Labels: `weightx = 0`
- Fields: `weightx = 1.0` (so they stretch to fill available width)
- After placing the field, reset `weightx = 0` and `gridwidth = 1`

### 4. Add `uniformButton(String text)` helper
- All action buttons should have the same preferred width (120 px) so they form a clean right
  column regardless of label text length.
- Replace every `new JButton(text)` call for action buttons with `uniformButton(text)`.

## Exact Code to Generate

### `addRow` (replace existing):
```java
private void addRow(JPanel p, GridBagConstraints c, int row, String lbl, JComponent field, int extra) {
    c.gridx = 0; c.gridy = row; c.gridwidth = 1; c.weightx = 0;
    p.add(label(lbl), c);
    c.gridx = 1; c.gridwidth = 1 + extra; c.weightx = 1.0;
    p.add(field, c);
    c.weightx = 0; c.gridwidth = 1;
}
```

### `uniformButton` (new method — add below `addRow`):
```java
/** Creates a button with a consistent preferred width so all action buttons align. */
private JButton uniformButton(String text) {
    JButton btn = new JButton(text);
    btn.setPreferredSize(new Dimension(120, btn.getPreferredSize().height));
    return btn;
}
```

### `buildJiraSettingsPanel()` — fixed button row:
```java
// Row 0: URL + Test Connection
addRow(panel, c, 0, "* Jira URL:", tfUrl, 1);           // field spans cols 1-2 only
c.gridx = 3; c.gridy = 0; c.gridwidth = 1;
c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
JButton btnTest = uniformButton("Test Connection");
btnTest.addActionListener(e -> onTestConnection());
panel.add(btnTest, c);

// Row 1: PAT (no button — field spans cols 1-3)
addRow(panel, c, 1, "* Personal Access Token:", tfToken, 2);

// Row 2: Checkbox (full width)
c.gridx = 0; c.gridy = 2; c.gridwidth = 4; c.weightx = 1;
// ... add checkbox ...

// Row 3: CardLayout panel (full width)
c.gridx = 0; c.gridy = 3; c.gridwidth = 4; c.weightx = 1;
// ... add card panel ...
```

### `buildFileSelectionPanel()` — fixed button rows:
```java
// Row 0: CSV Folder
addRow(panel, c, 0, "* CSV Folder:", tfCsvFolder, 1);
c.gridx = 3; c.gridy = 0; c.gridwidth = 1;
c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
JButton btnF = uniformButton("Browse…");
btnF.addActionListener(e -> onBrowseCsvFolder());
panel.add(btnF, c);

// Row 1: CSV File
addRow(panel, c, 1, "* CSV File:", cbCsvFiles, 1);
c.gridx = 3; c.gridy = 1; c.gridwidth = 1;
c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
JButton btnP = uniformButton("Preview");
btnP.addActionListener(e -> onPreview());
panel.add(btnP, c);

// Row 2: Config File
addRow(panel, c, 2, "* Config File:", tfConfigFile, 1);
c.gridx = 3; c.gridy = 2; c.gridwidth = 1;
c.fill = GridBagConstraints.HORIZONTAL; c.weightx = 0;
JButton btnC = uniformButton("Browse…");
btnC.addActionListener(e -> onBrowseConfigFile());
panel.add(btnC, c);
```

## Summary of Column Layout (after fix)
```
Col 0        | Col 1 ──── Col 2    | Col 3
─────────────┼─────────────────────┼────────────────
Label        │ Text Field (w=2)    │ Button (w=1)
Label        │ Text Field (w=3, no button)
Checkbox     │ (spans all 4 cols)
Card Panel   │ (spans all 4 cols)
```

## File to Edit
`src/main/java/com/estjira/ui/MainFrame.java`
