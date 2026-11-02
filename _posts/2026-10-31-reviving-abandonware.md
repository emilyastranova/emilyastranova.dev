---
title: "Reviving Abandonware: Resurrecting an Access Control System by Changing 5-Bytes"
date: 2026-10-31 16:45:00 -0600
categories: [Reverse Engineering]
tags: [reverse engineering, windows, proxmox, dotnet, access control, abandonware] # TAG names should always be lowercase
author: emilyastranova
description: How I rescued a bricked commercial door access control system from becoming e-waste after I triggerred poorly made anti-piracy controls on software my organization legally paid for.
toc: true
comments: false
---

I help a non-profit organization run their IT and security in my free time as a volunteer gig. Unfortunately, inheriting old IT infrastructure means situations like having to rescue an enterprise physical access control system after a VM migration completely bricked its licensing. When the virtual machine booted up on the new host, the licensing engine detected a hardware change and demanded online activation.

The catch? The software was released when I was still a child, more than a decade ago. The vendor's activation servers have been dead for years, and the manufacturer politely informed me that they would not help re-license the software.

Rather than ripping out thousands of dollars of working physical access controllers and rewiring an entire building, I decided to see what was actually going on under the hood. Here is how I extracted the files from a Proxmox backup without ever booting Windows, reverse-engineered its hybrid native/.NET licensing engine, and fixed the broken hardware check with a surgical **5-byte binary patch**.

## Introduction and Background

CardAccess 3000, originally built by Continental Access (a subsidiary of NAPCO Security Technologies), is the software brain behind commercial badge readers, alarm inputs, and access turnstiles. The core engine is a Windows background service (`HostNT.exe`) that constantly talks to serial and network field panels to grant or deny access when someone swipes a badge.

![Login Screen](assets/cardaccess3000/card_access_3000_login.png)

This wasn't even the first time I had danced with this licensing system. Last year, the software was running on an ancient, physically failing bare-metal Windows server. When I did a physical-to-virtual migration to rescue it from the dying hardware and move it into a Proxmox virtual machine, the hardware mismatch check tripped. Luckily, back then Continental was still operating their licensing servers, and they actually worked with me over email to re-activate the license on the new virtual machine.

At the time, I thought putting it into a VM was the ultimate insurance policy. I assumed running inside a hypervisor would permanently lock the hardware profile into place, no more dying motherboards, no more blown capacitors, and no more hardware change flags.

Upon migrating the VM to a newer Proxmox host during scheduled cluster maintenance, I found out the hard way that this wasn't true at all.

The moment the VM booted up on the new node, everything stopped working.

![Error Screen](assets/cardaccess3000/card_access_3000_error.png)

The management console popped up an error:

> Number of Users has been reached,
> or inactive stations have CCTV, Badging or a Com Server.
> Program execution not allowed.

The service refused to validate its license, effectively preventing me from migrating to a new hypervisor node. We kept this hypervisor node alive solely for this software while I waited on a vendor response to figure out how we could transfer the license again.

When I contacted the vendor this second time around, expecting the same straightforward re-activation as before, I checked what is left of their website post-acquisition and emailed the support address only to receive this error:

![Message blocked](assets/cardaccess3000/card_access_3000_message_blocked.png)

> Your message to **licensing@cicaccess.com** has been blocked. See technical details below for more information.

So I reached out again but this time to NAPCO's Continental tech support line only to be met with:

![Gatekeeping](assets/cardaccess3000/card_access_3000_denied.png)

> *"Please note our support is only provided to licensed security professionals. If you are a user looking for assistance with your product, please contact your local installer for help."*

It has been over a month and I've received no response.

(Also, I *am* a security professional, and I will do what I want with the hardware and software I own.)

I tried calling NAPCO, they forwarded me to Contintental and it made the dead phone line noise. My only assumption is they must be operating as a skeleton crew at this point and really only work with security integrators rather than directly with their customers. As a non-profit, we don't have the money to go to our security integrator every time something like this happens, we need full ownership of our own security hardware/software.

This essentially meant our only option was to rip out the old system, buy expensive new software, and replace perfectly functional access panels. Or... it meant I'd have to break out the disassembler :)

