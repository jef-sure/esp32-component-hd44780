# HD44780 ESP-IDF Component

ESP-IDF component for HD44780-compatible character LCD modules.

This driver supports three transport backends:

- PCF8574 I/O expander over I2C
- Direct GPIO 4-bit mode
- Direct GPIO 8-bit mode

The controller API is shared across all three backends, so the application logic stays the same after the bus is created.

## Features

- Supports common single-controller geometries: 8x2, 16x2, 16x4, 20x2, 20x4, 40x2
- Handles the HD44780 power-on and 4-bit reset sequence internally
- Works with PCF8574 LCD backpacks that use the supported fixed pin mapping
- Supports direct GPIO wiring in 4-bit and 8-bit modes
- Provides display, cursor, blink, entry-mode, clear, home, backlight, and CGRAM helpers
- Fails early if the bus backend cannot initialize or the LCD init sequence cannot complete

## Limitations

- Not internally synchronized. Use one dedicated task for all LCD access.
- Dual-enable modules such as 40x4 are not supported.
- The driver is write-only. It does not poll the HD44780 busy flag.
- Direct GPIO backends assume RW is tied to GND.
- Backlight control is only available when the active backend implements it, such as the PCF8574 backpack backend.

## Public Header

Include the driver with:

```c
#include "hd44780.h"
```

## Supported Geometries

Use one of these geometry values when calling `lcd_init()`:

- `HD44780_GEOMETRY_8X2`
- `HD44780_GEOMETRY_16X2`
- `HD44780_GEOMETRY_16X4`
- `HD44780_GEOMETRY_20X2`
- `HD44780_GEOMETRY_20X4`
- `HD44780_GEOMETRY_40X2`

## Bus Creation

### I2C Backpack via PCF8574

For simple applications, create an HD44780 bus and let the component own the ESP-IDF I2C master bus:

```c
lcd_bus_hd44780_t *lcd_bus_pcf8574_i2c_create(
    i2c_port_t i2c_num,
    uint8_t i2c_addr,
    gpio_num_t sda,
    gpio_num_t scl,
    uint32_t scl_hz
);
```

For applications that already own an ESP-IDF I2C master bus shared with other devices, attach only the PCF8574 device:

```c
lcd_bus_hd44780_t *lcd_bus_pcf8574_i2c_create_on_bus(
    i2c_master_bus_handle_t i2c_bus,
    uint8_t i2c_addr,
    uint32_t scl_hz
);
```

Supported backpack bit mapping expected by the driver:

| PCF8574 pin | LCD signal |
| --- | --- |
| P0 | RS |
| P1 | RW |
| P2 | E |
| P3 | Backlight, active high |
| P4 | D4 |
| P5 | D5 |
| P6 | D6 |
| P7 | D7 |

Notes:

- `scl_hz = 0` selects a conservative 100 kHz default.
- Common backpack addresses are `0x27` and `0x3F`.
- Some PCF8574 LCD backpacks use different signal mappings or backlight polarity; this factory does not currently make those configurable.
- The driver probes the address before attaching the device.
- `lcd_bus_pcf8574_i2c_create()` creates and owns the master bus. The first device created on an I2C port creates that bus; later devices on the same port reuse it and must use the same SDA/SCL pins.
- `lcd_bus_pcf8574_i2c_create_on_bus()` uses an externally owned master bus. The caller remains responsible for creating and deleting that master bus; this component only removes its device handle when destroyed.

### Direct GPIO 4-Bit Mode

Create a write-only 4-bit bus with:

```c
lcd_bus_hd44780_t *lcd_bus_gpio4_create(
    gpio_num_t rs,
    gpio_num_t en,
    gpio_num_t d4,
    gpio_num_t d5,
    gpio_num_t d6,
    gpio_num_t d7
);
```

### Direct GPIO 8-Bit Mode

Create a write-only 8-bit bus with:

```c
lcd_bus_hd44780_t *lcd_bus_gpio8_create(
    gpio_num_t rs,
    gpio_num_t en,
    gpio_num_t d0,
    gpio_num_t d1,
    gpio_num_t d2,
    gpio_num_t d3,
    gpio_num_t d4,
    gpio_num_t d5,
    gpio_num_t d6,
    gpio_num_t d7
);
```

All GPIO pins passed to the direct backends must be output-capable GPIOs.

## Controller Lifetime

Bind a bus to the controller with:

```c
hd44780_t *lcd_init(lcd_bus_hd44780_t *bus, hd44780_geometry_t geometry, bool owns_bus);
```

If `owns_bus` is `true`, `lcd_deinit()` also destroys the bus object.

```c
esp_err_t lcd_deinit(hd44780_t **lcd);
```

## Minimal I2C Example

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "hd44780.h"

