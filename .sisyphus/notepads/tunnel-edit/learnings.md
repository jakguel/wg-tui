## parse_tunnel_config Implementation

### Parsing Strategy
- Used peer_count to track which [Peer] section we're in (only extract from first peer)
- Increment peer_count when encountering [Peer] section header
- Only extract peer fields when peer_count == 1
- Returns empty strings for missing optional fields (DNS, MTU, ListenPort, PersistentKeepalive)

### Key Patterns
- Section tracking: Check line.starts_with('[') && line.ends_with(']')
- Case-insensitive comparison: .eq_ignore_ascii_case()
- Skip comments: line.starts_with('#') || line.starts_with(';')
- Field extraction: line.split_once('=') then trim both key and value

### Fields Extracted
**Interface section:**
- Address (required for editing)
- DNS (optional)
- ListenPort (optional)
- MTU (optional)

**First Peer section:**
- Endpoint (required for editing)
- AllowedIPs (required for editing)
- PersistentKeepalive (optional)

### Not Extracted (security-sensitive)
- PrivateKey, PublicKey, PresharedKey - not editable in UI

## update_tunnel_config Implementation

### Line-by-Line Modification Approach
- Read file content, iterate line by line
- For each line, determine if it's an editable field in the correct section
- Preserve original lines exactly unless modification is needed
- Join modified lines back with \n and write to file

### Key Implementation Details
1. **Section tracking**: Same pattern as parsing - track current_section and peer_count
2. **Preserve formatting**: Keep key_part (including whitespace before '=') unchanged
3. **Inline comment preservation**: Extract and re-append inline comments (# or ; after value)
4. **Empty draft values**: If draft value is empty, keep original line (don't clear fields)
5. **Non-existent fields**: Don't add new fields - only modify existing ones

### Lifetime Annotation
- `get_draft_value_for_field` returns `Option<&'a str>` tied to `draft`'s lifetime
- Only the draft reference needs the lifetime, section/key are independent

### Editable vs Protected Fields
**Editable in [Interface]:** Address, DNS, ListenPort, MTU
**Editable in [Peer] (first only):** Endpoint, AllowedIPs, PersistentKeepalive
**Protected (never modified):** PrivateKey, PublicKey, PresharedKey, Pre/PostUp/Down, SaveConfig