Our organization had a fully paid, perpetual license valid until the year **2115**. I wasn't trying to pirate anything or add unentitled features, I just wanted our legally purchased door controller software to keep working the way it always had.

## Ethics, Legal, and the Right to Repair

Before opening up a disassembler, let's talk about the rules of engagement. Reverse engineering production infrastructure software isn't something you do lightly.

1. **Lawful Ownership:** Our organization purchased a legitimate perpetual license. I wasn't trying to bypass user quotas, increase panel limits, or unlock optional paid add-ons.
2. **Exhausting Legitimate Channels:** You always ask the vendor first. I attempted to ask Continental for relicensing support several times and unfortunately I could not verify that they're even still around. If they are, they didn't want to talk to me.
3. **DMCA Section 1201 Exemptions:** The US Library of Congress explicitly grants anti-circumvention exemptions for computer programs when access to an external verification or activation server has been permanently discontinued, specifically to enable the continued lawful operation of lawfully acquired software.
4. **Preserving Security Boundaries:** The goal was a quick, clean, minimal surgical fix that preserved the cryptographic signatures on the license file while bypassing only the dead hardware comparison. My goal was not to enable someone without a license to pirate this software, and the patch I did create does not do that.

## Getting at the Files Without Booting

### Why You Never Boot a Broken License

When node-locking breaks, your first instinct might be to fire up the VM in safe mode or open the GUI to see what happens. **Don't do this.** If you absolutely must, remove the virtual NIC(s) from the VM configuration.

Software licensing engines (especially commercial ones from the 2000s) often contain anti-tamper routines:

- They decrement hidden trial/grace-period counters stored in isolated storage or registry hives.
- If an activation attempt fails over the network, some routines will actually overwrite or blank out the existing activated license file (`CardAccess.lic`), replacing a recoverable profile hash with an empty string.
- Once overwritten, your original activation state is gone.

I needed to pull the application binaries and the active license file off the disk completely offline, directly from the backup archive.

### Proxmox VMA Archives

Proxmox saves VM snapshots in its proprietary Virtual Machine Archive format (e.g. `backup.vma.zst` or `vzdump-qemu-*.vma.zst`).

Restoring a 70 GiB virtual SCSI disk using Proxmox's built-in `qmrestore` requires a live Proxmox VE cluster with configured LVM or ZFS storage pools. On my Linux workstation at home (not on-site at HQ), neither the `qm` tools nor the standalone `vma` binary were available to me. Furthermore, the backup was 52% sparse zero-data, decompressing 70 GiB of raw disk would eat up massive disk space and I/O.

### Writing a Stream Extractor

I wrote a small Python utility (`vma_extract.py`) to parse the VMA format directly from a `zstd` pipe and stream out a sparse raw disk image:

Usage:

```bash
zstd -dc backup.vma.zst | python3 tools/vma_extract.py work
```

Code:

