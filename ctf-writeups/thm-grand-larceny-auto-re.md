# Grand Larceny Auto — TryHackMe Reverse Engineering

**Platform:** TryHackMe
**Room:** [grandlarcenyauto](https://tryhackme.com/room/grandlarcenyauto)
**Category:** Reverse Engineering
**Difficulty:** Medium
**Time:** ~60 min
**Date:** 2026-07-26

---

## Description

> Los Vanto's dumbest crime sim. Steal traffic cones, "borrow" hatchbacks, annoy the locals, and outrun the cops. Legend has it that a sealed safehouse vault is tucked away somewhere — a vault the developers made unreachable.

**Objective:**
- Reverse-engineer the Godot 4 game (C#/.NET 8.0) to find how to open the safehouse vault
- Extract the flag

---

## Initial Analysis

### Files

```
GrandLarcenyAuto-windows/
├── GrandLarcenyAuto.exe          — Windows PE Godot engine (irrelevant)
├── GrandLarcenyAuto.pck          — Godot package file (2.3KB — essentially empty, decoy)
└── data_GrandLarcenyAuto_windows_x86_64/
    ├── GodotSharp.dll
    ├── GrandLarcenyAuto.dll      ── 91KB — C# game logic (target)
    └── System.*.dll
```

**Key insight:** The `.pck` is only 2.3KB. In Godot, `.pck` files normally hold all game data/scripts. A tiny one means the game logic is compiled directly into the C# DLL — nothing hidden in engine assets.

### Decompile the .NET assembly

```bash
$ ikdasm GrandLarcenyAuto.dll > grandlarceny.il
# 21,234 lines of CIL — obfuscated with Confuser.Core 1.6.0
```

---

## Investigation

### Step 1 — Find key classes and methods

```bash
$ strings -n 6 GrandLarcenyAuto.dll | grep -iE "vault|safehouse|unlock|stars"
CheckCrimesAndVault
MakeVault
SafehouseVault
TryVault
UnlockStars
vaultDot
vaultDotLabel
vaultPos
```

Key targets identified:

| Method | Class | Purpose |
|--------|-------|---------|
| `TryOpen` | `SafehouseVault` | Decrypts and returns vault contents |
| `DeriveKey` | `CryptoUtil` | SHA256 key derivation from `"GLA::vault::key::v1::stars=" + stars` |
| `Xor` | `CryptoUtil` | Byte-wise XOR decryption |
| `CheckCrimesAndVault` | `GameController` | Tracks wanted stars via in-game crimes |
| `TryVault` | `GameController` | Calls vault.TryOpen(), triggers win if successful |

### Step 2 — Analyze the vault logic

The `SafehouseVault.TryOpen()` method (heavily obfuscated with control-flow flattening):

```
1. Check: if player.WantedStars < 6 → reject ("You need SIX stars")
2. Derive key: SHA256("GLA::vault::key::v1::stars=" + WantedStars)
3. Decrypt: XOR(SealedBlob, key)
4. Return: "VAULT UNSEALED\n" + plaintext
```

The `UnlockStars` constant is `6`.

### Step 3 — The trick: gate vs. crypto mismatch

This is the core puzzle:

| Check | Logic |
|-------|-------|
| Gate condition | `WantedStars >= 6` (unlocks access) |
| Key derivation | `SHA256("...stars=" + WantedStars)` (uses actual value) |

The `SealedBlob` was encrypted using a key derived from **`stars=6`**. But the gate allows **any** value `>= 6`. If you have 7+ stars, the vault appears to open but decrypts to garbage because the XOR key doesn't match.

The intended solution requires exactly **6** wanted stars — no more, no less.

### Step 4 — Extract the flag via reflection

Instead of playing the game to land on exactly 6 stars, we can call `TryOpen()` directly with a `PlayerState` set to 6:

```csharp
// FlagReader/Program.cs
using System;
using GrandLarcenyAuto;

class Program {
    static void Main() {
        PlayerState player = new PlayerState();
        player.WantedStars = 6;
        SafehouseVault vault = new SafehouseVault(player);
        Console.WriteLine(vault.TryOpen());
    }
}
```

```bash
$ dotnet run
# Output:
# VAULT UNSEALED
# THM{h0tf1x3d_my_0wn_w4nt3d_l3v3l}
```

### Step 5 — Red herring: CheatConsole

There's a `CheatConsole.Submit()` method that checks for code `"L0SV4NT0S247"` and returns:

```
THM{ch34t_c0d3s_4r3_f0r_t0ur1sts}
```

This is a **decoy flag**. The actual challenge flag comes from the vault decryption path only.

---

## The Flag

```
THM{h0tf1x3d_my_0wn_w4nt3d_l3v3l}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `ikdasm` (mono-utils) | Decompile .NET DLL to CIL |
| `strings` | Quick enumeration of method/string references |
| `dotnet-sdk-8.0` | .NET runtime to execute the game DLL via reflection |
| IL inspection | Manual analysis of obfuscated switch-based control flow |

---

## Lessons Learned

1. **Check the `.pck` size first.** A tiny `.pck` in Godot = logic is in the DLL, not the engine assets. Saves hours of pointless extraction.
2. **Gate checks vs. crypto keys can diverge.** The vault accepts `>= 6` but the key hard-depends on `== 6`. This is a classic logic bug pattern in real auth/crypto systems too.
3. **Cheat codes aren't always the answer.** `CheatConsole` gives a convincing-looking but wrong flag — always verify against the actual target logic.
4. **Reflection beats reverse engineering.** Once you understand the algorithm, calling the real compiled method directly is faster and more reliable than trying to reproduce in-game state.
5. **Don't trust every string you find.** Confuser obfuscation hides strings behind resolvers, but the underlying data flow (load key → XOR → decode) is still readable once you isolate the method.

---

*Writeup by Kyle — July 2026*