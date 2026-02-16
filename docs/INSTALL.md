<h1 align="center"> Force Install Guide </h1>

> [!CAUTION]
> Normally this is rather safe and can be reverted easily if it causes a BSOD, however I must say
> Modifying INF files along with forcing drivers to install for unsupported hardware and disabling driver signature enforcement ***may*** come with a risk. **I am not responsible for any bad outcomes. Proceed at your own risk.**

  You have probably attempted random installs of other drivers if you found this, however the issue on why those didn't work was likely because your MT79XX (or similar) driver INF didn't include your specific board's **Subsystem ID** (the `SUBSYS_XXXXXXXX` part). 

  The fix is simple though! Just add your Hardware ID to the INF file and install the driver. That's literally all it takes lmfao

---

<h2 align="center"> Step 1: Find Your Hardware ID </h2>

  - Open **Device Manager** (Windows + R and type: `devmgmt.msc`)
  - Find your WiFi adapter, it will likely be under **Network adapters** 
> [!NOTE]
  > If not check under **Other devices**, it will be listed with a ⚠️ warning icon if no driver is installed
  - Right-click → **Properties** tab → **Details** tab
  - In the **Property** dropdown, select **Hardware Ids**
  - You'll see one or more strings like:
   ```
   PCI\VEN_14C3&DEV_7925&SUBSYS_XXXX1043&REV_01
   PCI\VEN_14C3&DEV_7925&SUBSYS_XXXX1043
   PCI\VEN_14C3&DEV_7925
   ```
  - **Copy the most long string** (usually the first one)
    - Right-click → Copy
    
> [!TIP]
> **Understanding the Hardware ID:**
> - `VEN_14C3` = is the Vendor ID, in this case it is MediaTek
> - `DEV_7925` = is the Device ID or the chipset, in this case it is the MT7925 chipset
> - `SUBSYS_XXXX1043` = is the specific board variant (this is what differs per OEM)
> - `REV_XX` = is the revision (pretty useless honestly, ive never found a driver that rejects based on REV)

<h2 align="center"> Step 2: Edit the INF File </h2>

Here's exactly how I got it working, all you need is a text editor.

- Open `mtkwl6ex.inf` in **Notepad++** or **VS Code**
> [!IMPORTANT] 
> These files are encoded in UTF-16, regular Notepad may cause issues when saving)
- Find the **Win10 MT7925 section** (around line 40-44):
   ```ini
   ;*******************************************************************************************
   ; Mediatek2 Win10 specific entries
   ;*******************************************************************************************
   [MediaTek2.NTAMD64.10.0...16299]
   ; MT7925
   ```
- Add a new line **at the end of the existing MT7925 entries** in that section, using the `DEFAULT` mode config and **your** Hardware ID:
   ```ini
   %MT7925.DeviceDescExA%          = MTK7925_DEFAULT.ndi,   PCI\VEN_14C3&DEV_7925&SUBSYS_XXXX1043
   ```
   Replace `SUBSYS_XXXX1043` with **your actual SUBSYS value** from Step 1.

- Save the file.

> [!TIP]
> If your card is a different chipset (MT7920, MT7921, MT7922, MT7927), add your line to the matching section and use the corresponding install section name (e.g., `MTK7922_MODE9.ndi` for MT7922). If you are in doubt, copy an existing line for the same chipset and just replace the `SUBSYS` portion.


> **For `mtkwecx.inf` (Windows 11):** I don't get why you'd need this but it's the exact aame process, but instead find the `[MediaTek.NTAMD64.10.0...22000]` section instead. 


<h2 align="center"> Step 3: Install the Driver </h2>

### Option A: Device Manager

- Open **Device Manager** (`devmgmt.msc`)
- Right-click your WiFi adapter → **Update driver**
- Select **"Browse my computer for drivers"**
- Select **"Let me pick from a list of available drivers on my computer"**
- Click **"Have Disk..."**
- Click **Browse** and navigate to the folder containing the modified `.inf` file
- Select the `.inf` file → **OK** → **Next**
- If prompted with a security warning about an unsigned driver, click **"Install this driver software anyway"**
- Wait for installation to complete, then reboot

### Option B: pnputil (Command Line)

Open an **Administrator Command Prompt**, navigate to the driver folder, and run:
```cmd
pnputil /add-driver mtkwl6ex.inf /install
```

> [!IMPORTANT]
> ## If Windows Blocks the Install (Driver Signature Enforcement) 
> Editing the INF breaks the digital signature, and Windows *may* refuse to install it. If this happens, you'll need to temporarily disable Driver Signature Enforcement

### Method A: One-Time Boot Option

- Hold **Shift** and click **Restart** from the Start Menu
- Navigate: **Troubleshoot** → **Advanced options** → **Startup Settings** → **Restart**
- Press **7** or **F7** to select **"Disable driver signature enforcement"**
- Windows will boot normally — you now have one session to install the unsigned driver
- Go back to Step 3 and install the driver

### Method B: Enable Test Signing (Persistent)

- Open **Command Prompt** as **Administrator** and run:
```cmd
bcdedit /set testsigning on
```
- Reboot. You'll see a "Test Mode" watermark on your desktop. Install the driver, then turn it off by opening **Command Prompt** as **Administrator**
```cmd
bcdedit /set testsigning off
```

> [!NOTE]
> If **Secure Boot** is enabled in your BIOS, Method B will not work. You'll either need to disable Secure Boot in BIOS or use Method A.

---

<h2 align="center"> Post-Installation Verification </h2>

After rebooting:

1. Open **Device Manager** and confirm your adapter shows under **Network adapters** without any warning icons
2. The device name should appear as **"MediaTek Wi-Fi 7 MT79XX Wireless LAN Card"** (or similar)
3. Check the **Driver** tab to confirm the version matches:
   - `mtkwl6ex`: Version **3.04.00.1304**
   - `mtkwecx`: Version **5.6.0.4093**
4. Try connecting to a WiFi network

---

<h2 align="center"> Troubleshooting </h2>

**Driver installs but WiFi doesn't work / device shows error code 10:**
- Try a different MODE configuration. Edit the INF and change `MTK7925_DEFAULT.ndi` to `MTK7925_MODE1.ndi` (or another available mode for your chipset) and reinstall.

**"Windows cannot verify the digital signature" during install:**
- Make sure you completed Step 3 (disable driver signature enforcement) before attempting installation.

**Device not showing in Device Manager at all:**
- Your WiFi card may be disabled in BIOS or physically disconnected. Check BIOS settings and reseat the M.2/PCIe card.

**INF file won't open correctly / shows garbled text:**
- These INF files are **UTF-16 LE** encoded. Use **Notepad++** or **VS Code** instead of regular Notepad. In Notepad++, check the encoding in the bottom-right status bar.

**Driver was working but stopped after a Windows Update:**
- Windows Update may have replaced the driver. Re-apply the modified driver using the same process. Consider pausing driver updates: Settings → Windows Update → Advanced options → Optional updates.

---

> [!NOTE]
> If your specific hardware doesn't work even after adding the correct Hardware ID, it's possible that your board variant requires a configuration mode that isn't included in these driver packages.

---

> [!IMPORTANT]
> # These drivers are the property of **MediaTek Inc.** and were extracted from publicly available OEM driver packages. 
> ## This repository only redistributes them for convenience and educational purposes.
> ## This is an **unofficial, community-driven** effort and is not affiliated with or endorsed by MediaTek, ASUS, Dell, or any other manufacturer.

---

