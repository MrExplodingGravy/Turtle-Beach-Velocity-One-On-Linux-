Short version for a ProtonDB report (SimplePlanes 2, app 2840470)
Turtle Beach VelocityOne Flightstick (10f5:7055) will bind pitch and roll but not yaw or the throttle and flaps levers until you add these launch options:

PROTON_ENABLE_HIDRAW=0x10f5/0x7055 SDL_JOYSTICK_HIDAPI=0 %command%
Do not add SDL_GAMECONTROLLER_IGNORE_DEVICES — the game loses the controller entirely.

Separate issue, not game-specific: Steam still claims this stick as a generic Xbox pad even with Steam Input disabled per-title, which can cause a lever to fire Steam screenshots. See ValveSoftware/steam-for-linux#10376.

Also note the stick's own left/right handed orientation setting reorders axis and button indices, which will silently break every binding if you change it.
