set(srcs "src/hd44780.c")

idf_component_register(
	SRCS "${srcs}"
	REQUIRES esp_driver_i2c esp_driver_gpio
	INCLUDE_DIRS "include"
)
