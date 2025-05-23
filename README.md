# MentalTi
Mentally ill Microsoft-Windows-Threat-Intelligence parser that somehow works for the most part

### Features:
- Select only the events you are interested in
- Select only specific members you want from an event (with a shitty UI)
- Easily usable JSON output
- Almost _full_ event definitions (check `EtwTi.hpp` for more info)
- Symbol resolving for addresses in 64bit KnownDlls (setup `_NT_SYMBOL_PATH` for better results)

### Compiling:
- WDK for KMentalTi
- C++17 for MentalTi
- In `Main.cpp` define the statistics interval (in milliseconds)

### Running:
Since PPL is needed for EtwTi, you need to enable testsigning mode and reboot:
```
bcdedit /set testsigning on
```
After that load the driver (name doesn't matter)
```
sc create random type= kernel binpath= C:\Users\someone\Downloads\KMentalTi.sys
sc start random
```

Specify if you want to monitor all processes with `all`, or specify a PID:

`-proc` flags:
- `<pid>` - Log specified events only from a specific process.
- `all` - Log specified events from all processes. Check `ParseUserKeywords` in `Utils.cpp`, depending on the specified events, event logging is modified for each running process and later created process (some event emissions are disabled unless enabled. Some processes have these enabled, some don't)
- `all-og` - Log specified events from all processes. Don't modify flags, keep them as they are for each process.

```
MentalTi.exe -proc all "0x40 | 0x1000" log.json
MentalTi.exe -proc all-og "0x100000 | 0x200000" log.json
MentalTi.exe -proc 6969 "0x4 | 0x4000" log.json
```
Third argument defines all of the events you are interested in. Full list of available events is displayed when running the program with incorrect amount of arguments.
For each event, select the members you actually want to retrieve:

![image](https://github.com/user-attachments/assets/790a0271-8f21-4548-ba06-bd283bfb8229)

_SPACEBAR to select/deselect member, Arrow UP/DOWN for movement, Enter to confirm and continue_

Fourth argument specifies the log file you wish to write the data to. This can be a relative or full path.
Output is compact, just use `jq` or similar to look at it.

After starting it, console is updated at the defined interval. Events and their log count is displayed:

![image](https://github.com/user-attachments/assets/cd2e009f-8948-436a-b4ba-497a5a90b98d)


### Example output parsing:
1. Detect memory fluctuations
```bash
cat log.json | jq -s '[.[] | select(.Metadata.EventId == 7) | {PID: .Metadata.ProcessId, Base: .Data.BaseAddress, Exe: .Metadata.Exe, Flip: [.Data.ProtectionMask, .Data.LastProtectionMask]}] | group_by(.Base) | map(. as $group | {ProcessId: $group[0].PID, BaseAddress: $group[0].Base, Exe: $group[0].Exe, Flips: (reduce $group[] as $item ([]; if length == 0 or .[-1][0] != $item.Flip[1] then . + [$item.Flip] else . end) | length - 1)} | select(.Flips > 3))'
```
Example output:
```json
[
  {
    "ProcessId": 6928,
    "BaseAddress": [
      "0x2316bac0000"
    ],
    "Exe": "C:\\ProgramData\\Microsoft\\Windows Defender\\Platform\\4.18.25010.11-0\\MsMpEng.exe",
    "Flips": 4
  },
  {
    "ProcessId": 25052,
    "BaseAddress": [
      "0x7ff7cad80000"
    ],
    "Exe": "C:\\Windows\\Temp\\defnotabeacon.exe",
    "Flips": 14
  }
]
```

2. Detect RW -> RX remote protection change
```bash
cat log.json | jq -s '[.[] | select(.Metadata.EventId == 2 and .Data.LastProtectionMask == 4 and .Data.ProtectionMask == 32)]'
```
Example output:
```json
[
  {
    "Data": {
      "BaseAddress": [
        "0x20017950000"
      ],
      "CallingProcessId": 13060,
      "LastProtectionMask": 4,
      "ProtectionMask": 32,
      "RegionSize": [
        "0x10000"
      ],
      "TargetProcessId": 7468
    },
    "Metadata": {
      "EventId": 2,
      "Exe": "A:\\random.exe",
      "ProcessId": 13060
    }
  },
  {
    "Data": {
      "BaseAddress": [
        "0x7ffc8f50d000"
      ],
      "CallingProcessId": 7468,
      "LastProtectionMask": 4,
      "ProtectionMask": 32,
      "RegionSize": [
        "0x10"
      ],
      "TargetProcessId": 1084
    },
    "Metadata": {
      "EventId": 2,
      "Exe": "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe",
      "ProcessId": 7468
    }
  }
]
```

3. Detect APC injection (unbacked memory)
```bash
cat log.json | jq -s '[ .[] | select(.Metadata.EventId == 4) | select(.Data.ApcRoutineVadMmfName == 0) | { CallingPID: .Data.CallingProcessId, CallingTID: .Data.CallingThreadId, TargetPID: .Data.TargetProcessId, TargetTID: .Data.TargetThreadId, ApcRoutine: .Data.ApcRoutine }]'
```
Example output:
```json
[
  {
    "CallingPID": 9412,
    "CallingTID": 12552,
    "TargetPID": 13060,
    "TargetTID": 9388,
    "ApcRoutine": [
      "0x1bdf95e0000"
    ]
  }
]
```

4. Detect Remote Writes to Addresses in 64bit KnownDlls
```bash
cat log.json | jq -s '[ .[] | select(.Metadata.EventId == 14) | select(.Data.BaseAddress | length > 1)]'
```
Example output:
```json
[
  {
    "Data": {
      "BaseAddress": [
        "0x7ff8c59106c0",
        "ntdll!ZwOpenProcessTokenEx"
      ],
      "BytesCopied": [
        "0x10"
      ],
      "CallingProcessCreateTime": "07:48:16.306",
      "CallingProcessId": 33176,
      "CallingProcessProtection": 0,
      "OperationStatus": 0,
      "TargetProcessCreateTime": "07:49:37.279",
      "TargetProcessId": 35948,
      "TargetProcessProtection": 0
    },
    "Metadata": {
      "EventId": 14,
      "Exe": "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe",
      "ProcessId": 33176
    }
  },
  {
    "Data": {
      "BaseAddress": [
        "0x7ff8c59106a0",
        "ntdll!NtOpenThreadTokenEx"
      ],
      "BytesCopied": [
        "0x10"
      ],
      "CallingProcessCreateTime": "07:48:16.306",
      "CallingProcessId": 33176,
      "CallingProcessProtection": 0,
      "OperationStatus": 0,
      "TargetProcessCreateTime": "07:49:37.279",
      "TargetProcessId": 35948,
      "TargetProcessProtection": 0
    },
    "Metadata": {
      "EventId": 14,
      "Exe": "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe",
      "ProcessId": 33176
    }
  },
  {
    "Data": {
      "BaseAddress": [
        "0x7ff8c59105c0",
        "ntdll!NtMapViewOfSection"
      ],
      "BytesCopied": [
        "0x10"
      ],
      "CallingProcessCreateTime": "07:48:16.306",
      "CallingProcessId": 33176,
      "CallingProcessProtection": 0,
      "OperationStatus": 0,
      "TargetProcessCreateTime": "07:49:37.279",
      "TargetProcessId": 35948,
      "TargetProcessProtection": 0
    },
    "Metadata": {
      "EventId": 14,
      "Exe": "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe",
      "ProcessId": 33176
    }
  },
  {
    "Data": {
      "BaseAddress": [
        "0x7ff8c5910600",
        "ntdll!NtUnmapViewOfSection"
      ],
      "BytesCopied": [
        "0x10"
      ],
      "CallingProcessCreateTime": "07:48:16.306",
      "CallingProcessId": 33176,
      "CallingProcessProtection": 0,
      "OperationStatus": 0,
      "TargetProcessCreateTime": "07:49:37.279",
      "TargetProcessId": 35948,
      "TargetProcessProtection": 0
    },
    "Metadata": {
      "EventId": 14,
      "Exe": "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe",
      "ProcessId": 33176
    }
  }
]

```

### Notes & Issues:
- Single header dependency ([nlohmann/json](https://github.com/nlohmann/json))
- Event ID 4 and 5 (`QUEUE_REMOTE_APC`/`SETTHREAD_CONTEXT_REMOTE`) always has PID as 4 in the event header. Because of this, when you are targeting a specific PID, all of these events will be logged so you don't miss them.
- `IsSandboxedToken` in event 35 (`SYSCALL_USAGE`) not working.
- `PreviousTokenTrustLevel`, `PreviousTokenGroups`, `CurrentTokenTrustLevel` and `CurrentTokenGroups` not included in events 33 and 36 (`IMPERSONATION_UP`/`IMPERSONATION_DOWN`) since they use some X type for those that I can't be bothered to figure out atm.
- Uses buffered io - when trying to take a look at the output while it's running, it might not be valid json, or it might be empty all together. When exiting the program, stuff gets flushed to disk and you get everything.
- Maybe some other issues and edge cases somewhere

### Why:
- Made it because all other parsers are pretty shit (no event data definitions, need to hardcode members, can't select stuff you want etc)
- To lose braincells with ETW and C++
- To get juicy data so that I know how to better blend in, and to write detections for my own EDR
- To make a base for the usermode agent of my EDR


