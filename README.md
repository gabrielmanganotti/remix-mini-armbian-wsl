# 🛠️ Armbian Boot on Remix Mini via Windows + WSL2 (FEL Mode)

This repository documents the step-by-step process to revive a **Remix Mini (Allwinner H3)**, forcing the execution of the **Armbian Linux** bootloader directly into the processor's RAM using **Windows 10/11 with WSL2**.

This method is ideal for repurposing old hardware into a lightweight home server (such as a **Pi-hole**), especially when the internal storage (eMMC) is locked in hardware "Read-Only" mode, preventing a standard direct boot from the MicroSD card slot.

This project was inspired by [r4nd3l's revived_remix_mini_pc](https://github.com/r4nd3l/revived_remix_mini_pc?tab=readme-ov-file) repository, with a focus on the Windows & WSL2 environment.

---

## 📌 Hardware Requirements

* 1x Remix Mini
* 1x MicroSD Card (16GB or higher recommended)
* 1x Male-to-Male USB OTG Cable (connected to the PC and the Remix Mini's top USB data port)
* 1x SIM ejection tool or paperclip (to press the recessed Reset button)
* 1x Host PC running Windows 10/11 with virtualization features enabled

---

## 📥 Required Files & Downloads

Before starting, create a project directory on your Windows machine (e.g., `C:\Users\YourUsername\Documents\remix_mini_armbian`) and gather the following components:

1. **sunxi-tools Source:** Available on [GitHub](https://github.com/linux-sunxi/sunxi-tools) (installed via package manager inside WSL).
2. **Armbian OS Image:** [Download here](https://mega.nz/file/50VBCACQ#xCP81v3K2QvWXiz7r8W8Reb4efk2fhbUZ2_tunzrq6M). Flash this image to your MicroSD card using **Rufus**, BalenaEtcher, or the raw Linux `dd` utility.
3. **U-Boot binary (`u-boot-sunxi-with-spl.bin`):** [Download here](https://mega.nz/file/xlkXmYCb#iaTcHRlwDMlfetCnCsdAoo-5bezEHaNHilkekJCbC_w). The specialized SPL and U-Boot combined binary compiled specifically for the Allwinner H3 architecture.

---

## 🏗️ Step 1: Preparing the Windows Host Environment

### 1. Motherboard Virtualization Prerequisite

Since WSL2 relies on Hyper-V features, nested hardware virtualization must be enabled in your BIOS setup.

* *Example for Gigabyte Aorus Motherboards:* Reboot your PC → Enter BIOS → Navigate to **Advanced CPU Settings** → Locate **SVM mode** → Set to **Enabled**.

### 2. Install WSL2 and Ubuntu

Open Windows **PowerShell as Administrator** and install Ubuntu subsystem:

```powershell
wsl --install
```

### 3. Install USBIPD-WIN

By default, WSL2 does not have direct access to physical USB hardware. Use `usbipd-win` to forward the interface:

```powershell
winget install wingetcreate
winget install usbipd
```

*(Restart after installation if necessary.)*

### 4. Put the Remix Mini into FEL Mode

* Disconnect the power cable from the Remix Mini.
* Connect the USB OTG cable to the PC and the top USB port (data port) of the Remix Mini.
* Using a SIM ejection tool, press and hold the physical Reset button inside the small hole on the back (located between the 3.5mm headphone jack and the HDMI port).
* While holding the Reset button, plug the power cable back in.
* Wait about 5 to 8 seconds, then release the Reset button. The device screen will remain completely black.

### 5. Attach the Device via PowerShell

In Administrator PowerShell, list the connected USB devices to find the Remix Mini:

```powershell
usbipd list
```

The output will look similar to this:

```text
Connected:
BUSID  VID:PID    DEVICE                                                        STATE
1-12   1b1c:1b64  USB Input Device                                              Not shared
1-15   03f0:0c8e  USB Input Device                                              Not shared
→ 1-16   1f3a:efe8  Allwinner Technology sunxi SoC OTG connector                  Not shared
2-4    0000:0002  Unknown USB Device (Device Descriptor Request Failed)         Not shared
```

Locate your Remix Mini's VID:PID (e.g., `1f3a:efe8`) (it might show up as `Unknown device` or `Allwinner Technology sunxi SoC OTG connector`). Bind and then attach it to your WSL instance:

```powershell
usbipd bind --busid <BUSID>
usbipd attach --wsl --busid <BUSID>
```

---

## 🐧 Step 2: Configuring WSL2 & Injecting the Bootloader

From this point forward, keep the WSL session authenticated and execute the commands inside your Linux terminal.

1. Open Ubuntu terminal. Verify that the forwarded USB instance is visible to the kernel by checking `lsusb` (it should display the VID:PID).

2. Install `sunxi-tools` to interact with the processor:

   ```bash
   sudo apt update && sudo apt install sunxi-tools -y
   ```

3. Navigate to the Windows directory where you placed the downloaded `u-boot-sunxi-with-spl.bin` file:

   ```bash
   cd /mnt/c/Users/YourUsername/Documents/remix_mini_armbian
   ```

4. Inject the bootloader binary directly into the Remix Mini's RAM:

   ```bash
   sudo sunxi-fel uboot u-boot-sunxi-with-spl.bin
   ```

(The command will execute silently. After, the Remix Mini will exit the black screen, read the inserted MicroSD card, and begin loading Armbian onto your connected display.)

---

## 🛡️ Step 3: Installing Pi-hole on Armbian

Once booted into Armbian, follow these configurations:

1. Map a Static IP Address using:

   ```bash
   sudo armbian-config
   ```

(Go to **Network** → Select your interface → Toggle to **Static** and fill in your home network routing parameters).

2. Refresh your system packages:

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

3. Deploy the automated Pi-hole deployment script:

   ```bash
   curl -sSL https://install.pi-hole.net | bash
   ```

4. Follow the configuration prompts on setup, assign your upstream DNS providers (Cloudflare, Google, etc.), and save the password defined at the end.

---

## ⚠️ Known Limitations & Troubleshooting

### Error: `Input/output error` during write attempts (`dd`)

During adjustments using device cloning tools like `dd` (e.g., `sudo dd if=/dev/zero of=/dev/mmcblk2`), the system blocks or rejects structural flags with device Input/output warnings.

* **Diagnosis:** The factory-installed internal eMMC storage device (`/dev/mmcblk2`) on the Remix Mini has hardware-level permanent write-protection state (Read-Only Mode).
* **Solution:** Because the internal hardware Boot ROM enforces a strict native layout priority check, a cold reboot will force the board back into the stock Android instance. To loop back into Armbian, you must repeat the payload injection step (Steps 1 and 2) whenever the device powers off. Given that a running instance of a Pi-hole server draws such low power, the best solution is to keep the device continuously powered on.

---

## 📝 References & Further Reading

* Inspired by: [r4nd3l/revived_remix_mini_pc](https://github.com/r4nd3l/revived_remix_mini_pc)
* Learn more about Linux byte streams: [The Complete Guide to the dd Command in Linux](https://blog.kubesimplify.com/the-complete-guide-to-the-dd-command-in-linux)
