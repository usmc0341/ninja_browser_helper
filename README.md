# NinjaOne Browser Helper Script
# Browser Extensions Inventory (NinjaOne Automation)

> **Script:** `getextensions_ninja_automation.ps1`  
> **Purpose:** Enumerate installed browser extensions for Chrome, Edge, and Firefox across local user profiles on Windows, and optionally publish the results into NinjaOne custom fields.

---

## Overview

This script scans a Windows endpoint for installed browser extensions and aggregates the findings per user profile. It produces human‑readable output and can update two NinjaOne custom fields:

- **Multiline** text field (`browserExtensions` by default) containing a formatted list  
- **WYSIWYG** field (`browserExtensionshtml` by default) containing an HTML table

If custom fields aren’t configured or you set them blank, the script will still print results to STDOUT for auditing and troubleshooting.

---

## What it collects

For each detected extension:

- **Browser**: Chrome, Edge, or Firefox  
- **User**: Windows account name  
- **Profile**: Friendly profile name for Chrome/Edge, `N/A` for Firefox  
- **Name**: Extension display name  
- **Extension ID**: Chrome/Edge extension ID or Firefox add‑on ID  
- **Description**: Truncated to 75 characters for readability

> Note: Chrome/Edge profile names are resolved using `Local State`. Firefox scanning targets `*.default-release` profiles.

---

## How it works (high level)

1. **Privilege check**: Exits with error if not running elevated (Administrator).  
2. **Browser presence**: Looks for Chrome, Edge, and Firefox via registry “Uninstall” keys at both machine and user hive scopes.  
3. **User profiles**: Enumerates Windows profiles from `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList` and loads user hives as needed.  
4. **Browser-specific collection**  
   - **Chrome/Edge**: Parses each profile’s `Preferences` JSON under `User Data`, excluding the “System Profile.” Resolves friendly profile names using `Local State`.  
   - **Firefox**: Reads `extensions.json` under `AppData\Roaming\Mozilla\Firefox\Profiles\*.default-release`.  
5. **Output shaping**: Trims descriptions, sorts items, then:  
   - Writes to console  
   - Optionally updates the **Multiline** and/or **WYSIWYG** custom fields with size‑aware truncation
6. **Exit code**: `0` on success, `1` on error.

---

## Requirements

- **OS:** Windows 10 or Windows Server 2016 and newer  
- **PowerShell:** 5.1  
- **Rights:** Local Administrator (required to load user hives and read per‑user data)  
- **NinjaOne:** When used inside NinjaOne, the standard property functions are expected to be available:
  - `Ninja-Property-Set-Piped`
  - `Ninja-Property-Docs-Set`
  - Related helpers the script references internally

---

## Parameters and Ninja variables

### Script parameters
```powershell
param (
    [Alias('CustomFieldName')] [string] $MultilineCustomField = 'browserExtensions',
    [string] $WysiwygCustomField = 'browserExtensionshtml'
)
```

- Set either parameter to an empty string `""` to **disable** writing to that field.
- The script will **error** if both parameters are non‑empty and identical.

### NinjaOne dynamic variables

These environment variables override the parameters when present and not equal to the literal string `"null"`:

- `multilineCustomFieldName`
- `wysiwygCustomFieldName`

This is convenient for configuring field names per organization or policy without modifying the script.

---

## Field size handling

- **Multiline** field: the script keeps output under ~9,500 characters. If exceeded, it adds a truncation notice and removes entries until under the limit.  
- **WYSIWYG** field: the HTML fragment is kept under ~199,500 characters to remain below the CLI’s 200k ceiling. Excess rows are removed after a notice is added.

These safeguards prevent failed updates due to field length constraints.

---

## Usage

### In NinjaOne

1. **Add custom fields**
   - Create a **Multiline** field named `browserExtensions`  
   - Create a **WYSIWYG** field named `browserExtensionshtml`  
   Or choose your own names and set them via script parameters or Ninja variables.

2. **Upload the script** to the library (`getextensions_ninja_automation.ps1`).

3. **Configure variables** (optional)
   - Set `multilineCustomFieldName` and/or `wysiwygCustomFieldName` on the policy, organization, or device as needed.

4. **Run as Administrator** (Ninja’s default elevated context is required).

5. **Review results**
   - Custom fields populate on the device record
   - The Activity Log shows console output and any truncation warnings

### Standalone / local testing

```powershell
# Run with default field names (requires admin)
.\getextensions_ninja_automation.ps1

# Disable custom field updates, print to console only
.\getextensions_ninja_automation.ps1 -MultilineCustomField "" -WysiwygCustomField ""

# Use custom field names
.\getextensions_ninja_automation.ps1 -MultilineCustomField "ExtInventory" -WysiwygCustomField "ExtInventoryHtml"
```

> Make sure you start PowerShell “Run as administrator.”

---

## Sample console output

