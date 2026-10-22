# MOTORSLICE — Research & Build Notes

Game slug: `motorslice`. Session: 2026-10-22.

Mod repository built in this session: <https://github.com/SubmitCodes/motorslice-coop> (`RE-UE4SS` UE 5.7 Lua mod + Python UDP/Steam Spacewar relay, MIT).

## Phase 0 & 1: Triage and Research (2026-10-22)

### Identity (rule 5)
| Field | Value | Source |
|---|---|---|
| Title | MOTORSLICE (2026, Regular Studio / Top Hat Studios) | Steam store appid `2830030`, `appmanifest_2830030.acf` |
| Store | Steam (AppID `2830030`, Depot `2830031`, Manifest `859196545199328247`) | `appmanifest_2830030.acf` |
| Install path | `<game dir>` (owner's Steam library; untouched) | read-only scan of `libraryfolders.vdf` |
| Version / build id | Steam buildid **`25166532`** (`StateFlags` 4, `SizeOnDisk` 1,810,392,826 bytes) | `appmanifest_2830030.acf`, 2026-10-22 |
| Launcher exe SHA-256 | `CE589DEDD5138093CC6C02323B222CEA2EAD06F10AE41835B963991019DB32C2` — `<game dir>\MotorSlice.exe` (313,856 bytes) | `Get-FileHash`, 2026-10-22 |
| Main shipping exe SHA-256 | `FE2C4EDA115F71C8A37C2645D46286E3408F27D3BB5852CB1F1C4FB939A48DB6` — `<game dir>\MotorSlice\Binaries\Win64\MotorSlice-Win64-Shipping.exe` (156,616,192 bytes) | `Get-FileHash`, 2026-10-22 |
| Engine | **Unreal Engine 5.7.2.0** (`FileVersionRaw: 5.7.2.0`, `UE5-CL-0`), IoStore archives (`MotorSlice-Windows.pak` v12 unencrypted + `.ucas`/`.utoc` + `global.ucas`/`.utoc`) | exe version info, `coop_mcp.detect_engine`, pak footer read |
| Bundled Steamworks DLL | `<game dir>\Engine\Binaries\ThirdParty\Steamworks\Steamv161\Win64\steam_api64.dll` (Steamworks v1.61, SHA-256 `670D654AA3255C5061CF0236A1CDBB2F5076CBD5AE44F613B6F4921558E83E2D`) | `Manifest_NonUFSFiles_Win64.txt`, `Get-FileHash` |
| OS | Windows 64-bit | PE header (`dumpbin /headers`) |

### Answers to the triage questions (`playbook/triage.md`)
1. **Which exact game?** MOTORSLICE (2026), Steam build `25166532`.
2. **Is it already done?** **No.**
   - `DIRECTORY.md` and `games/` in `cranebeaker/coop-anything`: no entry or folder.
   - GitHub issues and PRs in `cranebeaker/coop-anything` (`q=repo:cranebeaker/coop-anything+motorslice`): `total_count: 0` (checked 2026-10-22).
   - GitHub repository search (`q=motorslice`): 1 repository (`LightningShopkeeper/Motorslice-Trainer`, an empty/stub trainer repo, no co-op).
   - Web search (`"MOTORSLICE" game engine Steam co-op mod`): Nexus Mods has visual tweaks/reshades/character swaps only (`https://www.nexusmods.com/motorslice`), Steam Workshop for Garry's Mod has an exported player model (`MOTORSLICE - P [PM & NPC]`), no co-op or multiplayer mod exists.
3. **Which engine?** **Unreal Engine 5.7.2** (`Unreal Engine 4/5` family in `playbook/engine-families.md`). Sibling work in this hub: Poppy Playtime (`games/poppy-playtime`, UE4SS Lua + Python/Steam relay), Half Sword (`Dudiebug/halfsword-multiplayer-mod`, `dwmapi.dll` C++ proxy + Steam relay), Ghostrunner (`games/ghostrunner`, UE4SS probe).
4. **How much source is there?** Closed shipping binary (`MotorSlice-Win64-Shipping.exe`, unwrapped PE32+ x64 with intact `.text` 114.4 MB and `.pdata` 5.7 MB) plus UE5 IoStore cooked Blueprints indexed in plaintext in `<game dir>\Manifest_UFSFiles_Win64.txt`.
5. **Does the engine have any multiplayer?** MOTORSLICE is single-player only on Steam. However, `MotorSlice-Win64-Shipping.exe` retains UE5's networking and local-multiplayer strings (`IpNetDriver`, `ServerReplicateActors`, `ReplicatedMovement`, `OnlineSubsystemSteam`, `bUseSplitscreen`, `CreatePlayer`) and ships `OnlineSubsystem`, `OnlineSubsystemSteam`, `OnlineSubsystemUtils`, and `MsQuic` plugins in `MotorSlice-Windows.pak` plus `Steamv161\Win64\steam_api64.dll`.
6. **Shipping check.** Single-player game with no anti-cheat (no EAC/BattlEye/Denuvo; PE sections are standard MSVC `.text`/`.rdata`/`.data`/`.pdata`). A co-op mod ships as a standalone UE4SS Lua mod + relay without redistributing any game files.

### Observations (verified by read-only inspection, 2026-10-22)
1. **Unwrapped executable and proxy load points:** `dumpbin /headers` and `/dependents` on `MotorSlice-Win64-Shipping.exe` confirm standard unencrypted PE sections and direct imports of `dwmapi.dll`, `WINMM.dll`, `VERSION.dll`, `WS2_32.dll`, `dxgi.dll`, plus delay-loaded `XINPUT1_4.dll`, `d3d11.dll`, `d3d12.dll`, and `steam_api64.dll`.
2. **Bundled `steam_api64.dll` supports the flat C API out of the box:** Checked via Python `ctypes.WinDLL` against `<game dir>\Engine\Binaries\ThirdParty\Steamworks\Steamv161\Win64\steam_api64.dll` (SHA-256 `670D654AA3255C5061CF0236A1CDBB2F5076CBD5AE44F613B6F4921558E83E2D`). It exports `SteamAPI_InitFlat`, `SteamAPI_ManualDispatch_Init`, `SteamAPI_ManualDispatch_RunFrame`, `SteamAPI_ManualDispatch_GetNextCallback`, `SteamAPI_ManualDispatch_FreeLastCallback`, `SteamAPI_ManualDispatch_GetAPICallResult`, `SteamAPI_SteamUser_v023`, `SteamAPI_SteamMatchmaking_v009`, `SteamAPI_SteamFriends_v017` (note: `v017`, whereas Steamworks 1.64 uses `v018`), `SteamAPI_SteamNetworkingMessages_SteamAPI_v002`, and `SteamAPI_SteamNetworkingUtils_SteamAPI_v004`. This means a Spacewar (AppID 480) relay can load the game's own `steam_api64.dll` with zero external Steamworks downloads.
3. **Plaintext asset manifest (`Manifest_UFSFiles_Win64.txt`):** The install root ships `Manifest_UFSFiles_Win64.txt` (1,964,500 bytes), which lists every cooked asset and map inside the IoStore containers:
   - **Core Blueprint classes (`/Game/Blueprints/Essentials/` & `/Game/Blueprints/MainCharacter/`):**
     - Player character: `/Game/Blueprints/MainCharacter/MainChar` (`MainChar_C`), skeletal mesh `/Game/Blueprints/MainCharacter/P/Mesh/P_Mesh`, skeleton `P_Mesh_Skeleton`, anim blueprint `P_Anim`, companion drone `/Game/Blueprints/OrbDrone/Rig/OrbieActor`.
     - Plain animation sequences on `P_Mesh_Skeleton`: `/Game/Blueprints/MainCharacter/P/Anim/Walk/IDLE`, `Walk`, `Run`, `Fast`, `/Game/Blueprints/MainCharacter/P/Anim/Jump/Jump`, `Fall`, `/Game/Blueprints/MainCharacter/P/Anim/Attack/Attack`, plus `WallRun`, `CrouchSlide`, `Parkour`, `HangOff`, `BeamWalk`, `Pillar`, `Pole`.
     - Core framework: `MyController`, `MyGameInstance`, `MyGameMode`, `MyPlayerCamera`, `MySaveGame`, `SaveVars`, `SaveSlot`, `GlobalConfigSave`, `ConfigSave`, `LoadMapArea`, `LoadDetection`, `CheatList`, `CheatWidget`.
     - World & combat actors: `Checkpoint` (`/Game/Blueprints/Checkpoint/Checkpoint`), `Door2Normal`, `Door4Enemy`, `MetalGate`, `Valve` (`/Game/Blueprints/ValveSwitch/Valve`), `HackCube`/`HackPlatform`, `DataCore`/`DataCoreManager`, `EnemyManager`, `ActivateEnemyBox`, `EnemyArenaWall`, `GuardPatrol`/`GuardAI`, `SentinelDrone`, `BerserkTank`/`AITank`, `BulldozerTruck`, `BasicMotor` and bosses `Boss01`–`Boss08`.
   - **Map structure (`/Game/Maps/`):**
     - Title screen: `/Game/Maps/00_StartScreen/StartScreenMap`.
     - Megastructure world maps: `/Game/Maps/Megastructure`, `/Game/Maps/WorldMap`, `/Game/Maps/NewObelisk`, plus chapter sublevels `00_Tutorial`, `01_City`, `02_Canal`, `03_Falls`, `04_Overpass`, `05_Pillars`, `06_Rig`, `07_Quarry`, `08_Desert`, `09_Ending` (each split into `*_Environment`, `*_Enemy`, `*_Events`, and main chapter `.umap`).
4. **Save system (`%LOCALAPPDATA%\MotorSlice\Saved\SaveGames`):**
   - Inspected `FurthestSave.sav` (2,881 bytes), `GameSave.sav` (1,220,925 bytes), and `GlobalConfig.sav` (2,551 bytes) in read-only mode and backed them up to `local/save-backup/`.
   - Standard UE5 `GVAS` format. Campaign progress is centralized in `/Game/Blueprints/Essentials/MySaveGame.MySaveGame_C` (`SaveVars` struct):
     - `LevelID_2_175700E740984B5FD50D82AC1E4761D8` (`IntProperty`)
     - `OrbsCollected_42_189ED8F946BB5943DF9014AAC05E4C00` (`ArrayProperty<IntProperty>`)
     - `DeathCount_29_6AD641DE4BB3C3CE19F1098130987105` (`ArrayProperty<StructProperty: PerLevelCount>`)
     - `TimerCount_30_394BC1824E4764E517D4438AD622BB1C`, `DeathTotal_50_656B3EB4487C3E83149FCCB7BA0ACD4C`, `Playtime_51_C9A1E21143E88562D0E26A9568F0379F`, `Achievements_56_0E296E1F49E4A988344AC5AB3C1C16F8`, `Money_62_0D3B24564C325E43164F5E9DF98E1773`.

---

## Phase 2 & 3: Workspace, UE4SS Hook, and Runtime Class Dump (2026-10-22)

### Workspace & Save Isolation
- Created two isolated hardlinked test copies (`instance-a` and `instance-b`) from `<game dir>`.
- Downloaded `RE-UE4SS` `experimental-latest` (`UE4SS_v3.0.1-1181-g83330635.zip`, 8,748,315 bytes, SHA-256 `B9A166A756605B21058A64CDE93BCAAB1A855DBA3553E5C7E6CE0FA8FDFAD7B9`) and extracted it into `MotorSlice\Binaries\Win64` in both copies (`dwmapi.dll` in `Binaries\Win64\`, runtime and `Mods/` in `Binaries\Win64\ue4ss\`).
- **Observation — `-nullrhi -nosound -UserDir="<dir>"` works cleanly:** Launching `MotorSlice-Win64-Shipping.exe -windowed -ResX=640 -ResY=360 -NoSplash -nullrhi -nosound -UserDir="..."` with `steam_appid.txt` (`2830030`) boots directly into `/Game/Maps/WorldMap.WorldMap` in ~3 seconds with zero GPU/window focus overhead and keeps `%LOCALAPPDATA%\MotorSlice\Saved\SaveGames\*` 100% untouched (verified by SHA-256 before and after every run).
- **Observation — UTF-8 BOM trap in UE4SS Lua loader:** Writing `main.lua` via PowerShell `Set-Content -Encoding UTF8` (which writes a `0xEF 0xBB 0xBF` BOM in Windows PowerShell 5.1) caused UE4SS to fail with `main.lua:1: unexpected symbol near '<\239>'`. Writing UTF-8 without BOM fixes it.

### Runtime Class & Property Observations (UE 5.7.2)
1. **Default boot world is `/Game/Maps/WorldMap.WorldMap`:**
   - Immediately on startup, `MyController_C`, `MainChar_C`, `MyGameInstance_C`, and `MyGameMode_C` are live in `WorldMap`.
   - `WorldMap` starts with 8 main `Checkpoint_C` actors (`LevelID` = `20`, `1520`, `2520`, `3520`, `4520`, `5520`, `6520`, `7520`).
   - Calling `cp:TriggerCheckpoint()` + `cp:ReloadMaps()` on the `Checkpoint_C` with `LevelID = 20` immediately streams the Chapter 0/1 sublevels (`61` `Checkpoint_C` actors, `6` `Door2Normal_C`, `10` `Door4Enemy_C`, `2` `MetalGate_C`, `2` `Valve_C`, `21` `DataCore_C`, `19` `GuardPatrol_C`, `1` `EnemyManager_C`) and sets `MyGameMode_C.LevelID = 20`.
2. **UE 5.7 `SkeletalMeshComponent` API change (`GetSkeletalMeshAsset` vs `SkeletalMesh`):**
   - In UE 5.7, reading `pawn.Mesh.SkeletalMesh` or `pawn.Mesh.SkeletalMeshAsset` as a Lua property returns `nil`, and `SetSkeletalMesh` is deprecated (`GetSkeletalMesh_DEPRECATED`).
   - Calling the `UFunction` methods `pawn.Mesh:GetSkeletalMeshAsset()`, `pmesh:SetSkeletalMeshAsset(skel)`, and `pmesh:SetSkinnedAssetAndUpdate(skel, true)` works reliably:
     ```text
     Dressed! skel=SkeletalMesh /Game/Blueprints/MainCharacter/P/Mesh/P_Mesh.P_Mesh mats=5 animMode=1 isPlaying=true animInst=AnimSingleNodeInstance ...
     ```
   - `P_Mesh` carries all 5 materials (`MID_MA_P_HairCian`, `MID_MA_P_Face`, `MID_EyeMaterial_Inst`, `MID_MA_P_Body`, `MID_CharacterEffect_Inst`) automatically, and `CharacterMesh0` scale is `(1.000, 1.000, 1.000)`.
   - Calling `pmesh:PlayAnimation(seq, true)` with `/Game/Blueprints/MainCharacter/P/Anim/Walk/Run.Run` switches `AnimationMode` to `1` (`AnimationSingleNode`), creates an `AnimSingleNodeInstance`, and plays the sequence (`isPlaying=true`, `len=1.00`) with zero crashes.

---

## Phase 4, 5 & 6: Co-op Mod, Relay, and Automated QA (2026-10-22)

### Architecture (`MotorSliceCoop`)
- **`MotorSliceCoop/Scripts/main.lua`:**
  - Runs a 50 ms `LoopAsync` tick wrapped in `ExecuteInGameThread`.
  - Reads local `MainChar_C` location, rotation, velocity, movement flags (`isWallRunning`, `SlidingEnable`, `Jumping`, `HittingGround`), `MyGameMode_C.LevelID`, and sublevel-qualified world events (`<Sublevel>.<ActorName>` for `Door2Normal_C.Open`, `Door4Enemy_C.Opened`, `Valve_C.Turned`, `DataCore_C.Collected`, and enemy `Dead`/`Annihilated` flags).
  - Writes `<netdir>/<playerId>.state` and reads `<netdir>/<peerId>.state`.
  - Summons a `SkeletalMeshActor` via `summon SkeletalMeshActor`, dresses it with `P_Mesh`, disables actor collision, and updates its location (`Z - 90` capsule-to-mesh offset), yaw (`Yaw - 90`), and single-node animation (`IDLE`, `Walk`, `Run`, `Fast`, `Jump`, `Fall`, `WallRun`, `Slide`).
  - When the peer's `LevelID` advances to a new checkpoint, calls `TriggerCheckpoint()` + `ReloadMaps()` on the matching `Checkpoint_C`.
  - Applies new peer world events idempotently (`OpenDoor()`, `CompleteTurning()`, `StartCollect()`, `DestroyEvent()`).
- **`relay/cooprelay.py`:**
  - `--transport udp`: non-blocking UDP bridge between `<netdir>/<self>.state` and `<netdir>/<peer>.state`.
  - `--transport steam`: loads the game's bundled `Steamv161\Win64\steam_api64.dll` via `ctypes`, initializes Spacewar (`AppID 480`) via `SteamAPI_InitFlat`, creates/joins a lobby via `ISteamMatchmaking`, and exchanges state lines over `ISteamNetworkingMessages` channel 0.
  - **Windows file-lock note:** When both Python (`os.replace`) and UE4SS Lua (`io.open`) poll `<peer>.state` at 20–60 Hz on Windows, `os.replace` intermittently raises `PermissionError` if Lua has the file open for reading. Wrapping `os.replace` in `safe_write` with a direct write fallback resolved dropped state updates completely.
  - **Sublevel actor naming note:** Multiple streamed sublevels contain actors named `Door2Normal_C_0` (e.g., `00_01_Tutorial:PersistentLevel.Door2Normal_C_0` and `01_01_City:PersistentLevel.Door2Normal_C_0`). Qualifying actor IDs as `<Sublevel>.<ActorName>` (`01_01_City.Door2Normal_C_0`) ensures the exact door instance opens on the peer.

### Verification Evidence (`test-relay.ps1` + mutation twin + `test-steam-smoke.ps1`)
1. **Two-client end-to-end test (`test-relay.ps1`):**
   ```text
   netA/A.state: -801.91 150.00 122.14 0.00 0.00 0.00 0.0 0.0 0.0 IDLE 20 D2:01_01_City.Door2Normal_C_0
   netB/A.state: -801.91 150.00 122.14 0.00 0.00 0.00 0.0 0.0 0.0 IDLE 20 D2:01_01_City.Door2Normal_C_0
   netB/B.state: -1254.61 -180.00 122.16 0.00 0.00 0.00 0.0 0.0 0.0 IDLE 20 D2:01_01_City.Door2Normal_C_0
   netA/B.state: -1254.61 -180.00 122.16 0.00 0.00 0.00 0.0 0.0 0.0 IDLE 20 D2:01_01_City.Door2Normal_C_0
   Log A SYNC lines: 14 (last: [23:56:01][A] SYNC: peer=B target=(-1254.6, -180.0, 32.2) puppet=(-1254.6, -180.0, 32.2) anim=IDLE lvl=20 ev=D2:01_01_City.Door2Normal_C_0)
   Log B SYNC lines: 14 (last: [23:56:01][B] SYNC: peer=A target=(-801.9, 150.0, 32.1) puppet=(-801.9, 150.0, 32.1) anim=IDLE lvl=20 ev=D2:01_01_City.Door2Normal_C_0)
   Log B checkpoint sync: [23:55:51][B] Activated checkpoint LevelID=20 at (-1200, 0, 125)
   Log B door sync: [23:55:55][B] Synced Door2Normal open: 01_01_City.Door2Normal_C_0
   RESULT: PASS
   ```
2. **Mutation twin (`test-relay.ps1 -MutateDisableSync`):**
   - With `sync_enabled.txt = 0`, `test-relay.ps1` exits with code `1` and prints `RESULT: FAIL (missing relayed state files)`.
3. **Real Steam Spacewar lobby smoke test (`test-steam-smoke.ps1`):**
   ```text
   [relay:steam] loading ...\Engine\Binaries\ThirdParty\Steamworks\Steamv161\Win64\steam_api64.dll
   [relay:steam] SteamAPI_InitFlat ok, appid 480
   [relay:steam] logged_on=True
   [relay:steam] LobbyCreated result=1 lobby=109775241888613317
   [relay:steam] LOBBY READY. Friend joins with: --lobby 109775241888613317
   [relay:steam] LobbyEnter lobby=109775241888613317 response=1
   RESULT: PASS
   ```
4. **Save integrity verification:**
   - `%LOCALAPPDATA%\MotorSlice\Saved\SaveGames\*` hashes verified identical to pre-session backup (`FurthestSave.sav` `0D40450D...`, `GameSave.sav` `C1F3D90A...`, `GlobalConfig.sav` `639E9791...`, `steam_autocloud.vdf` `0C2D3F6B...`).
