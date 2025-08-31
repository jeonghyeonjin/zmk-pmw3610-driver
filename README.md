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

## macOS Optimization

If you experience slow cursor movement or lag on macOS, the driver includes specific optimizations:

### Recommended Configuration for macOS

Add these options to your `board.config`:

```conf
# Enable macOS-specific optimizations
CONFIG_PMW3610_MACOS_OPTIMIZATION=y

# Enable moving average filter for smoother movement
CONFIG_PMW3610_MOVING_AVERAGE=y

# Adjust CPI for better macOS performance
CONFIG_PMW3610_CPI=1000
CONFIG_PMW3610_CPI_DIVIDOR=2
```

### What These Optimizations Do

1. **CONFIG_PMW3610_MACOS_OPTIMIZATION**: 
   - Increases SPI timing delays for better stability
   - Adds interrupt handling optimizations
   - Improves overall responsiveness on macOS

2. **CONFIG_PMW3610_MOVING_AVERAGE**: 
   - Applies a moving average filter to smooth cursor movement
   - Reduces jitter and lag
   - Provides more consistent tracking

3. **CPI Adjustments**: 
   - Higher CPI with dividor can provide better precision
   - Reduces the need for large movements

### Troubleshooting macOS Issues

If you still experience issues on macOS:

1. Try reducing the SPI frequency in your overlay:
   ```dts
   spi-max-frequency = <1000000>;  // Reduce from 2MHz to 1MHz
   ```

2. Increase the CPI dividor for more controlled movement:
   ```conf
   CONFIG_PMW3610_CPI_DIVIDOR=3
   ```

3. Disable smart algorithm if it causes issues:
   ```conf
   CONFIG_PMW3610_SMART_ALGORITHM=n
   ```
