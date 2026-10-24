# Penguin-Colony-VR
Mod for Penguin Colony "WIP"


# Penguin Colony VR

A VR mod for **Penguin Colony** (Blue Fang / GoolWorks) — Unity 2022.3.62f3, Mono x64.

The game is a flat third-person penguin game with no XR support at all. This mod adds
room-scale VR through **OpenXR**, driven by a BepInEx 5 plugin.

---

## Requirements

| | |
|---|---|
| Game | Penguin Colony (Windows, Mono build) |
| Mod loader | BepInEx 5.4.23.5 x64 (already in your install — the log you attached shows it loading) |
| Runtime | Any **OpenXR** runtime: Virtual Desktop, Quest Link, SteamVR, ALVR, … |
| Graphics | Direct3D 11 or 12. If the game starts on Vulkan, add `-force-d3d11` |

---

## Install

1. Close the game.
2. unzip the folder into the
   game's install folder (the one with `Penguin Colony.exe`).
   Result:
   ```
   <game>/
     BepInEx/
       plugins/
         PenguinColonyVR/
           PenguinColonyVR.dll
           OpenXR.dll
           LICENSES/
   ```
3. Start your VR runtime (Virtual Desktop, Quest Link, SteamVR …).
4. Launch the game. The plugin logs to `BepInEx/LogOutput.log`.

---

## Controls

| Input | Action |
|---|---|
| **Left stick** | Move (relative to where you are looking) |
| **Right trigger** | Interact / use |
| **Right grip** | Push |
| **Right primary** (A / X) | Hop |
| **Right secondary** (B / Y) | Bark |
| **Left primary** (X / Y) | Double bark |
| **Left secondary** (Y / B) | Rewind |
| **Left grip** | Sprint |
| **Left trigger** | First / third person toggle |
| **Left stick click** | Take photo |
| **Right stick click** | Lock on |

### Hotkeys

| Input | Action |
|---|---|
| `F1` | Toggle VR on / off |
| `F2` | Show / hide controller models |
| `F3` | Show / hide the pointer ray |
| `F4` | World-space UI on / off |
| `F5` | Snap turn on / off |
| `F6` | Re-anchor the VR rig |
| `F7` | Cycle the per-eye render scale (100% / 85 / 70 / 55 / 40%) |

**Hold both stick clicks for a second** to toggle VR from the controller. With a
streaming headset the keyboard belongs to the headset app, so the `F` keys may
never reach the game.

---

## How it works

The game has no XR packages, so nothing in the engine talks to the headset. The mod
drives it from the outside.

### Runtime

The mod uses **OpenXR**, through the same native bridge (`OpenXR.dll`) that the
Mortuary Assistant VR mod uses. That bridge hides the OpenXR instance, session,
swapchain and action plumbing behind a small C interface, which is exactly what a
Unity game without any XR package needs.

OpenXR rather than OpenVR matters in practice: **Virtual Desktop's own runtime
(VDXR) is an OpenXR runtime**, so this mod works with Virtual Desktop out of the
box, with no SteamVR and no runtime juggling.

### Rendering

The game renders through one camera into a `CustomRenderTexture` that a UI quad
displays — a mono pipeline. For VR the mod:

1. Keeps the game's camera alive, and **parks it only once the game is really
   being played** (the menu still draws on the monitor). The game still writes
   position, field of view, culling mask and cutscene framing into it.
2. Mirrors that camera into two extra cameras, one per eye, each with the runtime's
   own projection matrix for that eye.
3. Copies both eye images into the bridge's textures and queues them for the
   swapchain present.

Because the eye cameras inherit the game's camera settings, colour grading,
underwater effects and shadows all keep working.

### Locomotion

This game is already camera relative:

```csharp
float target = Mathf.Atan2(vector.x, vector.z) * 57.29578f + camTF.eulerAngles.y;
```

The penguin's heading is the stick direction *plus the camera's yaw*. So the mod
simply puts the headset's yaw into that camera transform and writes the VR stick
into the game's own `leftstick` field — movement ends up head relative with no
re-implementation of the movement code.

The runtime tracks the head in its own space, whose origin is wherever the player
happened to be when it started. The mod re-anchors that origin on the penguin's
head and keeps the difference as the room-scale offset, so leaning moves the view
while the penguin stays where it is. The penguin's head bone is turned with the
headset so the avatar reads correctly.

### Input

The game drives everything through Unity's Input System with a generated
`PenguinActions` wrapper (`Move`, `Look`, `Kick`, `Push`, `Hop`, `Bark`,
`DoubleBark`, `Interact`, `Rewind`, `Spin`, `LockOn`, `FPV`, `TakePhoto`, …).

Rather than re-implement any of it, the mod:

1. Reads the **real binding paths** of every action at runtime.
2. Works out which gamepad control that action listens on, and claims that control
   for the VR button driving it.
3. Presses those controls on a **virtual Xbox pad injected into Unity's Input
   System**, so the game runs its own input code path unchanged.
