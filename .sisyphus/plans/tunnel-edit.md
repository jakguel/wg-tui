# WireGuard Tunnel Edit Feature

## TL;DR

> **Quick Summary**: Add Edit functionality to wg-tui that allows modifying tunnel configuration fields (DNS, Address, MTU, etc.) through a wizard-style modal, with automatic tunnel restart for active tunnels.
> 
> **Deliverables**: 
> - Edit modal triggered by keybind (e.g., `E`)
> - Editable fields: Address, DNS, ListenPort, MTU, Endpoint, AllowedIPs, PersistentKeepalive
> - Automatic down/up cycle for active tunnels
> - Config preservation (comments, non-editable fields kept intact)
> 
> **Estimated Effort**: Medium
> **Parallel Execution**: NO - sequential (parser depends on types, UI depends on parser)
> **Critical Path**: Types -> Parser -> Edit Wizard -> UI Integration -> Testing

---

## Context

### Original Request
User wants to edit WireGuard tunnel settings from within the TUI, specifically to change DNS server to use local DNS instead of VPN DNS.

### Interview Summary
**Key Discussions**:
- Edit approach: Structured fields (not raw text or external editor)
- Editable fields: Address, DNS, ListenPort, MTU (Interface) + Endpoint, AllowedIPs, PersistentKeepalive (Peer)
- Active tunnel handling: Automatic down/up cycle
- User profile: Client-only (single peer per config)
- Test strategy: Manual verification (no test infrastructure)

**Research Findings**:
- WireGuard configs stored in `/etc/wireguard/{name}.conf`
- Existing codebase has partial parsing (`parse_interface_value()`) but no full config parser/serializer
- Existing wizard patterns: `NewClientWizard`, `NewServerWizard` in app.rs
- `wg syncconf` only updates Peer sections, NOT Interface params

### Metis Review
**Identified Gaps** (addressed):
- Keybind conflict: `e` is Export -> Use `E` (Shift+e) for Edit
- Config preservation: Line-by-line modification to preserve comments
- SaveConfig=true risk: Read file before down to avoid losing runtime state
- Multi-peer handling: Edit first peer only, ignore additional peers
- Validation level: Match existing wizard level (minimal, non-empty checks)

---

## Work Objectives

### Core Objective
Enable users to edit existing WireGuard tunnel configurations directly in the TUI, focusing on commonly-changed fields like DNS, without requiring external editor access.

### Concrete Deliverables
- `EditTunnelDraft` struct for holding editable field values
- `EditTunnelWizard` with step-by-step field editing
- Config parser to extract current values from .conf file
- Config writer that modifies specific fields while preserving format
- Edit modal UI using existing `render_input()` pattern
- Keybind `E` to open edit modal for selected tunnel

### Definition of Done
- [ ] Pressing `E` on selected tunnel opens Edit modal
- [ ] All editable fields shown with current values pre-filled
- [ ] DNS field can be changed and saved
- [ ] Saved changes persist to `/etc/wireguard/{name}.conf`
- [ ] Active tunnels automatically restart after save
- [ ] Non-editable fields (PrivateKey, etc.) preserved exactly
- [ ] Comments in config file preserved

### Must Have
- Edit modal for selected tunnel
- Pre-fill with current config values
- Save changes to .conf file
- Automatic tunnel restart for active tunnels
- Preserve non-editable fields and comments

### Must NOT Have (Guardrails)
- NO editing of PrivateKey, PublicKey, PresharedKey
- NO editing of Pre/PostUp/Down scripts (too complex)
- NO validation beyond non-empty checks (match existing wizards)
- NO multi-peer editing (only first peer)
- NO tunnel rename feature (name change = file rename, out of scope)
- NO backup/undo functionality
- NO diff view before save
- NO syntax highlighting in edit fields
- NO connectivity testing after DNS change
- NO concurrent edit protection (v1 scope)

---

## Verification Strategy (MANDATORY)

### Test Decision
- **Infrastructure exists**: NO
- **User wants tests**: Manual-only
- **Framework**: none

### Manual Verification Procedures

Each TODO includes automated verification using shell commands:

