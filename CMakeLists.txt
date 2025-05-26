set(srcs "src/esp_ssd1306.c")
set(include "include")
set(priv_requires driver)

idf_component_register(
    SRCS ${srcs}
    INCLUDE_DIRS ${include}
    PRIV_REQUIRES "${priv_requires}"
)