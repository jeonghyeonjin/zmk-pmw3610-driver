PMW3610 driver implementation for ZMK with at least Zephyr 3.5

This work is based on [ufan's implementation](https://github.com/ufan/zmk/tree/support-trackpad) of the driver.

## Installation

Only GitHub actions builds are covered here. Local builds are different for each user, therefore it's not possible to cover all cases.

Include this project on your ZMK's west manifest in `config/west.yml`:

```yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/petejohanson
    - name: inorichi
      url-base: https://github.com/inorichi
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: feat/pointers-move-scroll
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: inorichi
      revision: main
  self:
    path: config
```

Then, edit your `build.yml` to look like this, 3.5 is now on main:

```yml
on: [workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

Now, update your `board.overlay` adding the necessary bits (update the pins for your board accordingly):

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 6 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

        /*   optional features   */
        // snipe-layers = <1>;
        // scroll-layers = <2 3>;
        // automouse-layer = <4>;
    };
};

/ {
  trackball_listener {
    compatible = "zmk,input-listener";
    device = <&trackball>;

  };
};
```

Now enable the driver config in your `board.config` file (read the Kconfig file to find out all possible options):

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_MOUSE=y
CONFIG_PMW3610=y
```

## Bluetooth Optimization

This driver includes special optimizations for Bluetooth connections, particularly useful for macOS and other Bluetooth hosts that may experience latency issues.

### Recommended Configuration for Bluetooth

For optimal Bluetooth performance, especially on macOS, use these settings:

```conf
# Enable Bluetooth optimizations
CONFIG_PMW3610_BLUETOOTH_OPTIMIZATION=y

# Keep original polling rate for responsiveness
CONFIG_PMW3610_POLLING_RATE_250=y

# Enable smooth motion for better cursor movement
CONFIG_PMW3610_SMOOTH_MOTION=y
CONFIG_PMW3610_SMOOTH_FACTOR=8

# Enable adaptive batching for dynamic performance
CONFIG_PMW3610_ADAPTIVE_BATCHING=y

# Adjust motion threshold (higher = less sensitive but smoother)
CONFIG_PMW3610_MOTION_THRESHOLD=2

# Batch size for motion reports (higher = less Bluetooth traffic)
CONFIG_PMW3610_BT_BATCH_SIZE=3
```

### How Bluetooth Optimization Works

1. **Motion Thresholding**: Small movements below the threshold are ignored to reduce noise
2. **Motion Batching**: Multiple motion samples are combined before sending to reduce Bluetooth traffic
3. **Motion Smoothing**: Uses interpolation to create smooth cursor movement instead of jerky updates
4. **Adaptive Batching**: Dynamically adjusts batch size based on motion speed for optimal responsiveness
5. **Timeout Handling**: Ensures motion data is sent even if the batch isn't full

### Motion Smoothing Features

- **Interpolation**: Smoothly interpolates between motion samples for fluid cursor movement
- **Configurable Smoothing**: Adjustable smoothing factor (1-20) for different preferences
- **Adaptive Response**: Fast movements use smaller batches for immediate response
- **Efficient Processing**: Only processes motion when needed to save power

### Troubleshooting Bluetooth Issues

If you experience lag or stuttering on Bluetooth connections:

1. **For jerky movement**: Enable motion smoothing with `CONFIG_PMW3610_SMOOTH_MOTION=y`
2. **For responsiveness**: Adjust smoothing factor (lower = more responsive, higher = smoother)
3. **For fast movements**: Enable adaptive batching with `CONFIG_PMW3610_ADAPTIVE_BATCHING=y`
4. **For general issues**: Increase the motion threshold to 3-5
5. **For extreme cases**: Try reducing the polling rate to 100Hz
6. Ensure your Bluetooth host supports the HID protocol properly

These optimizations should significantly improve the trackball performance on macOS and other Bluetooth hosts.
