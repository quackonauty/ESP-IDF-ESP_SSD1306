| Supported Targets | ESP32 | ESP32-C2 | ESP32-C3 | ESP32-C5 | ESP32-C6 | ESP32-C61 | ESP32-H2 | ESP32-S3 |
| ----------------- | ----- | -------- | -------- | -------- | -------- | --------- | -------- | -------- |

# SSD1306 OLED I2C Display Driver for ESP-IDF

## I. Overview

SSD1306 OLED I2C Display Driver is a lightweight and efficient component written entirely in C for the ESP-IDF framework. It provides easy control of SSD1306-based OLED displays over the I2C bus and integrates seamlessly into any ESP32 project.

✅ Tested and Proven: Successfully validated on the ESP32 (38 and 30-pin variants) with a 128x64, 0.96" OLED display. Minor adjustments may be needed for other boards or resolutions.

### Component Background

This component is an improved version of a previous implementation available [here](https://github.com/quackonaut/ESP-IDF-I2C_SSD1306). The main difference is that the earlier version used the deprecated `i2c.h` header, while this version uses the modern `i2c_master.h` driver, ensuring better functionality and compatibility with future ESP-IDF releases.

**Legacy ESP-IDF I2C Driver:**

- `i2c.h`: Header file for the deprecated legacy I2C API.

**New ESP-IDF I2C Driver:**

- `i2c_master.h`: Provides APIs for I2C master mode using the new standard driver.

- `i2c_slave.h`: Provides APIs for I2C slave mode using the new standard driver.

## II. Integration into Your Project

Follow these steps to integrate the component into your ESP-IDF project.

### 1. Hardware Setup

- **OLED Connection**:  
    By default, the ESP32 uses **GPIO22** for `SCL` (clock) and **GPIO21** for `SDA` (data). Connect these pins to the corresponding lines on your OLED display. If you prefer to use different GPIOs, make sure they are available and not reserved by other peripherals. Refer to your ESP32 board’s pinout for details.

    ⚙️ **Note**: For stable I2C communication, ensure that both SCL and SDA lines have pull-up resistors. If your chosen GPIOs lack internal pull-ups, add external ones (typically 4.7 kΩ).

### 2. Adding the Component as a Dependency

To integrate the component into your project, follow these steps:

1.  Clean your project before updating dependencies to avoid potential build issues. (**OPTIONAL BUT RECOMMENDED**)

    ``` bash
    idf.py clean
    ```

2. Add the component dependency by running:

    ```bash
    idf.py add-dependency --git https://github.com/quackonauty/ESP-IDF-ESP_SSD1306.git --git-ref v1.0.1 quackonauty/esp_ssd1306
    ```

    Or include it as a dependency in the idf_component.yml file, which is typically located in the project's main directory. If this file does not exist, simply create a new one with that name and add the component dependency as shown below:

    ```yaml
    dependencies:
    quackonauty/esp_ssd1306:
        git: https://github.com/quackonauty/ESP-IDF-ESP_SSD1306.git
        version: v1.0.1
    ```

3. After adding the dependency, update the components by running:

    ``` bash
    idf.py update-dependencies
    ```
4. Finally, rebuild your project:

    ``` bash
    idf.py build
    ```

## III. Component Workflow and Usage

### 1. Include Required Headers

Before proceeding with configuration and initialization, make sure to include the necessary headers in your application:

``` c
#include <driver/i2c_master.h>      // ESP-IDF I2C master driver
#include <esp_ssd1306.h>            // SSD1306 component header
```

### 2. I2C Master Initialization

Configure the I2C master bus with the following parameters:

``` c
/* I2C Master Configuration */
static i2c_master_bus_config_t i2c_master_bus_config = {
    .i2c_port = I2C_NUM_0,
    .scl_io_num = GPIO_NUM_22,
    .sda_io_num = GPIO_NUM_21,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true};
static i2c_master_bus_handle_t i2c_master_bus;
```

⚙️ **Note**: This configuration is the recommended default, including port number, clock source, glitch filter, internal pull-ups, and GPIO assignments (**GPIO22** for `SCL` and **GPIO21** for `SDA`). However, all parameters are fully customizable based on your hardware design and project requirements. If you choose different GPIO pins or settings, ensure they are updated accordingly in your code and compatible with your board. This flexibility is also detailed in **Section II**, "**Integration into Your Project**" → **Subsection 1**, "**Hardware Setup**".

### 3. SSD1306 Driver Initialization

After setting up the I2C bus, initialize the SSD1306 display with the following configuration:

``` c
static i2c_ssd1306_config_t i2c_ssd1306_config = {
    .i2c_device_address = 0x3C,
    .i2c_scl_speed_hz = 400000,
    .width = 128,
    .height = 64,
    .wise = SSD1306_TOP_TO_BOTTOM};
static i2c_ssd1306_handle_t i2c_ssd1306;
```

⚙️ **Note**: The values above represent common defaults for most 128x64 OLED displays. However, each parameter is fully customizable to match your specific display model and project requirements. Ensure you adjust the device address, resolution, and I2C speed according to your hardware specifications.

### 4. Initialization of I2C and SSD1306 Using Defined Configurations

Integrate both the I2C master and SSD1306 initializations into your application’s main entry point (app_main) using the previously defined configuration structures:

``` c
void app_main(void)
{
    /* I2C Master */
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_master_bus_config, &i2c_master_bus));
    /* SSD1306 */
    ESP_ERROR_CHECK(i2c_ssd1306_init(i2c_master_bus, &i2c_ssd1306_config, &i2c_ssd1306));
}
```

⚙️ **Note**: The function i2c_new_master_bus() is part of the ESP-IDF framework and requires including the header `<driver/i2c_master.h>`. The function i2c_ssd1306_init() is defined in this SSD1306 component, and requires including `<i2c_ssd1306.h>`. Make sure both headers are included at the top of your source file to avoid compilation errors.

### 5. Functions and Methods of the Driver

This driver provides a wide range of functions for drawing pixels, rendering text (including integers and floating-point values), and displaying images. It operates using an i2c_ssd1306_handle_t structure, which acts as a logical "object" that manages an internal buffer. This buffer organizes data into **pages** and **segments**, mirroring the architecture of the SSD1306 controller. For more technical details, the datasheet for the SSD1306 controller can be found [here](https://cdn-shop.adafruit.com/datasheets/SSD1306.pdf).

The display workflow is divided into two functional groups:

**A. Functions for Modifying the Internal Buffer**

These functions operate on the **internal buffer**, which stores graphical and textual data in memory before being pushed to the display.

  - `i2c_ssd1306_buffer_fill(i2c_ssd1306, fill)`: Fills the entire internal buffer with either all white or all black pixels.
    - `fill = true`: sets all segments to `0xFF` (all pixels ON).
    - `fill = false`: sets all segments to `0x00` (all pixels OFF).

  - `i2c_ssd1306_buffer_fill_pixel(i2c_ssd1306, x, y, fill)`: Sets or clears a single pixel at the specified (x, y) position.
    - Bounds checking ensures `x < width` and `y < height`.
    - If `fill = true`, the bit is set; otherwise, it is cleared.

  - `i2c_ssd1306_buffer_fill_space(i2c_ssd1306, x1, x2, y1, y2, fill)`: Fills or clears a rectangular area in the buffer.
    - Coordinates define a box from top-left `(x1, y1)` to bottom-right `(x2, y2)`.
    - Pixels outside the screen range or if `x1 > x2` / `y1 > y2` trigger an error.
    - If `fill = true`, the bits are set; otherwise, they are cleared.

  - `i2c_ssd1306_buffer_text(i2c_ssd1306, x, y, text, invert)`: Draws a string on the buffer using an 8x8 monospaced font.
    - The string starts at `(x, y)`, with vertical offset handled across pages.
    - If `invert = true`, the font bits are inverted.
    - **Note**: This function does not clear the area first — for dynamic text (e.g. sensor readings), you must clear the region manually before redrawing.

  - `i2c_ssd1306_buffer_int(i2c_ssd1306, x, y, value, invert)`: Converts an int value to a string and renders it with buffer_text().
    - Same behavior and limitations as the text function.

  - `i2c_ssd1306_buffer_float(i2c_ssd1306, x, y, value, decimals, invert)`: Formats and displays a floating-point number with a fixed number of decimal places.
    - Renders with buffer_text(), so clearing the previous area is necessary to avoid overlap artifacts.

  - `i2c_ssd1306_buffer_image(i2c_ssd1306, x, y, image, img_width, img_height, invert)`: Renders a monochrome image at the specified position.
    - The `image` array must be organized in vertical bytes (columns per page).
    - Handles clipping if the image overflows screen bounds.
    - Inversion flips image bits if `invert = true`.
    - **Note**: As with text, the area is not cleared before rendering; you must clear it if needed.

**B. Functions for Transferring the Internal Buffer to the SSD1306 RAM**

After drawing into the internal buffer, these functions commit changes to the actual display memory of the SSD1306 controller:

- `i2c_ssd1306_segment_to_ram(i2c_ssd1306, page, segment)`: Transfers a single segment (column) from a specific page of the buffer to the SSD1306 display RAM.

- `i2c_ssd1306_segments_to_ram(i2c_ssd1306, page, segment_start, segment_end)`: Transfers a continuous range of segments (columns) from a single page of the buffer to the display.

- `i2c_ssd1306_page_to_ram(i2c_ssd1306, page)`: Transfers an entire page (a horizontal band of 8 vertical pixels across all columns) from the buffer to the display RAM.

- `i2c_ssd1306_pages_to_ram(i2c_ssd1306, page_start, page_end)`: Transfers a range of full pages (horizontal pixel bands) from the buffer to the display RAM.

- `i2c_ssd1306_buffer_to_ram(i2c_ssd1306)`: Transfers the entire contents of the internal buffer to the SSD1306 display, performing a full screen refresh.

## IV. Basic Example

This example demonstrates how to initialize the SSD1306 OLED display using the ESP-IDF I2C master driver, draw content to the internal buffer (including text, numbers, and images), and finally transfer the buffer to the display RAM to visualize the output. The example showcases text rendering, image display, and screen clearing operations.

```c
#include <stdio.h>
#include <esp_log.h>
#include <driver/i2c_master.h>
#include <esp_ssd1306.h>

static const uint8_t ssd1306_esp_logo_img[4][32] = {
    {0x00, 0x00, 0x00, 0xC0, 0x60, 0x18, 0x00, 0x00, 0x70, 0x78, 0x78, 0x78, 0xF8, 0xF8, 0xF0, 0xF0,
     0xF2, 0xE6, 0xE6, 0xCE, 0x9E, 0x9C, 0x3C, 0x78, 0xF8, 0xF0, 0xE0, 0xC0, 0x80, 0x00, 0x00, 0x00},
    {0x00, 0xFC, 0x07, 0x60, 0xF8, 0xFC, 0xFE, 0xFE, 0x9E, 0x9E, 0x9E, 0x3E, 0x3E, 0x7C, 0x7C, 0xF9,
     0xF9, 0xF3, 0xE7, 0xCF, 0x9F, 0x3F, 0x7F, 0xFE, 0xFC, 0xF1, 0xE3, 0x8F, 0x1F, 0xFE, 0xF8, 0x00},
    {0x00, 0x07, 0x3C, 0xE0, 0x81, 0x03, 0x07, 0xC7, 0xE7, 0xC7, 0xCF, 0x1F, 0x7F, 0xFE, 0xFC, 0xF8,
     0xE1, 0x07, 0x3F, 0xFF, 0xFF, 0xFE, 0xF0, 0x01, 0x0F, 0xFF, 0xFF, 0xFF, 0x3C, 0x00, 0x00, 0x00},
    {0x00, 0x00, 0x00, 0x00, 0x01, 0x03, 0x06, 0x0D, 0x19, 0x11, 0x30, 0x20, 0x24, 0x4F, 0x4F, 0x4F,
     0x4F, 0x40, 0x40, 0x4F, 0x4F, 0x6F, 0x27, 0x20, 0x10, 0x10, 0x08, 0x0C, 0x04, 0x00, 0x00, 0x00},
};

/* I2C Master */
static const i2c_master_bus_config_t i2c_master_bus_config = {
    .i2c_port = I2C_NUM_0,
    .scl_io_num = GPIO_NUM_22,
    .sda_io_num = GPIO_NUM_21,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true};
static i2c_master_bus_handle_t i2c_master_bus;

/* SSD1306 */
static const i2c_ssd1306_config_t i2c_ssd1306_config = {
    .i2c_device_address = 0x3C,
    .i2c_scl_speed_hz = 400000,
    .width = 128,
    .height = 64,
    .wise = SSD1306_TOP_TO_BOTTOM};
static i2c_ssd1306_handle_t i2c_ssd1306;

void app_main(void)
{
    ESP_ERROR_CHECK(i2c_new_master_bus(&i2c_master_bus_config, &i2c_master_bus));
    ESP_ERROR_CHECK(i2c_ssd1306_init(i2c_master_bus, &i2c_ssd1306_config, &i2c_ssd1306));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_image(&i2c_ssd1306, 48, 16, (const uint8_t *)ssd1306_esp_logo_img, 32, 32, false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_to_ram(&i2c_ssd1306));
    vTaskDelay(3000 / portTICK_PERIOD_MS);

    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill(&i2c_ssd1306, false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 12, 0, "Hello, World!", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill_space(&i2c_ssd1306, 0, 127, 8, 8, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 10, "ABCDEFGHIJKLMNOP", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 18, "QRSTUVWXYZabcdfe", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT( i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 26, "ghijklmnopqrstuv", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 34, "wxyz1234567890!(", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 42, ")-=+[]{};:'\",.<>", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_text(&i2c_ssd1306, 0, 50, "?/\\|_`~@#$%^&*", false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill_space(&i2c_ssd1306, 0, 127, 58, 63, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_to_ram(&i2c_ssd1306));
    vTaskDelay(3000 / portTICK_PERIOD_MS);

    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill(&i2c_ssd1306, false));
    int x = 1234567890;
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_int(&i2c_ssd1306, 24, 0, x, false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_float(&i2c_ssd1306, 20, 8, x / 100000.0, 5, false));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_image(&i2c_ssd1306, 48, 20, (const uint8_t *)ssd1306_esp_logo_img, 32, 32, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_fill_space(&i2c_ssd1306, 0, 127, 58, 63, true));
    ESP_ERROR_CHECK_WITHOUT_ABORT(i2c_ssd1306_buffer_to_ram(&i2c_ssd1306));
}
```

## V. Convert an Image to a C Array for OLED Display with Python

To render static graphics or logos on the OLED display using the `i2c_ssd1306_buffer_image(i2c_ssd1306, x, y, image, img_width, img_height, invert)` function, the following Python script allows you to convert standard image files (PNG, JPG) into `uint8_t` C arrays compatible with the SSD1306 format.

💡 This script is an optional utility intended to assist in preparing image data for display. It is not required for using the component but is highly useful when integrating custom graphics into embedded applications.

### Features

- **Automatic Thresholding**: Uses Otsu's method to determine the best threshold for binarization.
- **Invert Colors Option**: Allows inversion of black/white to support display logic polarity.
- **C Array Formatting**: Outputs the image as a properly formatted C array.
- **Customizable Resolution**: Supports resizing to match OLED dimensions (e.g., 128x64 or 64x64).
- **Console Preview**: Simulates the image output directly in your terminal.

### Requirements

Make sure you have Python 3 and the **Pillow** library installed:

``` bash
pip install pillow
```

### Usage Instructions

1. Open the script in any text editor
2. Set the image path:
   ``` python
    # 🪟 Windows
    image_path = r"C:\Path\to\your\image.png"

    # 🍎 macOS
    image_path = "/Users/yourname/Path/to/your/image.png"

    # 🐧 Linux
    image_path = "/home/yourname/Path/to/your/image.png"
   ```
3. Set the output dimensions to match your display:
   ``` python
   width, height = 64, 64
   ```
4. Run the script:
   ``` bash
   python ImageToCArray.py
   ```

5. The script will:

   - Show a preview of the processed image
   - Print the byte array in console using characters
   - Output a C array that you can copy into your firmware source code

### Function Descriptors

- `imageToByteArray(image_path, width, height, invert=False, threshold=None)`: Converts an image to a 2D list of bytes for OLED display.
  - `image_path`: path to the image
  - `width/height`: resolution (e.g., 128x64)
  - `invert`: inverts black and white logic if needed
  - `threshold`: manually set threshold; uses Otsu’s method if None
  
  Returns: List[List[int]] of bytes

- `formatAsCArray(py_array, c_array_name="img", line_break=16)`: Formats a 2D list of bytes into a printable C array string.
  - `py_array (List[List[int]])`: The input 2D byte array.
  - `c_array_name (str, optional)`: The name of the C array. Default is "img".
  - `line_break (int, optional)`: Number of bytes per line in the output. Default is 16.

  Returns: A string representing the formatted C array.

- `printArray(py_array)`: Prints a console representation of the image using blocks (█) to simulate pixels.
  - `py_array (List[List[int]])`: The 2D byte array to visualize.

### Full Source Code

``` python
from PIL import Image