```python
#!/usr/bin/env python3
"""Extract disk images from a Proxmox VMA backup (reads uncompressed VMA on stdin).

Usage: zstd -dc backup.vma.zst | python3 vma_extract.py OUT_DIR
Writes OUT_DIR/<devname>.raw as sparse raw images.
"""
import os
import struct
import sys

CLUSTER = 65536
BLOCK = 4096
EXTENT_HDR = 512


def read_exact(f, n):
    buf = bytearray()
    while len(buf) < n:
        chunk = f.read(n - len(buf))
        if not chunk:
            if not buf:
                return None
            raise EOFError("truncated VMA stream")
        buf += chunk
    return bytes(buf)


def main():
    out_dir = sys.argv[1]
    os.makedirs(out_dir, exist_ok=True)
    f = sys.stdin.buffer

    # Header: magic(4) version(4) uuid(16) ctime(8) md5(16)
    # blob_off(4) blob_size(4) header_size(4) ...
    hdr = read_exact(f, 60)
    version = struct.unpack(">I", hdr[4:8])[0]
    if hdr[:4] != b"VMA\0" or version != 1:
        sys.exit("not a VMA v1 stream")
    blob_off, blob_size, header_size = struct.unpack(">III", hdr[48:60])
    hdr += read_exact(f, header_size - len(hdr))
    blob = hdr[blob_off:blob_off + blob_size]

    def blob_str(ptr):
        ln = struct.unpack("<H", blob[ptr:ptr + 2])[0]
        return blob[ptr + 2:ptr + 2 + ln].rstrip(b"\0").decode()

    # dev_info[256] starts at offset 4096 in the header, 32 bytes each
    devs = {}
    for i in range(1, 256):
        off = 4096 + i * 32
        name_ptr, _, size = struct.unpack(">IIQ", hdr[off:off + 16])
        if name_ptr:
            name = blob_str(name_ptr)
            path = os.path.join(out_dir, name + ".raw")
            fh = open(path, "wb")
            fh.truncate(size)
            devs[i] = fh
            print(f"device {i}: {name} {size} bytes -> {path}", flush=True)

    done = 0
    while True:
        eh = read_exact(f, EXTENT_HDR)
        if eh is None:
            break
        if eh[:4] != b"VMAE":
            sys.exit(f"bad extent magic at {done}")
        for i in range(59):
            bi = struct.unpack(">Q", eh[40 + i * 8:48 + i * 8])[0]
            if bi == 0:
                continue
            mask = (bi >> 48) & 0xFFFF
            dev = (bi >> 32) & 0xFF
            cluster = bi & 0xFFFFFFFF
            fh = devs[dev]
            for b in range(16):
                if mask & (1 << b):
                    data = read_exact(f, BLOCK)
                    fh.seek(cluster * CLUSTER + b * BLOCK)
                    fh.write(data)
            done += 1
            if done % 50000 == 0:
                print(f"{done * 64 // 1024} MiB of clusters processed", flush=True)

    for fh in devs.values():
        fh.close()
    print("done", flush=True)


if __name__ == "__main__":
    main()

```

#### VMA Under the Hood

The Proxmox VMA format is surprisingly clean once you unpack the spec:

- **Header (First 60 bytes):** Starts with magic bytes `VMA\0` followed by a big-endian version `1`, UUID, timestamp, MD5, and string blob table offsets.
- **Device Table (Offset 4096):** Up to 255 device entries (32 bytes each) listing device IDs and raw capacities. The target drive (`drive-scsi1`) clocked in at `75,161,927,680` bytes (70 GiB).
- **Extent Headers (`VMAE` blocks):** Backups are divided into 64 KiB clusters (`CLUSTER = 65536`), made up of sixteen 4 KiB blocks (`BLOCK = 4096`).
- **Bitmasks:** Each 512-byte extent header defines up to 59 64-bit descriptors. A 16-bit bitmask in bits `63..48` specifies which 4 KiB blocks actually contain non-zero data.

By seeking directly to the cluster offsets (`fh.seek(cluster * CLUSTER + b * BLOCK)`) and truncating the sparse raw file, Python let Linux handle the sparse block allocation. The 70 GiB disk was extracted in minutes without wasting space on empty sectors.

Once I had `drive-scsi1.raw`, looking at the partition table showed a hidden sector offset of `718848` (~351 MiB). Carving out that NTFS partition gave me `work/c_drive.ntfs`, from which I extracted:

- `C:\Program Files (x86)\CardAccess3000\` (the application binaries)
- `C:\ProgramData\ContinentalAccess\` (the license and state files)

## Mid-2000s Anti-Piracy Controls

Looking at the extracted files in `extracted/CardAccess3000/`, I found a the expected architecture of really old commercial software:

| Binary | Type | Role |
| --- | --- | --- |
| `CardAccess3000.exe` | Native PE32 (x86) | The main Win32 administrative GUI. |
| `HostNT.exe` | Native PE32 (x86) | The primary service communicating with field panels. |
| `haspms32.dll`, `AKSHASP.dll` | Native PE32 (x86) | Legacy Aladdin HASP USB dongle drivers. |
| `caEncrypt.dll`, `caPassword.dll` | Native PE32 (x86) | Native Win32 encryption helpers. |
| `CardAccess.License.dll` | .NET Assembly | Application-specific licensing interface. |
| `Continental.Licensing.dll` | .NET Assembly | Commercial licensing & copy protection engine. |

The core building control engine (`HostNT.exe`) is written in legacy C/C++. Earlier versions of the software used physical Aladdin HASP dongles plugged into server USB ports. But in later revisions, the vendor bolted on a modern software licensing stack implemented in .NET.

Opening `Continental.Licensing.dll` in a decompiler revealed the truth: Continental Access didn't write this licensing code. It was a rebranded OEM version of **Xheo DeployLX v2.1**, a widely used commercial .NET licensing and anti-piracy framework from the mid-2000s. I tried to find information about it and discovered their website no longer existed. Thankfully, the Internet Archive had [a copy of their page](https://web.archive.org/web/20100701164513/http://xheo.com/products/code-protection):

![Xheo Website](assets/cardaccess3000/xheo_website_archive.png)

Something that caught my eye was their claims of tamper resistance:

> "Tamper-Resistence makes it extremely difficult for hackers to modify the assembly and remove licensing and other restrictions."

We'll see about that :)

## Anatomy of the License and Hardware Checks

Inside `C:\ProgramData\ContinentalAccess\CardAccess.lic`, I found the active license file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Licenses>
  <License signature="XXXXXXX/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX..." version="2.0">
    <Component>*</Component>
    <Expires>2115-XX-XXT00:00:00</Expires>
    <SerialNumber>XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX</SerialNumber>
    <Limits>
      <Limit type="Activation" 
             url="http://www.continentalserialserver.com/Licensing/ContinentalActivationService.asmx" 
             tolerance="3" 
             profileHash="1.1-XXXXXXX-XXXXXXXX-XXXXXXX-XXXX-XXXXXXXX-XXXXXXX-XXXXXXXX-XXXXXXXX-XXXX-XXXXXXXX-XXXXXXXX-XXXX-XXXXXXXX" 
             canActivateByKey="true" />
      <Limit type="Runtime" />
      <Limit type="Registration" ... />
      <Limit type="LicenseServer" allowedFailures="5" />
    </Limits>
    <Values>
      <Value key="PanelCount" value="256" />
      <Value key="ReaderCount" value="2048" />
      <Value key="WorkstationCount" value="5" />
      <Value key="SoftwareCode" value="9" />
    </Values>
  </License>
</Licenses>
```

Look closely at those limits:

- **RSA Digital Signature:** The entire XML blob is signed with an RSA private key. The public key is embedded inside `CardAccess.License.dll` (`CardAccess.License.plsk`). If you edit so much as one character in this XML (like bumping `tolerance="3"` to `tolerance="99"`) DeployLX throws `E_InvalidSignature` and aborts.
- **Hardware Profile Hash:** The `profileHash` attribute holds a cryptographic fingerprint of the machine taken when the license was originally activated in 2015.
- **The Tolerance Check:** Notice `tolerance="3"`. DeployLX allows up to 3 hardware differences before flagging the machine as "unauthorized."

### How DeployLX Calculates Hardware Drift

In `Continental.Licensing/MachineProfile.cs`, DeployLX queries WMI and system APIs to build a profile:

1. Active MAC addresses (`00MacAddress`)
2. CPU microarchitecture and processor count (`00Processors`)
3. Windows Product ID (`00ProductId`)
4. SCSI, IDE, and storage controllers (`0SCSIControllers`, `0IDEControllers`)
5. Motherboard serial number, BIOS UUID, and RAM capacity (`0PhysicalMemory`)

Each component is hashed and concatenated with hyphens: `1.1-<Token1>-<Token2>-...`.

During validation, `MachineProfile.CompareHash()` compares the current host's fingerprint against the stored `profileHash`. Each difference incurs a penalty:

- Mismatched profile version: `+10.0`
- Changed CPU or disk/controller: `+2.0` or `+1.0`
- Changed RAM or peripheral: `+0.25`

If the total penalty exceeds `3.0`, the check fails.

### Why the VM Migration Tripped the Limit

People (including myself before taking on this project) often assume virtualizing a system locks its hardware fingerprint forever. Turns out that's not true.

When the VM moved to the new hypervisor node:

- The underlying physical CPU model changed, exposing different instruction flags to KVM.
- The virtual SCSI bus re-enumerated drive serial numbers.
- The SMBIOS UUID shifted.

The penalty score blew well past `3.0`. DeployLX flagged a "Profile Mismatch", and since `continentalserialserver.com` was offline, online reactivation was impossible.

> **Note:** While writing this blog post, I did discover that `continentalserialserver.com` is back online again.

