# Changes

## 0.1.1 - 2026-10-05

### Fixed

- Corrected the 8-bit function-set command so GPIO 8-bit initialization sends `0x38` (8-bit, 2-line, 5x8) instead of the invalid `0x18`.
- Made `lcd_init(..., true)` consume the bus on every failure path, including allocation failure; examples no longer destroy the bus after a failed init.
- Validated the LCD handle and string argument in `lcd_try_write_str()` and `lcd_try_write_strn()` so `NULL` handles and `NULL` strings are rejected. Empty strings and zero-length writes remain valid no-ops.
- Rejected duplicate RS, EN, and data pins in the direct-GPIO bus factories and validated all pins before configuring any, so an invalid pin cannot leave the other pins reconfigured.
- Raised the minimum ESP-IDF version to 5.3 to match the `esp_driver_i2c` and `esp_driver_gpio` component requirements.

### Changed

- Documented the cursor-clamping behavior of `lcd_set_cursor()` / `lcd_try_set_cursor()` and the `owns_bus` ownership contract.

## 0.1.0 - 2026-09-18

### Added

- Added `lcd_try_*()` runtime API variants that return `esp_err_t` for applications that need to detect LCD bus failures after initialization.
- Added `lcd_bus_pcf8574_i2c_create_on_bus()` for attaching a PCF8574 LCD backpack to an externally owned ESP-IDF I2C master bus.

### Changed

- Kept the existing `lcd_*()` helpers as convenience wrappers that intentionally ignore runtime bus errors after logging.
- Made PCF8574 I2C bus reuse validate that repeated calls on the same I2C port use the same SDA/SCL pins.
- Documented the supported fixed PCF8574 backpack mapping, including P3 active-high backlight polarity.

### Fixed

- Fixed cleanup when PCF8574 bus creation fails after creating an internal I2C master bus.
- Avoided configuring direct-GPIO pins before `gpio_bus_t` allocation succeeds.
