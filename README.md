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

# Use lower polling rate for Bluetooth
CONFIG_PMW3610_POLLING_RATE_100=y
# or
CONFIG_PMW3610_POLLING_RATE_50=y

# Adjust motion threshold (higher = less sensitive but smoother)
CONFIG_PMW3610_MOTION_THRESHOLD=2

# Batch size for motion reports (higher = less Bluetooth traffic)
CONFIG_PMW3610_BT_BATCH_SIZE=3
```

### How Bluetooth Optimization Works

1. **Motion Thresholding**: Small movements below the threshold are ignored to reduce noise
2. **Motion Batching**: Multiple motion samples are combined before sending to reduce Bluetooth traffic
3. **Adaptive Polling**: Lower polling rates reduce system load and improve Bluetooth stability
4. **Timeout Handling**: Ensures motion data is sent even if the batch isn't full

### Troubleshooting Bluetooth Issues

If you experience lag or stuttering on Bluetooth connections:

1. Try reducing the polling rate to 50Hz or 100Hz
2. Increase the motion threshold to 3-5
3. Increase the batch size to 4-5
4. Ensure your Bluetooth host supports the HID protocol properly

These optimizations should significantly improve the trackball performance on macOS and other Bluetooth hosts.