4. If an action has no usable gamepad binding, the mod calls the game's own
   private method with the same argument the generated callback would use.

The log tells you which path each action took:

```
[Info] Action Interact: gamepad control <Gamepad>/buttonWest
[Info] Action Hop: no usable gamepad binding, invoking the game method directly
```

The virtual pad is also paired with any `PlayerInput` in the scene, so player
actions and menus see it.

### UI

The game's canvases keep their screen space layout, so the game lays them out
exactly as it always did. A separate camera below the world draws them into a
texture, and a panel in front of the player shows that texture. The panel is the
only thing that moves, so it follows the headset and the interface cannot come
apart.

Pointing and clicking go through a **virtual mouse** that is added to Unity's
input system: the controller's ray is worked out against the panel, the mouse is
put at that spot in the texture, and the game's own interface module does the
rest. Every button, slider and scroll bar therefore behaves the way it does
with a real mouse.

---

## Configuration

`BepInEx/config/PenguinColonyVR.cfg`:

| Setting | Default | Meaning |
|---|---|---|
| `Enabled` | `true` | Master switch |
| `ShowHands` | `true` | Controller models |
| `ShowPointer` | `true` | Pointer ray on the right controller |
| `WorldSpaceHud` | `true` | Put the interface on a panel in the headset |
| `MenuUiInHeadset` | `true` | Show the menu in the headset as well |
| `VirtualGamepad` | `true` | Inject a virtual Xbox pad |
| `NoHeadBob` | `true` | Ignore the game's camera bob |
| `SnapTurn` | `true` | Snap turning with the right stick |
| `SmoothTurn` | `true` | Smooth turning with the right stick |
| `TurnSpeed` | `90` | Smooth turn speed, degrees a second |
| `SnapAngle` | `30` | Snap angle in degrees |
| `MoveSpeed` | `1` | Movement stick multiplier |
| `WorldScale` | `1` | How far your real leaning moves the view |
| `HeadHeight` | `0.05` | How far above the penguin the camera sits |
| `RenderScale` | `1` | Per-eye render resolution scale (0.25 – 1.5) |
| `FlatMirror` | `true` | Also draw the left eye to the monitor |
| `UiDistance` | `1.4` | How far in front of you the interface panel hangs |
| `UiWidth` | `1.5` | How wide the interface panel is |
| `UiSupersample` | `2` | How much sharper the interface is drawn than the game screen (1 – 3) |
| `UiHideBackgrounds` | `true` | Take the pieces of interface that cover the whole screen out of the panel |
| `UiLift` | `0` | How far the interface panel sits above eye level |
| `UiFlip` | `false` | Turn the interface panel upside down |
| `DisableRealMouse` | `true` | Switch the desk mouse off while the interface is in the headset |
| `UiSubmit` | `true` | Also press the interface's submit button on the trigger (menu only) |

---

## Streaming (Virtual Desktop, ALVR, Quest Link)

The mod speaks OpenXR, so **any** OpenXR runtime works. With Virtual Desktop that
means its default runtime is fine — no SteamVR needed.

Two things to know:

- **The game renders twice**, once per eye, so the GPU cost is roughly double the
  flat game. The game normally renders into an internal `CustomRenderTexture` and
  scales it, but **that path is bypassed in VR** — the eye cameras render straight
  into the eye targets at full resolution. That is what `RenderScale` is for: set
  it to 70% and the frame rate usually settles.
- **Touch controllers have no thumbstick.** The touchpad *is* the stick, so
  movement is a touchpad drag and stick-click is a touchpad click.

If the frame rate is unstable: lower `RenderScale` (or press `F7`), lower the
game's own quality settings, and lower the streaming frame rate rather than
letting the game tear.

---

## Known limitations

Worth knowing before you file a bug:

- **Snap turning drifts.** The headset is the only orientation authority, so a
  snapped turn leaves your physical and virtual headings out of sync by the
  snapped angle. Turn it off with `F5`, or just turn your head.
- **Cutscenes are flat.** During a cutscene the game drives the camera, so it is
  rendered as a stereo pair without head tracking. Gameplay is fully head tracked.
- **The in-game photo** grabs the left eye image, so it is a flat picture.
- **The UI is a floating panel**, not a diegetic HUD. It follows the headset's
  heading but not its pitch, so it does not swim when you look around. `UiWidth`
  and `UiDistance` set how big it is and how far away it hangs.
- **The penguin's own body** is seen from inside, exactly like the game's own
  first person mode.
- **The controller models are primitives.** They are placeholders, not hand models.

---

## Credits

- `OpenXR.dll` — the native OpenXR bridge by **Astien (© 2025)**, used by the
  Mortuary Assistant VR mod. Free redistribution and modification, no commercial
  use. The full licence text is in `LICENSES/`.
- BepInEx 5 and Harmony — BepInEx contributors.
- The reverse engineering of the game itself was done against the `Managed`
  assemblies of the installed game, with no decompiled source shipped in this
  package.
