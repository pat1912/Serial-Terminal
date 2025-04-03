# Requirements Document: Web Serial Terminal Application (Consolidated V6.0 - Code Aligned)

## 1. Introduction

### 1.1. Project Goal

To develop a static web application (HTML, CSS, JavaScript) that functions as a serial terminal using the browser's **Web Serial API**. The application will allow users to connect to serial devices, configure connection parameters (Baud, Data/Stop Bits, Parity) via controls in the **left panel**, send predefined and custom commands (primarily Hex-based), view incoming data, log communication, manage configurations via projects, and utilize a guided **Script Wizard** for creating and managing automation scripts with AI assistance, without requiring any local software installation.

### 1.2. Core Technology

The application **must** exclusively use the **Web Serial API** for serial port communication. It must run entirely client-side in compatible web browsers (e.g., Chrome, Edge, Opera) that support this API.

### 1.3. Target Users

Users who need to interact with serial devices (microcontrollers, IoT devices, debugging hardware, etc.) directly from their web browser, often dealing with byte-oriented (Hex) protocols, without installing native terminal applications. Users may also desire assistance in creating automation scripts.

## 2. Functional Requirements

### 2.1. Serial Port Connection Management

*   **FR2.1.1:** Provide a clear "Connect" button in the header to initiate the serial port connection process.
*   **FR2.1.2:** Upon clicking "Connect", the application shall use `navigator.serial.requestPort()` to trigger the browser's native dialog for port selection.
*   **FR2.1.3:** **(CODE ALIGNED)** Provide UI controls within a standard `fieldset` in the **left panel** allowing the user to configure connection parameters *before* connecting:
    *   Controls include:
        *   Baud Rate (Dropdown: 9600-921600).
        *   Data Bits (Dropdown: 7, 8).
        *   Stop Bits (Dropdown: 1, 2).
        *   Parity (Dropdown: None, Even, Odd).
    *   These controls shall be **disabled** when a connection is active or attempting to connect (indicated visually, e.g., grayed out).
    *   *(These parameters are part of the saveable Project state - FR2.9.2)*
*   **FR2.1.4:** Display the current connection status clearly in the header (e.g., "Disconnected", "Connecting...", "Connected to [Port Name / Info]").
*   **FR2.1.5:** When connected, display the active connection parameters summary (e.g., "115200, 8N1") next to the status in the header.
*   **FR2.1.6:** Provide a "Disconnect" button in the header, enabled only when connected, to close the serial port connection gracefully.
*   **FR2.1.7:** Handle and display connection errors clearly to the user (e.g., "Port access denied", "Device disconnected", "Failed to open port", "No port selected").

### 2.2. Command Management & Transmission (TX)

*   **FR2.2.1:** Provide a dedicated UI section (`fieldset`) in the left panel for managing reusable commands.
*   **FR2.2.2:** Allow users to create/edit commands with: Name (unique), Hex Data (validated), optional Description.
*   **FR2.2.3:** Provide a form (`fieldset`, initially hidden) for adding/editing TX commands within the left panel.
*   **FR2.2.4:** Display a list (`ul`) of saved commands. Each list item shows: Name (truncated with ellipsis if needed) and Actions (Send, Edit, Del). Display full details (Name, Hex, Description) on hover (tooltip) over the name.
*   **FR2.2.5:** Provide mechanisms (Edit/Del buttons) to Edit and Delete saved commands.
*   **FR2.2.6:** "Send" button converts Hex Data to `Uint8Array`, transmits via `write()`, and logs (Timestamp, TX, Hex Data, Command Name). Disable Send if not connected.

### 2.3. Data Reception & Display (RX)

*   **FR2.3.1:** Continuously listen for incoming data on the active serial connection using a `readLoop`.
*   **FR2.3.2:** **Heuristic Data Display:** Log incoming data using `formatReceivedData` function, prioritizing printable ASCII characters and using placeholders (e.g., `<0A>`, `<C3>`) for non-printable or control bytes.
*   **FR2.3.3:** Display the raw Hex representation of the log line's data chunk as a tooltip on hover over the data portion of the log entry.

### 2.4. Message Delimiting / Line Cutting (RX)

*   **FR2.4.1:** Provide configuration options in the left panel (`fieldset`) for defining a single, global message delimiter (Hex bytes, validated using `isValidDelimiterHex`). *(Project setting - FR2.9.2)*
*   **FR2.4.2:** Buffer incoming data (`rxBuffer`) until the delimiter sequence is detected (if enabled and delimiter is defined). Uses `findDelimiterIndex`.
*   **FR2.4.3:** Treat buffered data (up to and including the delimiter) as a single message unit for logging (`processReceivedMessage`) and script `waitForData` checking.
*   **FR2.4.4:** Provide an option (checkbox `useDelimiter`) in the left panel to enable/disable delimiter-based line cutting. *(Project setting - FR2.9.2)*

