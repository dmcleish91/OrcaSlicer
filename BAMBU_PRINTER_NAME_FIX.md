# Bambu Printer Name Display Fix for OrcaSlicer

## Problem Description

When using OrcaSlicer with Bambu Lab printers, the print dialog shows different name formats depending on the platform and connection mode:

- **macOS**: Shows user-friendly names like "My Bambu Printer"
- **Windows**: Shows device IDs/serial numbers like "01P00A123456789A"

This inconsistency makes it difficult to identify printers, especially when managing multiple devices.

## Root Cause Analysis

### Key Findings from Bambu Lab Forum Research

From [Bambu Lab community forum](https://forum.bambulab.com/t/how-rename-printer-or-is-that-a-no-no/4693):

> **geotekds (July 2023):** "I do not believe the rename will work in the LAN Only mode."
> 
> **ben_r (November 2023):** "In LAN Only mode you cannot rename the printers from their weird default name."
> 
> **user_3635837386 (February 2025):** "Well, it's 2025 and you still can't rename a printer that's in LAN only mode… which is a problem, as I've now bought enough X1E's that they've shipped me multiple with duplicate names."

### Technical Root Cause

The issue occurs in the network discovery process where printer names are retrieved:

**File**: `src/slic3r/GUI/DeviceManager.cpp` (line ~6185)
```cpp
std::string dev_name = j["dev_name"].get<std::string>();
```

**Connection Mode Behavior**:
- **Cloud-connected**: `dev_name` contains user-set friendly names
- **LAN-only mode**: `dev_name` contains raw device identifiers (serial numbers)

### Code Flow Analysis

1. **Discovery**: `DeviceManager::on_machine_alive()` parses JSON and sets `obj->dev_name`
2. **Display**: `SelectMachine.cpp` line 2362 directly displays raw `dev_name`:
   ```cpp
   wxString dev_name_text = from_u8(it->second->dev_name);
   ```
3. **Result**: LAN-only printers show cryptic device IDs instead of friendly names

## Available Data Fields

The `MachineObject` class contains additional fields that can be used for friendly names:

```cpp
// From DeviceManager.hpp
std::string dev_name;           // Raw device name (problematic for LAN-only)
std::string printer_type;       // Model ID (e.g., "C11")  
std::string product_name;       // Product name from discovery
wxString get_printer_type_display_str(); // Formatted display name (e.g., "Bambu Lab X1 Carbon")
```

**JSON Fields Available** (from `DeviceManager.cpp` line ~6690):
- `dev_name` - Raw device identifier
- `dev_model_name` - Model identifier 
- `dev_product_name` - Product name
- `dev_type` - Printer type string

## Solution Implementation

### File to Modify
`src/slic3r/GUI/SelectMachine.cpp`

### Required Changes

#### 1. Add Include Statement
At the top of `SelectMachine.cpp`, add:
```cpp
#include <algorithm>  // for std::all_of
```

#### 2. Replace Line ~2362
**Current code:**
```cpp
wxString dev_name_text = from_u8(it->second->dev_name);
```

**New code:**
```cpp
// Create a user-friendly display name
wxString dev_name_text;
std::string raw_dev_name = it->second->dev_name;

#ifdef _WIN32
// On Windows, if dev_name looks like a device ID (alphanumeric only, >8 chars),
// create a friendlier name using printer type info
if (raw_dev_name.length() > 8 && 
    std::all_of(raw_dev_name.begin(), raw_dev_name.end(), 
    [](char c) { return std::isalnum(c); })) {
    
    // Use printer type display string if available
    wxString printer_type_display = it->second->get_printer_type_display_str();
    if (!printer_type_display.IsEmpty() && printer_type_display != _L("Unknown")) {
        // Show "Printer Type (last 4 chars of ID)"
        std::string short_id = raw_dev_name.length() > 4 ? 
            raw_dev_name.substr(raw_dev_name.length() - 4) : raw_dev_name;
        dev_name_text = wxString::Format("%s (%s)", printer_type_display, short_id);
    } else if (!it->second->product_name.empty()) {
        // Fallback to product name if available
        std::string short_id = raw_dev_name.length() > 4 ? 
            raw_dev_name.substr(raw_dev_name.length() - 4) : raw_dev_name;
        dev_name_text = wxString::Format("%s (%s)", 
            wxString::FromUTF8(it->second->product_name), short_id);
    } else {
        // Last resort: use raw name
        dev_name_text = from_u8(raw_dev_name);
    }
} else {
    // Use raw name if it looks user-friendly already
    dev_name_text = from_u8(raw_dev_name);
}
#else
// On macOS and other platforms, use the raw name as it's already friendly
dev_name_text = from_u8(raw_dev_name);
#endif
```

### Logic Explanation

1. **Detects Windows platform** using `#ifdef _WIN32`
2. **Identifies device ID format** - checks if name is all alphanumeric and >8 characters
3. **Creates friendly names** using:
   - Primary: `get_printer_type_display_str()` (e.g., "Bambu Lab X1 Carbon")
   - Fallback: `product_name` field
   - Last resort: Raw device name
4. **Adds unique identifier** - appends last 4 characters of device ID
5. **Preserves other platforms** - no change for macOS/Linux behavior

### Expected Results

**Before**: `01P00A123456789A`
**After**: `Bambu Lab X1 Carbon (789A)`

## Build Instructions

### Prerequisites
- Visual Studio 2022 with C++ development tools
- CMake
- Git

### Build Steps
```cmd
# From OrcaSlicer root directory
build_release_vs2022.bat

# Run the built executable
cd build/OrcaSlicer
OrcaSlicer.exe
```

## Testing Plan

### Primary Test Case
1. **Setup**: Bambu printer in LAN-only mode showing device ID
2. **Test**: Open modified OrcaSlicer → Device tab → Check printer dropdown
3. **Expected**: See "Bambu Lab X1 Carbon (1234)" instead of device ID

### Test Scenarios

#### A. LAN-Only Printers (Primary Issue)
- ✅ Device IDs become friendly names
- ✅ Unique identifiers preserved for multiple same-model printers
- ✅ "(LAN)" indicator still appears

#### B. Cloud-Connected Printers (Regression Test)
- ✅ Existing user-friendly names unchanged
- ✅ No impact on cloud-connected printer behavior

#### C. Edge Cases
- ✅ Short names (<8 chars) use raw name
- ✅ Names with special characters use raw name
- ✅ Missing printer type/product name falls back gracefully
- ✅ Performance unchanged

### Debug Testing
Add temporary logging to verify code execution:
```cpp
#ifdef _WIN32
    BOOST_LOG_TRIVIAL(info) << "Original dev_name: " << raw_dev_name;
    BOOST_LOG_TRIVIAL(info) << "Formatted name: " << dev_name_text.ToStdString();
    BOOST_LOG_TRIVIAL(info) << "Printer type: " << it->second->get_printer_type_display_str().ToStdString();
#endif
```

### Validation Checklist
- [ ] Print job submission works
- [ ] Printer selection functions correctly  
- [ ] Device monitoring unchanged
- [ ] UI responsive and properly formatted
- [ ] No memory leaks or crashes
- [ ] Startup performance unchanged

## Additional Context

### Related Code Locations

**Other files that use `dev_name` for display**:
- `src/slic3r/GUI/SendToPrinter.cpp` (line ~906)
- `src/slic3r/GUI/CalibrationPanel.cpp` (line ~114)
- `src/slic3r/GUI/SelectMachinePop.cpp` (line ~169)
- `src/slic3r/GUI/BindDialog.cpp` (line ~923)

*Note: May need similar fixes if same issue appears in these dialogs*

### Alternative Name Sources
If `get_printer_type_display_str()` doesn't provide good names, check:
- `printer_type` field mapping in `MachineObject::get_preset_printer_model_name()`
- Direct JSON fields: `dev_model_name`, `dev_product_name`

### Bambu Printer Models
Common models that benefit from this fix:
- X1 Carbon, X1, P1P, P1S, A1 mini
- All show device IDs in LAN-only mode

## Rollback Plan

```cmd
# Backup original file before changes
copy src\slic3r\GUI\SelectMachine.cpp src\slic3r\GUI\SelectMachine.cpp.backup

# Restore if needed
copy src\slic3r\GUI\SelectMachine.cpp.backup src\slic3r\GUI\SelectMachine.cpp
```

## Future Considerations

1. **Upstream Contribution**: Consider submitting this fix to the main OrcaSlicer repository
2. **Bambu Lab Feature Request**: LAN-only printer renaming capability
3. **Configuration Option**: Add user preference for name display format
4. **Additional Dialogs**: Apply similar fixes to other printer selection dialogs

## References

- [Bambu Lab Forum: How to rename printer](https://forum.bambulab.com/t/how-rename-printer-or-is-that-a-no-no/4693)
- [OrcaSlicer Source Code](https://github.com/SoftFever/OrcaSlicer)
- Device discovery implementation: `src/slic3r/GUI/DeviceManager.cpp`
- Printer selection UI: `src/slic3r/GUI/SelectMachine.cpp` 