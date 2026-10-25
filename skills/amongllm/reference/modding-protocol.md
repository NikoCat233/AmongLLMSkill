# Among Us Modding Protocol

This reference explains the two common Among Us mod models and the protocols used to register them with a game server.

The protocol is a compatibility boundary, not an implementation detail. Before changing a networking patch, identify the mod model and the protocol already used by the project.

## Mod Models

### All-client mods

An all-client mod requires every client in the lobby to run the same mod, or a compatible build of it. This model gives the mod more freedom to change gameplay, roles, state, and user interface behavior, but it reduces the set of players who can join.

All-client mods normally identify themselves with a stable mod GUID through the Among Us Modded Client Identification (AMCI) path. The GUID is used to distinguish the modded lobby and to filter matchmaking results.

### Host-only mods

A host-only mod requires the host to be modded while remote players may remain on an otherwise compatible vanilla client. The host-authority protocol is used to let the host decide or distribute state that would normally be validated by the server.

Host-only mods are easier for unmodded players to join. Their gameplay logic runs on the modded host.

## Server Types

### Official servers

The official backend is operated by Innersloth and uses the `among.us` domain family. The default regional matchmakers are:

| Region | Matchmaker |
| --- | --- |
| North America | `matchmaker.among.us` |
| Europe | `matchmaker-eu.among.us` |
| Asia | `matchmaker-as.among.us` |

Official servers apply stricter anti-cheat and validation checks than typical private servers. Official servers support Among Us releases from 2024 onward.

### Private servers

A private server is an independently operated backend, commonly reached through a custom region or endpoint. Most private Among Us servers are based on Impostor and weaken anti-cheat checks to support a wider range of mods. Their matchmaking, protocol handling, anti-cheat checks, and admission rules are controlled by the operator.

## Registration Requirement

Under the Among Us Mod Policy, every mod that changes Among Us functionality must register through at least one modding-protocol mechanism. Use AMCI for mod identification, the `+25` protocol revision for host authority, or both when the mod needs both.

The two mechanisms relevant to this guide are:

1. **AMCI registration**, using a stable mod GUID to identify the modded lobby.
2. **The `+25` modded protocol**, which enables host authority for a host-only lobby.

These mechanisms can be used together when the mod needs both client identification and host authority.

## AMCI Registration

AMCI registration applies to all mod models. It identifies the modded lobby using a stable mod GUID and supports matchmaking filters. AMCI provides no anti-cheat exemption on any server; official-server anti-cheat checks still apply after registration.

The game includes a built-in AMCI registration helper:

- `CurrentModRegistration.ModRegistrationGuidString` stores the configured GUID string.
- `CurrentModRegistration.TryGetModRegistrationGuid(out Guid guid)` rejects an empty value and validates the GUID format.
- `CurrentModRegistration.UpdateFilterSetWithModRegistrationSettings(...)` adds or updates the `mod` matchmaking filter when a valid GUID exists and removes stale `mod` filters when it does not.
- `InnerNetClient.HostGame(...)` selects `InnerNet.Tags.HostModdedGame` and appends the GUID bytes when registration succeeds.

`InnerNet.Tags.HostModdedGame` is the AMCI lobby-registration message tag. Its numeric value is `25`; it is unrelated to adding `25` to the broadcast-protocol revision for host authority.

If a project depends on a framework that already performs AMCI registration, such as a Reactor-like mod framework, use the framework's registration path and do not register AMCI a second time in the project.

## The `+25` Host-authority Protocol

Host-authority mods usually patch both `Constants.GetBroadcastVersion` and `Constants.IsVersionModded`. Add `25` to the protocol revision only when it is below `25`, and make `IsVersionModded` return `true` so the host uses the modded-host code paths.

The game recognizes the modded revision as host authority:

- `Constants.GetBroadcastVersion()` returns the integer protocol version.
- `Constants.GetVersionComponents(...)` decodes the integer into year, month, day, and revision.
- `Constants.IsVersionModded()` treats a revision of `25` or greater as modded.
- `InnerNetClient.AmModdedHost` is true only for the host when `Constants.IsVersionModded()` is true.

Use this patch pair for a host-only mod that enables host authority with `+25`:

```csharp
[HarmonyPatch(typeof(Constants), nameof(Constants.GetBroadcastVersion))]
class ServerUpdatePatch
{
    static void Postfix(ref int __result)
    {
        if (GameStates.IsOnlineGame && __result % 50 < 25)
        {
            __result += 25;
        }
    }
}

[HarmonyPatch(typeof(Constants), nameof(Constants.IsVersionModded))]
public static class IsVersionModdedPatch
{
    public static bool Prefix(ref bool __result)
    {
        __result = true;
        return false;
    }
}
```

Among Us encodes the protocol revision in the remainder after division by `50`; revisions `25` through `49` are already marked as modded. The check above adds the offset only to an unmodified revision.

Only patch `IsVersionModded` when using `+25` for host authority. It forces the host client to act as if the session uses `+25` even when the broadcast revision does not, sending host-authority behavior to a server that expects `+0` and triggering anti-cheat checks. Use AMCI alone for mod identification without host authority.

## Authority Changes Caused by `+25`

### Player information ownership

In the unmodified `+0` protocol, the server creates `NetworkedPlayerInfo` with `OwnerId = -4` and does not actively update it afterward. Player information changes through RPCs sent after joining.

The game changes this path when the host is modded. The player-creation coroutine computes:

```csharp
bool isOwnerOfPlayerData =
    NetworkMode == NetworkModes.LocalGame ||
    AmongUsClient.Instance.AmModdedHost ||
    NetworkMode == NetworkModes.FreePlay;
```

With `+25`, the host allocates the player id and creates `NetworkedPlayerInfo` with `OwnerId = -2`. The host owns its ongoing updates and marks player data dirty so the lobby receives them. Under `+0`, a normal remote client waits for the server-created player information with `OwnerId = -4`.

This is why `+25` is more than a matchmaking marker: it changes who owns and refreshes the player-information state in a modded lobby.

### Command checks

In the normal `+0` path used by Innersloth and most private servers, the server validates and responds to `Murder`, `Appear`, and `Vanish` checks. The host handles `Name`, `Color`, `Shapeshift`, `Zipline`, and `Protect` checks. Other checks follow their vanilla dispatch path. The host also applies `ShipStatus.UpdateSystem`.

With a `+25` broadcast-protocol revision, all `cmd check` decisions are routed to and processed by the host. The host then sends the resulting state to the other clients.

Command dispatch follows this pattern:

| Operation | `+0` handler | `+25` handler |
| --- | --- | --- |
| Murder (`CmdCheckMurder`) | Server validates and responds. | Host validates and distributes the result. |
| Appear (`CmdCheckAppear`) | Server validates and responds. | Host validates and distributes the result. |
| Vanish (`CmdCheckVanish`) | Server validates and responds. | Host validates and distributes the result. |
| Name (`CmdCheckName`) | Host runs `CheckName`; clients send the request to `HostId`. | Host runs `CheckName` and distributes the result. |
| Color (`CmdCheckColor`) | Host runs `CheckColor`; clients send the request to `HostId`. | Host runs `CheckColor` and distributes the result. |
| Shapeshift (`CmdCheckShapeshift`) | Host runs `CheckShapeshift`; clients send the request to `HostId`. | Host runs `CheckShapeshift` and distributes the result. |
| Zipline (`CmdCheckUseZipline`) | Host runs `CheckUseZipline`; clients send the check RPC to `HostId`. | Host runs `CheckUseZipline` and distributes the result. |
| Protect (`CmdCheckProtect`) | Host runs `CheckProtect`. | Host runs `CheckProtect` and distributes the result. |
| Other checks | Follow their vanilla server or host handler. | Host handles the check and distributes the result. |

### System updates

The host always handles `ShipStatus.UpdateSystem`, in both `+0` and `+25` protocol modes.

### Anti-cheat and ownership checks

With `+25`, most gameplay anti-cheat checks are disabled, including `InnerNetObject` ownership checks, allowing the host to send the relevant packets. Server rate limits still apply.

## First-contact Project Workflow

When a project is first encountered:

1. Determine whether it is an all-client mod or a host-only mod.
2. Search existing patches for `CurrentModRegistration`, `ModRegistrationGuidString`, `GetBroadcastVersion`, `IsVersionModded`, `AmModdedHost`, `HostModdedGame`, and framework registration helpers.
3. Confirm which protocol is already in use before adding a new registration path.
4. If the mode or protocol cannot be determined from the project, ask the user before changing networking behavior.
5. Record the confirmed mod model and protocol in project memory so later patches use the same assumptions.

