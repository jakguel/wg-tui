# Edit Form UI Redesign

## TL;DR

> **Quick Summary**: Replace step-by-step wizard with a modern form-based UI showing all fields at once, like nmtui.
> 
> **Deliverables**: 
> - Form popup with all 7 fields visible
> - Tab/Arrow navigation between fields
> - tui-input for field editing
> - Enter to save, Esc to cancel
> 
> **Estimated Effort**: Medium
> **Parallel Execution**: NO - sequential refactoring
> **Critical Path**: Add dependency -> Create form state -> Render form -> Handle input

---

## Context

### User Feedback
Current wizard (7 separate screens, one field at a time) is **ugly and cumbersome**.
User wants a form-based UI like nmtui - all fields visible at once.

### Current Implementation
- EditTunnelWizard: Step-by-step through 7 fields
- Each field on separate screen
- Must press Enter 7 times to navigate through all

### Desired UI
```
┌─────────── Edit: mein-vpn ───────────┐
│                                       │
│  Address:     [10.0.0.2/32        ]  │
│  DNS:         [192.168.1.1        ]  │ ← active field
│  ListenPort:  [                   ]  │
│  MTU:         [                   ]  │
│                                       │
│  Endpoint:    [vpn.example.com:51820] │
│  AllowedIPs:  [0.0.0.0/0          ]  │
│  Keepalive:   [25                 ]  │
│                                       │
│  [Tab/↑↓] navigate  [Enter] save  [Esc] cancel │
└───────────────────────────────────────┘
```

---

## Work Objectives

### Core Objective
Replace wizard with form-based edit UI that shows all fields simultaneously with Tab/Arrow navigation.

### Concrete Deliverables
- tui-input dependency added to Cargo.toml
- EditFormState struct for managing field focus and input states
- render_edit_form() function in ui.rs
- consume_edit_form() for input handling in app.rs
- Remove old EditTunnelWizard code

### Definition of Done
- [ ] tui-input added as dependency
- [ ] All 7 fields visible in one popup
- [ ] Tab/Shift+Tab navigates between fields
- [ ] Arrow keys navigate between fields
- [ ] Active field is highlighted
- [ ] Enter saves, Esc cancels
- [ ] 't' still toggles tunnel
- [ ] Old wizard code removed

---

## TODOs

- [ ] 1. Add tui-input dependency

  **What to do**:
  - Add `tui-input = "0.10"` to Cargo.toml dependencies
  - Run `cargo check` to verify it builds

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**: NO - Sequential (blocks task 2)

  **Acceptance Criteria**:
  ```bash
  grep "tui-input" Cargo.toml
  cargo check
  ```

  **Commit**: YES
  - Message: `deps: add tui-input for form field editing`

---

- [ ] 2. Create EditFormState struct

  **What to do**:
  - Add EditFormState to app.rs with:
    - `inputs: Vec<tui_input::Input>` (7 fields)
    - `focused_field: usize` (0-6)
    - `tunnel_name: String`
    - `was_active: bool`
  - Add methods:
    - `new(name: String, draft: EditTunnelDraft, was_active: bool) -> Self`
    - `next_field(&mut self)` - focus next field
    - `prev_field(&mut self)` - focus previous field
    - `to_draft(&self) -> EditTunnelDraft` - convert inputs to draft

  **Must NOT do**:
  - Don't remove EditTunnelDraft (still needed for parsing/writing)
  - Don't remove parse_tunnel_config or update_tunnel_config

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**: NO - Sequential

  **References**:
  - tui-input docs: https://docs.rs/tui-input/latest/tui_input/

  **Acceptance Criteria**:
  ```bash
  grep "struct EditFormState" src/app.rs
  cargo check
  ```

  **Commit**: YES
  - Message: `feat(app): add EditFormState for form-based editing`

---