```shell
emily@pc:~$ nslookup continentalserialserver.com
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   continentalserialserver.com
Address: 167.206.164.235
```

```shell
emily@pc:~$ curl continentalserialserver.com
<!DOCTYPE html>
<html>
<head>
        <title>License Server</title>
</head>
<body>
        <h1>You reached Continental License Server</h1> 
</body>
```

## Decompilation and Finding the Validation Logic

I'll be honest. Originally, I just threw this at an LLM. I know, I know. Trust me, I feel guilty about it. But its 2026, I'm busy, this software is critical for my organization and LLMs have gotten damn good at reverse engineering software. One of the silver linings of the LLM craze has been that they give regular people the ability to take back control of the software landscape, especially in the software presevation space. In an unexpected turn of events, LLMs have made the software playing field just a little bit more level. 2026 has been the year of LLM-assisted reverse engineering. We've gotten a [Mario Kart recompilation](https://github.com/patchzyy/wiicompiled), [Adobe software clean-room reimplementations](https://github.com/storytold/photocraft), and I personally have reimplemented some USB over IP software recently that I'll be making another post about.

The way I heard someone describe this phenomenon was a spin on the [infamous Ida Auken quote](https://en.wikipedia.org/wiki/You%27ll_own_nothing_and_be_happy):

> "You **will** own everything and you'll be happy"

I anticipated some crazy obfuscated mess that was going to take months of my time. That, however, was not at all the case and I should have lowered my expectations for early anti-piracy controls.

After a quick look I saw that this application was written in .NET, one of the easiest languages to reverse engineer. .NET applications are easy to decompile because they compile into Common Intermediate Language (CIL) rather than raw, native machine code. This was going to take 15 minutes in ILSpy to find and patch, I just knew it. To inspect the assemblies on Linux, I set up a batch decompilation pipeline using `ilspycmd`:

```bash
export DOTNET_ROOT=~/.dotnet PATH=~/.dotnet:~/.dotnet/tools:$PATH
mkdir -p decompiled
for a in Continental.Licensing CardAccess.License; do
    ilspycmd -p -o decompiled/$a extracted/CardAccess3000/$a.dll
done
```

This gave me the complete C# source tree.

Next, I checked whether `CardAccess.License.dll` verified the integrity or strong-name signature (Strong-name signing in .NET uses a cryptographic key pair to give an assembly a unique identity, version, and public key token) of `Continental.Licensing.dll`:

- Checked `CardAccess.License.csproj`: It used a standard `<HintPath>` reference with `PublicKeyToken=null`.
- Grepped for `ComputeHash`, `GetPublicKeyToken`, or anti-tamper checksums: **None.**
- Strong-name signing: Neither assembly was strong-named.

That meant if I modified `Continental.Licensing.dll`, the CLR would load it without complaining.

### Tracing Validation Logic in `ActivationLimit.cs`

Tracing through `Continental.Licensing.ActivationLimit.Validate()`, I found the decision logic:

```csharp
public override bool Validate(LicenseContext context, Type type, object instance)
{
    bool flag = false;
    if (CanActivateByKey && ActivationKey != null && UnlockHash != null && 
        base.License.PublicKey.ValidateActivationUnlockKey(UnlockHash, ActivationKey))
    {
        flag = true;
    }

    // Branch 1: If the license was NEVER activated, force activation
    if ((ProfileHash == null || ProfileHash.Length == 0) && !flag)
    {
        if (!Activate(type))
        {
            base.License.InvalidReason = StaticResourceProvider.CurrentProvider.GetString("E_DidNotActivateOrContinue");
            return false;
        }
        return true;
    }

    // Branch 2: The license WAS previously activated -> Check the hardware
    MachineProfile machineProfile = new MachineProfile();
    if (MachineProfile.CompareHash(ProfileHash, machineProfile.GetComparableHash()) > Tolerance && 
       (ActivationKey == null || UnlockHash == null || MachineProfile.CompareHash(UnlockHash, machineProfile.GetComparableHash()) > Tolerance))
    {
        if (!Activate(type, bProfileMismatch: true))
        {
            base.License.InvalidReason = StaticResourceProvider.CurrentProvider.FormatResourceString("E_ProfileMismatch", base.License.LicensePack.Location);
            base.License.ProfileMismatch = true;
            return false;
        }
        return true;
    }
    return true;
}
```

Notice what happens here:

1. First, it checks if `ProfileHash` is empty. If it's a fresh install that was never activated, it must activate.
2. If `ProfileHash` is already present (meaning this was a legitimate, previously activated license), execution reaches line 20: `MachineProfile machineProfile = new MachineProfile();`.
3. It compares the current machine against the stored hash. If they mismatch, it attempts to call `Activate()`. The activation server fails, `ProfileMismatch` is set to `true`, and the license is rejected.

## Designing the Patch

I could have tried several patching approaches:

- **Patching the XML?** Nope. Breaks the RSA signature.
- **Emulating the Web Service?** Messy. You'd need to mock an ASMX SOAP service, redirect DNS in `hosts`, and reverse the encrypted SOAP response handshake.
- **Cracking `CALicenseManager`?** Incomplete. Native services (`HostNT.exe`) load DeployLX through different hooks.

The cleanest possible solution was to patch `ActivationLimit.Validate()` right after Branch 1 completes. If a license is already activated (`ProfileHash != null`), I simply tell the method to return `true`.

### Inspecting the CIL

Looking at the compiled bytecode for `ActivationLimit.Validate` around offset `IL_00a4`:

```text
// Branch 1 finishes checking for empty profileHash...
IL_009f: ldloc.1
IL_00a0: brtrue.s  IL_00e0

// IL_00a4: Instantiation of MachineProfile
IL_00a4: newobj    instance void Continental.Licensing.MachineProfile::.ctor()
IL_00a9: stloc.2
IL_00aa: ldarg.0
...
```

At file offset `0x44b8` (RVA `0x6208` + 12-byte fat method header + `0x00A4`), there is the 5-byte instruction:

```text
73 e9 03 00 06    ; newobj MachineProfile::.ctor()
```

I could replace those exact 5 bytes with:

```text
17                ; ldc.i4.1 (push boolean true / 1 onto stack)
2a                ; ret      (return immediately)
00                ; nop      (padding)
00                ; nop      (padding)
00                ; nop      (padding)
```

### Why I Went This Route

1. **Zero Shift in Offsets:** The replacement is exactly 5 bytes. No branch offsets change, no exception handler tables are corrupted, and no PE sections need resizing.
2. **Signature Checking Intact:** The RSA signature verification in `ExtendedLicense` happens before this method. Modifying the XML license will still break.
3. **Capacity Limits Preserved:** The 2,048 reader and 256 panel quotas stored in the signed XML are still strictly enforced.
4. **Unactivated Licenses Blocked:** An unactivated license with a blank profile hash still hits Branch 1 and gets rejected.

### Patcher Script (`tools/patch_activation.py`)

I wrote a small Python script to locate the exact method RVA, verify the surrounding fat header and opcodes, and apply the patch safely:

```python
#!/usr/bin/env python3
import hashlib, struct, sys

EXPECTED_SHA256 = "e9d88edeafc125961a75a70e552baac35350228e294de15fd86252d09f73e382"
METHOD_RVA = 0x6208
HEADER_SIZE = 12
IL_OFFSET = 0x00A4
NEW_BYTES = bytes([0x17, 0x2A, 0x00, 0x00, 0x00])  # ldc.i4.1 ; ret ; nop x3

def rva_to_offset(pe, rva):
    pe_off = struct.unpack_from("<I", pe, 0x3C)[0]
    nsec = struct.unpack_from("<H", pe, pe_off + 6)[0]
    opt_size = struct.unpack_from("<H", pe, pe_off + 20)[0]
    sec = pe_off + 24 + opt_size
    for i in range(nsec):
        s = sec + i * 40
        vsize, vaddr, rsize, raddr = struct.unpack_from("<IIII", pe, s + 8)
        if vaddr <= rva < vaddr + max(vsize, rsize):
            return rva - vaddr + raddr
    raise ValueError("RVA not found in section table")

def main():
    src, dst = sys.argv[1], sys.argv[2]
    data = bytearray(open(src, "rb").read())
    if hashlib.sha256(data).hexdigest() != EXPECTED_SHA256:
        sys.exit("Input hash mismatch; refusing to patch unknown DLL")

    hdr = rva_to_offset(data, METHOD_RVA)
    site = hdr + HEADER_SIZE + IL_OFFSET
    if data[site] != 0x73:
        sys.exit(f"Expected 0x73 (newobj) at offset {site:#x}, found {data[site]:#02x}")

    data[site:site + 5] = NEW_BYTES
    open(dst, "wb").write(data)
    print(f"Patched successfully! New SHA-256: {hashlib.sha256(data).hexdigest()}")

if __name__ == "__main__":
    main()
```

Running it against `Continental.Licensing.dll`:

```text
patching file offset 0x44b8: 73e9030006 -> 172a000000
wrote patched/Continental.Licensing.dll sha256=2cb9d1e6c24df497af30e9f20d564a6fa7c177d6395b48b464268417395ae5f8
```

Decompiling the patched binary in ILSpy confirmed that `ActivationLimit.Validate` now cleanly decompiles to:

```csharp
if ((ProfileHash == null || ProfileHash.Length == 0) && !flag)
{
    if (!Activate(type))
    {
        base.License.InvalidReason = StaticResourceProvider.CurrentProvider.GetString("E_DidNotActivateOrContinue");
        return false;
    }
    return true;
}
return true;
```

The entire hardware profile mismatch check was completely gone.

## Verification and Deployment

Now that I had a patch, here are the steps I took to revive the server:

1. **Take a VM Snapshot First:** Take a snapshot of the virtual machine before touching anything.
2. **Stop the Background Services:** Open `services.msc` and stop:
   - `CardAccess HostNT Service`
   - `CardAccess Licensing Service`
   - `CardAccessArchiveService`
   - `CardAccessCICDataService`
   ![Services](assets/cardaccess3000/card_access_3000_services.png)
3. **Swap the Binary:**
   - In `C:\Program Files (x86)\CardAccess3000\`, rename `Continental.Licensing.dll` to `Continental.Licensing.dll.original`.
   - Copy in the patched `Continental.Licensing.dll`.
4. **Verify the License File:**
   - Check `C:\ProgramData\ContinentalAccess\CardAccess.lic`.
   - Ensure `profileHash="1.1-..."` is populated. If failed activation popups wiped or corrupted it, restore it from the backup copy (`CardAccess.lic.from-backup`).
5. **Start Services and Test:**
   - Start `CardAccessCICDataService`, then `CardAccessArchiveService`, `CardAccess Licensing Service` and `CardAccess HostNT Service`.
   - Launch `CardAccess3000.exe`.
   - Log in and observe the error is gone :)

If anything goes wrong, reverting `Continental.Licensing.dll.original` restores the exact pre-patch state.

## Key Takeaways and Lessons for Engineers

- **Cloud Activation is a Business Risk:** Relying on proprietary cloud activation servers for critical infrastructure is an enormous operational liability. When vendors abandon products or get acquired, perfectly operational hardware turns into e-waste unless you have the skills to intervene.
- **Tiny Patches Are Best Patches:** Strip-mining an entire binary or writing crude loader bypasses is sloppy. The best patches change the absolute fewest bytes possible, leaving cryptographic checks and sanity boundaries untouched.
- **Offline Forensics Protects Your Evidence:** Never troubleshoot a failing node-locked license inside a booted Windows OS. One bad reboot can wipe grace-period records or overwrite valid license files. If you must boot the VM, remove the virtual NIC from it's configuration completely.
- **VMs Do See Hardware Changes:** KVM CPU flags, SMBIOS UUIDs, and virtual SCSI controller mappings change between hypervisor nodes. If a legacy licensing engine monitors hardware profiles, a simple migration can trigger a lockout.

## Resources

- **ECMA-335 Common Language Infrastructure (CLI) Specification**:  
  <https://www.ecma-international.org/publications-and-standards/standards/ecma-335/>
- **U.S. Copyright Office DMCA Section 1201 Exemptions (Defunct Server Software)**:  
  <https://www.copyright.gov/1201/>
- **Proxmox Virtual Machine Archive (VMA) Format Specification**:  
  <https://git.proxmox.com/?p=pve-qemu.git;a=blob;f=vma-spec.txt>
- **ILSpy .NET Decompiler**:  
  <https://github.com/icsharpcode/ILSpy>