### 2.5. Communication Log Display

*   **FR2.5.1:** Implement a main display area (`div.log-display`) in the right panel for the log.
*   **FR2.5.2:** Each entry represents a TX, RX, SCRIPT, SYS, or ERROR message/data chunk.
*   **FR2.5.3:** **Single-Line Log Entries:** Ensure log entries do not wrap and enable horizontal scrolling for the entire log display area (`overflow: auto`, `white-space: nowrap` on `.log-display`).
*   **FR2.5.4:** Each entry (`div.log-entry`) displays: Timestamp (`span.timestamp`), Direction (`span.direction`), Data (formatted, `span.data` with hex tooltip), and optional Description (`span.description`).
*   **FR2.5.5:** Use distinct visual styling (CSS classes like `direction-tx`, `data-rx`, theme classes `log-theme-*`) for entry types and parts.
*   **FR2.5.6:** Automatically scroll log display to the bottom on new entries, unless the user has manually scrolled up (`isScrolledToBottom` check).
*   **FR2.5.7:** Provide "Clear Log", "Copy Log", "Save Log (.log)" buttons in the log header.
*   **FR2.5.8:** **Log Display Theming:** Provide a dropdown (`logThemeSelect`) in the log header for themes (Default Dark, Dark, Light, High Contrast), affecting log area colors, persisted via `localStorage`.

### 2.6. RX Message Pattern Matching (Optional - Deferred)

*   *(Deferred)*

### 2.7. RX Message Description Mapping (Command Byte Lookup)

*   **FR2.7.1:** Implement mechanism for adding descriptions based on a specific byte in received messages:
    *   UI: "RX Message Definitions" section (`fieldset`) in the left panel.
    *   User defines: **Command Byte Index** (`cmdBytePosInput`, numeric input, zero-based).
    *   User defines: **Command Mapping Table** via an add/edit form (`addRxDefForm`, initially hidden) and list (`ul#rxDefList`). Each mapping defines: `Value (Hex)` (1-2 hex chars, unique, validated via `isValidHexByte` and uniqueness check), `Description` (required).
    *   Display list shows `Value (Hex)` (in `item-primary` styled span) and `Description` (in `item-secondary` styled span, truncated with ellipsis, full description on hover tooltip).
    *   *(Project setting - FR2.9.2)*
*   **FR2.7.2:** **Logic:** Implemented in `getRxDescription`. When an RX message is processed, check if message length > Cmd Byte Index. If so, extract the byte at the defined index, convert it to a padded 2-char uppercase Hex string (e.g., '0A', 'C3'), and look it up in the `state.rxCmdMap`. Append the mapped description to the log entry if a match is found.

### 2.8. Advanced Scripting Feature (with AI-Assisted Wizard)

*   **FR2.8.1:** **Scripting Goal:** Allow users to define, save, and execute automated JavaScript sequences, facilitated by a guided wizard (`scriptWizardModal`) that assists in generating code prompts for external AI tools.
*   **FR2.8.2:** **Scripting Language:** JavaScript.
*   **FR2.8.3:** **Script Execution Environment:** Sandboxed using `AsyncFunction`. Mandatory security warnings are displayed (FR2.8.12, FR2.5.5 equivalent in wizard UI/logs).
*   **FR2.8.4:** **Scripting API:** Expose API within scripts via `buildScriptApi`:
    *   `log(message: string)`
    *   `await sleep(milliseconds: number): Promise<void>`
    *   `await sendCommand(commandName: string): Promise<boolean>`
    *   `await sendHex(hexString: string): Promise<boolean>`
    *   `await sendAscii(asciiString: string): Promise<boolean>`
    *   `await waitForData(options: object): Promise<object | null>` (`options`: `{ pattern?: string | RegExp, timeout?: number, format?: 'ascii' | 'hex' | 'description' | 'raw' }`). Checks pending waits via `checkPendingWaits`.
    *   `getCommands(): object[]` (Returns deep copy of `state.txCommands`).
    *   `getRxDefinitions(): object` (Returns deep copy of RX settings: `{ commandBytePosition, commandMap }`).
