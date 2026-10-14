# Turtle Beach VelocityOne Flightstick (10f5:7055) on Linux — findings

Adding a data point to this issue nearly three years on, since it's still the top search result for this stick and there's no other writeup anywhere.

**System:** CachyOS (Arch-based), Steam native (non-Flatpak), October 2026 client. Tested against SimplePlanes 2 (Steam app 2840470), which is a Windows build run through Proton — there is no native Linux build.

## The device enumerates correctly

For anyone wondering whether the hardware works at all: it does. `lsusb` reports it, and udev creates the expected nodes:

```
usb-Turtle_Beach_VelocityOne_Flightstick_<serial>@A-event-joystick -> ../event19
usb-Turtle_Beach_VelocityOne_Flightstick_<serial>@A-hidraw         -> ../../hidraw5
usb-Turtle_Beach_VelocityOne_Flightstick_<serial>@A-joystick       -> ../js1
usb-Turtle_Beach_VelocityOne_Flightstick_<serial>@A-if01-event-mouse -> ../event20
usb-Turtle_Beach_VelocityOne_Flightstick_<serial>@A-if01-hidraw      -> ../../hidraw6
usb-Turtle_Beach_VelocityOne_Flightstick_<serial>@A-if01-mouse       -> ../mouse1
```

Note interface 01. **This stick presents a second USB interface that the kernel binds as a mouse**, with both an evdev node and a legacy `/dev/input/mouse1` node. I haven't seen this mentioned anywhere. I did not confirm whether it emits spurious events, so treat it as a thing to be aware of rather than a known bug.

## Confirmed working: getting a Proton game to see the real device

Initial symptom was that the game saw the stick and bound pitch and roll fine, but yaw (twist) and the throttle and flaps levers could not be bound at all. That is the signature of the device being flattened into a six-axis XInput pad somewhere below the game.

These launch options fixed it — all axes became bindable:

```
PROTON_ENABLE_HIDRAW=0x10f5/0x7055 SDL_JOYSTICK_HIDAPI=0 %command%
```

If `PROTON_ENABLE_HIDRAW` appears to do nothing, check permissions on the hidraw node first. Proton fails quietly if it can't open it:

```
ls -l /dev/hidraw*
```

## Confirmed broken: do not add SDL_GAMECONTROLLER_IGNORE_DEVICES

Adding `SDL_GAMECONTROLLER_IGNORE_DEVICES=0x10f5/0x7055` to the game's launch options caused the game to stop seeing the controller entirely. Worth documenting because it's an obvious thing to reach for and it's a dead end in this configuration.

## Gotcha that is not a bug: orientation setting

The Flightstick has a left-handed / right-handed orientation setting configured on the device. **Changing it reorders the axis and button indices.**

Symptoms if you flip it without realising: the throttle knob swaps from one side to the other, accessory buttons swap sides, some controls stop responding entirely, and your in-game bindings all still look correct because the game stores them by index. It presents exactly like a descriptor or enumeration bug. It isn't. Set orientation back to whichever side you originally configured.

This cost me an evening of chasing phantom problems, so it's worth having in writing.

## Still unresolved: Steam grabs the device as a generic Xbox pad

Corroborating the original report. On replug while in-game, Steam shows a "generic Xbox controller detected" toast. Downstream effect for me is that a lever crossing a threshold fires a Steam screenshot.

Things that did **not** stop it:

- "Disable Steam Input" in the game's per-title controller properties
- Generic / third-party controller support toggle in Big Picture → Settings → Controller (was already off)
- Clearing or reassigning the keyboard screenshot shortcut, which has no effect because the trigger is controller-level rather than a keypress

This matches the original poster's `controller.txt`, where the device is found as `type: 10f5 7055, path: sdl://1` with the correct product name, then re-reported seconds later as `Generic X-Box pad` with an XInput slot reserved.

## Untested idea for the Steam grab

Since Steam claims the device through SDL, it may be possible to hide it from the Steam client without hiding it from the game. Start Steam with the ignore hint set, then cancel it for the game process via launch options, which inherit Steam's environment:

```
# start Steam itself with:
SDL_GAMECONTROLLER_IGNORE_DEVICES=0x10f5/0x7055 steam

# game launch options:
PROTON_ENABLE_HIDRAW=0x10f5/0x7055 SDL_JOYSTICK_HIDAPI=0 SDL_GAMECONTROLLER_IGNORE_DEVICES= %command%
```

**I have not verified this works.** Posting it as a direction rather than a fix. If Steam's controller stack doesn't honour the SDL hint it will simply do nothing.

There may also be a plain-text binding in the generic controller config set, worth grepping with Steam closed:

```
~/.local/share/Steam/steamapps/common/Steam Controller Configs/<id>/config/configset_controller_generic.vdf
```

## Related

There is a WIP userspace driver for this exact device at `PatrykBukowski/hid-velocityone-flightstick` — Python over hidapi and python-evdev, republishing to uinput. Early stage, but it's the right architecture if the long-term answer turns out to be bypassing Steam's and Wine's interpretation of the device entirely.

---

# Short version for a ProtonDB report (SimplePlanes 2, app 2840470)

Turtle Beach VelocityOne Flightstick (10f5:7055) will bind pitch and roll but not yaw or the throttle and flaps levers until you add these launch options:

```
PROTON_ENABLE_HIDRAW=0x10f5/0x7055 SDL_JOYSTICK_HIDAPI=0 %command%
```

Do not add `SDL_GAMECONTROLLER_IGNORE_DEVICES` — the game loses the controller entirely.

Separate issue, not game-specific: Steam still claims this stick as a generic Xbox pad even with Steam Input disabled per-title, which can cause a lever to fire Steam screenshots. See ValveSoftware/steam-for-linux#10376.

Also note the stick's own left/right handed orientation setting reorders axis and button indices, which will silently break every binding if you change it.
