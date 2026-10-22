# MOTORSLICE (2026)

| | |
|---|---|
| Engine | Unreal Engine 5.7.2 |
| Versions tested | Steam build `25166532` (Windows 64-bit, `MotorSlice-Win64-Shipping.exe` SHA-256 `FE2C4EDA115F71C8A37C2645D46286E3408F27D3BB5852CB1F1C4FB939A48DB6`) |
| Projects | [MotorSliceCoop](https://github.com/SubmitCodes/motorslice-coop) (UE4SS Lua mod + UDP & Steam Spacewar P2P relay, MIT) |
| QA level | automated |
| Last updated | 2026-10-22 |

## Where things stand

You can play MOTORSLICE in two-player co-op using **MotorSliceCoop**, a `RE-UE4SS` (UE 5.7 experimental build) Lua mod paired with a zero-dependency Python relay (`cooprelay.py`) that uses the game's own bundled `Steamv161\Win64\steam_api64.dll` flat C API over Steam "Spacewar" (AppID 480) lobbies and `ISteamNetworkingMessages` P2P relay (plus a localhost/LAN UDP mode).

Both players see each other's animated character (`P_Mesh`) at 20 Hz, stream into the same megastructure chapter checkpoints (`LevelID`), and share opened doors (`Door2Normal_C`, `Door4Enemy_C`), turned valves (`Valve_C`), collected orbs (`DataCore_C`), and enemy slice/death states.

## Existing work

Searched 2026-10-22, nothing found prior to this work:

- **This repository (`DIRECTORY.md`, `games/`, issues and PRs):** 0 matches for `motorslice`.
- **GitHub repository search (`q=motorslice`):** 1 result (`LightningShopkeeper/Motorslice-Trainer`, a single-player trainer stub; no co-op or networking code).
- **Nexus Mods (`nexusmods.com/motorslice`):** visual reshades and character model swaps only; no multiplayer mod.
- **Steam Workshop / Community:** an exported player model addon for Garry's Mod (`MOTORSLICE - P [PM & NPC]`), no co-op mod for MOTORSLICE itself.

## How the game works (for co-op)

- **Engine and binaries:** Unreal Engine 5.7.2 (`FileVersionRaw: 5.7.2.0`). `MotorSlice\Binaries\Win64\MotorSlice-Win64-Shipping.exe` (156.6 MB) has no DRM wrapper and directly imports `dwmapi.dll` (used by `RE-UE4SS`), `WINMM.dll`, `VERSION.dll`, and `WS2_32.dll`.
- **Asset layout (`Manifest_UFSFiles_Win64.txt`):** Cooked assets live in IoStore containers (`MotorSlice-Windows.ucas`/`.utoc` and unencrypted Pak v12 `MotorSlice-Windows.pak`), with every asset path listed in `Manifest_UFSFiles_Win64.txt`:
  - **Player ("P") & Companion ("Orbie"):** `/Game/Blueprints/MainCharacter/MainChar` (`MainChar_C`), skeletal mesh `/Game/Blueprints/MainCharacter/P/Mesh/P_Mesh`, skeleton `P_Mesh_Skeleton`, anim blueprint `P_Anim_C`, and standalone animation sequences under `/Game/Blueprints/MainCharacter/P/Anim/` (`Walk/IDLE`, `Walk/Walk`, `Walk/Run`, `Walk/Fast`, `Jump/Jump`, `Jump/Fall`, `Attack/Attack`, `WallRun/WallRun`, `CrouchSlide/CrouchSlide`), plus `/Game/Blueprints/OrbDrone/Rig/OrbieActor`.
  - **Core classes:** `/Game/Blueprints/Essentials/MyController` (`MyController_C`), `MyGameInstance_C`, `MyGameMode_C`, `MyPlayerCamera_C`, `MySaveGame_C`, `SaveVars`, `GlobalConfigSave_C`, `LoadMapArea_C`.
  - **World & combat actors:** `Checkpoint_C` (`/Game/Blueprints/Checkpoint/Checkpoint`), `Door2Normal_C`, `Door4Enemy_C`, `MetalGate_C`, `Valve_C`, `HackCube_C`, `HackPlatform_C`, `DataCore_C`, `DataCoreManager_C`, `EnemyManager_C`, `GuardPatrol_C`, `SentinelDrone_C`, `BerserkTank_C`, `BulldozerTruck_C`, `BasicMotor_C`, and `Boss01`–`Boss08`.
- **Map & streaming structure:** On launch, the game boots into `/Game/Maps/WorldMap.WorldMap` (the megastructure persistent level) with 8 main `Checkpoint_C` actors (`LevelID` `20`, `1520`, `2520`, `3520`, `4520`, `5520`, `6520`, `7520`). Calling `TriggerCheckpoint()` + `ReloadMaps()` on a `Checkpoint_C` streams its chapter sublevels (`00_Tutorial`, `01_City`, etc.) into `WorldMap` and populates all chapter doors, valves, `DataCore` orbs, and enemies.
- **Bundled Steamworks:** `Engine\Binaries\ThirdParty\Steamworks\Steamv161\Win64\steam_api64.dll` (Steamworks 1.61, SHA-256 `670D654AA3255C5061CF0236A1CDBB2F5076CBD5AE44F613B6F4921558E83E2D`) exports `SteamAPI_InitFlat`, `SteamAPI_ManualDispatch_*`, `SteamAPI_SteamUser_v023`, `SteamAPI_SteamMatchmaking_v009`, `SteamAPI_SteamFriends_v017`, and `SteamAPI_SteamNetworkingMessages_SteamAPI_v002`, enabling a Spacewar (AppID 480) relay without downloading external Steamworks binaries.

## What we changed

Built **MotorSliceCoop**:
1. **`MotorSliceCoop/Scripts/main.lua` (`RE-UE4SS` Lua mod):**
   - Summons a `SkeletalMeshActor` on the game thread (`ExecuteInGameThread`) via `KismetSystemLibrary:ExecuteConsoleCommand(..., "summon SkeletalMeshActor", ...)` to avoid `UWorld::SpawnActor` crashes.
   - Dresses the puppet using UE 5.7's `GetSkeletalMeshAsset()` + `SetSkeletalMeshAsset()` + `SetSkinnedAssetAndUpdate(skel, true)` with `/Game/Blueprints/MainCharacter/P/Mesh/P_Mesh.P_Mesh` (all 5 material slots resolve automatically from the mesh asset), copies `CharacterMesh0` scale (`1.0`), disables actor collision, and plays single-node `AnimSequence` assets (`PlayAnimation(seq, true)`) driven by peer velocity and movement state (`IDLE`, `Walk`, `Run`, `Fast`, `Jump`, `Fall`, `WallRun`, `Slide`).
   - Syncs `MyGameMode_C.LevelID` so when the host or joiner enters a chapter checkpoint (`Checkpoint_C`), the peer automatically activates the matching `Checkpoint_C` (`TriggerCheckpoint()` + `ReloadMaps()`).
   - Syncs sublevel-qualified world events (`<Sublevel>.<ActorName>`) for `Door2Normal_C` (`OpenDoor()`), `Door4Enemy_C` (`OpenDoor()`), `Valve_C` (`CompleteTurning()`), `DataCore_C` (`StartCollect()`), and enemy kills (`DestroyEvent()`).
2. **`relay/cooprelay.py` + `deploy.ps1` + `host.bat` / `join.bat`:**
   - Bridges `<netdir>/<self>.state` and `<netdir>/<peer>.state` over either localhost/LAN UDP (`--transport udp`) or Steam Spacewar AppID 480 lobbies + `ISteamNetworkingMessages` (`--transport steam`) using the game's own `Steamv161\Win64\steam_api64.dll`.

## QA

- **QA level:** `automated`
- **`test-relay.ps1` (two real game instances + two UDP relays on one PC):**
  - Launches `instance-a` and `instance-b` concurrently (`-windowed -ResX=640 -ResY=360 -NoSplash -nullrhi -nosound` with isolated `-UserDir` save folders), triggers Tutorial `Checkpoint_C` (`LevelID=20`) on Host A, verifies Client B automatically streams `LevelID=20`, teleports both players to distinct coordinates (`A` at `-801.9, 150.0, 122.1`, `B` at `-1254.6, -180.0, 122.2`), opens `01_01_City.Door2Normal_C_0` on Host A, and asserts bidirectional puppet placement (`moved=true`), checkpoint streaming sync, and door open sync (`RESULT: PASS`, exit code `0`).
  - **Mutation twin (`test-relay.ps1 -MutateDisableSync`):** Disables state sync (`sync_enabled.txt = 0`) and confirms the test harness fails loudly (`RESULT: FAIL (missing relayed state files)`, exit code `1`).
- **`test-steam-smoke.ps1` (real Steam Spacewar lobby test):**
  - Loads the game's bundled `Steamv161\Win64\steam_api64.dll`, initializes `SteamAPI_InitFlat` on AppID `480`, creates a lobby (`LobbyCreated result=1`), sets lobby metadata (`motorslice_coop=1`), and enters the lobby (`LobbyEnter response=1`, `RESULT: PASS`, exit code `0`).
- **Owner save integrity:** Verified SHA-256 hashes of `%LOCALAPPDATA%\MotorSlice\Saved\SaveGames\*` (`FurthestSave.sav`, `GameSave.sav`, `GlobalConfig.sav`, `steam_autocloud.vdf`) before and after all test runs—100% untouched.

## How to play

1. Extract [`UE4SS_v3.0.1-1181-g83330635.zip`](https://github.com/UE4SS-RE/RE-UE4SS/releases/download/experimental-latest/UE4SS_v3.0.1-1181-g83330635.zip) (SHA-256 `B9A166A756605B21058A64CDE93BCAAB1A855DBA3553E5C7E6CE0FA8FDFAD7B9`) into `<MOTORSLICE>\MotorSlice\Binaries\Win64`.
2. **Host:** Run `host.bat "C:\Path\To\MOTORSLICE" steam` and share the printed `--lobby <LobbyID>` number with your friend.
3. **Friend:** Run `join.bat "C:\Path\To\MOTORSLICE" <LobbyID> steam`.

## Known issues

- Steam `ISteamNetworkingMessages` P2P packet exchange between two separate Steam accounts on two separate PCs has not been tested yet (lobby creation and entry on real Steam are verified; two-client gameplay sync is verified over the UDP relay).
- Puppet animations use single-node `AnimSequence` playback (`IDLE`, `Walk`, `Run`, `Fast`, `Jump`, `Fall`, `WallRun`, `CrouchSlide`) rather than the procedural `P_Anim_C` blueprint (which casts its owner to `MainChar_C` and cannot be attached to a `SkeletalMeshActor`).
- Boss phase transitions (`Boss01`–`Boss08`) are not individually hooked yet; standard enemies (`GuardPatrol`, `SentinelDrone`, `BerserkTank`, `BulldozerTruck`, `BasicMotor`), doors, valves, `DataCore` orbs, and checkpoints sync.

Detailed measurements, UE 5.7 API findings, and test logs are in [NOTES.md](NOTES.md).
