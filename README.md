# ZMK Driver for Azoteq IQS5XX Trackpads

Note: This driver was originally developed by @stelmakhdigital and I wanted to add some features/bug fixes.  But I don't read Russian, so I asked my AI buddy to make this English [translated](https://github.com/geeksville/zmk_driver_azoteq/commit/4c56195658bbf31851b9892c63f758f50ea48d4d) version.  I hope you find it useful.  It now seems to work well for me.  If you see problems please open issues and I'll do what I can to help.

## Compatibility

This driver should work with any IQS5XX-based trackpad (TPS43 or TPS65).

## Features

- Trackpad movement.
- Touch reporting: finger contact is reported as `INPUT_BTN_TOUCH` (1 when one or more fingers are on the pad, 0 when all fingers are lifted).
- Single tap: registered as left click.
- Two-finger tap: registered as right click.
- Press and hold: registered as continuous left click (drag).
- Vertical and horizontal scrolling - Modern 'gesture based' two finger swiping
- Left/right/up/down swipe send INPUT_BTN_WEST/EAST/NORTH/SOUTH events.  You can map them to other keys/actions as you wish in your dts file. (Either 3-finger or 1-finger directional swiping is supported)
- Pinch-to-zoom (reported as movement on the INPUT_REL_MISC 'wheel' - you can add dts rules to map this to something your apps can understand)

## Usage

- In the configuration file (for the appropriate half), add `CONFIG_INPUT_TPS43` to enable the driver

```
CONFIG_INPUT_TPS43=y
```

- In the `.overlay` file, specify `compatible = "azoteq,tps43"` inside the i2c node where the trackpad will be used (example below).

> For complete trackpad configuration information, see the file: [available trackpad settings](./dts/bindings/input/azoteq,tps43-common.yaml)

```
&i2c0 {
    status = "okay";
    clock-frequency = <I2C_BITRATE_FAST>;
    pinctrl-0 = <&i2c0_default>;  /* Configuration for SDA and SCL */
    pinctrl-1 = <&i2c0_sleep>;    /* Configuration for SDA and SCL */
    pinctrl-names = "default", "sleep";

    tps43_trackpad: trackpad@74 {
        compatible = "azoteq,tps43";
        reg = <0x74>;
        status = "okay";
        
        /* GPIO connections */
        rdy-gpios = <&pro_micro 21 GPIO_ACTIVE_HIGH>;  /* RDY pin */
        rst-gpios = <&pro_micro 20 GPIO_ACTIVE_HIGH>;  /* RST pin */

        enable-power-management;
        
        sensitivity = <100>;           /* 100% = normal state */
        scroll-sensitivity = <50>;     /* 50% = normal state */
        zoom-sensitivity = <50>;       /* 50% = normal state */

        filter-settings=<0x0B>;        /* See filter description in `available settings` */
        // filter-dynamic-bottom=<7>;     /* Dynamic filter bottom beta (optional) */
        // filter-dynamic-lower=<6>;      /* Dynamic filter lower speed (optional) */
        // filter-dynamic-upper=<0xFA>;   /* Dynamic filter upper speed (optional) */
        x-resolution=<2048>;            /* X resolution in pixels */
        y-resolution=<1792>;            /* Y resolution in pixels */

        // swipe-initial-distance=<500>;          /* Swipe initial distance in px (optional) */
        // swipe-initial-time=<200>;              /* Swipe initial time in ms (optional) */
        // swipe-angle=<20>;                      /* Max swipe angle in degrees (optional) */
        // swipe-consecutive-distance=<200>;      /* Swipe consecutive distance in px (optional) */
        // swipe-consecutive-time=<150>;          /* Swipe consecutive time in ms (optional) */
        // scroll-initial-distance=<100>;         /* Scroll initial distance in px (optional) */
        // scroll-angle=<30>;                     /* Max scroll angle in degrees (optional) */
        // zoom-initial-distance=<100>;           /* Zoom initial distance in px (optional) */
        // zoom-consecutive-distance=<50>;        /* Zoom consecutive distance in px (optional) */

        scroll;
        two-finger-tap;
        single-tap;
        press-and-hold;
        swipes;
        // three-finger-swipe;                    /* 3-finger swipe only, without enabling 1-finger swipe (optional) */
        // three-finger-swipe-throttle-ms=<300>;  /* Min ms between 3-finger swipe actuations, 0 disables (optional, default 300) */
        zoom;

        switch-xy;
        invert-scroll-y;
    };
};
```