*   **FR2.8.5:** **Script Management UI (Left Panel):**
    *   Section (`fieldset`) listing saved scripts (`ul#scriptList`) by Name. Script names exceeding available space are truncated with ellipsis, showing the full name and user description (if available) on hover tooltip.
    *   Buttons per script: "Run", "Edit", "Delete". Disable Run if not connected.
    *   Button "Add New Script (Wizard)" (`addScriptBtn`) to launch the modal wizard.
    *   *(Saved scripts are part of Project state - FR2.9.2)*
*   **FR2.8.6:** **Script Wizard UI (Modal Dialog):**
    *   Wizard appears as a modal dialog (`scriptWizardModal`) for adding or editing scripts.
    *   Clear visual separation for steps (`wizardStep1`, `wizardStep2`, `wizardStep3`).
    *   Mechanisms to open (`addScriptBtn`, Edit button) and close (`wizardCloseBtn`, `wizardCancelBtn`, Save button) the modal.
*   **FR2.8.7:** **Wizard Data Structure:** Stores script `name`, user's natural language `description`, and `code` (JavaScript including the final explanation comment) per script in `state.scripts`. Uses temporary `wizardData` during editing.
*   **FR2.8.8:** **Wizard Flow - Step 1: Define Logic:** UI (`wizardStep1`) includes Script Name (`wizardScriptNameInput`, validated unique) and Script Logic Description (`wizardScriptDescriptionInput`). Pre-fills on edit. "Next: Generate Prompt" button (`wizardNextBtn`) advances after validation.
*   **FR2.8.9:** **Wizard Flow - Step 2: Generate & Review AI Prompt:** Dynamically generates AI prompt via `generateWizardPrompt` (incorporating Objective, API, Current Project Context, User Request, AI Instructions - requesting single JS code block with final explanation comment). Displays prompt in read-only textarea (`wizardAiPromptDisplay`). Provides "Copy Prompt" button (`wizardCopyPromptBtn`), "Next: Paste Code" (`wizardNextBtn`), and "Back" (`wizardBackBtn`) buttons. Includes user instructions text.
*   **FR2.8.10:** **Wizard Flow - Step 3: Paste, Review Explanation, & Save Code:** UI (`wizardStep3`) displays AI's Explanation (read-only `div#wizardAiExplanationDisplay`, extracted from code comment using `extractAndDisplayWizardExplanation`). Textarea (`wizardPastedCodeInput`) for pasting AI response (code + comment block). Pre-fills on edit. Explanation display updates dynamically on paste/input. Buttons: "Save Script" (`wizardSaveBtn`), "Back" (`wizardBackBtn`), "Cancel" (`wizardCancelBtn`). **Security Warning** message is displayed.
*   **FR2.8.11:** **Script Execution Flow:** "Run" button triggers `handleRunScript`, which instantiates the sandboxed environment (`AsyncFunction`), injects the API (`buildScriptApi`), executes the script code, and logs output/errors. `waitForData` uses `pendingScriptWaits` list managed by `checkPendingWaits`.
*   **FR2.8.12:** **Security Warning:** Clear, persistent warning message is displayed in the wizard's Step 3 UI and errors during script execution are logged with an 'ERROR' type.

### 2.9. Project Management (Configuration Persistence)

*   **FR2.9.1:** **Project Concept:** Encapsulates the application's configurable state for a specific device/task, allowing users to save and load different configurations.
*   **FR2.9.2:** **Project Contents:** The `state` object managed by the `App` class includes:
    *   Project Name (`projectName`)
    *   Saved TX Commands (`txCommands`: Array of { name, hex, description }) (FR2.2).
    *   RX Message Definitions (`rxCmdBytePos`: number, `rxCmdMap`: object {'HEX': description}) (FR2.7).
    *   General RX Delimiter Settings (`rxDelimiterHex`: string, `rxUseDelimiter`: boolean) (FR2.4).
    *   Serial Connection Parameters (`baudRate`, `dataBits`, `stopBits`, `parity`) (FR2.1.3).
    *   Saved Scripts (`scripts`: Array of { name, description, code }) (FR2.8).