```
A Google Chrome installation was detected. Searching Chrome for browser extensions...
A Microsoft Edge installation was detected. Searching Microsoft Edge for browser extensions...
A Firefox installation was detected. Searching Firefox for browser extensions...
Browser extensions were detected.

Browser      : Chrome
User         : jdoe
Profile      : Default
Name         : uBlock Origin
Extension ID : cjpalhdlnbpafiamejdnhcphjbkeiagm
Description  : Finally, an efficient wide-spectrum content blocker...

Browser      : Firefox
User         : jdoe
Profile      : N/A
Name         : Bitwarden – Free Password Manager
Extension ID : {446900e4-71c2-419f-a6a7-df9c091e268b}
Description  : Secure and free password manager for all of your devices...
```

---

## Exit codes

- `0` Success (extensions found or not)  
- `1` Failure (not elevated, property update error, or unexpected exception)

---

## Known limitations

- **Firefox profiles**: Only scans `*.default-release`. If your estate uses additional or differently named profiles, extend the profile selector.  
- **Status & version**: The script does not currently record “enabled/disabled” state or version numbers.  
- **Portable or non‑standard installs**: Browsers installed in non‑standard paths may be missed if neither uninstall keys nor expected profile folders exist.  
- **Description length**: Descriptions are trimmed to 75 characters in the output model for readability.

---

## Security notes

- Requires elevation; reads user profile data and loads hives under `HKEY_USERS`.  
- Output destined for WYSIWYG is HTML generated by `ConvertTo-Html -Fragment`. The content originates from local JSON and may include untrusted strings; don’t render that field in contexts where raw HTML could be executed without sanitization.  
- Least privilege isn’t an option here; enumeration across all user profiles necessitates admin.

---

## Troubleshooting

- **“Access Denied. Please run with Administrator privileges.”**  
  Run the script in an elevated context.

- **Custom field update fails**  
  - Verify the field exists and the **type** matches your expectation (Multiline vs WYSIWYG).  
  - Ensure field names are correct if using overrides.  
  - Check for size warnings; the script will auto‑trim but will log if limits were hit.

- **No extensions found**  
  - Confirm user profiles exist and contain browser data.  
  - Validate that Chrome/Edge `User Data` and Firefox `Profiles` folders are present for discovered users.  
  - Ensure the browsers are actually installed; detection relies on Uninstall keys and profile folders.

---

## Performance

Roughly linear in the number of profiles and profiles’ preference files:

- Time complexity: **O(U + P + E)**  
  - U = Windows user profiles  
  - P = Browser profile folders across Chrome/Edge/Firefox  
  - E = Total extensions inspected  
- Memory use: proportional to the number of discovered extensions retained for output.

---

## Changelog

- **Initial release**: Windows 10 / Server 2016+ support; Chrome, Edge, and Firefox inventory; optional Multiline and WYSIWYG custom field updates with size‑aware truncation.

---

## Legal

Use of this script is subject to NinjaOne’s applicable terms. The script is provided “as is,” without warranties. Test in a non‑production scope before wide deployment.

---

## Appendix: Implementation details (for reviewers)

- **Privilege test:** `Test-IsElevated` checks membership in `Administrators`.  
- **User hive loading:** Uses `reg.exe load` to mount per‑user hives at `HKU\<SID>` when needed.  
- **Install detection:** `Find-InstallKey` searches 32/64‑bit Uninstall registry paths at both machine and user scopes.  
- **Chrome/Edge enumeration:** Reads `Preferences` JSON and `Local State` to map friendly profile names.  
- **Firefox enumeration:** Reads `extensions.json` for `*.default-release`.  
- **Field updates:** Internal helper `Set-NinjaProperty` pipes values to `Ninja-Property-Set-Piped` or `Ninja-Property-Docs-Set` and enforces conservative character ceilings (≈9,500 for Multiline, ≈199,500 for WYSIWYG).

---

## Appendix: SRS checklist (starter outline)

1. **Draft SRS**  
   - Purpose, scope, definitions, references, assumptions, constraints  
   - Functional requirements: enumeration logic per browser, output shaping, custom field updates, error handling  
   - Non‑functional requirements: performance bounds, security, portability, observability

2. **UML (as‑built)**
   - **Use case**: “Inventory browser extensions,” “Publish to custom fields”  
   - **Class/structural**: Script modules/functions and data objects (BrowserExtensions item)  
   - **Behavioral**: Flow from privilege check to detection, enumeration, shaping, publish  
   - **Activity**: Per‑browser enumeration flows  
   - **Sequence**: Calls from main process to helpers and property writers  
   - **Data Flow**: Files/registry → parser → in‑memory list → console/custom fields

3. **Implementation Guide**  
   - Deployment in NinjaOne, parameterization, variable overrides, safe defaults

4. **Testing Plan**  
   - Unit-ish: Mock JSON inputs for Chrome/Edge/Firefox parsers  
   - Integration: Devices with single/multiple profiles, missing browsers, large extension counts  
   - Negative: Not elevated, missing fields, field oversize, malformed JSON

---

## Roadmap ideas

- Capture **version** and **enabled/disabled** state  
- Support more Firefox profile patterns beyond `*.default-release`  
- Include **publisher/author** and **homepage** if available  
- Optional CSV/JSON attachment export for offline analysis  
- Toggle to include the **Default** Windows profile if desired
