# Changes

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
