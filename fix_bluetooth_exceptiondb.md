# Fix Stuck Bluetooth Pairing in Windows by Clearing ExceptionDB

When Windows fails to pair a Bluetooth device (e.g., FILCO Majestouch Convertible 2) and no longer detects it, the cause can be a stale entry in the Bluetooth **ExceptionDB**.  
Removing this entry forces Windows to treat the device as new.

---

## ⚠️ Safety Notes

- Editing the registry can break Windows if done incorrectly. Always **backup your registry** first.
- These steps will remove stored "blocked" device addresses, so any device previously in the ExceptionDB will be allowed to pair again.

---

## 1️⃣ Method A — Using `regedit` (Manual)

### Step 1 — Backup the Registry
1. Press **Win + R**, type:
   ```
   regedit
   ```
   and press **Enter**.
2. In Registry Editor, go to **File → Export…**.
3. Set **Export range** to **All** and save the `.reg` file somewhere safe.

### Step 2 — Navigate to ExceptionDB
In Registry Editor, browse to:
```
Computer\HKEY_USERS\.DEFAULT\Software\Microsoft\Windows\CurrentVersion\Bluetooth\ExceptionDB\Addrs
```

### Step 3 — Remove Device Entry
- Each folder under **Addrs** is a Bluetooth MAC address in hex format.
- Right-click the one matching your stuck device → **Delete**.
- If unsure, delete **all** subfolders under `Addrs` (Windows will rebuild them automatically).

### Step 4 — Close & Reboot
- Exit Registry Editor.
- Restart Windows.
- Re-pair the Bluetooth device via **Settings → Bluetooth & devices → Add device**.

---

## 2️⃣ Method B — Using PowerShell (Automatic)

### Step 1 — Open PowerShell as Administrator
- Press **Win + X** → **Windows PowerShell (Admin)** or **Terminal (Admin)**.

### Step 2 — Run the Command
To **delete all ExceptionDB entries**:
```powershell
Remove-Item "HKU\.DEFAULT\Software\Microsoft\Windows\CurrentVersion\Bluetooth\ExceptionDB\Addrs" -Recurse -Force
```

### Step 3 — Restart Windows
- Reboot your PC.
- Put your device into pairing mode and add it in Bluetooth settings.

---

## ℹ️ Why This Works

Windows’ `ExceptionDB` stores “do not pair” or failed pairing records for specific Bluetooth MAC addresses.  
If your device is listed here, Windows will silently refuse to connect again.  
Removing the entry clears that block, allowing pairing as if the device is brand new.

---

**Tested on:**  
- Windows 10 Pro 22H2  
- Windows 11 Pro 23H2  
- Device: FILCO Majestouch Convertible 2