**For Config Changes** (using Bash grep):
```bash
# Agent runs:
grep "DNS = 192.168.1.1" /etc/wireguard/test-tunnel.conf
# Assert: Returns matching line (exit 0)
```

**For TUI Interaction** (using interactive_bash tmux):
```bash
# Agent runs via tmux:
1. Start: sudo wg-tui
2. Navigate to test tunnel
3. Press 'E' to open edit
4. Verify modal appears
5. Edit field, press Enter
6. Verify save
7. Press 'q' to quit
```

**For Tunnel State** (using Bash):
```bash
# Agent runs:
ip link show test-tunnel 2>/dev/null && echo "UP" || echo "DOWN"
# Assert: Expected state
```

---

## Execution Strategy

### Sequential Execution (No Parallelization)

This feature requires sequential implementation due to dependencies:

```
Task 1: Define EditTunnelDraft struct (types.rs)
    ↓
Task 2: Implement config parser (wireguard.rs)
    ↓
Task 3: Implement config writer (wireguard.rs)
    ↓
Task 4: Create EditTunnelWizard (app.rs)
    ↓
Task 5: Integrate into App state machine (app.rs)
    ↓
Task 6: Add keybind and help (app.rs, ui.rs)
    ↓
Task 7: Manual integration testing
```

### Dependency Matrix

| Task | Depends On | Blocks |
|------|------------|--------|
| 1 | None | 2, 4 |
| 2 | 1 | 3, 4 |
| 3 | 2 | 5 |
| 4 | 1, 2 | 5 |
| 5 | 3, 4 | 6 |
| 6 | 5 | 7 |
| 7 | 6 | None |

---

## TODOs

- [ ] 1. Define EditTunnelDraft and EditWizardStep

  **What to do**:
  - Add `EditTunnelDraft` struct to `types.rs` with fields:
    - `name: String` (read-only, for reference)
    - `address: String`
    - `dns: String`
    - `listen_port: String`
    - `mtu: String`
    - `peer_endpoint: String`
    - `peer_allowed_ips: String`
    - `peer_persistent_keepalive: String`
  - Fields should be `String` to match existing wizard pattern

  **Must NOT do**:
  - Do NOT add PrivateKey, PublicKey, or other sensitive fields
  - Do NOT create separate struct for Peer (single peer assumption)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple struct definition, minimal logic
  - **Skills**: []
    - No special skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Tasks 2, 4
  - **Blocked By**: None (can start immediately)

  **References**:
  - `src/types.rs:30-48` - Follow `NewTunnelDraft` and `NewServerDraft` struct patterns

  **Acceptance Criteria**:
  ```bash
  # Agent runs:
  grep -A 10 "struct EditTunnelDraft" src/types.rs
  # Assert: Returns struct definition with expected fields
  ```

  **Commit**: YES
  - Message: `feat(types): add EditTunnelDraft struct for tunnel editing`
  - Files: `src/types.rs`
  - Pre-commit: `cargo check`

---