def printArray(py_array):
    """
    Prints a 2D byte array as a monochrome image, simulating the OLED display.

    @param py_array: List[List[int]]
        A 2D list of byte blocks representing the monochrome image.

    @return: None
    """
    for array in py_array:
        segment = ["" for _ in range(8)]

        for byte in array:
            bin = format(byte, "08b")
            for i in range(8):
                segment[7 - i] += "█" if bin[i] == "1" else " "

        for linea in segment:
            print(linea)

def formatAsCArray(py_array, c_array_name="img", line_break=16):
    """
    Formats a 2D byte array as a C array string.

    @param py_array: List[List[int]]
        A 2D list of byte blocks representing the monochrome image.
    @param c_array_name: str, optional
        Name of the C array. Defaults to "img".
    @param line_break: int, optional
        Number of bytes per line. Defaults to 16.

    @return: str
        A string representing the C array.
    """
    rows = len(py_array)
    cols = len(py_array[0]) if rows > 0 else 0

    c_array = f"uint8_t {c_array_name}[{rows}][{cols}] = {{\n"
    for row in py_array:
        c_array += "    {"
        for i in range(0, len(row), line_break):
            chunk = row[i : i + line_break]
            c_array += ", ".join(f"0x{byte:02X}" for byte in chunk)
            if i + line_break < len(row):
                c_array += ",\n     "
        c_array += "},\n"
    c_array = c_array.rstrip(",\n") + "\n};"
    return c_array

