# Modern Device Provisioning: Autopilot, Entra ID, and the End of Imaging

I used to tolerate imaging. WDS, MDT, SCCM OSD, task sequences that felt like a hostage negotiation — all of it. It worked, sort of, if your definition of *worked* includes praying that drivers, domain join, software deployment, and user state all behaved in the exact same five-minute window.

Modern provisioning is better. Not because Microsoft made it magical. It isn't. It's better because the whole model changes: identity first, policy first, compliance first, then apps and access. That is a much saner way to build a Windows estate.

If you want the condensed portfolio version, I already have a companion project card here: [SCCM to Intune & Autopilot Migration](/projects/sccm-intune-migration.html). Related repos:

- [zero-trust-homelab](https://github.com/mdziegiel/zero-trust-homelab)
- [powershell-scripts](https://github.com/mdziegiel/powershell-scripts)
- [powershell-scripts / User-Provisioning](https://github.com/mdziegiel/powershell-scripts/tree/main/User-Provisioning)

## 1. The problem with imaging

Classic imaging has a few recurring sins:

- You spend time building a “gold image” that is obsolete the minute you finish it.
- Driver injection becomes a special religion.
- OOBE is not really OOBE. It's a pile of scripts pretending to be one.
- The machine is often useful only after someone on the IT side has already touched it three or four times.

The deeper problem is that imaging is artifact-centered. Autopilot is identity-centered.

That matters because the real job is not “build a perfect image.” The real job is “make a device trustworthy, compliant, and ready for a user with as little drama as possible.”

## 2. Prerequisites

Before I even think about shipping Autopilot, I want the boring pieces squared away:

- Microsoft Entra ID P1 or equivalent licensing
- Microsoft Intune licensing
- MDM authority set correctly
- Devices registered in Autopilot
- A clean separation between pilot, pre-production, and production device groups
- Conditional Access ready to consume compliance signals

Microsoft's own Zero Trust guidance is basically the same thesis: require healthy and compliant devices before access is granted. That is the part that actually matters.

Useful references:

- [Plan your Microsoft Entra device deployment](https://learn.microsoft.com/en-us/entra/identity/devices/plan-device-deployment)
- [Windows Autopilot profiles](https://learn.microsoft.com/en-us/autopilot/profiles)
- [Use Conditional Access with Microsoft Intune compliance policies](https://learn.microsoft.com/en-us/intune/device-security/conditional-access-integration/overview)

## 3. Autopilot deployment profile setup

This is where people usually overcomplicate things.

I think about Autopilot in three modes:

| Mode | Best for | Why I use it |
|---|---|---|
| User-driven | Standard employee laptops and desktops | User signs in, policy does the rest, and I don't have to babysit the hardware |
| Self-deploying | Kiosks, shared devices, and very controlled scenarios | No user affinity nonsense; the device provisions itself |
| Pre-provisioning / white glove | When I want IT to stage the device before handoff | Great for reducing first-login pain and letting me verify apps before the user gets it |

Microsoft documents the same general split, and they're right for once. I also like the fact that pre-provisioning and self-deploying are explicitly profile-driven instead of being some weird side quest.

If I had to keep this brutally simple: for normal staff devices, I want user-driven. For shared devices, I want self-deploying. For executive machines or anything that needs to be staged cleanly before handoff, I use pre-provisioning.

Docs:

- [Configure Windows Autopilot profiles](https://learn.microsoft.com/en-us/autopilot/profiles)
- [Windows Autopilot self-deploying mode](https://learn.microsoft.com/en-us/autopilot/self-deploying)
- [Windows Autopilot for pre-provisioned deployment](https://learn.microsoft.com/en-us/autopilot/pre-provision)

## 4. Getting the hardware hash into Autopilot

This part still trips people up because it feels like something that should be automatic and, naturally, isn't.

### Manual collection

For a one-off device, I still like the old reliable route:

```powershell
Install-Script -Name Get-WindowsAutopilotInfo
Get-WindowsAutopilotInfo.ps1 -OutputFile .\AutopilotHWID.csv
```

Microsoft's guidance is straightforward here: the script gives you the hardware hash and serial number, and the CSV can be imported into Autopilot.

### Bulk import with Graph

For real deployments, I prefer to stop pretending I'll only ever touch one device at a time. Bulk import is cleaner.

```powershell
Connect-MgGraph -Scopes "DeviceManagementServiceConfig.ReadWrite.All"

$devices = Import-Csv .\AutopilotHWID.csv

foreach ($device in $devices) {
    New-MgDeviceManagementImportedWindowsAutopilotDeviceIdentity -BodyParameter @{
        serialNumber      = $device.'Device Serial Number'
        productKey         = $device.'Windows Product ID'
        importId           = [guid]::NewGuid().Guid
        hardwareIdentifier = [Convert]::FromBase64String($device.'Hardware Hash')
        groupTag           = 'Corp'
    }
}
```

If you want to verify what came in, the Graph cmdlets are there too:

```powershell
Get-MgDeviceManagementWindowsAutopilotDeviceIdentity
```

Relevant docs:

- [Manually Register Devices with Windows Autopilot](https://learn.microsoft.com/en-us/autopilot/add-devices)
- [Import-MgDeviceManagementImportedWindowsAutopilotDeviceIdentity](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devicemanagement.enrollment/import-mgdevicemanagementimportedwindowsautopilotdeviceidentity?view=graph-powershell-1.0)
- [Get-MgDeviceManagementWindowsAutopilotDeviceIdentity](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devicemanagement.enrollment/get-mgdevicemanagementwindowsautopilotdeviceidentity?view=graph-powershell-1.0)

## 5. Entra ID join vs hybrid join

This is the decision point where a lot of environments get stuck in the past.

| Join type | I use it when | I avoid it when |
|---|---|---|
| Entra ID join | The device can live cloud-native and the app stack doesn't need on-prem Kerberos dependency | The only reason to choose it is fear of changing the old model |
| Hybrid join | I still have an unavoidable legacy dependency on domain auth, old line-of-business behavior, or a server-side control plane that refuses to die | I can honestly move away from it without breaking production |

My bias is simple: if I can make the device Entra ID joined and keep access controlled with Intune + Conditional Access, that's the cleaner design.

Hybrid join still exists because reality is annoying. Some environments have old apps, old auth assumptions, and old vendors who think Kerberos is a personality trait. Fine. Use hybrid when you have to. Just don't pretend it's modern because it has the word *cloud* somewhere in the documentation.

## 6. Intune configuration profiles and compliance

Autopilot without policy is just a nicer way to log into a machine that still isn't managed.

What I want in place:

- Baseline configuration profiles
- BitLocker enforcement
- Defender or endpoint protection settings
- LAPS for local admin recovery
- Wi-Fi / VPN profiles where needed
- Compliance policies that map to actual access decisions

The key idea is this: compliance should not be decorative. It should drive Conditional Access.

If a device is compliant, it gets access. If it isn't, it doesn't. That is the whole point of Zero Trust. Not vibes. Not checkboxes. Access.

References:

- [Step 3. Set up compliance policies for devices with Intune](https://learn.microsoft.com/en-us/security/zero-trust/manage-devices-with-intune-compliance-policies)
- [Step 4. Require healthy and compliant devices with Intune](https://learn.microsoft.com/en-us/security/zero-trust/manage-devices-with-intune-require-compliance)
- [Zero Trust deployment approach with Microsoft Intune](https://learn.microsoft.com/en-us/intune/fundamentals/zero-trust-deployment)

## 7. App deployment

This is where a lot of teams still carry dead habits around like an old tool belt.

If the app is a Windows app and it matters, I package it as Win32. That is the sane route now. Microsoft has made it pretty clear that Win32 is the path forward, and the old Microsoft Store for Business world is dead enough that we can stop talking about it like it's coming back.

What I usually do:

- Package the app with `IntuneWinAppUtil.exe`
- Define proper install and uninstall commands
- Use detection rules that actually prove the app is there
- Mark the app Required when it's critical to the user's first day
- Use Company Portal for optional user-driven installs

Example packaging command:

```powershell
.\IntuneWinAppUtil.exe -c .\Source -s Install.ps1 -o .\Output -q
```

The important part is not the tool. The important part is the discipline:

- silent install support
- reliable detection
- dependency ordering
- no dumb GUI prompts during provisioning

Relevant docs:

- [Win32 App Management in Microsoft Intune](https://learn.microsoft.com/en-us/intune/app-management/deployment/win32)
- [Add and Assign Win32 Apps to Microsoft Intune](https://learn.microsoft.com/en-us/intune/app-management/deployment/add-win32)
- [Prepare a Win32 App to Be Uploaded to Microsoft Intune](https://learn.microsoft.com/en-us/intune/app-management/deployment/create-win32-package)

## 8. OOBE walkthrough

Here's the part the end user actually sees.

On a good day:

1. They unbox the device.
2. They connect to Wi-Fi or Ethernet.
3. They sign in once with their Entra ID account.
4. Autopilot recognizes the device and assigns the right profile.
5. ESP holds the desktop until the required policies and apps are in place.
6. The machine lands on the desktop already branded, compliant, and mostly boring.

That last word is the goal. Boring is good. Boring means I don't have to get involved.

I don't have a trustworthy fleet timing benchmark from your environment, so I am not inventing one here.

[NEEDS INPUT: your usual first-login timing from a pilot Autopilot device, if you want a real-world number in this section]

## 9. Lessons learned and gotchas

This is where the fantasy meets the floor.

### Sync is not instant

In my user provisioning work, I literally wait for sync and poll Graph because identity systems are rarely immediate. The GUI provisioning script waits **300 seconds** for AD Connect sync and then polls Entra ID repeatedly until the user shows up.

That is not a random implementation detail. That is reality.

Relevant source: [New User Provisioning GUI](https://github.com/mdziegiel/powershell-scripts/blob/main/User-Provisioning/NewUserProvisioningGUI.ps1)

### Entra-only changes the auth game

I also have a real example of the opposite problem: a pure Entra-joined device does **not** magically satisfy old Windows Integrated Authentication assumptions.

From the NiceLabel 2019 troubleshooting notes:

- Entra-only devices have no domain machine trust
- the browser can work while the client auth flow still fails
- `whoami /groups` returned nothing useful because there was no domain security context

That is the kind of thing that gets ignored until someone says, “but it works on my machine,” which is usually code for “my machine has the old join type.”

### What I do *not* have a clean source for here

I don't have a vault-captured Autopilot ESP timeout, VPN profile push timing failure, or driver-delivery postmortem that I trust enough to quote as fact.

[NEEDS INPUT: one real ESP timeout or driver-delivery incident from your Autopilot rollout, if you want a concrete war story here]

## 10. Closing

Autopilot is not just a better way to install Windows. It's a better way to think about Windows.

Once you stop centering the image and start centering the identity, the rest of the stack makes more sense:

- Entra ID owns the identity
- Intune owns the policy
- Conditional Access owns the gate
- Win32 packaging owns the ugly legacy apps
- Autopilot owns the first impression

And if the estate is still stuck on imaging because someone is emotionally attached to SCCM task sequences, I get it. I've been there. It was useful. It also belongs in the museum.

The next thing I'm watching is [Autopilot device preparation](https://learn.microsoft.com/en-us/autopilot/device-preparation/faq). That feels like the direction this whole mess is heading, which is probably for the best.

If you want the shorter portfolio version of this story, start here:

- [SCCM to Intune & Autopilot Migration](/projects/sccm-intune-migration.html)

And if you want the infrastructure philosophy behind the rest of my lab, this is the one that matters:

- [zero-trust-homelab](https://github.com/mdziegiel/zero-trust-homelab)