void app_main(void)
{
    lcd_bus_hd44780_t *bus = lcd_bus_pcf8574_i2c_create(
        I2C_NUM_0,
        0x27,
        GPIO_NUM_21,
        GPIO_NUM_22,
        100000
    );

    if (!bus) {
        return;
    }

    hd44780_t *lcd = lcd_init(bus, HD44780_GEOMETRY_20X4, true);
    if (!lcd) {
        if (bus->destroy) {
            bus->destroy(&bus);
        }
        return;
    }

    lcd_backlight_on(lcd);
    lcd_clear_screen(lcd);
    lcd_set_cursor(lcd, 0, 0);
    lcd_write_str(lcd, "Hello, HD44780");
    lcd_set_cursor(lcd, 0, 1);
    lcd_write_str(lcd, "over I2C");
}
```

## Shared I2C Bus Example

```c
#include "driver/i2c_master.h"
#include "hd44780.h"

void app_main(void)
{
    i2c_master_bus_config_t bus_config = {
        .i2c_port = I2C_NUM_0,
        .sda_io_num = GPIO_NUM_21,
        .scl_io_num = GPIO_NUM_22,
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };

    i2c_master_bus_handle_t i2c_bus = NULL;
    if (i2c_new_master_bus(&bus_config, &i2c_bus) != ESP_OK) {
        return;
    }

    lcd_bus_hd44780_t *bus = lcd_bus_pcf8574_i2c_create_on_bus(i2c_bus, 0x27, 100000);
    if (!bus) {
        i2c_del_master_bus(i2c_bus);
        return;
    }

    hd44780_t *lcd = lcd_init(bus, HD44780_GEOMETRY_20X4, true);
    if (!lcd) {
        bus->destroy(&bus);
        i2c_del_master_bus(i2c_bus);
        return;
    }

    lcd_write_str(lcd, "Shared I2C bus");
    lcd_deinit(&lcd);
    i2c_del_master_bus(i2c_bus);
}
```

## Minimal GPIO 4-Bit Example

```c
#include "hd44780.h"

void app_main(void)
{
    lcd_bus_hd44780_t *bus = lcd_bus_gpio4_create(
        GPIO_NUM_4,
        GPIO_NUM_5,
        GPIO_NUM_18,
        GPIO_NUM_19,
        GPIO_NUM_21,
        GPIO_NUM_22
    );

    if (!bus) {
        return;
    }

    hd44780_t *lcd = lcd_init(bus, HD44780_GEOMETRY_16X2, true);
    if (!lcd) {
        if (bus->destroy) {
            bus->destroy(&bus);
        }
        return;
    }

    lcd_clear_screen(lcd);
    lcd_write_str(lcd, "GPIO 4-bit mode");
}
```

## Common Operations

Most operations have two forms. The `lcd_*()` convenience helpers keep the API compact and log/ignore runtime bus errors. The matching `lcd_try_*()` helpers return `esp_err_t` so applications can detect an I2C/device failure after initialization.

Display and cursor control:

- `lcd_display_on()` / `lcd_display_off()`
- `lcd_try_display_on()` / `lcd_try_display_off()`
- `lcd_cursor_on()` / `lcd_cursor_off()`
- `lcd_try_cursor_on()` / `lcd_try_cursor_off()`
- `lcd_blink_on()` / `lcd_blink_off()`
- `lcd_try_blink_on()` / `lcd_try_blink_off()`
- `lcd_backlight_on()` / `lcd_backlight_off()`
- `lcd_try_backlight_on()` / `lcd_try_backlight_off()`

Cursor and text:

- `lcd_set_cursor()`
- `lcd_try_set_cursor()`
- `lcd_home()`
- `lcd_try_home()`
- `lcd_clear_screen()`
- `lcd_try_clear_screen()`
- `lcd_write_char()`
- `lcd_try_write_char()`
- `lcd_write_str()`
- `lcd_try_write_str()`
- `lcd_write_strn()`
- `lcd_try_write_strn()`

Entry mode:

- `lcd_left_to_right()`
- `lcd_try_left_to_right()`
- `lcd_right_to_left()`
- `lcd_try_right_to_left()`
- `lcd_autoscroll_on()`
- `lcd_try_autoscroll_on()`
- `lcd_autoscroll_off()`
- `lcd_try_autoscroll_off()`

Custom characters:

- `lcd_write_cgram()` writes one 8-byte character pattern into CGRAM slot 0-7
- `lcd_try_write_cgram()` returns the first runtime bus error while writing CGRAM

## Thread Safety

This component is not thread-safe.

Use one FreeRTOS task as the sole owner of the display. If other tasks need to update the LCD, send messages to that display task instead of calling `lcd_*()` from multiple tasks.

## Troubleshooting

- No output on I2C: verify the backpack address, SDA/SCL pins, power, and pull-ups.
- Initialization fails immediately: confirm the selected geometry matches the actual module.
- Text appears on unexpected rows: double-check the geometry, especially for 16x4 and 20x4 modules.
- Backlight helpers appear to do nothing: direct GPIO backends do not implement backlight control.
- Random display corruption: keep all LCD access in one task.
