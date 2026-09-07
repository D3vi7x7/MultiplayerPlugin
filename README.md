# MultiplayerSessions Plugin

A lightweight Unreal Engine 5 plugin that wraps the `OnlineSubsystem` session API into a simple, delegate-driven `UGameInstanceSubsystem`. Drop it into any project to get session creation, session search, joining, and lobby travel working with just a few function calls and UI button bindings — no need to touch the raw `IOnlineSession` interface yourself.

Built on top of `OnlineSubsystem` / `OnlineSubsystemSteam`, but designed so the calling code (your Menu widget, game mode, etc.) never has to know which subsystem is active underneath.

---

## Features

- 🔹 **Create Session** — host a session with a configurable player count and match type
- 🔹 **Find Session** — search for open, joinable sessions filtered by presence
- 🔹 **Join Session** — join a found session and travel the client to the host
- 🔹 **Destroy Session** — safely tear down and re-host a session without race conditions
- 🔹 **Start Session** — mark a session as started/in-progress
- 🔹 All operations are exposed as **custom multicast delegates**, so any number of classes (UI, GameMode, HUD, etc.) can bind to the results without tight coupling to the subsystem internals

---

## Requirements

- Unreal Engine 5.x
- `OnlineSubsystem` and `OnlineSubsystemSteam` (or another `OnlineSubsystem` implementation) enabled
- A Steam AppID for testing (`480` — the public Spacewar test AppID — works fine during development)

---

## 1. Installation

1. Copy the plugin folder (containing `MultiplayerSessions.uplugin` and the `Source/` directory) into your project's `Plugins/` folder:
   ```
   YourProject/
     Plugins/
       MultiplayerSessions/
         MultiplayerSessions.uplugin
         Source/
           MultiplayerSessions/
             MultiplayerSessionsSubsystem.h / .cpp
             Menu.h / .cpp
   ```
2. Regenerate project files (right-click your `.uproject` → **Generate Visual Studio project files**, or `Generate Xcode project` on Mac).
3. Open the project — Unreal will prompt to build the plugin. Accept and let it compile.
4. Enable the plugin: **Edit → Plugins → search "MultiplayerSessions" → enable it** (it should already be enabled if it came bundled in `Plugins/`).

---

## 2. Add module dependencies

In your **project's** `Build.cs` (e.g. `YourProject.Build.cs`), make sure the following are listed under `PublicDependencyModuleNames` or `PrivateDependencyModuleNames`:

```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "Engine", "InputCore",
    "OnlineSubsystem",
    "OnlineSubsystemSteam",
    "MultiplayerSessions"
});
```

And in the **plugin's own** `MultiplayerSessions.Build.cs`, `OnlineSubsystem` should already be listed:

```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "Engine", "UMG", "Slate", "SlateCore",
    "OnlineSubsystem", "OnlineSubsystemSteam"
});
```

---

## 3. Enable the Online Subsystem in `DefaultEngine.ini`

Add this to your project's `Config/DefaultEngine.ini`:

```ini
[OnlineSubsystem]
DefaultPlatformService=Steam

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480

[/Script/Engine.GameEngine]
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemSteam.SteamNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")

[/Script/OnlineSubsystemSteam.SteamNetDriver]
NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
```

> If you're testing on LAN instead of Steam, set `DefaultPlatformService=Null` and skip the Steam-specific blocks — the plugin automatically detects a `NULL` subsystem and switches session settings to LAN mode.

Also make sure `steam_appid.txt` (containing `480`) sits next to your project's editor/game executable if you're testing Steam sessions outside of Steam itself.

---

## 4. Set up the UI (Menu Widget)

The plugin ships with a `UMenu` widget base class (`Menu.h` / `Menu.cpp`) that you extend in Blueprint:

1. Create a **Widget Blueprint** that inherits from `Menu` (parent class = `Menu`).
2. Add two `Button` widgets to the canvas and name them exactly:
   - `Host_Btn`
   - `Join_Btn`
   (these are `BindWidget` properties, so the names must match exactly or the bind will silently fail.)
