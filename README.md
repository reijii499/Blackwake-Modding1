[README.md](https://github.com/user-attachments/files/34219736/README.md)
# Blackwake Modding Guide

### An unofficial technical reference for mod developers

> **Document status:** community research. This is not official
> documentation and is not affiliated with the developers of Blackwake.

This guide collects technical findings that may help with creating
client-side and server-side **Blackwake** mods. It is based on analysis
of the client and server assemblies (`Assembly-CSharp.dll`), scene
files, BepInEx logs, and practical experience developing plugins for
**BepInEx 5**.

## Contents

-   [1. Game Technical Overview](#1-game-technical-overview)
-   [2. Tools and Environment Setup](#2-tools-and-environment-setup)
-   [3. Obfuscation and Compatibility](#3-obfuscation-and-compatibility)
-   [4. Game Class Reference](#4-game-class-reference)
-   [5. Server-Side Modding](#5-server-side-modding)
-   [6. Client-Side Modding](#6-client-side-modding)
-   [7. Known Limitations and
    Pitfalls](#7-known-limitations-and-pitfalls)
-   [8. Recommended Workflow](#8-recommended-workflow)
-   [9. Open Questions](#9-open-questions)
-   [10. Research Sources and
    Limitations](#10-research-sources-and-limitations)

## Evidence labels

  -----------------------------------------------------------------------
  Label                               Meaning
  ----------------------------------- -----------------------------------
  **\[Tested\]**                      Confirmed by running the game or
                                      server, reviewing logs, or
                                      performing a practical test.

  **\[Code analysis\]**               Inferred from assembly or IL
                                      analysis; not yet verified in a
                                      live test.

  **\[Hypothesis\]**                  An assumption that still needs to
                                      be tested.
  -----------------------------------------------------------------------

> **Note:** Obfuscated identifiers, including strings such as
> `òäóæåòîððòä`, belong to a specific game build and may change after an
> update.

------------------------------------------------------------------------

## 1. Game Technical Overview

  -----------------------------------------------------------------------
  Property                            Details
  ----------------------------------- -----------------------------------
  Engine                              Unity **5.4.4** (scenes are
                                      serialized as `5.4.4p3`)
                                      **\[Tested\]**

  Runtime                             Mono, .NET **3.5** profile (BepInEx
                                      reports CLR `3.0.40818`)
                                      **\[Tested\]**

  Platform                            Windows x64, Steam, AppID `420290`
                                      **\[Tested\]**

  Game version                        Servers are tagged `[3.9]a`; the
                                      loading screen contains a 3.88
                                      changelog dated November 1, 2019.
                                      It may be outdated **\[Tested\]**

  Networking                          Custom RPC layer built on top of
                                      Unity (`WakeNetObject`), not
                                      UNET/HLAPI **\[Code analysis\]**

  Server                              Separate dedicated server,
                                      `BlackwakeServer`, distributed
                                      through SteamCMD **\[Tested\]**

  Protection                          Obfuscated member names,
                                      server-side action validation
                                      (BWAC), and a global ban list
                                      **\[Code analysis\]**
  -----------------------------------------------------------------------

### Client directory structure

``` text
Blackwake/
├── Blackwake.exe
├── Blackwake_Data/
│   ├── Managed/
│   │   ├── Assembly-CSharp.dll
│   │   ├── Assembly-CSharp-firstpass.dll
│   │   ├── Assembly-UnityScript*.dll
│   │   ├── UnityEngine*.dll
│   │   └── Rewired*, NSpeex, ...
│   ├── level0, level1, ...       # scenes; level0 is the loading screen
│   ├── sharedassets*.assets
│   ├── resources.assets
│   └── globalgamemanagers*
└── BepInEx/                      # if installed
    ├── core/
    ├── plugins/
    ├── config/
    └── LogOutput.log
```

The server has a similar layout, but uses `BlackwakeServer_Data` instead
of `Blackwake_Data`. The server code also references `Server.cfg`,
`Admin.txt`, `ServerAnnouncements.txt`, and `ServerMessage.txt` **\[Code
analysis\]**.

Besides the game's own code, `Assembly-CSharp.dll` contains third-party
components including **PlayWay Water**, **UltimateFracturing**,
**Amplify Color**, **SE Natural Bloom**, and **SSAOPro**. **Rewired**,
**NSpeex**, and **Steamworks.NET** are supplied as separate assemblies
**\[Code analysis\]**.

## 2. Tools and Environment Setup

### 2.1. Recommended environment

1.  Install **BepInEx 5** in the game's root directory. The tested
    version is `5.4.23.5`. On first launch, `BepInEx/config` and
    `LogOutput.log` should be created.
2.  Use **HarmonyX**, which is bundled with BepInEx 5
    (`BepInEx/core/0Harmony.dll`).
3.  Target `net35` for plugin projects. Reference `BepInEx.dll`,
    `0Harmony.dll`, and `UnityEngine.dll`; add `Assembly-CSharp.dll`,
    `Assembly-CSharp-firstpass.dll`, and `UnityEngine.UI.dll` when
    needed.
4.  The NuGet package `Microsoft.NETFramework.ReferenceAssemblies`
    version `1.0.3` can provide .NET 3.5 reference assemblies.
5.  Put the compiled plugin DLL in `BepInEx/plugins/`.
6.  Build and test the plugin yourself, preferably on a local server.
    You can pass `-p:GameDir="..."` to `dotnet build`.

### 2.2. Inspecting assemblies and game assets

Use **dnSpy**, **ILSpy**, or **dotPeek** to inspect types, fields,
methods, and IL code.

For Unity assets, useful tools include:

-   **AssetStudio** --- browse and extract Unity assets;
-   **UABE** --- inspect and edit Unity asset files.

If a decompiler is unavailable, .NET metadata analysis can still be
useful. You can inspect type lists, method IL, field references, and
call sites.

### 2.3. Finding the relevant code

-   **Search string literals.** Messages such as `"Reached level "`,
    `"Failed to retrieve stats for "`, and `"BWAC: Shot denied for "`
    generally remain readable because obfuscation does not alter string
    literals.
-   **Search public names.** Unity callbacks (`Update`, `OnGUI`) and
    some game identifiers (`PlayerInfo`, `takeDamage`, `requestDamage`,
    `sendChat`, `respawn`) retain their names in the analyzed build
    **\[Code analysis\]**.
-   **Search by signature.** If a member is obfuscated, identify it by
    parameter types. For example, a `WakeNetObject` RPC method has the
    form `(string, <recipient>, object[])`.
-   **Follow Steamworks references.** Look for calls to
    `SteamUserStats`, `SteamFriends`, and other Steamworks.NET types.

## 3. Obfuscation and Compatibility

Obfuscation replaces private and many public member names with random
strings, sometimes containing diacritical characters. Unity callbacks
(`Update`, `Start`, `OnGUI`, `OnEnable`) and several game classes
(`PlayerInfo`, `GameMode`, `ShipHealth`, `Chat`, `WakeNetObject`) retain
their original names.

**Practical implication:** a mod that relies on a specific obfuscated
identifier may break after a game update.

Recommended practices:

-   identify methods and fields by signature whenever possible, rather
    than relying only on names;
-   check that required members exist during plugin initialization;
-   if a member cannot be found, disable the affected feature and write
    a clear message to the log;
-   document the game build your mod targets.

## 4. Game Class Reference

### 4.1. Players and game modes

  -----------------------------------------------------------------------
  Class / component                   Findings
  ----------------------------------- -----------------------------------
  `PlayerInfo`                        Player data includes `team`,
                                      `dead`, `spectating`, `flyspec`,
                                      `isAdmin`, `isMod`, `steamID`
                                      (`ulong`), `ping`, `level`,
                                      `trueScore`, `prestige`, `wno`,
                                      `character`, `playerHealth`,
                                      `gold`, statistics (`statKills`,
                                      `statKD`, `statWins`, etc.), and
                                      appearance fields. **\[Code
                                      analysis\]**

  `GameMode`                          Singleton `GameMode.Instance`;
                                      contains the player list and
                                      methods such as
                                      `getPlayerInfo(...)` and
                                      `getPlayerBySocket(int)`, as well
                                      as `teamParents` and `winning`.

  Game modes                          `TDMS`, `TDMS2v2`,
                                      `TDMSGalleon2v2`, `TDMS64Combo`,
                                      `TDMGalleon1v1`, `CTF1`,
                                      `DynamicConquest`, `Payload`, and
                                      `ChestTeam`. Siege modes are
                                      enabled through the
                                      `allowFortModes` setting.

  Bots                                `BotHandler`, `BotPlayer`,
                                      `BotNode`, and `SyncBot`.
                                      `BotPlayer.Update` calls
                                      `Physics.OverlapSphere` several
                                      times per frame, which may be
                                      expensive.
  -----------------------------------------------------------------------

### 4.2. Health, damage, and ships

  -----------------------------------------------------------------------
  Class                               Findings
  ----------------------------------- -----------------------------------
  `PlayerHealth`                      Methods include `takeDamage(...)`,
                                      `requestDamage(...)`, `die(...)`,
                                      `respawn(...)`, and
                                      `SetOnFire(bool, string, info)`.
                                      Server-side code analysis suggests
                                      `takeDamage` and `SetOnFire` handle
                                      effects, while `requestDamage`
                                      performs the damage-condition
                                      checks. **\[Code analysis\]**

  `ShipHealth`                        Contains the `SailHealth` array,
                                      hull data, and RPC methods
                                      `updateHullDamage`, `setSunk`, and
                                      `setDestroyed`. Observations
                                      indicate the intact-hull fraction
                                      is calculated as
                                      `1 − damage field value`.
                                      **\[Tested\]**

  `ShipMovement`, `Boyancy`,          Components responsible for ship
  `ShipLOD`, `SyncShip`               movement, buoyancy, LOD, and
                                      synchronization.

  `CannonUse`                         Tracks cannon state, its ship
                                      reference, loading stage, and
                                      projectile type (enum). Loading
                                      stage `4` means the cannon is
                                      loaded. Known projectile values:
                                      `10` --- Grape, `11` --- Grap, `23`
                                      --- Cannon, `42` --- Bar, `43` ---
                                      Chain. **\[Tested\]**
  -----------------------------------------------------------------------

### 4.3. Chat and commands

-   `Chat.sendChat(int type, int, string sender, string message, info)`
    runs on the server for every chat message. Types: `0` --- global,
    `1` --- faction, `2` --- squad. **\[Tested\]**
-   Message methods include `Chat.SendChatMessageAll(string)`,
    `SendChatMessageTeam(string, int)`, and
    `SendChatMessageLocal(string)`.
-   Client commands in `Chat.parseCommand` include `/ui`, `/netdebug`,
    and `/suicide`.
-   Administrative commands `/restart`, `/spec`, `/endround`, `/alert`,
    and `/ff` check `isAdmin` on the server. **\[Code analysis\]**
-   Chat lines are stored in `Chat.chatLines` (a `Text` array); the
    input field is `InputField input`.

### 4.4. Networking and RPC

The base networking component is `WakeNetObject`. RPC calls are made
through reflection using the target method name as a string. Examples
include `"setLevel"`, `"SetProfile"`, `"updateSailDamage"`, and
`"requestChestAndCaptures"` **\[Code analysis\]**.

Sending methods accept a method name, a recipient, and an `object[]`
argument array. In the analyzed client build, the ModPack map-change
feature calls the `SetProfile` RPC on `EnvironmentHandler`.

The receiving side uses an `info` structure containing, among other
things, the sender's socket number and network flags.
`GameMode.getPlayerBySocket(socket)` can resolve the corresponding
`PlayerInfo` **\[Tested\]**.

The client sends score and prestige through
`PlayerInfo.setLevel(trueScore, prestige, info)`, while the server
calculates the level through `ScoreHandler`. This suggests that some
data originates on the client and could potentially be spoofed; BWAC
checks are also present on the server **\[Code analysis\]**.

### 4.5. Statistics and rank

-   Player score and prestige are stored through **Steam User Stats**,
    using `SteamUserStats.GetUserStat/SetStat`. The client contains a
    static wrapper with stat-name strings and methods to read and write
    stats. **\[Tested\]**
-   `ScoreHandler` converts score and prestige into a level and badge.
-   `MainMenu` displays the level, badge, and prestige.

### 4.6. User interface

-   The interface uses **uGUI** (`UnityEngine.UI.Text`, `InputField`,
    and `RawImage`).
-   Menu text is stored in `level*` scene files, not exclusively in
    code. Therefore, translating strings through a dictionary only
    covers the strings that have been found. **\[Tested\]**
-   Known UI components include `MainMenu`, `ServerList`, `OptionsMenu`,
    `PlayerOptions`, `Scoreboard2F`, `NamePlates`, `TeamSelect`, and
    `CharacterCustomizationUI`.
-   `ServerList` includes columns for password, server name, mode,
    player count, ping, and Discord.
-   `OptionsMenu` contains Gameplay, Graphics, Key & Mouse, and Sound
    tabs.
-   The loading screen is in `level0`, which also contains English tips
    and the changelog.

### 4.7. Graphics and audio

-   Ocean: **PlayWay Water**; reflections: `BWPlanarReflection`,
    `RealtimeReflection`.
-   Volumetric smoke: `VolumetricSmokeCamera`; weather:
    `WeatherFXHandler`.
-   Post-processing (`OnRenderImage`): `SSAOPro`,
    `SENaturalBloomAndDirtyLens`, `AmplifyColorBase`, `WaterDropsIME`,
    and `UnderwaterIME`.
-   Music is handled by `MusicHandler`, which contains `AudioSource`
    arrays; some of its fields are obfuscated.
-   `Camera.main` is called by many scripts every frame. Caching it may
    be an optimization opportunity, but this has not been measured.
    **\[Hypothesis\]**

## 5. Server-Side Modding

### What has been tested

BepInEx runs on the dedicated server. A practical plugin can:

-   receive chat messages from all channels;
-   identify the sender through the socket and `PlayerInfo`;
-   send global chat messages and private messages through a private
    RPC;
-   track kills, deaths, and wins based on damage and death calls.

### Important considerations

1.  **Administrator permissions.** Check `PlayerInfo.isAdmin` on the
    server. Do not rely on client-side permission claims.
2.  **Client-provided data.** The server receives some data from
    clients, including score, prestige, and name. Do not automatically
    treat it as authoritative.
3.  **Configuration files.** The game itself reads `Server.cfg` and
    `Admin.txt`. Store plugin data separately. For safer writes,
    consider an atomic `.tmp` → `.bak` approach.
4.  **Testing.** Start with a LAN setup and a local dedicated server.

### Features not provided out of the box

No ready-made server commands were found for killing, setting on fire,
or sinking a player/ship by player name. Such functionality would need
to be implemented, with `isAdmin` checked on the server.

Based on the discovered calls, sinking a ship involves
`updateHullDamage`, `setSunk`, and `setDestroyed`. The complete sequence
still needs to be tested on a running server **\[Hypothesis\]**.

## 6. Client-Side Modding

  -----------------------------------------------------------------------
  Feature                             Status
  ----------------------------------- -----------------------------------
  IMGUI menus and overlays (`OnGUI`)  **\[Tested\]** --- used by ModPack

  Player-name override through a      **\[Tested\]**
  `SteamFriends.GetPersonaName` patch 

  UI translation by overriding        **\[Partially tested\]** --- menus
  `Text.m_Text` during                are translated; other strings are
  `OnPopulateMesh`                    added to a dictionary

  Displayed-rank modification through **\[Code analysis\]** --- effect on
  stat read/write patches             other players' view has not been
                                      tested

  Texture replacement through         **\[Code analysis\]** --- untested
  `Texture2D.LoadImage`               

  Custom music through `WWW` and      **\[Code analysis\]** --- untested
  `AudioSource` (OGG/WAV)             

  `Camera.main` caching and graphics  **\[Code analysis\]** --- untested
  settings                            
  -----------------------------------------------------------------------

## 7. Known Limitations and Pitfalls

### 7.1. Technical limitations

1.  **Obfuscation.** Member names may change between game builds.

2.  **Legacy Unity and .NET 3.5.** The environment uses an older .NET
    profile. The original research notes limitations involving
    `string.IsNullOrWhiteSpace`, `Tuple`, `async/await`, parts of LINQ,
    and `Task`. Target C# 7.3 and verify which APIs are actually
    available.

3.  **Unity API differences.** In the analyzed `UnityEngine.dll`,
    `QualitySettings` does not contain `shadows` or `softParticles`.
    Check the DLL you are compiling against; use reflection where
    necessary.

4.  **HarmonyX.** BepInEx 5 uses HarmonyX. In this environment, the
    older five-argument `Harmony.Patch(...)` call is obsolete and causes
    a compilation error. The approach documented in the research is:

    ``` csharp
    harmony.CreateProcessor(method)
        .AddPrefix(prefix)
        .AddPostfix(postfix)
        .Patch();
    ```

    When a method has overloads, resolve it using exact parameter types;
    searching by name alone may result in `Ambiguous matching`.

5.  **Legacy mod loaders.** Some mods, such as `lvlFix.dll`, were built
    for Harmony 1.x (`HarmonyInstance`) and read configuration from
    `Managed/Mods/Configs`. They are not BepInEx plugins. Do not combine
    patches targeting the same method without checking compatibility.

6.  **Fonts and Cyrillic.** The menu font does not include Cyrillic
    characters. Translation therefore requires a fallback font (the
    analyzed mod used Arial). Cyrillic is displayed in player names;
    Cyrillic support in chat has not been confirmed.

7.  **Scene text.** UI strings are stored in `level*` files. They can be
    inspected with AssetStudio or an in-game scanner.

8.  **Cursor and input.** The game locks the cursor. An IMGUI menu must
    unlock the cursor and block gameplay input correctly; otherwise,
    clicks may pass through to the game.

9.  **The game may overwrite values.** Fog, camera distance, and quality
    settings may be reset in `Update` or when the environment profile
    changes. Reapplying settings in `LateUpdate` or before rendering may
    be necessary.

10. **Debugging.** Available debugging information is limited, and
    testing on public servers is risky. Use a local dedicated server.

### 7.2. Networking limitations

1.  **Desynchronization.** A client mod that changes gameplay state
    (positions, damage, ship state) may desynchronize from the server or
    have its changes reverted.
2.  **Server validation.** The server validates actions, distances, and
    teams. A log message such as `BWAC: Shot denied...` is one example.
    Client-side changes not accepted by the server may fail and can
    result in a kick.
3.  **Client trust.** Some data, including score, prestige, and name,
    comes from the client. This does not mean the server accepts every
    value without validation.

### 7.3. Rules and fair play

Respect server and community rules. Do not use mods to disrupt gameplay,
bypass server restrictions, or gain an unfair advantage. When publishing
a mod, clearly state what it does, where it works, and what its
limitations are.

## 8. Recommended Workflow

1.  Install BepInEx on the client and on a local dedicated server.
    Confirm that `LogOutput.log` is created.
2.  Create a minimal `BaseUnityPlugin` plugin that writes a message to
    the log.
3.  Open `Assembly-CSharp.dll` in a decompiler. Find the target class
    and record the method signature.
4.  Add a Harmony Prefix or Postfix. Initially, only log the method
    arguments.
5.  Test with two clients on a local server.
6.  Add configuration through `Config.Bind` and safe fallback behavior
    if the target method cannot be found.
7.  Prepare a README, license, disclaimer, and supported game-version
    information.

### Pre-release checklist

-   [ ] The project targets `net35`.
-   [ ] Game files are not included in the repository.
-   [ ] The plugin has a unique GUID.
-   [ ] Access to obfuscated members is guarded by checks.
-   [ ] Exceptions are handled and logs are not flooded with repeated
    messages.
-   [ ] The mod has been tested on a local server and in a solo session.
-   [ ] The README explains what the mod does and does not do.
-   [ ] Limitations and potential risks are documented, especially those
    involving ranks, names, and statistics.

## 9. Open Questions

1.  How do other players receive the displayed rank: from the client
    through `setLevel`, or does the server read Steam stats itself?
2.  Which server RPCs check `isAdmin` beyond ban, kick, and moderator
    assignment? Is there an existing path for kill, ignite, or sink
    commands?
3.  What is the complete list of UI strings in scenes such as `level1`,
    `level4`, and `sharedassets*`?
4.  Does the chat font support Cyrillic? What are the input field's
    length and character restrictions?
5.  Does texture replacement through `LoadImage` work with the game's
    compressed textures?
6.  Which settings are fully supported by `Server.cfg`? The currently
    identified entries include `compCTB`, `allowFortModes`, and
    bot-related settings.
7.  Which obfuscated identifiers change when the game is updated?

## 10. Research Sources and Limitations

This guide is based on analysis of the client and server assemblies, the
`level0`, `level2`, and `level3` scenes, BepInEx logs, and
plugin-development experience.

The plugin code referenced in the research was not compiled by the
original analyst. Therefore, all findings marked **\[Code analysis\]**
or **\[Hypothesis\]** require additional verification in a running game.

**This is an unofficial community resource.** Blackwake and related
marks belong to their respective owners. This document is not affiliated
with the game's developers.