def imageToByteArray(image_path, width, height, invert=False, threshold=None):
    """
    Converts an image to a byte array representation for use in monochrome displays.

    @param image_path: str
        Path to the image file.
    @param width: int
        Width of the output image, maximum for SSD1306 is 128.
    @param height: int
        Height of the output image, maximum for SSD1306 is 64.
    @param invert: bool, optional
        If True, inverts the black and white colors in the final image (black becomes white and vice versa).
        Useful for displays where pixel logic is reversed. Defaults to False.
    @param threshold: int, optional
        Pixel intensity threshold (0-255) for binarization. Pixels above this value become white (1),
        and those below become black (0). If None, the script automatically determines the optimal threshold using Otsu's method.

    @return: List[List[int]]
        A 2D list of byte blocks representing the monochrome image.
    """
    image = Image.open(image_path).convert("L")
    image = image.resize((width, height))

    if threshold is None:
        histogram = image.histogram()
        total_pixels = sum(histogram)
        sum_brightness = sum(i * histogram[i] for i in range(256))
        sum_b = 0
        max_variance = 0
        threshold = 0
        w_b = 0

        for t in range(256):
            w_b += histogram[t]
            w_f = total_pixels - w_b
            if w_b == 0 or w_f == 0:
                continue

            sum_b += t * histogram[t]
            m_b = sum_b / w_b
            m_f = (sum_brightness - sum_b) / w_f
            variance_between = w_b * w_f * (m_b - m_f) ** 2

            if variance_between > max_variance:
                max_variance = variance_between
                threshold = t

    print(f"Threshold: {threshold}")

    image = image.point(lambda p: 255 if p > threshold else 0, mode="1")
    if invert:
        image = image.point(lambda p: 255 if p == 0 else 0, mode="1")
    image.show()

    pixels = list(image.getdata())

    rows = [pixels[i * width : (i + 1) * width] for i in range(height)]

    byte_blocks = []
    for group_start in range(0, height, 8):
        group = rows[group_start : group_start + 8]
        block = []
        for col in range(width):
            byte = 0
            for row_idx, row in enumerate(group):
                if row[col] == 0:
                    byte |= 1 << row_idx
            block.append(byte)
        byte_blocks.append(block)

    return byte_blocks

image_path = r"C:\Path\to\your\image.png"
width, height = 64, 64

byte_array = imageToByteArray(image_path, width, height, True)
c_array = formatAsCArray(byte_array)
printArray(byte_array)
print(c_array)
```

## VI. Do you have any questions, suggestions, or have you found any errors?

If you have any questions or suggestions about the operation of the component, or if you encountered any errors while compiling it on your ESP32 board, please don't hesitate to leave your comment.