*   **FR2.9.3:** **Project File Format:** JSON. Uses `JSON.stringify` for saving and `JSON.parse` for loading.
*   **FR2.9.4:** **Export/Save Project:** "Save Project As..." button (`saveProjectBtn`) triggers `saveProject`, which serializes the current `state` (excluding transient properties like the port object) to a JSON string and prompts the user to download it as a `.json` file named after the project.
*   **FR2.9.5:** **Import/Open Project:** "Open Project..." button (`openProjectBtn`) triggers a hidden file input (`loadProjectInput`). On file selection, `handleProjectFileSelected` reads the file, parses the JSON, validates the structure (`isValidProjectData`), disconnects any active serial connection, loads the valid data into the application `state` (`loadProjectData`), updates the UI (`updateUIFromState`), and saves the loaded state to `localStorage`. Handles file read and parse errors with alerts and logs.
*   **FR2.9.6:** **Project Name Display:** Displays the current project name (`state.projectName`) prominently in the header (`projectNameSpan`). Updates from the filename (base name) on project load. Shows the full name on hover via the `title` attribute.
*   **FR2.9.7:** **Auto-Save:** Uses `localStorage` (key `webSerialTerminalState_v6_0`) to automatically persist the *current working state* (`state` object). Saving is triggered via a debounced function (`debouncedSaveState`) after changes to settings, commands, definitions, scripts, etc. A log message confirms loading from localStorage if data exists. File export is emphasized for explicit backup/sharing.

### 2.10. Log Export and Persistence

*   **FR2.10.1:** Store log data in an in-memory array (`logEntries`), limited by `MAX_LOG_ENTRIES` constant (oldest entries removed first).
*   **FR2.10.2:** Provide "Copy Log" button (`copyLogBtn`) to copy the current log content (formatted text) to the clipboard using `navigator.clipboard`.
*   **FR2.10.3:** Provide "Save Log" button (`saveLogBtn`) to trigger a download of the current log content as a `.log` file (`saveLogToFile`).
*   **FR2.10.4:** Acknowledge potential high memory usage with many log entries (mitigated by `MAX_LOG_ENTRIES`).

## 3. Non-Functional Requirements

### 3.1. Usability & User Interface (UI)

*   **NFR3.1.1:** Clean, intuitive user interface.
*   **NFR3.1.2:** **(CODE ALIGNED)** Logical layout with a fixed-width left panel for settings and management (TX commands, RX definitions, Scripts) and a main right panel for the communication log. Forms for adding/editing items within the left panel appear/disappear dynamically. A modal dialog (`scriptWizardModal`) is used for the focused task of script creation/editing.
*   **NFR3.1.3:** Clear labels, tooltips for truncated items and details, visual feedback for button states, connection status, and input validation errors.
*   **NFR3.1.4:** Visually distinct color scheme for log entries, supporting multiple themes selectable by the user (FR2.5.8).
*   **NFR3.1.5:** Aim for simple operation. The Script Wizard simplifies the process of generating automation script prompts.
*   **NFR3.1.6:** Ensure UI elements handle content appropriately. Implement text truncation with ellipsis for list items (TX command names, RX definition descriptions, script names) and project name in the header, showing full text/details on hover via tooltips.
*   **NFR3.1.7:** Maintain consistent visual presentation between similar list elements (TX Commands, RX Definitions, Scripts) using shared CSS classes (`item-list`, `item-details`, `item-actions`, `item-primary`, `item-secondary`).

### 3.2. Performance

*   **NFR3.2.1:** UI should remain responsive under typical serial data rates. Log rendering optimized by appending entries rather than full re-renders.
*   **NFR3.2.2:** Efficient data processing: Use appropriate data structures (`Uint8Array`, objects), debounced saving, limit in-memory log size. Script execution runs asynchronously.

### 3.3. Compatibility

*   **NFR3.3.1:** Must function in modern desktop browsers implementing the Web Serial API (e.g., Chrome, Edge, Opera). Displays a message and disables connection button if the API is not detected (`checkWebSerialSupport`).
*   **NFR3.3.2:** Must be served over HTTPS or accessed via `localhost` for the Web Serial API to function.

### 3.4. Maintainability

*   **NFR3.4.1:** Code structured with HTML, CSS, and JavaScript. JavaScript organized into an `App` class with distinct methods for different functionalities (connection, logging, UI updates, state management, wizard logic, etc.). Includes helper functions. Comments explain key parts.

### 3.5. Security

*   **NFR3.5.1:** Relies on the browser's Web Serial API user-permission model (user must explicitly grant port access).
*   **NFR3.5.2:** Handles data display using basic HTML escaping (`escapeHtml`). Implements clear security warnings regarding the execution of potentially untrusted code generated via the Script Wizard (FR2.8.12). Uses sandboxed `AsyncFunction` for script execution.

## 4. Future Considerations (Optional)

*   More advanced log filtering/searching.
*   Data charting/visualization.
*   Binary file transfer support.
*   More sophisticated script validation within the wizard.
*   **Interactive panel resizing** allowing users to adjust the width ratio between the left (settings) and right (log) panels. *(Still Optional/Not Implemented)*