- Now you need to configure a listener for tracking touches:

> Important: If your trackpad `.overlay` configuration is on the central device, use `Option 1`. If the trackpad is located on the peripheral half of the keyboard, use `Option 2`

---
**Option 1**

Simply specify the listener on the central device itself.

```
/ {
    tps43_input: tps43_input {
        compatible = "zmk,input-listener";
        device = <&tps43_trackpad>;
    };
};
```

---

... Otherwise ...

---

**Option 2**

Configure `split_inputs` where the trackpad is used (on the peripheral half of the keyboard)

```
/ {
    split_inputs {
        #address-cells = <1>;
        #size-cells = <0>;

        tps43_split: tps43_split@0 {
            compatible = "zmk,input-split";
            reg = <0>;
            device = <&tps43_trackpad>;
        };
    };
};
```

Now on the central part, you need to specify a listener but without specifying `device` (device is specified where the trackpad is used - on the peripheral part)

```
/ {
    split_inputs {
        #address-cells = <1>;
        #size-cells = <0>;

        tps43_split: tps43_split@0 {
            compatible = "zmk,input-split";
            reg = <0>;
            /* No device property here - this is a proxy on the central side */
        };
    };

    tps43_listener: tps43_listener {
        compatible = "zmk,input-listener";
        device = <&tps43_split>;
        status = "okay";
    };
};
```
---


> Configuring the Azoteq trackpad requires 5 pins!

Power:
3V on nice!nano -> VDD on IQS5xx.
GND (Ground) on nice!nano -> GND on IQS5xx.

I2C signals:
SDA on nice!nano -> SDA on IQS5xx.
SCL on nice!nano -> SCL on IQS5xx.

"DR" or "RDY" pin on IQS5xx -> Any available GPIO on nice!nano. In the devicetree, this pin is specified as rdy-gpios.
"RST" pin is used to initialize device reset. In the devicetree, this pin is specified as rst-gpios.


## Proper Driver Operation Sequence

```txt
1. Power on / Hardware reset
   └─> Wait 10ms
   └─> RST: LOW (10ms) → HIGH
   └─> Wait ~600ms for firmware loading

2. Check SHOW_RESET flag (0x000F bit 0)
   └─> Poll until flag appears

3. Acknowledge reset
   └─> Write ACK_RESET (0x0431 = 0x80)

4. Device configuration
   └─> System Config 1 (0x058F) - event modes
   └─> XY Config (0x0669) - axis settings
   └─> Filter settings, gestures, etc.

5. Complete setup
   └─> Write SETUP_COMPLETE (0x058E = 0x40)

6. Configure GPIO interrupt (RDY)
   └─> AFTER full configuration

```

## Power Management

The driver supports two independent power management mechanisms to reduce power consumption:

### Integration with ZMK Power Management System

The trackpad automatically enters sleep mode when the keyboard transitions to idle/sleep state.

**How it works:**
- The `tps43_idle_sleeper.c` module subscribes to ZMK `zmk_activity_state_changed` events
- When ZMK transitions to `SLEEP` state, the trackpad enters sleep mode
- When ZMK transitions to `IDLE` state, the trackpad enters sleep mode only if the `idle-sleep` property is set; otherwise it stays active
- When returning to `ACTIVE` state, the trackpad wakes up

**ZMK States:**
- `ZMK_ACTIVITY_ACTIVE` - keyboard active, trackpad operational
- `ZMK_ACTIVITY_IDLE` - keyboard in idle mode, trackpad in sleep only if `idle-sleep` is set
- `ZMK_ACTIVITY_SLEEP` - keyboard in sleep mode, trackpad in sleep

**Important:** This mechanism only works if `enable-power-management` is enabled.

The optional `idle-sleep` property controls whether the IDLE state also puts the
trackpad to sleep:

```
&i2c0 {
    tps43_trackpad: trackpad@74 {
        compatible = "azoteq,tps43";
        /* ... */
        enable-power-management;
        idle-sleep;   /* also sleep the trackpad while ZMK is idle */
    };
};
```

### Technical Details

**Control register:**
- Suspend mode is controlled via the `SYSTEM_CONTROL_1` register (0x0432)
- The `TPS43_SUSPEND` bit (BIT(1)) is set to enter suspend mode
- In suspend mode, the trackpad consumes minimal power and does not process touches

**Wake-up:**
- Automatic wake-up occurs when activity is detected via RDY interrupt
- On wake-up, the trackpad automatically processes the first touch

> Without `enable-power-management`, power management is completely disabled