- [ ] 2. Implement config file parser

  **What to do**:
  - Add function `parse_tunnel_config(name: &str) -> Result<EditTunnelDraft, Error>` to `wireguard.rs`
  - Read config file from `/etc/wireguard/{name}.conf`
  - Extract all editable fields using existing parsing pattern
  - Parse both `[Interface]` and first `[Peer]` sections
  - Handle missing optional fields gracefully (return empty string)
  
  **Implementation approach**:
  - Follow `parse_interface_value()` pattern (lines 249-271)
  - Add helper for parsing `[Peer]` section values
  - Use `fs::read_to_string()` to read file

  **Must NOT do**:
  - Do NOT parse comments (not needed for editing)
  - Do NOT store raw file content (parse only needed fields)
  - Do NOT parse multiple peers (only first peer)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-low`
    - Reason: Moderate complexity, follows existing patterns
  - **Skills**: []
    - No special skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Tasks 3, 4
  - **Blocked By**: Task 1

  **References**:
  - `src/wireguard.rs:249-271` - `parse_interface_value()` pattern to follow
  - `src/wireguard.rs:216-247` - `parse_interface_addresses()` for multi-value parsing
  - `src/wireguard.rs:589-620` - `parse_peer_allowed_ips()` for peer section parsing

  **Acceptance Criteria**:
  ```bash
  # Pre-condition: Create test config
  echo -e "[Interface]\nPrivateKey = test\nAddress = 10.0.0.2/32\nDNS = 1.1.1.1\n\n[Peer]\nPublicKey = peer\nEndpoint = vpn.example.com:51820\nAllowedIPs = 0.0.0.0/0" | sudo tee /etc/wireguard/test-edit.conf
  
  # Run Rust code to parse:
  cargo build && cargo run -- --parse-test test-edit
  # Note: Will need temporary CLI flag or unit test
  
  # Alternative: Check code compiles
  cargo check
  # Assert: Exit 0
  ```

  **Commit**: YES
  - Message: `feat(wireguard): add parse_tunnel_config for extracting editable fields`
  - Files: `src/wireguard.rs`
  - Pre-commit: `cargo check`

---

- [ ] 3. Implement config file writer with line-by-line modification

  **What to do**:
  - Add function `update_tunnel_config(name: &str, draft: &EditTunnelDraft) -> Result<(), Error>` to `wireguard.rs`
  - Read existing config file line by line
  - For each editable field:
    - If field exists: replace the value
    - If field doesn't exist but draft has value: do NOT add (keep minimal changes)
    - If field exists but draft is empty: keep original (don't remove)
  - Write modified content back to file
  - Preserve comments, blank lines, and non-editable fields exactly

  **Implementation approach**:
  - Track current section (`[Interface]` or `[Peer]`)
  - Use regex or string matching to identify field lines
  - Build modified content line by line
  - Use `fs::write()` to save

  **Must NOT do**:
  - Do NOT add new fields that didn't exist
  - Do NOT remove fields (even if draft value is empty)
  - Do NOT modify comments or formatting
  - Do NOT reorder sections

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Complex line-by-line parsing logic, edge cases
  - **Skills**: []
    - No special skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 5
  - **Blocked By**: Task 2

  **References**:
  - `src/wireguard.rs:216-247` - Section parsing pattern
  - `src/wireguard.rs:457-471` - Config writing pattern in `create_tunnel()`

  **Acceptance Criteria**:
  ```bash
  # Pre-condition: Config has DNS = 1.1.1.1
  grep "DNS = 1.1.1.1" /etc/wireguard/test-edit.conf
  
  # Action: Call update_tunnel_config with DNS = 192.168.1.1
  # (via test or TUI)
  
  # Verification:
  grep "DNS = 192.168.1.1" /etc/wireguard/test-edit.conf
  # Assert: Exit 0 (line found)
  
  # Verify PrivateKey unchanged:
  grep "PrivateKey = test" /etc/wireguard/test-edit.conf
  # Assert: Exit 0 (still present)
  ```

  **Commit**: YES
  - Message: `feat(wireguard): add update_tunnel_config for safe config modification`
  - Files: `src/wireguard.rs`
  - Pre-commit: `cargo check`

---

- [ ] 4. Create EditTunnelWizard with step enum

  **What to do**:
  - Add `EditWizardStep` enum to `app.rs` with variants:
    - `Address`, `Dns`, `ListenPort`, `Mtu`, `PeerEndpoint`, `PeerAllowedIps`, `PeerKeepalive`
  - Add `EditTunnelWizard` struct with:
    - `step: EditWizardStep`
    - `draft: EditTunnelDraft`
    - `tunnel_name: String`
    - `was_active: bool` (to know if restart needed)
  - Implement methods:
    - `new(name: String, draft: EditTunnelDraft, was_active: bool) -> Self`
    - `current_value(&self) -> &str`
    - `current_value_mut(&mut self) -> &mut String`
    - `ui(&self) -> (String, &'static str, Option<String>)` (title, prompt, hint)
    - `advance(&mut self) -> bool` (returns true when finished)

  **Must NOT do**:
  - Do NOT add validation beyond non-empty check patterns from existing wizards
  - Do NOT add PrivateKey or other sensitive steps

  **Recommended Agent Profile**:
  - **Category**: `unspecified-low`
    - Reason: Follow established wizard pattern closely
  - **Skills**: []
    - No special skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 5
  - **Blocked By**: Tasks 1, 2

  **References**:
  - `src/app.rs:949-983` - `ClientWizardStep` enum pattern
  - `src/app.rs:986-1083` - `NewClientWizard` implementation pattern
  - `src/app.rs:1085-1111` - `ServerWizardStep` enum pattern

  **Acceptance Criteria**:
  ```bash
  # Check code compiles with new struct
  cargo check
  # Assert: Exit 0
  
  # Verify struct exists
  grep -A 20 "struct EditTunnelWizard" src/app.rs
  # Assert: Shows struct with step, draft, tunnel_name, was_active fields
  ```

  **Commit**: YES
  - Message: `feat(app): add EditTunnelWizard for step-by-step config editing`
  - Files: `src/app.rs`
  - Pre-commit: `cargo check`

---

- [ ] 5. Integrate EditTunnelWizard into App state machine

  **What to do**:
  - Add `edit_tunnel: Option<EditTunnelWizard>` field to `App` struct
  - Initialize to `None` in `App::new()`
  - Add `consume_edit_tunnel(&mut self, key) -> bool` method:
    - Handle Enter: advance step or finish (save + restart if needed)
    - Handle Esc: cancel edit
    - Handle Backspace/Char: modify current field value
  - On finish:
    - Call `update_tunnel_config()` to save
    - If `was_active`, call `wg_quick("down")` then `wg_quick("up")`
    - Show success/error message
    - Refresh tunnel list
  - Add call to `consume_edit_tunnel()` in `handle_key()` method

  **Must NOT do**:
  - Do NOT use `sync_interface_with_content()` (doesn't work for Interface fields)
  - Do NOT skip the down/up cycle for active tunnels

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Integration with state machine, multiple code changes
  - **Skills**: []
    - No special skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 6
  - **Blocked By**: Tasks 3, 4

  **References**:
  - `src/app.rs:30-48` - App struct fields pattern
  - `src/app.rs:441-497` - `consume_new_tunnel_wizard()` pattern to follow exactly
  - `src/app.rs:179-218` - `handle_key()` consumer chain pattern
  - `src/app.rs:137-147` - `toggle_selected_with_name()` for down/up pattern

  **Acceptance Criteria**:
  ```bash
  # Check compilation
  cargo check
  # Assert: Exit 0
  
  # Verify App struct has edit_tunnel field
  grep "edit_tunnel:" src/app.rs
  # Assert: Shows Option<EditTunnelWizard>
  
  # Verify consume method exists
  grep "fn consume_edit_tunnel" src/app.rs
  # Assert: Function found
  ```

  **Commit**: YES
  - Message: `feat(app): integrate EditTunnelWizard into App state machine`
  - Files: `src/app.rs`
  - Pre-commit: `cargo check`

---

- [ ] 6. Add Edit keybind and update help screen

  **What to do**:
  - In `handle_global_key()`, add case for `KeyCode::Char('E')`:
    - Get selected tunnel
    - Check if tunnel exists
    - Read tunnel state (active/inactive)
    - Parse config with `parse_tunnel_config()`
    - Create `EditTunnelWizard` with parsed draft
    - Set `self.edit_tunnel = Some(wizard)`
  - In `ui.rs` `render_help()`, add entry for Edit:
    - `("E", "Edit tunnel config")`
  - In `draw()`, add rendering for edit wizard (before other modals):
    - Use existing `render_input()` with wizard's `ui()` output

  **Must NOT do**:
  - Do NOT use lowercase 'e' (that's Export)
  - Do NOT create new UI components (use existing render_input)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Small additions following existing patterns
  - **Skills**: []
    - No special skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 7
  - **Blocked By**: Task 5

  **References**:
  - `src/app.rs:571-624` - `handle_global_key()` keybind patterns
  - `src/ui.rs:286-329` - `render_help()` key list
  - `src/app.rs:691-701` - `render_input()` call pattern for wizards

  **Acceptance Criteria**:
  ```bash
  # Verify keybind added
  grep "'E'" src/app.rs
  # Assert: Found in handle_global_key
  
  # Verify help entry
  grep -i "edit" src/ui.rs | grep -i "tunnel"
  # Assert: Found help text
  ```

  **Commit**: YES
  - Message: `feat(app,ui): add Edit keybind (E) and update help screen`
  - Files: `src/app.rs`, `src/ui.rs`
  - Pre-commit: `cargo check`

---

- [ ] 7. Manual integration testing

  **What to do**:
  - Build release binary: `cargo build --release`
  - Create test tunnel with known values
  - Test each scenario:
    1. Edit inactive tunnel, change DNS, verify file changed
    2. Edit active tunnel, change DNS, verify restart happens
    3. Cancel edit (Esc), verify no changes saved
    4. Verify comments preserved in config
    5. Verify PrivateKey unchanged after edit
  
  **Test tunnel setup**:
  ```bash
  sudo bash -c 'cat > /etc/wireguard/test-edit.conf << EOF
  # Test tunnel for editing
  [Interface]
  PrivateKey = test-key-do-not-use
  Address = 10.0.0.2/32
  DNS = 1.1.1.1
  
  [Peer]
  PublicKey = test-peer-key
  AllowedIPs = 0.0.0.0/0
  Endpoint = vpn.test.com:51820
  EOF'
  ```

  **Must NOT do**:
  - Do NOT test with real VPN configs (use test tunnel)
  - Do NOT leave test tunnel in /etc/wireguard after testing

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Manual testing, straightforward verification
  - **Skills**: []
    - No special skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (final task)
  - **Blocks**: None
  - **Blocked By**: Task 6

  **References**:
  - (None - testing procedures only)

  **Acceptance Criteria**:
  ```bash
  # Test 1: Edit DNS of inactive tunnel
  # Pre: test-edit tunnel exists with DNS = 1.1.1.1, tunnel is DOWN
  # Action: Open TUI, select test-edit, press E, change DNS to 192.168.1.1, save
  # Verify:
  grep "DNS = 192.168.1.1" /etc/wireguard/test-edit.conf && echo "PASS" || echo "FAIL"
  
  # Test 2: Verify comments preserved
  grep "# Test tunnel" /etc/wireguard/test-edit.conf && echo "PASS" || echo "FAIL"
  
  # Test 3: Verify PrivateKey unchanged
  grep "PrivateKey = test-key-do-not-use" /etc/wireguard/test-edit.conf && echo "PASS" || echo "FAIL"
  
  # Cleanup:
  sudo rm /etc/wireguard/test-edit.conf
  ```

  **Commit**: NO (testing only)

---

## Commit Strategy

| After Task | Message | Files | Verification |
|------------|---------|-------|--------------|
| 1 | `feat(types): add EditTunnelDraft struct for tunnel editing` | types.rs | cargo check |
| 2 | `feat(wireguard): add parse_tunnel_config for extracting editable fields` | wireguard.rs | cargo check |
| 3 | `feat(wireguard): add update_tunnel_config for safe config modification` | wireguard.rs | cargo check |
| 4 | `feat(app): add EditTunnelWizard for step-by-step config editing` | app.rs | cargo check |
| 5 | `feat(app): integrate EditTunnelWizard into App state machine` | app.rs | cargo check |
| 6 | `feat(app,ui): add Edit keybind (E) and update help screen` | app.rs, ui.rs | cargo check |

---

## Success Criteria

### Verification Commands
```bash
# Build succeeds
cargo build --release
# Expected: Exit 0

# Help shows Edit keybind
cargo run -- --help 2>&1 | head -5  # Just verify it runs
# Expected: Program starts or shows help

# DNS change persists
grep "DNS = 192.168.1.1" /etc/wireguard/test-edit.conf
# Expected: Exit 0 (after editing)
```

### Final Checklist
- [ ] All "Must Have" present:
  - [ ] Edit modal opens with 'E' keybind
  - [ ] All editable fields shown with current values
  - [ ] Changes save to .conf file
  - [ ] Active tunnels restart after save
  - [ ] Non-editable fields preserved
  - [ ] Comments preserved
- [ ] All "Must NOT Have" absent:
  - [ ] No PrivateKey/PublicKey editing
  - [ ] No Pre/PostUp/Down editing
  - [ ] No multi-peer editing
  - [ ] No tunnel rename
  - [ ] No backup/undo
- [ ] All code compiles: `cargo check` passes