- [ ] 3. Create render_edit_form function

  **What to do**:
  - Add `render_edit_form(f: &mut Frame, state: &EditFormState)` to ui.rs
  - Render centered popup (80% width, 70% height)
  - Render all 7 fields as labeled input boxes
  - Highlight active field with different color/border
  - Show help text at bottom: "[Tab/↑↓] navigate  [Enter] save  [Esc] cancel  [t] toggle"
  - Use tui_input widget for rendering each field

  **Implementation approach**:
  - Use Layout::vertical to split popup into field rows
  - Each field: label (Yellow) + input box (White/Cyan if focused)
  - Active field gets Cyan border

  **Recommended Agent Profile**:
  - **Category**: `visual-engineering`
  - **Skills**: `["frontend-ui-ux"]`

  **Parallelization**: NO - Sequential

  **References**:
  - ui.rs existing render functions (render_input, etc.)
  - tui-input rendering: https://docs.rs/tui-input/latest/tui_input/#rendering

  **Acceptance Criteria**:
  ```bash
  grep "pub fn render_edit_form" src/ui.rs
  cargo check
  ```

  **Commit**: YES
  - Message: `feat(ui): add render_edit_form for multi-field form popup`

---

- [ ] 4. Replace consume_edit_tunnel with consume_edit_form

  **What to do**:
  - In App struct: change `edit_tunnel: Option<EditTunnelWizard>` to `edit_form: Option<EditFormState>`
  - Rename consume_edit_tunnel to consume_edit_form
  - Handle keys:
    - Tab: form.next_field()
    - BackTab (Shift+Tab): form.prev_field()
    - Up: form.prev_field()
    - Down: form.next_field()
    - Enter: save (call update_tunnel_config with form.to_draft()), restart if was_active
    - Esc: cancel (set edit_form = None)
    - Char('t'): toggle tunnel
    - Other keys: pass to tui_input for current field
  - Update draw() to call render_edit_form instead of render_input

  **Must NOT do**:
  - Don't break the 't' toggle functionality
  - Don't remove update_tunnel_config or parse_tunnel_config

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`

  **Parallelization**: NO - Sequential

  **References**:
  - tui-input input handling: https://docs.rs/tui-input/latest/tui_input/struct.Input.html#method.handle_event

  **Acceptance Criteria**:
  ```bash
  grep "edit_form:" src/app.rs
  grep "fn consume_edit_form" src/app.rs
  cargo check
  ```

  **Commit**: YES
  - Message: `feat(app): replace wizard with form-based editing`

---

- [ ] 5. Update Enter keybind to open form

  **What to do**:
  - In handle_global_key, update Enter handler to create EditFormState instead of EditTunnelWizard
  - Parse config with parse_tunnel_config
  - Create EditFormState::new() with parsed draft
  - Set self.edit_form = Some(form)

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**: NO - Sequential

  **Acceptance Criteria**:
  ```bash
  grep "EditFormState::new" src/app.rs
  cargo check
  ```

  **Commit**: YES
  - Message: `refactor(app): use EditFormState in Enter keybind`

---

- [ ] 6. Remove old wizard code

  **What to do**:
  - Remove EditWizardStep enum (lines ~1208-1242)
  - Remove EditTunnelWizard struct and impl (lines ~1245-1329)
  - Verify no references remain

  **Must NOT do**:
  - Don't remove EditTunnelDraft (still used)
  - Don't remove parse_tunnel_config or update_tunnel_config

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**: NO - Sequential

  **Acceptance Criteria**:
  ```bash
  ! grep "EditTunnelWizard" src/app.rs
  ! grep "EditWizardStep" src/app.rs
  cargo check
  ```

  **Commit**: YES
  - Message: `refactor(app): remove old wizard code in favor of form`

---

- [ ] 7. Manual testing

  **What to do**:
  - Build: `cargo build --release`
  - Test form UI:
    - All fields visible
    - Tab navigation works
    - Arrow navigation works
    - Active field highlighted
    - Enter saves
    - Esc cancels
    - 't' toggles

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**: NO - Sequential (final task)

  **Acceptance Criteria**:
  ```bash
  cargo build --release
  # Manual verification
  ```

  **Commit**: NO (testing only)

---

## Success Criteria

- [ ] Form shows all 7 fields at once
- [ ] Tab/Shift+Tab/Arrows navigate between fields
- [ ] Active field is visually distinct
- [ ] Enter saves changes
- [ ] Esc cancels
- [ ] 't' toggles tunnel while in form
- [ ] Code compiles cleanly
