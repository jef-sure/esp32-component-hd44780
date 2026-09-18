set(hd44780_root "${CMAKE_CURRENT_LIST_DIR}/../../../..")

idf_component_register(
    SRCS "${hd44780_root}/src/hd44780.c"
    REQUIRES driver esp_driver_i2c esp_driver_gpio esp_timer
    INCLUDE_DIRS "${hd44780_root}/include"
)