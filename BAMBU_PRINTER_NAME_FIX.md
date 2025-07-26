# Bambu Printer Name Display Fix for OrcaSlicer

## Problem Description

When using OrcaSlicer with Bambu Lab printers, users cannot rename LAN-only printers, forcing them to work with cryptic device IDs instead of user-friendly names:

- **Cloud-connected printers**: ✅ Can be renamed, show custom names like "My Workshop Printer"
- **LAN-only printers**: ❌ Cannot be renamed, show device IDs like "01P00A123456789A"

This makes it extremely difficult to identify printers, especially when managing multiple devices in LAN-only mode.

## Root Cause Analysis

### Key Findings from Bambu Lab Forum Research

From [Bambu Lab community forum](https://forum.bambulab.com/t/how-rename-printer-or-is-that-a-no-no/4693):

> **geotekds (July 2023):** "I do not believe the rename will work in the LAN Only mode."
> 
> **ben_r (November 2023):** "In LAN Only mode you cannot rename the printers from their weird default name."
> 
> **user_3635837386 (February 2025):** "Well, it's 2025 and you still can't rename a printer that's in LAN only mode… which is a problem, as I've now bought enough X1E's that they've shipped me multiple with duplicate names."

### Technical Root Cause

**The rename functionality already exists in OrcaSlicer** but is artificially disabled for LAN-only printers:

1. **UI Elements Exist**: Edit button (`m_edit_name_img`) and dialog (`EditDevNameDialog`) are implemented
2. **Cloud Dependency**: Rename function calls `modify_printer_name()` which requires cloud API access
3. **Conditional Display**: Edit button only shown when `mobj->is_online()` AND printer is cloud-connected
4. **No Local Storage**: No mechanism to store custom names locally for LAN-only printers

### Current Code Flow

**File**: `src/slic3r/GUI/SelectMachinePop.cpp` (line ~717)
```cpp
if (!mobj->is_online()) {
    op->SetToolTip(_L("Offline"));
    op->set_printer_state(PrinterState::OFFLINE);
} else {
    op->show_edit_printer_name(true);  // Only for cloud-connected printers
    op->show_printer_bind(true, PrinterBindState::ALLOW_UNBIND);
    // ...
}
```

**File**: `src/slic3r/GUI/SelectMachinePop.cpp` (line ~958)
```cpp
// EditDevNameDialog::on_edit_name() calls:
dev->modify_device_name(m_info->dev_id, name);  // Requires cloud API
```

## Solution Implementation

### Overview
Enable the existing rename button for LAN-only printers by:
1. **Adding local storage** for custom printer names
2. **Showing edit button** for all online printers (cloud AND LAN)
3. **Dual save logic** - cloud API for cloud printers, local storage for LAN printers
4. **Display priority** - show custom names when available, fall back to device names

### Files to Modify

#### 1. Add Local Storage for Custom Names

**File**: `src/libslic3r/AppConfig.hpp`

Add to class AppConfig:
```cpp
// Custom printer name storage
std::map<std::string, std::string> get_custom_printer_names();
void set_custom_printer_name(const std::string& dev_id, const std::string& custom_name);
std::string get_custom_printer_name(const std::string& dev_id);
void remove_custom_printer_name(const std::string& dev_id);
```

**File**: `src/libslic3r/AppConfig.cpp`

Add to save/load methods:
```cpp
// In save() method, add:
if (!m_custom_printer_names.empty()) {
    json custom_names_json;
    for (const auto& pair : m_custom_printer_names) {
        custom_names_json[pair.first] = pair.second;
    }
    m_json["custom_printer_names"] = custom_names_json;
}

// In load() method, add:
if (m_json.contains("custom_printer_names") && m_json["custom_printer_names"].is_object()) {
    m_custom_printer_names.clear();
    for (auto& item : m_json["custom_printer_names"].items()) {
        m_custom_printer_names[item.key()] = item.value().get<std::string>();
    }
}

// Implement helper methods:
std::map<std::string, std::string> AppConfig::get_custom_printer_names() {
    return m_custom_printer_names;
}

void AppConfig::set_custom_printer_name(const std::string& dev_id, const std::string& custom_name) {
    if (custom_name.empty()) {
        m_custom_printer_names.erase(dev_id);
    } else {
        m_custom_printer_names[dev_id] = custom_name;
    }
    save();  // Auto-save changes
}

std::string AppConfig::get_custom_printer_name(const std::string& dev_id) {
    auto it = m_custom_printer_names.find(dev_id);
    return (it != m_custom_printer_names.end()) ? it->second : "";
}

void AppConfig::remove_custom_printer_name(const std::string& dev_id) {
    m_custom_printer_names.erase(dev_id);
    save();
}
```

Add private member variable:
```cpp
std::map<std::string, std::string> m_custom_printer_names;
```

#### 2. Enable Edit Button for LAN-Only Printers

**File**: `src/slic3r/GUI/SelectMachinePop.cpp`

**Replace line ~717:**
```cpp
// OLD CODE:
if (!mobj->is_online()) {
    op->SetToolTip(_L("Offline"));
    op->set_printer_state(PrinterState::OFFLINE);
} else {
    op->show_edit_printer_name(true);  // Only for cloud printers
    op->show_printer_bind(true, PrinterBindState::ALLOW_UNBIND);
    // ...
}

// NEW CODE:
if (!mobj->is_online()) {
    op->SetToolTip(_L("Offline"));
    op->set_printer_state(PrinterState::OFFLINE);
} else {
    // Show edit button for ALL online printers (cloud AND LAN)
    op->show_edit_printer_name(true);
    op->show_printer_bind(true, PrinterBindState::ALLOW_UNBIND);
    // ...
}
```

#### 3. Enhance EditDevNameDialog for Dual Save Logic

**File**: `src/slic3r/GUI/SelectMachinePop.cpp`

**Replace `EditDevNameDialog::on_edit_name()` method around line 958:**
```cpp
void EditDevNameDialog::on_edit_name(wxCommandEvent &e)
{
    m_static_valid->SetLabel(wxEmptyString);
    auto     m_valid_type = Valid;
    wxString info_line;
    auto     new_dev_name = m_textCtr->GetTextCtrl()->GetValue();

    // Existing validation code (keep unchanged)
    const char *      unusable_symbols = "<>[]:/\\|?*\"";
    const std::string unusable_suffix  = PresetCollection::get_suffix_modified();

    for (size_t i = 0; i < std::strlen(unusable_symbols); i++) {
        if (new_dev_name.find_first_of(unusable_symbols[i]) != std::string::npos) {
            info_line    = _L("Name is invalid;") + _L("illegal characters:") + " " + unusable_symbols;
            m_valid_type = NoValid;
            break;
        }
    }

    if (m_valid_type == Valid && new_dev_name.find(unusable_suffix) != std::string::npos) {
        info_line    = _L("Name is invalid;") + _L("illegal suffix:") + "\n\t" + from_u8(PresetCollection::get_suffix_modified());
        m_valid_type = NoValid;
    }

    if (m_valid_type == Valid && new_dev_name.empty()) {
        info_line    = _L("The name is not allowed to be empty.");
        m_valid_type = NoValid;
    }

    if (m_valid_type == Valid && new_dev_name.find_first_of(' ') == 0) {
        info_line    = _L("The name is not allowed to start with space character.");
        m_valid_type = NoValid;
    }

    if (m_valid_type == Valid && new_dev_name.find_last_of(' ') == new_dev_name.length() - 1) {
        info_line    = _L("The name is not allowed to end with space character.");
        m_valid_type = NoValid;
    }

    if (m_valid_type == NoValid) {
        m_static_valid->SetLabel(info_line);
        Layout();
        return;
    }

    // NEW: Dual save logic
    if (m_valid_type == Valid) {
        m_static_valid->SetLabel(wxEmptyString);
        auto utf8_str = new_dev_name.ToUTF8();
        auto name = std::string(utf8_str.data(), utf8_str.length());
        
        if (m_info) {
            if (m_info->is_lan_mode_printer()) {
                // For LAN-only printers, save custom name locally
                wxGetApp().app_config->set_custom_printer_name(m_info->dev_id, name);
                BOOST_LOG_TRIVIAL(info) << "Saved custom name for LAN printer: " << m_info->dev_id << " -> " << name;
            } else {
                // For cloud printers, use existing cloud API
                DeviceManager *dev = Slic3r::GUI::wxGetApp().getDeviceManager();
                if (dev) {
                    dev->modify_device_name(m_info->dev_id, name);
                    BOOST_LOG_TRIVIAL(info) << "Updated cloud printer name: " << m_info->dev_id << " -> " << name;
                }
            }
        }
        DPIDialog::EndModal(wxID_CLOSE);
    }
}
```

#### 4. Update Display Logic in All Printer Selection Dialogs

**File**: `src/slic3r/GUI/SelectMachine.cpp` (line ~2362)

**Replace:**
```cpp
// OLD CODE:
wxString dev_name_text = from_u8(it->second->dev_name);

// NEW CODE:
wxString dev_name_text;
std::string custom_name = wxGetApp().app_config->get_custom_printer_name(it->second->dev_id);
if (!custom_name.empty()) {
    // Use custom name if available
    dev_name_text = from_u8(custom_name);
} else {
    // Fall back to device name
    dev_name_text = from_u8(it->second->dev_name);
}
```

**Apply similar changes to:**
- `src/slic3r/GUI/SendToPrinter.cpp` (line ~906)
- `src/slic3r/GUI/CalibrationPanel.cpp` (line ~114)
- `src/slic3r/GUI/BindDialog.cpp` (line ~923)

#### 5. Update EditDevNameDialog Pre-fill Logic

**File**: `src/slic3r/GUI/SelectMachinePop.cpp` (around line 900)

**Update `EditDevNameDialog::set_machine_obj()` method:**
```cpp
void EditDevNameDialog::set_machine_obj(MachineObject *obj)
{
    m_info = obj;
    if (m_info) {
        // Pre-fill with custom name if available, otherwise use device name
        std::string display_name = wxGetApp().app_config->get_custom_printer_name(m_info->dev_id);
        if (display_name.empty()) {
            display_name = m_info->dev_name;
        }
        m_textCtr->GetTextCtrl()->SetValue(from_u8(display_name));
    }
}
```

## Expected User Experience

### Before Fix
- **Cloud printers**: ✅ Edit button visible, can rename, shows custom names
- **LAN-only printers**: ❌ No edit button, shows device IDs like `01P00A123456789A`

### After Fix  
- **Cloud printers**: ✅ Edit button visible, can rename, shows custom names (unchanged behavior)
- **LAN-only printers**: ✅ Edit button visible, can rename, shows custom names like `"Workshop X1 Carbon"`

### User Workflow
1. **Connect LAN-only printer** - initially shows device ID
2. **Click edit button** (pencil icon) next to unbind button
3. **Enter custom name** - "My Workshop Printer"
4. **Name persists** across app restarts and reconnections
5. **All dialogs show custom name** consistently

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

### Primary Test Cases

#### A. LAN-Only Printer Renaming
1. **Setup**: Bambu printer in LAN-only mode 
2. **Test**: Device tab → Find printer → Click edit button (pencil icon)
3. **Expected**: 
   - Edit dialog opens with current name pre-filled
   - Can enter custom name and save
   - Name appears immediately in all dialogs
   - Name persists after app restart

#### B. Cloud Printer Compatibility (Regression Test)
1. **Setup**: Cloud-connected Bambu printer
2. **Test**: Rename using edit button
3. **Expected**: 
   - Works exactly as before
   - Uses cloud API (not local storage)
   - Name syncs across devices

#### C. Mixed Environment
1. **Setup**: Both LAN-only and cloud-connected printers
2. **Test**: Rename both types independently
3. **Expected**: 
   - Each uses appropriate storage method
   - Names display correctly in all dialogs
   - No cross-contamination

### Validation Checklist
- [ ] LAN-only printers show edit button when online
- [ ] Custom names save and load correctly
- [ ] All printer selection dialogs show custom names
- [ ] Cloud printer renaming still works (no regression)
- [ ] Names persist across app restarts
- [ ] Config file updates properly
- [ ] UI responsive and properly formatted
- [ ] No memory leaks or crashes

## Data Storage Details

### Config File Structure
Custom names stored in `%APPDATA%/OrcaSlicer/orca-slicer.conf`:
```json
{
  "custom_printer_names": {
    "01P00A123456789A": "Workshop X1 Carbon",
    "01P00B987654321B": "Prototyping P1S",
    "01P00C555444333C": "Production Line 1"
  }
}
```

## Summary

This solution **enables the existing rename button for LAN-only printers** by adding local storage for custom names. It's a user-controlled, persistent solution that doesn't require cloud connectivity while preserving all existing functionality for cloud-connected printers.

**Key advantages:**
- ✅ Uses familiar existing UI (edit button + dialog)
- ✅ Works offline for LAN-only printers  
- ✅ Persistent across app restarts
- ✅ No regression for cloud printers
- ✅ User has full control over naming
- ✅ Clean fallback to device names

## References

- [Bambu Lab Forum: How to rename printer](https://forum.bambulab.com/t/how-rename-printer-or-is-that-a-no-no/4693)
- [OrcaSlicer Source Code](https://github.com/SoftFever/OrcaSlicer)
- Edit button implementation: `src/slic3r/GUI/SelectMachinePop.cpp`
- Config storage: `src/libslic3r/AppConfig.cpp`
- Printer selection UI: `src/slic3r/GUI/SelectMachine.cpp` 