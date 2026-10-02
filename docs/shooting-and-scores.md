# Game Scene: shooting and scores

Press either controller trigger to fire one small sphere forward from that hand.
Hit another player's torso: you gain **1** point and they lose **1** point.
Both players see the same scores on the board in **Game Scene**. Negative scores are allowed.
Each new Game Scene starts a new round at zero.

Run the project through its existing Lobby connection flow. Shooting stays off in Lobby.
The connection, lobby, avatar tracking, and scene-transition scripts are unchanged.

## The whole lesson

```text
Trigger press → Spawn bullet → Move bullet → Hit torso
                                              ↓
                                  RPC: report the hit
                                              ↓
                           Master updates networked scores
                                              ↓
                                Everyone's text updates
```

## Read these three scripts in this order

All three are in `Assets/ShootingDemo/Scripts`.

1. **PlayerShooter.cs — create the bullet.**
   This component is attached to Network Player. It uses the existing hand transforms
   and trigger actions from `FusionNetworkPlayer`. `WasPressedThisFrame()` notices a
   new press; holding a trigger does not keep firing. `Update()` remembers the press,
   and `FixedUpdateNetwork()` calls `Runner.Spawn()` on the next network tick.
   Each hand has a short cooldown. Presses during cooldown are discarded.

2. **NetworkBullet.cs — move and detect a hit.**
   The shooter has State Authority because they spawned the bullet. Only that client
   moves it; `NetworkTransform` shares the position. The script checks the bullet's
   entire path each tick so it cannot jump past a torso between checks.
   The avatar's existing **Neck** collider is its torso. The shooter's own body,
   heads, hands, and floor markers do not score. Solid walls and tables stop bullets.
   Bullets also disappear after three seconds if they miss.

3. **MatchScoreboard.cs — accept the report and show the result.**
   The Game Scene's **Match Scoreboard** is a Master Client Object. It stores scores
   in a `[Networked] NetworkDictionary<PlayerRef, int>`: a shared table of player IDs
   and numbers. Each client reads this table to draw its local TextMeshPro display.

## The RPC to explain in class

```csharp
[Rpc(RpcSources.All, RpcTargets.StateAuthority)]
public void RPC_ReportHit(NetworkId bulletId, PlayerRef victim, RpcInfo info = default)
```

- **All:** any client may send a report.
- **StateAuthority:** only the score manager's authority processes it.
- **bulletId:** which bullet hit?
- **victim:** which player was hit?
- **info.Source:** Fusion tells us who sent the report.

The score manager checks that the sender owns that bullet, both players are in the
room, and they are different players. It then increases one score and decreases the
other. Fusion sends the resulting numbers to everyone. No ownership request is
needed for the scoreboard.

**RPC = a message. Networked property = the current result.**

## Why is there an AcceptedHits table?

A bullet may resend its report while waiting for confirmation. The score manager
records accepted bullet IDs, so the same bullet can never score twice while it is
alive. The bullet becomes invisible after a hit, but its network object stays alive
briefly until it sees that confirmation. Then it despawns. The scoreboard removes
the old ID after the bullet disappears. A five-second timeout prevents stuck bullets.
Scores and confirmations are networked, so they survive a normal Master Client change.

This small demo accepts the shooter's collision report. It does not independently
verify combat on a server or compensate for network delay.

## Inspector setup students can reproduce

- **Network Player prefab:** add `PlayerShooter` and assign
  `Assets/ShootingDemo/Prefabs/Bullet.prefab` to **Bullet Prefab**.
- **Bullet prefab:** a sphere with scale `0.07`, `NetworkObject`, `NetworkTransform`,
  and `NetworkBullet`. No Rigidbody is needed: the script moves it and explicitly
  checks for collisions. Leave authority override off and destroy-on-authority-leave on.
- **Game Scene:** add a world-space Canvas with a `NetworkObject` and `MatchScoreboard`.
  Enable **Is Master Client Object**, disable **Destroy When State Authority Leaves**,
  and assign its score text. The supplied scene already has this setup.
- Keep the supplied **Neck** collider on the player prefab. New torso shapes must
  remain under the `Neck` object referenced by `FusionNetworkPlayer`.

The base folder only changes by adding the shooter component to its player prefab.
All new gameplay scripts, the bullet prefab, and its material are in `Assets/ShootingDemo`.

## Two-headset check before class

1. Join from Lobby on both devices and enter Game. Both scores should be zero.
2. Press and release each trigger. One sphere appears per press; holding does not repeat.
3. Hit the other torso. The two displays agree: shooter +1, target -1.
4. Hit yourself, a hand, or a head. No points change. A single bullet never scores twice.
5. Shoot a wall or miss. The bullet disappears without changing scores.
6. Let the other player shoot and score too.
7. Leave Game and start another round. The scores reset, and shooting was off in Lobby.

Network checks can verify the scoring messages, but real headsets are still needed
to check controller direction, aiming comfort, and physical two-player interaction.