3. From your main menu level Blueprint (or `GameMode::BeginPlay`), create the widget and call **`MenuSetup`**:

   ```cpp
   // C++ example
   UMenu* Menu = CreateWidget<UMenu>(GetWorld(), MenuWidgetClass);
   if (Menu)
   {
       Menu->MenuSetup(/*NumberOfConnections=*/4, /*TypeOfMatch=*/TEXT("FreeForAll"));
   }
   ```

   Or call `MenuSetup` directly from Blueprint (it's `BlueprintCallable`) with your desired **max players** and **match type** string.

`MenuSetup` automatically:
- Adds the widget to the viewport and switches input to UI-only mode
- Grabs the `MultiplayerSessionsSubsystem` from the current `GameInstance`
- Binds all five completion delegates (Create / Find / Join / Destroy / Start) to the widget's internal handlers

---

## 5. Wire up Host and Join

Out of the box:

- **Host_Btn** → calls `MultiplayerSessionsSubsystem->CreateSession(NumPublicConnections, MatchType)`. On success, `OnCreateSession` runs `ServerTravel` to your lobby map (default: `/Game/ThirdPerson/Maps/Lobby?listen` — change this path in `Menu.cpp` to match your project).
- **Join_Btn** → calls `MultiplayerSessionsSubsystem->FindSession(MaxSearchResults)`. When results come back, `OnFindSession` filters them by the `MatchType` string set in `MenuSetup` and automatically calls `JoinSession` on the first match. On success, `OnJoinSession` resolves the connect string and calls `ClientTravel` to the host.

You don't need to write any of this glue yourself — just make sure your **lobby map path** and **match type strings** match what you pass into `MenuSetup`.

---

## 6. Using the Subsystem directly (without the Menu widget)

If you want to build your own UI instead of using `UMenu`, you can talk to the subsystem directly from any class with access to a `GameInstance`:

```cpp
UMultiplayerSessionsSubsystem* Subsystem =
    GetGameInstance()->GetSubsystem<UMultiplayerSessionsSubsystem>();

if (Subsystem)
{
    // Bind to results
    Subsystem->MultiplayerOnCreateSessionComplete.AddDynamic(this, &AMyClass::OnCreateSession);
    Subsystem->MultiplayerOnFindSessionsComplete.AddUObject(this, &AMyClass::OnFindSessions);
    Subsystem->MultiplayerOnJoinSessionComplete.AddUObject(this, &AMyClass::OnJoinSession);
    Subsystem->MultiplayerOnDestroySessionComplete.AddDynamic(this, &AMyClass::OnDestroySession);
    Subsystem->MultiplayerOnStartSessionComplete.AddDynamic(this, &AMyClass::OnStartSession);

    // Kick off a session
    Subsystem->CreateSession(4, TEXT("FreeForAll"));
}
```

### API reference

| Function | Description |
|---|---|
| `CreateSession(int32 NumPublicConnections, FString MatchType)` | Hosts a new session. If one already exists, it is destroyed first and recreated automatically once teardown completes. |
| `FindSession(int32 MaxSearchResults)` | Searches for joinable sessions advertising presence. |
| `JoinSession(const FOnlineSessionSearchResult& SearchResult)` | Joins a specific search result. |
| `DestroySession()` | Tears down the current session. |
| `StartSession()` | Marks the current session as started. |

| Delegate | Fires when |
|---|---|
| `MultiplayerOnCreateSessionComplete(bool bWasSuccessful)` | A create attempt finishes |
| `MultiplayerOnFindSessionsComplete(const TArray<FOnlineSessionSearchResult>&, bool bWasSuccessful)` | A search finishes |
| `MultiplayerOnJoinSessionComplete(EOnJoinSessionCompleteResult::Type Result)` | A join attempt finishes |
| `MultiplayerOnDestroySessionComplete(bool bWasSuccessful)` | A destroy finishes |
| `MultiplayerOnStartSessionComplete(bool bWasSuccessful)` | A start finishes |

---

## 7. Match type filtering

`MatchType` is just a free-form `FString` stored as a session setting (`"MatchType"`), advertised via `EOnlineDataAdvertisementType::ViaOnlineServiceAndPing`. Use it to separate game modes at the lobby-browser level — e.g. `"FreeForAll"`, `"TeamDeathmatch"`, `"Ranked"`. When searching, only sessions whose `MatchType` matches the value passed into `MenuSetup` will be joined automatically.

---

## 8. Troubleshooting

- **Nothing happens when I click Host/Join** — Make sure your buttons in the Widget Blueprint are literally named `Host_Btn` and `Join_Btn`. `BindWidget` fails silently if the names don't match.
- **"Session already exists" error on re-host** — Make sure you're on a plugin version where `CreateSession` waits for `OnDestroySessionComplete` before recreating; destroying and creating in the same frame is a known race condition against Steam.
- **Client never travels to the lobby after "Session Created"** — Check that all five completion delegates (`CreateSessionCompleteDelegate`, etc.) are bound in the subsystem's constructor via `FOnCreateSessionCompleteDelegate::CreateUObject(...)`. An unbound delegate registers successfully but never actually calls your handler.
- **No sessions found when searching** — Confirm `SEARCH_PRESENCE` is set on both the hosting and searching side, and that both instances are using the same `OnlineSubsystem` (e.g. both Steam, or both Null/LAN — not a mix of the two).

---

## Roadmap / Future Scope

This plugin is the networking foundation for a larger project — a free-for-all multiplayer shooter — with the following planned additions:

- 🔹 Dedicated server support and server-authoritative session start
- 🔹 Skill-based matchmaking via a backend microservices layer (Match / Player / Economy / Telemetry services)
- 🔹 Kafka-based event streaming between backend services
- 🔹 Redis-backed matchmaking queue
- 🔹 FastAPI ML inference service for matchmaking quality / skill prediction
- 🔹 In-lobby chat and ready-check system
- 🔹 Reconnect handling for dropped clients

---

## License

Add your license of choice here (MIT, Apache-2.0, etc.) before publishing or open-sourcing.
