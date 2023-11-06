# xmrig源码解析 13

# `src/3rdparty/fmt/color.h`

以上是一个包含常见颜色代码的列表。它们都可以用来表示不同的颜色，包括红色、绿色、蓝色、黄色、紫色等等。这些颜色代码在计算机图形学、编程语言等方面有着广泛的应用。


```cpp
// Formatting library for C++ - color support
//
// Copyright (c) 2018 - present, Victor Zverovich and fmt contributors
// All rights reserved.
//
// For the license information refer to format.h.

#ifndef FMT_COLOR_H_
#define FMT_COLOR_H_

#include "format.h"

FMT_BEGIN_NAMESPACE

enum class color : uint32_t {
  alice_blue = 0xF0F8FF,               // rgb(240,248,255)
  antique_white = 0xFAEBD7,            // rgb(250,235,215)
  aqua = 0x00FFFF,                     // rgb(0,255,255)
  aquamarine = 0x7FFFD4,               // rgb(127,255,212)
  azure = 0xF0FFFF,                    // rgb(240,255,255)
  beige = 0xF5F5DC,                    // rgb(245,245,220)
  bisque = 0xFFE4C4,                   // rgb(255,228,196)
  black = 0x000000,                    // rgb(0,0,0)
  blanched_almond = 0xFFEBCD,          // rgb(255,235,205)
  blue = 0x0000FF,                     // rgb(0,0,255)
  blue_violet = 0x8A2BE2,              // rgb(138,43,226)
  brown = 0xA52A2A,                    // rgb(165,42,42)
  burly_wood = 0xDEB887,               // rgb(222,184,135)
  cadet_blue = 0x5F9EA0,               // rgb(95,158,160)
  chartreuse = 0x7FFF00,               // rgb(127,255,0)
  chocolate = 0xD2691E,                // rgb(210,105,30)
  coral = 0xFF7F50,                    // rgb(255,127,80)
  cornflower_blue = 0x6495ED,          // rgb(100,149,237)
  cornsilk = 0xFFF8DC,                 // rgb(255,248,220)
  crimson = 0xDC143C,                  // rgb(220,20,60)
  cyan = 0x00FFFF,                     // rgb(0,255,255)
  dark_blue = 0x00008B,                // rgb(0,0,139)
  dark_cyan = 0x008B8B,                // rgb(0,139,139)
  dark_golden_rod = 0xB8860B,          // rgb(184,134,11)
  dark_gray = 0xA9A9A9,                // rgb(169,169,169)
  dark_green = 0x006400,               // rgb(0,100,0)
  dark_khaki = 0xBDB76B,               // rgb(189,183,107)
  dark_magenta = 0x8B008B,             // rgb(139,0,139)
  dark_olive_green = 0x556B2F,         // rgb(85,107,47)
  dark_orange = 0xFF8C00,              // rgb(255,140,0)
  dark_orchid = 0x9932CC,              // rgb(153,50,204)
  dark_red = 0x8B0000,                 // rgb(139,0,0)
  dark_salmon = 0xE9967A,              // rgb(233,150,122)
  dark_sea_green = 0x8FBC8F,           // rgb(143,188,143)
  dark_slate_blue = 0x483D8B,          // rgb(72,61,139)
  dark_slate_gray = 0x2F4F4F,          // rgb(47,79,79)
  dark_turquoise = 0x00CED1,           // rgb(0,206,209)
  dark_violet = 0x9400D3,              // rgb(148,0,211)
  deep_pink = 0xFF1493,                // rgb(255,20,147)
  deep_sky_blue = 0x00BFFF,            // rgb(0,191,255)
  dim_gray = 0x696969,                 // rgb(105,105,105)
  dodger_blue = 0x1E90FF,              // rgb(30,144,255)
  fire_brick = 0xB22222,               // rgb(178,34,34)
  floral_white = 0xFFFAF0,             // rgb(255,250,240)
  forest_green = 0x228B22,             // rgb(34,139,34)
  fuchsia = 0xFF00FF,                  // rgb(255,0,255)
  gainsboro = 0xDCDCDC,                // rgb(220,220,220)
  ghost_white = 0xF8F8FF,              // rgb(248,248,255)
  gold = 0xFFD700,                     // rgb(255,215,0)
  golden_rod = 0xDAA520,               // rgb(218,165,32)
  gray = 0x808080,                     // rgb(128,128,128)
  green = 0x008000,                    // rgb(0,128,0)
  green_yellow = 0xADFF2F,             // rgb(173,255,47)
  honey_dew = 0xF0FFF0,                // rgb(240,255,240)
  hot_pink = 0xFF69B4,                 // rgb(255,105,180)
  indian_red = 0xCD5C5C,               // rgb(205,92,92)
  indigo = 0x4B0082,                   // rgb(75,0,130)
  ivory = 0xFFFFF0,                    // rgb(255,255,240)
  khaki = 0xF0E68C,                    // rgb(240,230,140)
  lavender = 0xE6E6FA,                 // rgb(230,230,250)
  lavender_blush = 0xFFF0F5,           // rgb(255,240,245)
  lawn_green = 0x7CFC00,               // rgb(124,252,0)
  lemon_chiffon = 0xFFFACD,            // rgb(255,250,205)
  light_blue = 0xADD8E6,               // rgb(173,216,230)
  light_coral = 0xF08080,              // rgb(240,128,128)
  light_cyan = 0xE0FFFF,               // rgb(224,255,255)
  light_golden_rod_yellow = 0xFAFAD2,  // rgb(250,250,210)
  light_gray = 0xD3D3D3,               // rgb(211,211,211)
  light_green = 0x90EE90,              // rgb(144,238,144)
  light_pink = 0xFFB6C1,               // rgb(255,182,193)
  light_salmon = 0xFFA07A,             // rgb(255,160,122)
  light_sea_green = 0x20B2AA,          // rgb(32,178,170)
  light_sky_blue = 0x87CEFA,           // rgb(135,206,250)
  light_slate_gray = 0x778899,         // rgb(119,136,153)
  light_steel_blue = 0xB0C4DE,         // rgb(176,196,222)
  light_yellow = 0xFFFFE0,             // rgb(255,255,224)
  lime = 0x00FF00,                     // rgb(0,255,0)
  lime_green = 0x32CD32,               // rgb(50,205,50)
  linen = 0xFAF0E6,                    // rgb(250,240,230)
  magenta = 0xFF00FF,                  // rgb(255,0,255)
  maroon = 0x800000,                   // rgb(128,0,0)
  medium_aquamarine = 0x66CDAA,        // rgb(102,205,170)
  medium_blue = 0x0000CD,              // rgb(0,0,205)
  medium_orchid = 0xBA55D3,            // rgb(186,85,211)
  medium_purple = 0x9370DB,            // rgb(147,112,219)
  medium_sea_green = 0x3CB371,         // rgb(60,179,113)
  medium_slate_blue = 0x7B68EE,        // rgb(123,104,238)
  medium_spring_green = 0x00FA9A,      // rgb(0,250,154)
  medium_turquoise = 0x48D1CC,         // rgb(72,209,204)
  medium_violet_red = 0xC71585,        // rgb(199,21,133)
  midnight_blue = 0x191970,            // rgb(25,25,112)
  mint_cream = 0xF5FFFA,               // rgb(245,255,250)
  misty_rose = 0xFFE4E1,               // rgb(255,228,225)
  moccasin = 0xFFE4B5,                 // rgb(255,228,181)
  navajo_white = 0xFFDEAD,             // rgb(255,222,173)
  navy = 0x000080,                     // rgb(0,0,128)
  old_lace = 0xFDF5E6,                 // rgb(253,245,230)
  olive = 0x808000,                    // rgb(128,128,0)
  olive_drab = 0x6B8E23,               // rgb(107,142,35)
  orange = 0xFFA500,                   // rgb(255,165,0)
  orange_red = 0xFF4500,               // rgb(255,69,0)
  orchid = 0xDA70D6,                   // rgb(218,112,214)
  pale_golden_rod = 0xEEE8AA,          // rgb(238,232,170)
  pale_green = 0x98FB98,               // rgb(152,251,152)
  pale_turquoise = 0xAFEEEE,           // rgb(175,238,238)
  pale_violet_red = 0xDB7093,          // rgb(219,112,147)
  papaya_whip = 0xFFEFD5,              // rgb(255,239,213)
  peach_puff = 0xFFDAB9,               // rgb(255,218,185)
  peru = 0xCD853F,                     // rgb(205,133,63)
  pink = 0xFFC0CB,                     // rgb(255,192,203)
  plum = 0xDDA0DD,                     // rgb(221,160,221)
  powder_blue = 0xB0E0E6,              // rgb(176,224,230)
  purple = 0x800080,                   // rgb(128,0,128)
  rebecca_purple = 0x663399,           // rgb(102,51,153)
  red = 0xFF0000,                      // rgb(255,0,0)
  rosy_brown = 0xBC8F8F,               // rgb(188,143,143)
  royal_blue = 0x4169E1,               // rgb(65,105,225)
  saddle_brown = 0x8B4513,             // rgb(139,69,19)
  salmon = 0xFA8072,                   // rgb(250,128,114)
  sandy_brown = 0xF4A460,              // rgb(244,164,96)
  sea_green = 0x2E8B57,                // rgb(46,139,87)
  sea_shell = 0xFFF5EE,                // rgb(255,245,238)
  sienna = 0xA0522D,                   // rgb(160,82,45)
  silver = 0xC0C0C0,                   // rgb(192,192,192)
  sky_blue = 0x87CEEB,                 // rgb(135,206,235)
  slate_blue = 0x6A5ACD,               // rgb(106,90,205)
  slate_gray = 0x708090,               // rgb(112,128,144)
  snow = 0xFFFAFA,                     // rgb(255,250,250)
  spring_green = 0x00FF7F,             // rgb(0,255,127)
  steel_blue = 0x4682B4,               // rgb(70,130,180)
  tan = 0xD2B48C,                      // rgb(210,180,140)
  teal = 0x008080,                     // rgb(0,128,128)
  thistle = 0xD8BFD8,                  // rgb(216,191,216)
  tomato = 0xFF6347,                   // rgb(255,99,71)
  turquoise = 0x40E0D0,                // rgb(64,224,208)
  violet = 0xEE82EE,                   // rgb(238,130,238)
  wheat = 0xF5DEB3,                    // rgb(245,222,179)
  white = 0xFFFFFF,                    // rgb(255,255,255)
  white_smoke = 0xF5F5F5,              // rgb(245,245,245)
  yellow = 0xFFFF00,                   // rgb(255,255,0)
  yellow_green = 0x9ACD32              // rgb(154,205,50)
};                                     // enum class color

```

以上代码定义了一个名为 `terminal_color` 的枚举类型，它包含 8 个成员变量，每个成员变量都是一个无符号 8 位整数。

这个枚举类型定义了不同的终端颜色，包括黑色、红色、绿色、黄色、蓝色、紫色、白色和一种特殊的黑色，这种黑色的值是 90。

枚举类型是一种面向对象编程的技术，它可以帮助程序员在代码中使用预定义的常量，从而提高代码的可读性和可维护性。通过使用这个枚举类型，程序员可以在程序中使用预定义的颜色名称，而不是用数字或字符串表示颜色常量。例如，在下面的代码中，可以使用 `terminal_color.black` 来表示终端的黑色颜色。


```cpp
enum class terminal_color : uint8_t {
  black = 30,
  red,
  green,
  yellow,
  blue,
  magenta,
  cyan,
  white,
  bright_black = 90,
  bright_red,
  bright_green,
  bright_yellow,
  bright_blue,
  bright_magenta,
  bright_cyan,
  bright_white
};

```

这段代码定义了一个名为 "emphasis" 的枚举类型，它有四个成员，每个成员都是一个 8 位二进制数。这些成员被命名为 "bold"、"italic"、"underline" 和 "strikethrough"，分别对应于成员的值从 0 到 255。

此外，代码中定义了一个名为 "rgb" 的结构体类型，它包含 red、green 和 blue 三个成员变量。这个结构体类型的定义与成员函数的一致，所以可以认为 "rgb" 是 "emphasis" 枚举的成员变量。

整段代码主要定义了一个枚举类型和一个结构体类型，它们都是用来表示颜色的。


```cpp
enum class emphasis : uint8_t {
  bold = 1,
  italic = 1 << 1,
  underline = 1 << 2,
  strikethrough = 1 << 3
};

// rgb is a struct for red, green and blue colors.
// Using the name "rgb" makes some editors show the color in a tooltip.
struct rgb {
  FMT_CONSTEXPR rgb() : r(0), g(0), b(0) {}
  FMT_CONSTEXPR rgb(uint8_t r_, uint8_t g_, uint8_t b_) : r(r_), g(g_), b(b_) {}
  FMT_CONSTEXPR rgb(uint32_t hex)
      : r((hex >> 16) & 0xFF), g((hex >> 8) & 0xFF), b(hex & 0xFF) {}
  FMT_CONSTEXPR rgb(color hex)
      : r((uint32_t(hex) >> 16) & 0xFF),
        g((uint32_t(hex) >> 8) & 0xFF),
        b(uint32_t(hex) & 0xFF) {}
  uint8_t r;
  uint8_t g;
  uint8_t b;
};

```

这段代码定义了一个名为“color_type”的结构体，用于表示颜色。该结构体有以下几种颜色类型：

1. 基本颜色类型：通过一个纯函数类型“color_type()”定义，它是一个没有参数的函数类型，返回一个“color_type”类型的对象。这种类型将生成的对象作为纯函数，返回一个“color_type”类型的对象。

2. 带RGB颜色颜色类型：通过一个纯函数类型“color_type(color rgb_color)”定义。它接受一个“color_type”类型的参数“color rgb_color”，然后使用这个颜色对象生成一个“color_type”类型的对象。在这种情况下，函数类型需要一个参数，返回类型为“color_type”。

3. 带终端颜色颜色类型：通过一个纯函数类型“color_type(terminal_color)”定义。它接受一个“color_type”类型的参数“terminal_color”，然后使用这个颜色对象生成一个“color_type”类型的对象。在这种情况下，函数类型需要一个参数，返回类型为“color_type”。

4. 是/否类型：通过一个“is_rgb”函数类型定义。这个类型是一个“bool”类型，可以通过“is_rgb”函数判断对象是否为RGB颜色。

5. 颜色枚举类型：通过一个“color_union”枚举类型定义。它有“term_color”和“rgb_color”两个成员变量。

这段代码描述了一个结构体，用于表示颜色，提供了不同类型的颜色对象，可以生成或使用不同类型的颜色。


```cpp
namespace detail {

// color is a struct of either a rgb color or a terminal color.
struct color_type {
  FMT_CONSTEXPR color_type() FMT_NOEXCEPT : is_rgb(), value{} {}
  FMT_CONSTEXPR color_type(color rgb_color) FMT_NOEXCEPT : is_rgb(true),
                                                           value{} {
    value.rgb_color = static_cast<uint32_t>(rgb_color);
  }
  FMT_CONSTEXPR color_type(rgb rgb_color) FMT_NOEXCEPT : is_rgb(true), value{} {
    value.rgb_color = (static_cast<uint32_t>(rgb_color.r) << 16) |
                      (static_cast<uint32_t>(rgb_color.g) << 8) | rgb_color.b;
  }
  FMT_CONSTEXPR color_type(terminal_color term_color) FMT_NOEXCEPT : is_rgb(),
                                                                     value{} {
    value.term_color = static_cast<uint8_t>(term_color);
  }
  bool is_rgb;
  union color_union {
    uint8_t term_color;
    uint32_t rgb_color;
  } value;
};
}  // namespace detail

```



This is a C++ class that defines a text style that can have a foreground color and a background color. The `text_style` class has a constructor that takes a hint for the color type of the foreground and background color, and a destructor that resets these colors.

The class has a number of member functions, including `has_background()` and `get_emphasis()`, which check if the text style has a specified background color and an emphasis color, respectively.

The class also has a `get_foreground()` function that returns the color type of the text style's foreground color.

The `text_style` class is intended to be used in situations where a text color needs to be specified, such as for a console or terminal application. It can be instantiated with a combination of a color and an optional emphasis color, as seen in the example code provided.


```cpp
// Experimental text formatting support.
class text_style {
 public:
  FMT_CONSTEXPR text_style(emphasis em = emphasis()) FMT_NOEXCEPT
      : set_foreground_color(),
        set_background_color(),
        ems(em) {}

  FMT_CONSTEXPR text_style& operator|=(const text_style& rhs) {
    if (!set_foreground_color) {
      set_foreground_color = rhs.set_foreground_color;
      foreground_color = rhs.foreground_color;
    } else if (rhs.set_foreground_color) {
      if (!foreground_color.is_rgb || !rhs.foreground_color.is_rgb)
        FMT_THROW(format_error("can't OR a terminal color"));
      foreground_color.value.rgb_color |= rhs.foreground_color.value.rgb_color;
    }

    if (!set_background_color) {
      set_background_color = rhs.set_background_color;
      background_color = rhs.background_color;
    } else if (rhs.set_background_color) {
      if (!background_color.is_rgb || !rhs.background_color.is_rgb)
        FMT_THROW(format_error("can't OR a terminal color"));
      background_color.value.rgb_color |= rhs.background_color.value.rgb_color;
    }

    ems = static_cast<emphasis>(static_cast<uint8_t>(ems) |
                                static_cast<uint8_t>(rhs.ems));
    return *this;
  }

  friend FMT_CONSTEXPR text_style operator|(text_style lhs,
                                            const text_style& rhs) {
    return lhs |= rhs;
  }

  FMT_CONSTEXPR text_style& operator&=(const text_style& rhs) {
    if (!set_foreground_color) {
      set_foreground_color = rhs.set_foreground_color;
      foreground_color = rhs.foreground_color;
    } else if (rhs.set_foreground_color) {
      if (!foreground_color.is_rgb || !rhs.foreground_color.is_rgb)
        FMT_THROW(format_error("can't AND a terminal color"));
      foreground_color.value.rgb_color &= rhs.foreground_color.value.rgb_color;
    }

    if (!set_background_color) {
      set_background_color = rhs.set_background_color;
      background_color = rhs.background_color;
    } else if (rhs.set_background_color) {
      if (!background_color.is_rgb || !rhs.background_color.is_rgb)
        FMT_THROW(format_error("can't AND a terminal color"));
      background_color.value.rgb_color &= rhs.background_color.value.rgb_color;
    }

    ems = static_cast<emphasis>(static_cast<uint8_t>(ems) &
                                static_cast<uint8_t>(rhs.ems));
    return *this;
  }

  friend FMT_CONSTEXPR text_style operator&(text_style lhs,
                                            const text_style& rhs) {
    return lhs &= rhs;
  }

  FMT_CONSTEXPR bool has_foreground() const FMT_NOEXCEPT {
    return set_foreground_color;
  }
  FMT_CONSTEXPR bool has_background() const FMT_NOEXCEPT {
    return set_background_color;
  }
  FMT_CONSTEXPR bool has_emphasis() const FMT_NOEXCEPT {
    return static_cast<uint8_t>(ems) != 0;
  }
  FMT_CONSTEXPR detail::color_type get_foreground() const FMT_NOEXCEPT {
    FMT_ASSERT(has_foreground(), "no foreground specified for this style");
    return foreground_color;
  }
  FMT_CONSTEXPR detail::color_type get_background() const FMT_NOEXCEPT {
    FMT_ASSERT(has_background(), "no background specified for this style");
    return background_color;
  }
  FMT_CONSTEXPR emphasis get_emphasis() const FMT_NOEXCEPT {
    FMT_ASSERT(has_emphasis(), "no emphasis specified for this style");
    return ems;
  }

 private:
  FMT_CONSTEXPR text_style(bool is_foreground,
                           detail::color_type text_color) FMT_NOEXCEPT
      : set_foreground_color(),
        set_background_color(),
        ems() {
    if (is_foreground) {
      foreground_color = text_color;
      set_foreground_color = true;
    } else {
      background_color = text_color;
      set_background_color = true;
    }
  }

  friend FMT_CONSTEXPR_DECL text_style fg(detail::color_type foreground)
      FMT_NOEXCEPT;
  friend FMT_CONSTEXPR_DECL text_style bg(detail::color_type background)
      FMT_NOEXCEPT;

  detail::color_type foreground_color;
  detail::color_type background_color;
  bool set_foreground_color;
  bool set_background_color;
  emphasis ems;
};

```

This is a C++ implementation of the `FMT` macro, which is a feature-length floating-point number. The macro takes a single argument, which is a floating-point number, and returns a string representation of the number using the `std::format` library.

The implementation defines a class `Float` with a single member function `to_string()`, which takes a single argument of type `float` and returns a string representation of the number using the `std::format` library.

The `std::format` library is a C++ library that provides a set of macros for formatting and manipulating strings, and is included in the default `std::insert_1d` header.

The `FMT` macro can be used to generate a string representation of a floating-point number in different formats, including scientific, engineering, and i
```cpp


```
FMT_CONSTEXPR text_style fg(detail::color_type foreground) FMT_NOEXCEPT {
  return text_style(/*is_foreground=*/true, foreground);
}

FMT_CONSTEXPR text_style bg(detail::color_type background) FMT_NOEXCEPT {
  return text_style(/*is_foreground=*/false, background);
}

FMT_CONSTEXPR text_style operator|(emphasis lhs, emphasis rhs) FMT_NOEXCEPT {
  return text_style(lhs) | rhs;
}

namespace detail {

template <typename Char> struct ansi_color_escape {
  FMT_CONSTEXPR ansi_color_escape(detail::color_type text_color,
                                  const char* esc) FMT_NOEXCEPT {
    // If we have a terminal color, we need to output another escape code
    // sequence.
    if (!text_color.is_rgb) {
      bool is_background = esc == detail::data::background_color;
      uint32_t value = text_color.value.term_color;
      // Background ASCII codes are the same as the foreground ones but with
      // 10 more.
      if (is_background) value += 10u;

      size_t index = 0;
      buffer[index++] = static_cast<Char>('\x1b');
      buffer[index++] = static_cast<Char>('[');

      if (value >= 100u) {
        buffer[index++] = static_cast<Char>('1');
        value %= 100u;
      }
      buffer[index++] = static_cast<Char>('0' + value / 10u);
      buffer[index++] = static_cast<Char>('0' + value % 10u);

      buffer[index++] = static_cast<Char>('m');
      buffer[index++] = static_cast<Char>('\0');
      return;
    }

    for (int i = 0; i < 7; i++) {
      buffer[i] = static_cast<Char>(esc[i]);
    }
    rgb color(text_color.value.rgb_color);
    to_esc(color.r, buffer + 7, ';');
    to_esc(color.g, buffer + 11, ';');
    to_esc(color.b, buffer + 15, 'm');
    buffer[19] = static_cast<Char>(0);
  }
  FMT_CONSTEXPR ansi_color_escape(emphasis em) FMT_NOEXCEPT {
    uint8_t em_codes[4] = {};
    uint8_t em_bits = static_cast<uint8_t>(em);
    if (em_bits & static_cast<uint8_t>(emphasis::bold)) em_codes[0] = 1;
    if (em_bits & static_cast<uint8_t>(emphasis::italic)) em_codes[1] = 3;
    if (em_bits & static_cast<uint8_t>(emphasis::underline)) em_codes[2] = 4;
    if (em_bits & static_cast<uint8_t>(emphasis::strikethrough))
      em_codes[3] = 9;

    size_t index = 0;
    for (int i = 0; i < 4; ++i) {
      if (!em_codes[i]) continue;
      buffer[index++] = static_cast<Char>('\x1b');
      buffer[index++] = static_cast<Char>('[');
      buffer[index++] = static_cast<Char>('0' + em_codes[i]);
      buffer[index++] = static_cast<Char>('m');
    }
    buffer[index++] = static_cast<Char>(0);
  }
  FMT_CONSTEXPR operator const Char*() const FMT_NOEXCEPT { return buffer; }

  FMT_CONSTEXPR const Char* begin() const FMT_NOEXCEPT { return buffer; }
  FMT_CONSTEXPR const Char* end() const FMT_NOEXCEPT {
    return buffer + std::char_traits<Char>::length(buffer);
  }

 private:
  Char buffer[7u + 3u * 4u + 1u];

  static FMT_CONSTEXPR void to_esc(uint8_t c, Char* out,
                                   char delimiter) FMT_NOEXCEPT {
    out[0] = static_cast<Char>('0' + c / 100);
    out[1] = static_cast<Char>('0' + c / 10 % 10);
    out[2] = static_cast<Char>('0' + c % 10);
    out[3] = static_cast<Char>(delimiter);
  }
};

```cpp

这段代码定义了三个模板变量ansi_color_escape<Char>，用于表示颜色数据类型Char的颜色数据。

make_foreground_color函数模板参数为foreground，使用ansi_color_escape<Char>函数实现将foreground颜色数据转换为字符颜色数据。

make_background_color函数模板参数为background，使用ansi_color_escape<Char>函数实现将background颜色数据转换为字符颜色数据。

make_emphasis函数模板参数为emphasis，使用ansi_color_escape<Char>函数实现将emphasis类型转换为字符颜色数据。


```
template <typename Char>
FMT_CONSTEXPR ansi_color_escape<Char> make_foreground_color(
    detail::color_type foreground) FMT_NOEXCEPT {
  return ansi_color_escape<Char>(foreground, detail::data::foreground_color);
}

template <typename Char>
FMT_CONSTEXPR ansi_color_escape<Char> make_background_color(
    detail::color_type background) FMT_NOEXCEPT {
  return ansi_color_escape<Char>(background, detail::data::background_color);
}

template <typename Char>
FMT_CONSTEXPR ansi_color_escape<Char> make_emphasis(emphasis em) FMT_NOEXCEPT {
  return ansi_color_escape<Char>(em);
}

```cpp

这段代码定义了三个模板函数，分别是 fputs 和 reset_color，用于在文件中输出字符串和重置输出颜色。

fputs 函数接受两个参数，一个字符串 chars 和一个文件 stream。函数的作用是将字符串 chars 中的每个字符输出到 stream 中。在实现中，使用 std::fputs 函数来输出字符。如果函数失败，使用 FMT_NOEXCEPT 格式化代码来输出默认错误信息。

reset_color 函数也接受一个文件 stream，但是不接收任何参数。函数的作用是在文件中输出一个默认的颜色，通常用于表示成功或警告。在实现中，使用 fputs 函数来输出 "reset_color"。

wputs 函数与 fputs 函数类似，只是输出字符串的 wchar_t 版本。


```
template <typename Char>
inline void fputs(const Char* chars, FILE* stream) FMT_NOEXCEPT {
  std::fputs(chars, stream);
}

template <>
inline void fputs<wchar_t>(const wchar_t* chars, FILE* stream) FMT_NOEXCEPT {
  std::fputws(chars, stream);
}

template <typename Char> inline void reset_color(FILE* stream) FMT_NOEXCEPT {
  fputs(detail::data::reset_color, stream);
}

template <> inline void reset_color<wchar_t>(FILE* stream) FMT_NOEXCEPT {
  fputs(detail::data::wreset_color, stream);
}

```cpp

这段代码定义了两个函数，一个用于将格式化字符串中的颜色信息恢复，另一个用于将格式化字符串中的颜色信息格式化。这两个函数都接受一个字符缓冲区和一个文本样式对象。

第一个函数 `reset_color` 接受一个字符缓冲区 (`buffer`)，并在缓冲区的起始位置和结束位置之间插入颜色信息 (如 `data::reset_color`)。这样，当将格式化字符串中的颜色信息存储到缓冲区时，会覆盖原有内容。

第二个函数 `vformat_to` 接受一个字符缓冲区 (`buf`)、一个文本样式对象 (`ts`) 和格式化字符串。它将格式化字符串中的颜色信息格式化并存储到缓冲区中。它还使用 `detail::make_emphasis` 和 `detail::make_foreground_color` 函数来获取强调样式和前景颜色。如果 `ts.has_emphasis()` 和 `ts.has_foreground()` 为真，则会创建一个新的强调样式并将格式化字符串中的颜色信息添加到缓冲区中。

另外，`detail::reset_color` 函数接受一个字符缓冲区 (`buf`)，并在缓冲区的起始位置和结束位置之间插入颜色信息 (如 `data::reset_color`)。这个函数与第一个函数 `reset_color` 类似，但它只会在格式化字符串中的颜色信息生效时才执行，以避免覆盖原有内容。


```
template <typename Char>
inline void reset_color(buffer<Char>& buffer) FMT_NOEXCEPT {
  const char* begin = data::reset_color;
  const char* end = begin + sizeof(data::reset_color) - 1;
  buffer.append(begin, end);
}

template <typename Char>
void vformat_to(buffer<Char>& buf, const text_style& ts,
                basic_string_view<Char> format_str,
                basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  bool has_style = false;
  if (ts.has_emphasis()) {
    has_style = true;
    auto emphasis = detail::make_emphasis<Char>(ts.get_emphasis());
    buf.append(emphasis.begin(), emphasis.end());
  }
  if (ts.has_foreground()) {
    has_style = true;
    auto foreground = detail::make_foreground_color<Char>(ts.get_foreground());
    buf.append(foreground.begin(), foreground.end());
  }
  if (ts.has_background()) {
    has_style = true;
    auto background = detail::make_background_color<Char>(ts.get_background());
    buf.append(background.begin(), background.end());
  }
  detail::vformat_to(buf, format_str, args);
  if (has_style) detail::reset_color<Char>(buf);
}
}  // namespace detail

```cpp

这段代码定义了一个名为 `vprint` 的函数，用于将字符格式化并输出到文件中。

该函数接受三个参数：

1. 一个指向输出文件的文件流的指针 `f`；
2. 一个模板类 `S`，其中 `Char` 被定义为 `char_t<S>`，用于表示输入数据类型；
3. 一个格式字符串 `format`，其中包含用于格式化输出字符的ANSI escape序列。

函数内部首先创建一个名为 `buf` 的字符缓冲区，然后使用 `to_string_view` 函数将格式字符串转换为字符串，接着使用模板的 `basic_format_args` 类型将格式字符串和输入数据类型 `S` 组合，并将其存储到 `buf` 中。

随后，函数使用 `fputs` 函数将缓冲区中的字符串输出到文件流中，最后在字符串的结尾添加一个空字符以表示字符串的结束。

该函数可以用于将格式化后的字符串输出到文件中，例如：

```fmt
fmt::print("Hello, world!", fmt::emphasis::bold);
```cpp

会输出 "Hello, world!"，并且使用了 ANSI escape sequences。


```
template <typename S, typename Char = char_t<S>>
void vprint(std::FILE* f, const text_style& ts, const S& format,
            basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  basic_memory_buffer<Char> buf;
  detail::vformat_to(buf, ts, to_string_view(format), args);
  buf.push_back(Char(0));
  detail::fputs(buf.data(), f);
}

/**
  \rst
  Formats a string and prints it to the specified file stream using ANSI
  escape sequences to specify text formatting.

  **Example**::

    fmt::print(fmt::emphasis::bold | fg(fmt::color::red),
               "Elapsed time: {0:.2f} seconds", 1.23);
  \endrst
 */
```cpp

这段代码定义了一个名为 print 的模板函数，用于将给定的格式字符串和参数对象格式化并打印到标准输出文件中。

模板参数部分声明了一个类型参数 S、一个或多个参数 Args 和一个模板形参表＜typename... Args＞。它允许函数能够接受任何符合这些条件的参数。

函数体中，首先通过 `FMT_ENABLE_IF(detail::is_string<S>::value)` 语句启用 S 类型安全特性。接着定义了一个函数内部函数 `vprint`，它将格式化字符串和参数对象传递给 `fmt::print` 函数进行打印。

最后，函数的实现部分将格式字符串、参数对象和调用 `fmt::print` 函数封装在一起，使用 `fmt::make_args_checked<Args...>(format_str, args...)` 将参数对象调用 `fmt::print` 函数的形式化字符串。


```
template <typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_string<S>::value)>
void print(std::FILE* f, const text_style& ts, const S& format_str,
           const Args&... args) {
  vprint(f, ts, format_str,
         fmt::make_args_checked<Args...>(format_str, args...));
}

/**
  Formats a string and prints it to stdout using ANSI escape sequences to
  specify text formatting.
  Example:
    fmt::print(fmt::emphasis::bold | fg(fmt::color::red),
               "Elapsed time: {0:.2f} seconds", 1.23);
 */
```cpp

这段代码定义了两个模板函数，分别是：

1. print()：该函数接收三个参数，一个模板参数S，一个格式字符串 ts，以及一个或多个参数Args。函数内部实现了一个内联函数，接收一个或多个输出流对象stdout，将格式字符串和参数Args分别传递给print函数，最后将结果返回。

2. vformat()：该函数接收两个参数，一个模板参数S和一个格式字符串 ts，以及一个名为args的参数，该参数是一个或多个辅助格式化参数。函数内部实现了一个内联函数，接收一个基本输入流对象，将格式字符串和参数Args分别传递给vformat_to()函数，最后将结果转换成字符串类型并返回。

print()函数的作用是包装成一个内联函数，可以用来调用vformat()函数，将格式字符串和要输出的事物一起传递给vformat()函数，然后再将结果输出。

vformat()函数的作用是在调用print()函数之前，对格式字符串和参数进行标准化，以帮助印刷输出的过程中正确地解释和格式化数据。


```
template <typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_string<S>::value)>
void print(const text_style& ts, const S& format_str, const Args&... args) {
  return print(stdout, ts, format_str, args...);
}

template <typename S, typename Char = char_t<S>>
inline std::basic_string<Char> vformat(
    const text_style& ts, const S& format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  basic_memory_buffer<Char> buf;
  detail::vformat_to(buf, ts, to_string_view(format_str), args);
  return fmt::to_string(buf);
}

```cpp

这段代码是一个C++模板函数，名为`format`，它的作用是格式化一个字符串，将输入的格式字符和参数按照指定的顺序连接起来，最终返回一个正确格式化后的字符串。

具体来说，这个函数接受三个参数：

1. 一个`text_style`类型的参数，它是一个格式化字符串，可以使用ANSI escape sequences进行格式化，例如`fmt::emphasis::bold`可以用来加粗。
2. 一个格式字符串，可以使用`fmt::color::red`进行颜色强调，这个格式字符串也可以包含多个参数。
3. 一个或多个`Args`类型的参数，它们用来在格式字符中插入arg，通过`format_str`参数传递。

函数内部首先定义了一个`std::basic_string<Char>`类型的变量`message`，用来存储格式化后的字符串，这个字符串中的`Char`类型是输入参数中的类型。

然后定义了一个`template <typename S, typename... Args>`类型的函数`vformat`，这个函数是一个`std::basic_string<Char>`类型的函数，接受两个`text_style`类型的参数，和一个`std::params<Args...)`类型的参数，其中`Args...`是一个参数列表，通过`format_str`参数传递。

接着就是函数体，在函数体内部，首先使用`fmt::make_args_checked`将`format_str`和`args...`组装成一个参数列表，然后使用`vformat`函数将组装好的参数列表格式化，最后将格式化后的字符串赋值给`message`，即返回了格式化后的字符串。


```
/**
  \rst
  Formats arguments and returns the result as a string using ANSI
  escape sequences to specify text formatting.

  **Example**::

    #include <fmt/color.h>
    std::string message = fmt::format(fmt::emphasis::bold | fg(fmt::color::red),
                                      "The answer is {}", 42);
  \endrst
*/
template <typename S, typename... Args, typename Char = char_t<S>>
inline std::basic_string<Char> format(const text_style& ts, const S& format_str,
                                      const Args&... args) {
  return vformat(ts, to_string_view(format_str),
                 fmt::make_args_checked<Args...>(format_str, args...));
}

```cpp

这段代码定义了一个名为 `vformat_to` 的函数，用于将给定的字符串格式化并写入到指定的输出迭代器中。

函数接受三个参数：

1. `out`: 输出迭代器，用于将格式化后的字符串写入该迭代器中。
2. `ts`: 字符串样式，可以使用 `text_style` 类或手动传递给函数。
3. `format_str`: 要格式化的字符串，可以使用 `basic_string_view` 类来获取字符串中的字符和他们的格式。 `basic_format_args` 类用于传递给 `vformat_to` 函数的格式参数。

函数实现中，首先定义了一个名为 `buf` 的变量，用于存储格式化后的字符串。然后使用 `vformat_to` 函数来格式化给定的字符串，并将格式化后的字符串存储在 `buf` 中。最后，函数返回 `buf` 的迭代器。

该函数可以用来将给定的字符串格式化并写入到指定的输出迭代器中。例如，以下代码将格式化字符串 `"{}`(传递给 `fmt::format_to` 函数)，并将其写入到输出迭代器中：

```c++
std::vector<char> out;
fmt::format_to(std::back_inserter(out),
                  fmt::emphasis::bold | fg(fmt::color::red), "{}", 42);
```cpp

这将输出 `"{}` 和一个带有 `fmt::color::red` 强调的 `fmt::emphasis::bold` 的字符串，其内容为 `"42{}`。


```
/**
  Formats a string with the given text_style and writes the output to ``out``.
 */
template <typename OutputIt, typename Char,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value)>
OutputIt vformat_to(
    OutputIt out, const text_style& ts, basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  decltype(detail::get_buffer<Char>(out)) buf(detail::get_buffer_init(out));
  detail::vformat_to(buf, ts, format_str, args);
  return detail::get_iterator(buf);
}

/**
  \rst
  Formats arguments with the given text_style, writes the result to the output
  iterator ``out`` and returns the iterator past the end of the output range.

  **Example**::

    std::vector<char> out;
    fmt::format_to(std::back_inserter(out),
                   fmt::emphasis::bold | fg(fmt::color::red), "{}", 42);
  \endrst
```cpp

这段代码定义了一个名为`format_to`的函数模板，它的参数包括一个输出迭代器`out`、一个模板参数`S`和一个格式字符串`ts`。这个函数模板重载了`std::format`库中的`vformat_to`函数，同时也引入了`std::format_to`库中的`FMT_ENABLE_IF`特性。

具体来说，这段代码的作用是实现了一个格式化字符串的函数，它可以将一个格式字符串和多个参数（模板实参）连接起来，然后输出到给定的输出迭代器上。这个函数可以用于格式化文本输出，例如打印"Hello %s, %s%c %c"这个字符串。

对于每个调用`format_to`函数，它会首先解析格式字符串中的占位符，如果占位符是`%s`，那么它会被输出。否则，如果占位符是`%d`或`%f`，那么它们会被存储在`%3d`和`%f`的形式中，最终输出到`%s`。如果占位符是`%c`，那么它会被转换为`%c`的ASCII码并输出。

此外，`FMT_ENABLE_IF`特性还会在编译时检查`detail::is_output_iterator<OutputIt>::value`和`detail::is_string<S>::value`是否为真，如果是，那么`std::format_to`库将被启用，否则将不输出。


```
*/
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value&&
                            detail::is_string<S>::value)>
inline OutputIt format_to(OutputIt out, const text_style& ts,
                          const S& format_str, Args&&... args) {
  return vformat_to(out, ts, to_string_view(format_str),
                    fmt::make_args_checked<Args...>(format_str, args...));
}

FMT_END_NAMESPACE

#endif  // FMT_COLOR_H_

```cpp

# `src/3rdparty/fmt/compile.h`

这段代码是一个格式化库，它是针对C++的实验性格式字符串编译。它旨在提供一个方便的方式来格式化C++代码。该代码由Victor Zverovich和fmt contributors共同贡献，并在版权中注明了受版权保护。

该代码包含一个头文件fmt_compile.h，其中定义了一系列API，用于从输入文本中生成格式化字符串，并将它们编译成可执行文件。

具体来说，这段代码的作用是定义了一个抽象类A format，包含一个成员函数fmt_error，它接收两个格式化字符串作为参数，然后返回一个表示错误信息的整数。另外，还定义了几个成员函数，如fmt_parse_from_file，fmt_set_align，fmt_set_format等等，用于从输入文本中读取或设置格式。

fmt_compile.h是 FMT 库的入口点，FMT 库是一个用于格式化 C++ 代码的工具链。它允许开发人员使用简单的语法来格式化他们的 C++ 代码，而无需从头编写格式化字符串的代码。


```
// Formatting library for C++ - experimental format string compilation
//
// Copyright (c) 2012 - present, Victor Zverovich and fmt contributors
// All rights reserved.
//
// For the license information refer to format.h.

#ifndef FMT_COMPILE_H_
#define FMT_COMPILE_H_

#include <vector>

#include "format.h"

FMT_BEGIN_NAMESPACE
```cpp

这段代码定义了一个名为 "detail" 的命名空间，其中包含一个名为 "compiled_string" 的类，以及一个名为 "is_compiled_string" 的模板结构体。

作用：

1. "compiled_string" 是一个编译时生成的字符串类，它包含一个模板类型的参数 S，以及一个转换函数，可以将给定的字符串常量 *s* 转换为格式化字符串，并在编译时生成相应的代码。

2. "is_compiled_string" 是一个模板结构体，它继承自 "std::is_base_of<compiled_string, S>"，意为 "如果给定的模板类型参数 S 属于编译时生成的字符串类，那么 is_compiled_string 结构体将是一个基类，可以使用编译时检查检查它们是否基类，如果 is_base_of 编译器支持"。

3. 该代码提供了一个名为 "fmt::format" 的函数，它接受一个格式化字符串模板，该模板使用 "{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}`{}


```
namespace detail {

// A compile-time string which is compiled into fast formatting code.
class compiled_string {};

template <typename S>
struct is_compiled_string : std::is_base_of<compiled_string, S> {};

/**
  \rst
  Converts a string literal *s* into a format string that will be parsed at
  compile time and converted into efficient formatting code. Requires C++17
  ``constexpr if`` compiler support.

  **Example**::

    // Converts 42 into std::string using the most efficient method and no
    // runtime format string processing.
    std::string s = fmt::format(FMT_COMPILE("{}"), 42);
  \endrst
 */
```cpp

This is a C++ template that takes a value of type T and a potentially larger number of Tail arguments. It provides a function called first that returns the first argument in the Tuple<T, Tail> being processed.

The first argument is that the value of type T that is being processed. If the Tuple<T, Tail} is empty, the first argument is an default value of type T.

The second argument is a率和 a format string. The format string is a combination of literal text and replacement fields. The replacement fields are additional information that can be inserted into the format string at compile time, such as the index of the argument or the name of the argument.

The third argument is an optional piece of information that can be used to further specify the format string. It is a piece of information that is passed to the template metachthon to handle the format string in the absence of the other arguments.

The code for the first function is defined as follows:
```
template <typename T, typename... Tail>
const T& first(const T& value, const Tail&... tail) {
 return value;
}
```cpp
This function is a member of the template Metachron which allows the program to use the template parameter in the context of the function.

It is used in the following way:
```
int main() {
   const int x = 5;
   const char* str = "Hello %s\n";
   const char* format = R"(%d %s)";
   fmt::detail::compile_string<const char*>(str, format);
   const char* result = first(x, format);
   fmt::detail::print(result); // "Hello 5 Hello"
   return 0;
}
```cpp
It is worth noting that the code provided is a simplified version, in a real-world scenario, you would need to handle the errors, and provide more robustness.


```
#define FMT_COMPILE(s) FMT_STRING_IMPL(s, fmt::detail::compiled_string)

template <typename T, typename... Tail>
const T& first(const T& value, const Tail&...) {
  return value;
}

// Part of a compiled format string. It can be either literal text or a
// replacement field.
template <typename Char> struct format_part {
  enum class kind { arg_index, arg_name, text, replacement };

  struct replacement {
    arg_ref<Char> arg_id;
    dynamic_format_specs<Char> specs;
  };

  kind part_kind;
  union value {
    int arg_index;
    basic_string_view<Char> str;
    replacement repl;

    FMT_CONSTEXPR value(int index = 0) : arg_index(index) {}
    FMT_CONSTEXPR value(basic_string_view<Char> s) : str(s) {}
    FMT_CONSTEXPR value(replacement r) : repl(r) {}
  } val;
  // Position past the end of the argument id.
  const Char* arg_id_end = nullptr;

  FMT_CONSTEXPR format_part(kind k = kind::arg_index, value v = {})
      : part_kind(k), val(v) {}

  static FMT_CONSTEXPR format_part make_arg_index(int index) {
    return format_part(kind::arg_index, index);
  }
  static FMT_CONSTEXPR format_part make_arg_name(basic_string_view<Char> name) {
    return format_part(kind::arg_name, name);
  }
  static FMT_CONSTEXPR format_part make_text(basic_string_view<Char> text) {
    return format_part(kind::text, text);
  }
  static FMT_CONSTEXPR format_part make_replacement(replacement repl) {
    return format_part(kind::replacement, repl);
  }
};

```cpp

这是一个C++代码，定义了一个名为`part_counter`的结构体，用于跟踪文本中每个部分的数量。该结构体有以下几种成员函数：

1. `on_text`函数：在给定范围内遍历文本，并在遇到分号时增加`num_parts`变量。
2. `on_arg_id`函数：根据给定的参数类型返回一个整数，其中包括`num_parts`变量。
3. `on_arg_id`函数：根据给定的基本字符串 view 返回一个整数，其中包括`num_parts`变量。
4. `on_replacement_field`函数：当`num_parts`变量为整数时，执行此函数。
5. `on_format_specs`函数：返回给定格式specs的起始和结束位置，其中format_specs是一个指向char类型的指针，指定了格式（例如，{%，p，表示输出两个整数）的宽度。
6. `on_error`函数：当给定的字符串中包含错误时执行此函数。

该结构体的定义是在`<part_counter.h>`文件中。


```
template <typename Char> struct part_counter {
  unsigned num_parts = 0;

  FMT_CONSTEXPR void on_text(const Char* begin, const Char* end) {
    if (begin != end) ++num_parts;
  }

  FMT_CONSTEXPR int on_arg_id() { return ++num_parts, 0; }
  FMT_CONSTEXPR int on_arg_id(int) { return ++num_parts, 0; }
  FMT_CONSTEXPR int on_arg_id(basic_string_view<Char>) {
    return ++num_parts, 0;
  }

  FMT_CONSTEXPR void on_replacement_field(int, const Char*) {}

  FMT_CONSTEXPR const Char* on_format_specs(int, const Char* begin,
                                            const Char* end) {
    // Find the matching brace.
    unsigned brace_counter = 0;
    for (; begin != end; ++begin) {
      if (*begin == '{') {
        ++brace_counter;
      } else if (*begin == '}') {
        if (brace_counter == 0u) break;
        --brace_counter;
      }
    }
    return begin;
  }

  FMT_CONSTEXPR void on_error(const char*) {}
};

```cpp

This is a Rust implementation of the Structson formatting engine for Python. It includes the ability to handle text formatting, positional arguments, and a `FMT_CONSTEXPR` macro for specifying the maximum number of characters allowed in a format string.

The `on_text` function is used to handle text formatting. It takes a start and end character range and returns a reference to the formatted text.

The `on_arg_id` function is used to handle positional arguments. It takes an integer argument and returns an index into the `arg_index` array or a string argument if the argument is not an integer.

The `on_arg_name` function is used to handle named arguments. It takes a basic string argument and returns the index into the `arg_name` array.

The `on_replacement_field` function is used to handle replacement fields. It takes an integer argument and a pointer to a character string and returns the index into the `replacement` array.

The `on_format_specs` function is used to handle the maximum number of characters allowed in a format string. It takes a begin and end character range and returns a reference to the formatted text.

The `on_error` function is used to handle errors. It takes a string argument and returns a reference to the error object.


```
// Counts the number of parts in a format string.
template <typename Char>
FMT_CONSTEXPR unsigned count_parts(basic_string_view<Char> format_str) {
  part_counter<Char> counter;
  parse_format_string<true>(format_str, counter);
  return counter.num_parts;
}

template <typename Char, typename PartHandler>
class format_string_compiler : public error_handler {
 private:
  using part = format_part<Char>;

  PartHandler handler_;
  part part_;
  basic_string_view<Char> format_str_;
  basic_format_parse_context<Char> parse_context_;

 public:
  FMT_CONSTEXPR format_string_compiler(basic_string_view<Char> format_str,
                                       PartHandler handler)
      : handler_(handler),
        format_str_(format_str),
        parse_context_(format_str) {}

  FMT_CONSTEXPR void on_text(const Char* begin, const Char* end) {
    if (begin != end)
      handler_(part::make_text({begin, to_unsigned(end - begin)}));
  }

  FMT_CONSTEXPR int on_arg_id() {
    part_ = part::make_arg_index(parse_context_.next_arg_id());
    return 0;
  }

  FMT_CONSTEXPR int on_arg_id(int id) {
    parse_context_.check_arg_id(id);
    part_ = part::make_arg_index(id);
    return 0;
  }

  FMT_CONSTEXPR int on_arg_id(basic_string_view<Char> id) {
    part_ = part::make_arg_name(id);
    return 0;
  }

  FMT_CONSTEXPR void on_replacement_field(int, const Char* ptr) {
    part_.arg_id_end = ptr;
    handler_(part_);
  }

  FMT_CONSTEXPR const Char* on_format_specs(int, const Char* begin,
                                            const Char* end) {
    auto repl = typename part::replacement();
    dynamic_specs_handler<basic_format_parse_context<Char>> handler(
        repl.specs, parse_context_);
    auto it = parse_format_specs(begin, end, handler);
    if (*it != '}') on_error("missing '}' in format string");
    repl.arg_id = part_.part_kind == part::kind::arg_index
                      ? arg_ref<Char>(part_.val.arg_index)
                      : arg_ref<Char>(part_.val.str);
    auto part = part::make_replacement(repl);
    part.arg_id_end = begin;
    handler_(part);
    return it;
  }
};

```cpp

这段代码定义了一个名为 `format_arg` 的函数，它接受一个 `basic_format_parse_context` 和一个 `Context` 对象，并按给定的格式解析给定的 `arg_formatter` 函数。

具体来说，这个函数首先定义了一个模板参数组 `IS_CONSTEXPR`，该参数组包括 `bool` 类型和 `Char` 和 `PartHandler` 类型的参数。接着，它定义了一个名为 `compile_format_string` 的函数，该函数接受一个 `basic_string_view` 和一个 `PartHandler` 类型的函数作为参数，然后按照给定的格式解析字符串。

`compile_format_string` 函数首先定义了一个局部函数 `parse_format_string`，该函数也接受 `IS_CONSTEXPR` 和 `Char` 和 `PartHandler` 类型的参数，并将解析字符串的格式化器设置为给定的 `handler` 函数。然后，它调用 `parse_format_string` 函数，将解析字符串的格式化器设置为给定的 `handler` 函数，并将解析上下文中的 `arg_formatter` 参数设置为 `arg_formatter` 函数。

最后，`format_arg` 函数接受一个 `basic_format_parse_context` 和一个 `Context` 对象，并按给定的 `arg_formatter` 函数解析给定的 `arg_formatter` 函数。它通过 `arg_formatter` 函数将解析到的格式化信息与 `arg_id` 关联起来，并将解析结果输出到 `std::ostream` 对象中。


```
// Compiles a format string and invokes handler(part) for each parsed part.
template <bool IS_CONSTEXPR, typename Char, typename PartHandler>
FMT_CONSTEXPR void compile_format_string(basic_string_view<Char> format_str,
                                         PartHandler handler) {
  parse_format_string<IS_CONSTEXPR>(
      format_str,
      format_string_compiler<Char, PartHandler>(format_str, handler));
}

template <typename OutputIt, typename Context, typename Id>
void format_arg(
    basic_format_parse_context<typename Context::char_type>& parse_ctx,
    Context& ctx, Id arg_id) {
  ctx.advance_to(visit_format_arg(
      arg_formatter<OutputIt, typename Context::char_type>(ctx, &parse_ctx),
      ctx.arg(arg_id)));
}

```cpp

This is a Rust implementation of a `format_part_t` struct that represents a part of a formatted string. It has a `kind` field that specifies the kind of part this is (e.g. `format_part_t::kind::arg_index`), as well as a `to_end` method that advances to the end of the string.

The `arg_index` field is a field of the `kind` field that specifies the index of the argument to insert in the output.

The `arg_name` field is a field of the `kind` field that specifies the name of the argument.

The `replacement` field is a field of the `kind` field that specifies whether the argument should be replaced with replacement text. If it is, the `repl` field is an object that maps the `arg_id` to a `str` value.

The `handle_dynamic_spec` function is a helper function that takes a vector of `specs` and handles dynamic replacements.

The `numeric_specs_checker` function is a helper function that takes a numeric `specs` and checks that the input is valid.

The `check_sign` function is a helper function that takes a `sign` and checks that the input is valid.

The `require_numeric_argument` function is a helper function that takes a `type` and a `checker` object and checks that the `checker`'s `require_numeric_argument` method returns true.

The `require_numeric_argument` function is a helper function that takes a `type` and a `checker` object and checks that the `checker`'s `require_numeric_argument` method returns true.


```
// vformat_to is defined in a subnamespace to prevent ADL.
namespace cf {
template <typename Context, typename OutputIt, typename CompiledFormat>
auto vformat_to(OutputIt out, CompiledFormat& cf,
                basic_format_args<Context> args) -> typename Context::iterator {
  using char_type = typename Context::char_type;
  basic_format_parse_context<char_type> parse_ctx(
      to_string_view(cf.format_str_));
  Context ctx(out, args);

  const auto& parts = cf.parts();
  for (auto part_it = std::begin(parts); part_it != std::end(parts);
       ++part_it) {
    const auto& part = *part_it;
    const auto& value = part.val;

    using format_part_t = format_part<char_type>;
    switch (part.part_kind) {
    case format_part_t::kind::text: {
      const auto text = value.str;
      auto output = ctx.out();
      auto&& it = reserve(output, text.size());
      it = std::copy_n(text.begin(), text.size(), it);
      ctx.advance_to(output);
      break;
    }

    case format_part_t::kind::arg_index:
      advance_to(parse_ctx, part.arg_id_end);
      detail::format_arg<OutputIt>(parse_ctx, ctx, value.arg_index);
      break;

    case format_part_t::kind::arg_name:
      advance_to(parse_ctx, part.arg_id_end);
      detail::format_arg<OutputIt>(parse_ctx, ctx, value.str);
      break;

    case format_part_t::kind::replacement: {
      const auto& arg_id_value = value.repl.arg_id.val;
      const auto arg = value.repl.arg_id.kind == arg_id_kind::index
                           ? ctx.arg(arg_id_value.index)
                           : ctx.arg(arg_id_value.name);

      auto specs = value.repl.specs;

      handle_dynamic_spec<width_checker>(specs.width, specs.width_ref, ctx);
      handle_dynamic_spec<precision_checker>(specs.precision,
                                             specs.precision_ref, ctx);

      error_handler h;
      numeric_specs_checker<error_handler> checker(h, arg.type());
      if (specs.align == align::numeric) checker.require_numeric_argument();
      if (specs.sign != sign::none) checker.check_sign();
      if (specs.alt) checker.require_numeric_argument();
      if (specs.precision >= 0) checker.check_precision();

      advance_to(parse_ctx, part.arg_id_end);
      ctx.advance_to(
          visit_format_arg(arg_formatter<OutputIt, typename Context::char_type>(
                               ctx, nullptr, &specs),
                           arg));
      break;
    }
    }
  }
  return ctx.out();
}
}  // namespace cf

```cpp

这段代码定义了一个模板结构体 `basic_compiled_format`；

接着定义了一个模板结构体 `compiled_format_base` 继承自 `basic_compiled_format`；

`compiled_format_base` 中包含一个 `using` 短语，说明这个结构体中的一些成员使用的是模板参数的别名类型，具体来说继承自 `basic_compiled_format` 的 `using` 类型为 `void` ，因此继承自 `compiled_format_base` 的 `using` 类型也为 `void` ；

`compiled_format_base` 中包含一个 `using` 短语，说明这个结构体中的一些成员使用的是模板参数的别名类型，具体来说继承自 `basic_compiled_format` 的 `using` 类型为 `void` ，因此继承自 `compiled_format_base` 的 `using` 类型也为 `void` ；

`compiled_format_base` 中包含一个 `using` 短语，说明这个结构体中的一些成员使用的是模板参数的别名类型，具体来说继承自 `basic_compiled_format` 的 `using` 类型为 `void` ，因此继承自 `compiled_format_base` 的 `using` 类型也为 `void` ；

`compiled_format_base` 中包含一个 `using` 短语，说明这个结构体中的一些成员使用的是模板参数的别名类型，具体来说继承自 `basic_compiled_format` 的 `using` 类型为 `void` ，因此继承自 `compiled_format_base` 的 `using` 类型也为 `void` ；

`compiled_format_base` 中包含一个 `using` 短语，说明这个结构体中的一些成员使用的是模板参数的别名类型，具体来说继承自 `basic_compiled_format` 的 `using` 类型为 `void` ，因此继承自 `compiled_format_base` 的 `using` 类型也为 `void` ；

`compiled_format_base` 中包含一个 `using` 短语，说明这个结构体中的一些成员使用的是模板参数的别名类型，具体来说继承自 `basic_compiled_format` 的 `using` 类型为 `void` ，因此继承自 `compiled_format_base` 的 `using` 类型也为 `void` 。


```
struct basic_compiled_format {};

template <typename S, typename = void>
struct compiled_format_base : basic_compiled_format {
  using char_type = char_t<S>;
  using parts_container = std::vector<detail::format_part<char_type>>;

  parts_container compiled_parts;

  explicit compiled_format_base(basic_string_view<char_type> format_str) {
    compile_format_string<false>(format_str,
                                 [this](const format_part<char_type>& part) {
                                   compiled_parts.push_back(part);
                                 });
  }

  const parts_container& parts() const { return compiled_parts; }
};

```cpp

这段代码定义了一个模板结构体 `format_part_array`，其中 `Char` 是一个类型参数，`N` 是另一个类型参数，用于指定 `format_part_array` 的大小。`format_part_array` 是一个结构体，其中包含一个 `Char` 类型的数组 `data`，该数组长度为 `N`。此外，它还包含一个名为 `FMT_CONSTEXPR` 的类型别注释，用于声明一个常量表达式。

`compile_to_parts` 函数接收一个 `basic_string_view<Char>` 类型的参数 `format_str`，该参数表示要解析的格式字符串。函数首先定义了一个名为 `parts` 的 `format_part_array` 类型的变量，然后定义了一个名为 `counter` 的变量，用于跟踪已解析过的字符数。

接着，函数创建了一个名为 `collector` 的结构体，该结构体包含一个指向 `format_part<Char>` 的指针 `parts`，以及一个指向 `int` 的指针 `counter`。最后，函数调用 `compile_format_string<true>` 函数来解析 `format_str` 中的格式字符串，并获取解析结果，如果解析成功，则将结果存回 `parts`。如果解析失败，则继续解析下一个字符。

最终，函数返回 `parts` 类型，该类型包含了一个 `Char` 类型的数组，该数组长度为 `N`，其中包含解析格式字符串后的字符。


```
template <typename Char, unsigned N> struct format_part_array {
  format_part<Char> data[N] = {};
  FMT_CONSTEXPR format_part_array() = default;
};

template <typename Char, unsigned N>
FMT_CONSTEXPR format_part_array<Char, N> compile_to_parts(
    basic_string_view<Char> format_str) {
  format_part_array<Char, N> parts;
  unsigned counter = 0;
  // This is not a lambda for compatibility with older compilers.
  struct {
    format_part<Char>* parts;
    unsigned* counter;
    FMT_CONSTEXPR void operator()(const format_part<Char>& part) {
      parts[(*counter)++] = part;
    }
  } collector{parts.data, &counter};
  compile_format_string<true>(format_str, collector);
  if (counter < N) {
    parts.data[counter] =
        format_part<Char>::make_text(basic_string_view<Char>());
  }
  return parts;
}

```cpp

这段代码定义了一个模板结构体，名为compiled_format_base。该结构体包含一个函数模板，该函数模板使用了编译时格式化语法。

具体来说，该函数模板接受两个整型参数a和b，并返回a和b中的较大值。如果a小于b，则返回b，否则返回a。

该代码中还包含一个枚举类型S，以及一个结构体，名为compiled_format_base。该结构体包含一个basic_string_view类型的成员变量，用于存储输入的格式化字符串。

最后，该代码中还包含一个可选的评论，指出该代码是为了应对使用编译时格式化语法的旧编译器而设计的。


```
template <typename T> constexpr const T& constexpr_max(const T& a, const T& b) {
  return (a < b) ? b : a;
}

template <typename S>
struct compiled_format_base<S, enable_if_t<is_compile_string<S>::value>>
    : basic_compiled_format {
  using char_type = char_t<S>;

  FMT_CONSTEXPR explicit compiled_format_base(basic_string_view<char_type>) {}

// Workaround for old compilers. Format string compilation will not be
// performed there anyway.
#if FMT_USE_CONSTEXPR
  static FMT_CONSTEXPR_DECL const unsigned num_format_parts =
      constexpr_max(count_parts(to_string_view(S())), 1u);
```cpp

这段代码定义了一个名为`Template`的模板类，其中包含一个名为`<=`的析构右大括号。这个模板类有一个额外的成员变量`static const unsigned num_format_parts = 1;`，用于初始化模板中形式的数量。

接下来的代码使用模板元编程技术，根据`num_format_parts`的值来定义一个名为`format_part`的函数指针。这个函数指针类型被称为`<=char_type>`(因为`char_type`是模板中使用的数据类型)。

接着，代码创建了一个名为`parts_container`的成员变量，这个成员变量是一个`<=char_type>`类型的对象，它引用了`format_part`函数指针，并且这个函数指针的一个参数是一个整数类型的变量`unsigned int`。

最后，代码提供了一个名为`parts`的成员函数，这个函数返回一个`<=char_type>`类型的变量，它是`parts_container`对象的一个成员变量，这个成员变量的类型与创建它的函数中的参数相同。这个函数的实现与`<=char_type>`函数指针的定义相关联，根据`num_format_parts`的值返回适当的字符类型的子程序。


```
#else
  static const unsigned num_format_parts = 1;
#endif

  using parts_container = format_part<char_type>[num_format_parts];

  const parts_container& parts() const {
    static FMT_CONSTEXPR_DECL const auto compiled_parts =
        compile_to_parts<char_type, num_format_parts>(
            detail::to_string_view(S()));
    return compiled_parts.data;
  }
};

template <typename S, typename... Args>
```cpp

这段代码定义了一个名为 `compiled_format` 的类，该类从 `compiled_format_base` 继承而来，其基类也继承自 `char_type`。

该类的公共成员包括一个名为 `char_type` 的类型别嘌呤，以及一个名为 `format_str_` 的 `basic_string_view` 类型的成员变量，用于存储输入格式字符串。

该类有一个名为 `cf::vformat_to` 的私有成员函数，该函数接受一个输出迭代器 `out`、一个 `CompiledFormat` 引用 `cf` 和一个 `basic_format_args` 类型的参数。该函数将调用 `cf` 的 `vformat_to` 函数，将输入格式字符串 `format_str_` 和输入 `CompiledFormat` `cf` 以及 `basic_format_args` 类型的参数传递给 `vformat_to` 函数，然后返回 `cf` 的引用中的迭代器，该迭代器将 `format_str_` 中的格式字符串转换为输入格式字符串的格式。

该类的构造函数是一个虚函数，其目的是在基类中定义构造函数，但是该构造函数没有实现任何成员函数，因此不能用于实例化 `CompiledFormat`。

此外，由于 `basic_string_view` 需要显式地指定其初始值，因此 `compiled_format` 类的成员 `format_str_` 被声明为 `private`，从而防止从函数中直接访问它。


```
class compiled_format : private compiled_format_base<S> {
 public:
  using typename compiled_format_base<S>::char_type;

 private:
  basic_string_view<char_type> format_str_;

  template <typename Context, typename OutputIt, typename CompiledFormat>
  friend auto cf::vformat_to(OutputIt out, CompiledFormat& cf,
                             basic_format_args<Context> args) ->
      typename Context::iterator;

 public:
  compiled_format() = delete;
  explicit constexpr compiled_format(basic_string_view<char_type> format_str)
      : compiled_format_base<S>(format_str), format_str_(format_str) {}
};

```cpp

这段代码定义了一个模板结构体 `type_list` ，用于表示一系列输入参数的类型。其中，`template <typename... Args> struct type_list` 表示结构体 `type_list` 中的输入参数可以是任意数量的类型参数。

接下来，代码中定义了一个模板函数 `get` ，用于获取输入参数中的特定类型，并返回该类型的引用或静置值。

在模板函数 `get` 中，使用了 `constexpr` 特性，说明该函数可以被用于编译时类型检查。模板函数使用了一个 `template <int N, typename T, typename... Args>` 声明，表示该函数的输入参数包括一个整数 `N` 和一个或多个类型参数 `T` 和 `Args` 。

函数体中使用了 `static_assert` 语句，用于在编译时检查输入参数的类型是否正确。其中，第一个条件 `N < 1 + sizeof...(Args), "index is out of bounds"` 表示如果 `N` 大于 `1 + sizeof...(Args)`，那么编译器无法检查 `Args` 的类型，因此会报错。如果 `N` 等于 0，那么函数返回第一个输入参数 `first` 。否则，函数返回 `get<N - 1>(rest...)` 的结果，其中 `...` 表示省略了 `Args` 的类型参数列表。

接着，代码中定义了一个模板类 `get_type_impl`，其中 `template <int N, typename>` 表示该模板类可以被用于 `N` 类型的输入参数。在该模板类中，使用了 `constexpr` 特性，说明该模板类可以被用于编译时类型检查。


```
#ifdef __cpp_if_constexpr
template <typename... Args> struct type_list {};

// Returns a reference to the argument at index N from [first, rest...].
template <int N, typename T, typename... Args>
constexpr const auto& get([[maybe_unused]] const T& first,
                          [[maybe_unused]] const Args&... rest) {
  static_assert(N < 1 + sizeof...(Args), "index is out of bounds");
  if constexpr (N == 0)
    return first;
  else
    return get<N - 1>(rest...);
}

template <int N, typename> struct get_type_impl;

```cpp

这段代码定义了一个模板结构体 `get_type_impl`，其中 `N` 是模板参数，`Args` 是模板参数的序列。模板结构体的类型列表为 `type_list<Args...>`，其中 `Args` 是模板参数的序列。

`get_type_impl` 的作用是在编译时检查模板结构体 `get_type` 的实现，如果 `N` 是非零整数，则执行 `get<N>(std::declval<Args>()...)` 获取模板参数的值，否则忽略该值。然后通过 `remove_cvref_t<decltype(get<N>(std::declval<Args>()...)):>` 移除掉 `Args` 的类型注解，获取到模板参数的原始类型，最终返回该原始类型的别名类型。

`get_type` 的模板参数为 `N` 和一个类型参数 `T`，该类型参数允许用户定义模板结构体。`get_type` 的作用是在编译时检查模板结构体 `T` 的实现，如果 `N` 是非零整数，则返回 `get_type_impl<N, T>::type`，否则返回 `std::false_type`。

`text` 的模板参数为 `Char` 类型，该类型定义了一个 `basic_string_view<Char>` 类型的数据成员 `data`，以及一个 `char_type` 类型的别名成员 `char_type`，该别名成员引用 `Char` 类型。`text` 的模板部分定义了一个 `format` 函数，该函数接受一个输出迭代器 `out` 和一个或多个参数 `Args`。该函数通过调用 `write` 函数来将 `data` 中的内容写入 `out` 中，其中 `Args` 是模板参数的序列。


```
template <int N, typename... Args> struct get_type_impl<N, type_list<Args...>> {
  using type = remove_cvref_t<decltype(get<N>(std::declval<Args>()...))>;
};

template <int N, typename T>
using get_type = typename get_type_impl<N, T>::type;

template <typename T> struct is_compiled_format : std::false_type {};

template <typename Char> struct text {
  basic_string_view<Char> data;
  using char_type = Char;

  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&...) const {
    return write<Char>(out, data);
  }
};

```cpp

这段代码定义了一个模板结构体 `is_compiled_format`，模板参数为 `text<Char>`。

该模板结构体有一个 `std::true_type` 被继承自，说明这个模板结构体是一个真类型，即 `is_compiled_format` 中的类型必须是一个 valid 的 C++ type。

该代码还定义了一个模板函数 `make_text`，该函数接收一个 `basic_string_view<Char>` 类型的参数 `s` 和一个 `size_t` 类型的参数 `pos` 和 `size`。该函数返回一个 `code_unit` 类型的结构体，其中 `value` 是 `Char` 类型的变量，`char_type` 是 `Char` 类型。

该函数还有一个内部定义的模板函数 `format`，该函数接收一个 `OutputIt` 类型的参数 `out` 和一个或多个 `Args` 类型的参数。该函数使用 `write` 函数将 `value` 输出到 `out` 中，其中 `Args` 是传递给 `format` 的其他参数。

该代码还定义了一个代码单元 `code_unit` 类型，其中 `value` 是 `Char` 类型的变量，`char_type` 是 `Char` 类型。该类型还有一个模板参数 `OutputIt` 和一个模板参数组 `...`。该模板函数 `format` 的实现使用 `write` 函数将 `value` 输出到 `out` 中，其中 `Args` 是传递给 `format` 的其他参数。


```
template <typename Char>
struct is_compiled_format<text<Char>> : std::true_type {};

template <typename Char>
constexpr text<Char> make_text(basic_string_view<Char> s, size_t pos,
                               size_t size) {
  return {{&s[pos], size}};
}

template <typename Char> struct code_unit {
  Char value;
  using char_type = Char;

  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&...) const {
    return write<Char>(out, value);
  }
};

```cpp

这段代码定义了一个模板结构体 `field` 和一个模板类 `is_compiled_format` 。模板 `is_compiled_format` 的模板参数为 `code_unit<Char>`，意味着它可以是一个 `char` 类型的代码单元。

`is_compiled_format` 的作用是在编译时检查代码是否符合指定的格式。通过检查 `code_unit<Char>` 的参数，我们可以确保代码单元是 `char` 类型的。

`field` 的模板参数为 `Char`、`T` 和 `int N`。它定义了一个模板类，其中 `T` 是 `field` 的使用类型，`N` 是 `field` 的参数类型。

`field` 的模板包含一个通用的 `format` 函数，它的参数为 `OutputIt` 和 `Args` 类型的参数。这个函数的实现比较复杂，它通过 `write` 函数将 `T` 类型的参数写入到 `OutputIt` 类型的输出中。这里使用了 `const T&` 来确保输入参数的类型与 `field` 的参数类型匹配。

`is_compiled_format` 通过 `field` 的模板来检查代码是否符合指定的格式。如果检查通过，`is_compiled_format` 将输出 `true`，否则将输出 `false`。


```
template <typename Char>
struct is_compiled_format<code_unit<Char>> : std::true_type {};

// A replacement field that refers to argument N.
template <typename Char, typename T, int N> struct field {
  using char_type = Char;

  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&... args) const {
    // This ensures that the argument type is convertile to `const T&`.
    const T& arg = get<N>(args...);
    return write<Char>(out, arg);
  }
};

```cpp

这段代码定义了一个模板结构体 `is_compiled_format`，模板参数包括一个字符类型 `Char`、一个整型类型 `T` 和一个整数类型 `N`。

该模板结构体有一个成员变量 `spec_field`，其中 `using char_type` 被定义为 `Char` 类型，`formatter` 被定义为 `T` 类型，`fmt` 被定义为 `Char` 类型。

该模板结构体还有一个模板参数 `spec_field_args`，通过这个参数模板可以格式化 `spec_field` 中定义的模板，格式化参数是一个 `const T&` 类型的参数。

该代码的作用是定义了一个模板结构体 `is_compiled_format`，可以用来在编译时检查 `spec_field` 中定义的模板是否正确，如果模板定义正确，该结构体将返回 `std::true_type`。


```
template <typename Char, typename T, int N>
struct is_compiled_format<field<Char, T, N>> : std::true_type {};

// A replacement field that refers to argument N and has format specifiers.
template <typename Char, typename T, int N> struct spec_field {
  using char_type = Char;
  mutable formatter<T, Char> fmt;

  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&... args) const {
    // This ensures that the argument type is convertile to `const T&`.
    const T& arg = get<N>(args...);
    const auto& vargs =
        make_format_args<basic_format_context<OutputIt, Char>>(args...);
    basic_format_context<OutputIt, Char> ctx(out, vargs);
    return fmt.format(arg, ctx);
  }
};

```cpp

这段代码定义了一个模板结构体 `is_compiled_format`，模板参数包括三个类型参数 `Char`、`T` 和 `N`。这个模板结构体有一个真类型别名 `std::true_type`。

模板结构体的具体实现包括两个模板变体 `spec_field` 和 `concat`。

1. `spec_field` 模板变体定义了一个模板，通过 `spec_field<Char, T, N>` 可以获取到模板中 `Char`、`T` 和 `N` 的具体类型信息。该模板结构体是一个真类型别名，意味着它可以被赋值给某些其他类型的变量。

2. `concat` 模板结构体定义了一个模板，用于将两个字符串拼接成一个新的字符串。该模板结构体有两个成员变量 `L` 和 `R`，分别代表两个输入的字符串，以及一个用于格式化输出字符串的参数 `out`。

`concat` 的模板部分实现了一个通用的模板 `format`，该模板接受两个参数：`OutputIt` 和 `Args` 类型的参数。这个模板通过 `lhs.format(out, args...)` 将输入的字符串 `L` 的格式化输出，并通过 `rhs.format(out, args...)` 将输入的字符串 `R` 的格式化输出。最终将两个输出字符串连接成一个新的字符串并返回。

整段代码定义了一个模板结构体 `is_compiled_format`，以及两个模板变体 `spec_field` 和 `concat`。`is_compiled_format` 的模板形式是一个布尔类型，表示它可以被赋值给其他任何类型。`concat` 的模板则用于将两个字符串拼接成一个新的字符串，并可用于格式化输出字符串。


```
template <typename Char, typename T, int N>
struct is_compiled_format<spec_field<Char, T, N>> : std::true_type {};

template <typename L, typename R> struct concat {
  L lhs;
  R rhs;
  using char_type = typename L::char_type;

  template <typename OutputIt, typename... Args>
  OutputIt format(OutputIt out, const Args&... args) const {
    out = lhs.format(out, args...);
    return rhs.format(out, args...);
  }
};

```cpp

这段代码定义了一个模板结构体`is_compiled_format`，其中有两个类型参数`L`和`R`，用于指定两个不同类型的数据结构。这个模板结构体有一个成员变量`is_compiled_format<L, R>`，其类型为`std::true_type`，表示这个模板结构体是一个真类型。

这个模板结构体还定义了一个模板函数`make_concat<L, R>`，该函数将两个类型的数据结构`L`和`R`连接起来，并返回一个新的结构体类型的变量。

接下来，该代码定义了一个模板类`unknown_format`，其中有一个成员变量`unknown_format{}`，类型为未知类型。

最后，该代码定义了一个模板函数`size_t parse_text<Char>`，该函数接受一个`basic_string_view<Char>`类型的字符串`str`以及一个位置`pos`。该函数从字符串`str`中读取一个字符，并返回该位置。


```
template <typename L, typename R>
struct is_compiled_format<concat<L, R>> : std::true_type {};

template <typename L, typename R>
constexpr concat<L, R> make_concat(L lhs, R rhs) {
  return {lhs, rhs};
}

struct unknown_format {};

template <typename Char>
constexpr size_t parse_text(basic_string_view<Char> str, size_t pos) {
  for (size_t size = str.size(); pos != size; ++pos) {
    if (str[pos] == '{' || str[pos] == '}') break;
  }
  return pos;
}

```cpp

这段代码定义了一个模板结构体，名为`<typename Args, size_t POS, int ID, typename S>`，表示输入参数的类型为`Args`，输出参数的类型为`S`，参数的长度为`POS`，参数的ID为`ID`。

接下来是两个模板函数：

1. `compile_format_string`：

这是一个模板函数，接收一个`S`形式的格式字符串，返回编译器可以使用的相应格式字符的代码。这个函数的核心是：`constexpr auto compile_format_format_str(S format_str)`，其中`format_str`是输入的格式字符串，`S`是输出参数的类型，`constexpr`表示这个函数可以被编译为常量，而`auto`表示编译器可以自动推导出该函数的返回类型。

2. `parse_tail`：

这是一个模板函数，接收一个`T`类型的头和一个`S`形式的格式字符串，返回解析出的尾。头`T`类型没有被定义，因为它需要与输入参数的类型匹配。

`parse_tail`函数首先检查输入的格式字符串是否与头类型匹配，如果不匹配，则按照格式字符串的格式执行编译。接着，如果编译成功，就执行正常的解析操作，即删除头的前缀部分，并返回新的尾。如果编译失败，则返回输入的头。


```
template <typename Args, size_t POS, int ID, typename S>
constexpr auto compile_format_string(S format_str);

template <typename Args, size_t POS, int ID, typename T, typename S>
constexpr auto parse_tail(T head, S format_str) {
  if constexpr (POS !=
                basic_string_view<typename S::char_type>(format_str).size()) {
    constexpr auto tail = compile_format_string<Args, POS, ID>(format_str);
    if constexpr (std::is_same<remove_cvref_t<decltype(tail)>,
                               unknown_format>())
      return tail;
    else
      return make_concat(head, tail);
  } else {
    return head;
  }
}

```cpp

这段代码定义了一个模板结构体 `parse_specs_result`，其中 `T` 和 `Char` 是两个模板参数。

这个结构体有一个 `fmt` 成员变量，它是一个 `formatter` 对象，用于格式化 `T` 类型的数据。还有一个 `end` 成员变量，它是一个表示 `BasicStringView<Char>` 对象中当前字符位置的整数。还有一个 `next_arg_id` 成员变量，它是 `arg_id` 加上 1，用于跟踪输入参数的编号。

模板定义了一个名为 `parse_specs` 的函数，它接受一个 `basic_string_view<Char>` 类型的输入参数 `str`，以及一个整数参数 `arg_id`。这个函数首先将输入参数中的前缀去掉，然后创建一个 `BasicFormatParseContext` 对象 `ctx`，并将 `arg_id` 设为 1。接下来，它创建了一个 `formatter<T, Char>` 类型的对象 `f`，并调用 `f.parse` 函数来解析 `ctx` 中的输入参数。最后，它将解析结果返回到 `parse_specs_result` 结构体中，包括 `fmt`、`pos` 和 `next_arg_id` 三个成员变量。

具体来说，这段代码的作用是定义了一个模板结构体 `parse_specs_result`，其中包含了一个 `formatter` 对象 `fmt`，一个表示 `BasicStringView<Char>` 对象中当前字符位置的整数 `end`，和一个整数变量 `next_arg_id`。然后定义了一个名为 `parse_specs` 的函数，它接受一个 `basic_string_view<Char>` 类型的输入参数 `str` 和一个整数参数 `arg_id`。这个函数首先将输入参数中的前缀去掉，然后创建一个 `BasicFormatParseContext` 对象 `ctx`，并将 `arg_id` 设为 1。接下来，它创建了一个 `formatter<T, Char>` 类型的对象 `f`，并调用 `f.parse` 函数来解析 `ctx` 中的输入参数。最后，它将解析结果返回到 `parse_specs_result` 结构体中，包括 `fmt`、`pos` 和 `next_arg_id` 三个成员变量。


```
template <typename T, typename Char> struct parse_specs_result {
  formatter<T, Char> fmt;
  size_t end;
  int next_arg_id;
};

template <typename T, typename Char>
constexpr parse_specs_result<T, Char> parse_specs(basic_string_view<Char> str,
                                                  size_t pos, int arg_id) {
  str.remove_prefix(pos);
  auto ctx = basic_format_parse_context<Char>(str, {}, arg_id + 1);
  auto f = formatter<T, Char>();
  auto end = f.parse(ctx);
  return {f, pos + (end - str.data()) + 1, ctx.next_arg_id()};
}

```cpp

This is a C++ implementation of a `basic_string_view` template that manages a slice of a `char_type` string. The string can have the form `"..."` where `"..."` is a substring that is processed by the template.

The implementation includes two helper functions, `parse_specs` and `parse_tail`, which are used to parse the substring at the specified position and return the substring starting from that position, respectively.

The main function checks whether the given substring starts with `{` and matches the expected format string. If the substring matches, it either returns the substring starting from the matched position or all input if the substring does not match the expected format string. If the substring does not match the expected format string, it raises an error.

Here is a brief example of how this template can be used:
```
constexpr basic_string_view<char_type> str = format_str("{0}");
if (constexpr (str[POS] == '{') {
   if (POS + 1 == str.size())
     throw format_error("unmatched '{' in format string");
   if constexpr (str[POS + 1] == '{') {
     return parse_tail<Args, POS + 2, ID>(make_text(str, POS, 1), format_str);
   } else if constexpr (str[POS + 1] == '}') {
     using type = get_type<ID, Args>;
     return parse_tail<Args, POS + 2, ID + 1>(field<char_type, type, ID>(),
                                              format_str);
   } else if constexpr (str[POS + 1] == ':') {
     using type = get_type<ID, Args>;
     constexpr auto result = parse_specs<type>(str, POS + 2, ID);
     return parse_tail<Args, result.end, result.next_arg_id>(
         spec_field<char_type, type, ID>{result.fmt}, format_str);
   } else {
     return unknown_format();
   }
 } else if constexpr (str[POS] == '}') {
   if (POS + 1 == str.size())
     throw format_error("unmatched '}' in format string");
   return parse_tail<Args, POS + 2, ID>(make_text(str, POS, 1), format_str);
 } else {
   constexpr auto end = parse_text(str, POS + 1);
   if constexpr (end - POS > 1) {
     return parse_tail<Args, end, ID>(make_text(str, POS, end - POS),
                                      format_str);
   } else {
     return parse_tail<Args, end, ID>(code_unit<char_type>{str[POS]}, format_str);
   }
 }
}
```cpp
Note that this implementation includes a dummy implementation for the case where the input string does not match the expected format string, which is not very useful in practice.


```
// Compiles a non-empty format string and returns the compiled representation
// or unknown_format() on unrecognized input.
template <typename Args, size_t POS, int ID, typename S>
constexpr auto compile_format_string(S format_str) {
  using char_type = typename S::char_type;
  constexpr basic_string_view<char_type> str = format_str;
  if constexpr (str[POS] == '{') {
    if (POS + 1 == str.size())
      throw format_error("unmatched '{' in format string");
    if constexpr (str[POS + 1] == '{') {
      return parse_tail<Args, POS + 2, ID>(make_text(str, POS, 1), format_str);
    } else if constexpr (str[POS + 1] == '}') {
      using type = get_type<ID, Args>;
      return parse_tail<Args, POS + 2, ID + 1>(field<char_type, type, ID>(),
                                               format_str);
    } else if constexpr (str[POS + 1] == ':') {
      using type = get_type<ID, Args>;
      constexpr auto result = parse_specs<type>(str, POS + 2, ID);
      return parse_tail<Args, result.end, result.next_arg_id>(
          spec_field<char_type, type, ID>{result.fmt}, format_str);
    } else {
      return unknown_format();
    }
  } else if constexpr (str[POS] == '}') {
    if (POS + 1 == str.size())
      throw format_error("unmatched '}' in format string");
    return parse_tail<Args, POS + 2, ID>(make_text(str, POS, 1), format_str);
  } else {
    constexpr auto end = parse_text(str, POS + 1);
    if constexpr (end - POS > 1) {
      return parse_tail<Args, end, ID>(make_text(str, POS, end - POS),
                                       format_str);
    } else {
      return parse_tail<Args, end, ID>(code_unit<char_type>{str[POS]},
                                       format_str);
    }
  }
}

```cpp

这段代码定义了一个名为 `compile` 的模板函数，它的参数是一个格式字符串 `S` 和一个或多个参数 `Args`。它的作用是输出一个字符串，根据输入的格式字符串 `S` 是否为编译字符串，如果是，则执行编译操作并返回编译后的结果，否则直接返回输入的格式字符串。

模板元编程中的 `FMT_ENABLE_IF` 是一个用于在编译时检查代码中定义的条件句是否为真。如果条件成立，则会编译 `<...>`，否则不会编译，这样就可以避免在运行时产生未定义的行为。

该代码中，`is_compile_string<S>` 和 `is_compiled_string<S>` 是 `is_compile_string` 模板中的函数，用于检查给定的 `S` 是否为编译字符串，如果为，则执行编译操作，否则不执行。

`detail::make_text` 是 `detail::text` 函数的别名，用于将 `str` 变量初始化为指定的格式字符串，并返回一个新的字符串，其中 `str` 是模板形参，作为输入的 `S` 格式字符串。

`detail::compile_format_string` 是 `detail::format_string` 函数的别名，用于根据给定的格式字符串 `format_str` 和模板参数 `Args...` 编译字符串，其中 `str` 是模板形参，作为输入的 `S` 格式字符串，`Args...` 是模板参数，它们的类型列表作为 `format_str` 的第二组参数传递。

`detail::compiled_format` 是 `detail::format_format` 函数的别名，用于根据模板参数 `Args...` 和格式字符串编译字符串，其中 `str` 是模板形参，作为输入的 `S` 格式字符串，`Args...` 是模板参数，它们的类型列表作为 `format_str` 的第二组参数传递。

`to_string_view` 是 `std::string_view` 的别名，用于将 `format_str` 格式字符串转换为 `std::string_view` 类型，这样就可以作为模板形参传递给 `compile` 函数了。

最终，该代码的输出结果是，如果给定的 `S` 是编译字符串，则编译结果，否则直接返回输入的格式字符串。


```
template <typename... Args, typename S,
          FMT_ENABLE_IF(is_compile_string<S>::value ||
                        detail::is_compiled_string<S>::value)>
constexpr auto compile(S format_str) {
  constexpr basic_string_view<typename S::char_type> str = format_str;
  if constexpr (str.size() == 0) {
    return detail::make_text(str, 0, 0);
  } else {
    constexpr auto result =
        detail::compile_format_string<detail::type_list<Args...>, 0, 0>(
            format_str);
    if constexpr (std::is_same<remove_cvref_t<decltype(result)>,
                               detail::unknown_format>()) {
      return detail::compiled_format<S, Args...>(to_string_view(format_str));
    } else {
      return result;
    }
  }
}
```cpp

这段代码定义了两个函数，一个用于输出指定格式信息的编译输出，另一个是编译字符串模板，根据给定的模板参数进行编译。以下是代码的作用说明：

1. 第一个函数 `compile`：

这个函数接收一个格式字符串 `format_str`，并返回 `detail::compiled_format<S, Args...>`，其中 `S` 是格式字符串的类型参数，`Args...` 是模板实参的列表。

如果 `is_compile_string<S>::value` 为 `true`，那么编译器将会尝试将 `format_str` 解析为字符串，如果是字符串，则执行编译并返回。否则，返回 `detail::compiled_format<S, Args...>`，这个默认的实现会在编译时失败，需要手动调用。

2. 第二个函数 `compile_string`：

这个函数接收一个字符串常量 `format_str`，并返回 `detail::compiled_format<const Char*, Args...>`，其中 `const Char*` 是模板实参的类型参数，`Args...` 是模板实参的列表。

这个函数的实现与第一个函数 `compile` 类似，但是会将 `format_str` 转换为字符串类型，以便使用 `to_string_view` 函数进行字符串view操作。


```
#else
template <typename... Args, typename S,
          FMT_ENABLE_IF(is_compile_string<S>::value)>
constexpr auto compile(S format_str) -> detail::compiled_format<S, Args...> {
  return detail::compiled_format<S, Args...>(to_string_view(format_str));
}
#endif  // __cpp_if_constexpr

// Compiles the format string which must be a string literal.
template <typename... Args, typename Char, size_t N>
auto compile(const Char (&format_str)[N])
    -> detail::compiled_format<const Char*, Args...> {
  return detail::compiled_format<const Char*, Args...>(
      basic_string_view<Char>(format_str, N - 1));
}
}  // namespace detail

```cpp

这段代码是一个C++的模板，名为`FMT_DEPRECATED!`。它是一个不依赖任何实际参数的函数，用于返回其内部类型（即模板实参的类型）的别离型（即使用编译器时如何使用该类型）。

首先，我们定义了一个模板参数为`Args...`的函数`compile()`。这个函数采用`std::informative_group<int, std::remove_const<std::decltype<Args...>>..., std::arg_info<0>>`作为模板参数，其中`Args...`表示该函数可以接受任意数量的参数。

接着，我们实现了一个名为`detail::compile()`的函数，它采用`Args...`作为输入并返回一个指向`std::string`类型对象的指针。这个函数的具体实现方式在另一个名为`detail::compile()`的函数中，这个函数使用了`const_cast`来检查输入参数是否是编译器支持的字符类型（即`char_type`）。如果是，它就返回`std::string`类型对象的一个别离型，否则它返回一个空字符串。

最后，我们实现了一个名为`FMT_INLINE`的函数，它是一个编译时输出函数。它接受一个名为`std::basic_string<char_t>`的类型参数，这个类型参数是一个别离型，它表示字符串的别离型参数。然后，它接受一个参数`const CompiledFormat&`和一个参数`const Args&`，并将它们传递给函数的实现。函数的实现是使用`std::basic_string`的构造函数，这个构造函数将输入参数中的所有字符存储到一个内存缓冲区中，然后使用`std::basic_string`的结束标志（`std::ends`）将该缓冲区转换为一个字符串对象。最后，函数返回这个字符串对象的一个别离型。

总之，这段代码定义了一个模板函数`compile()`，它接受任意数量的参数，并返回一个指向`std::string`类型对象的指针。此外，我们还实现了一个名为`FMT_INLINE`的编译时输出函数，它可以将`std::basic_string<char_t>`的别离型作为输出。


```
// DEPRECATED! use FMT_COMPILE instead.
template <typename... Args>
FMT_DEPRECATED auto compile(const Args&... args)
    -> decltype(detail::compile(args...)) {
  return detail::compile(args...);
}

#if FMT_USE_CONSTEXPR
#  ifdef __cpp_if_constexpr

template <typename CompiledFormat, typename... Args,
          typename Char = typename CompiledFormat::char_type,
          FMT_ENABLE_IF(detail::is_compiled_format<CompiledFormat>::value)>
FMT_INLINE std::basic_string<Char> format(const CompiledFormat& cf,
                                          const Args&... args) {
  basic_memory_buffer<Char> buffer;
  cf.format(detail::buffer_appender<Char>(buffer), args...);
  return to_string(buffer);
}

```cpp

这段代码定义了一个模板，名为“format_to”，有两个参数：一个输出迭代器（OutputIt）和一个CompiledFormat类型，还带有一个省略号（out）和一个参数列表（Args）。通过这个模板，可以调用一个名为“format”的函数，该函数接受一个CompiledFormat类型的参数和若干个Args类型的参数。如果CF是true，则按照编译器定义的格式进行输出，否则会尝试从模板参数中推导出输出格式。

第一个模板声明了一个可以被 instantiated的模板，这个模板模板参数包括：一个输出迭代器out，一个CompiledFormat类型的参数cf，以及一个或多个Args类型的参数。通过这个模板，可以为Args类型定义了格式化输出所需的模板参数。

第二个模板声明了一个std::basic_string<Char>类型的函数，名为“format”，它的参数是CF和Args。函数使用了basic_memory_buffer<Char>和basic_function<char_type（CF.char_type）...，通过利用const CompiledFormat&和Args...来获取CF和Args的值，并使用detail::cf::vformat_to函数将格式化后的格式字符串输出到buffer中，最后使用to_string函数将buffer中的字符串转换为std::basic_string<Char>类型并返回。


```
template <typename OutputIt, typename CompiledFormat, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_format<CompiledFormat>::value)>
OutputIt format_to(OutputIt out, const CompiledFormat& cf,
                   const Args&... args) {
  return cf.format(out, args...);
}
#  endif  // __cpp_if_constexpr
#endif    // FMT_USE_CONSTEXPR

template <typename CompiledFormat, typename... Args,
          typename Char = typename CompiledFormat::char_type,
          FMT_ENABLE_IF(std::is_base_of<detail::basic_compiled_format,
                                        CompiledFormat>::value)>
std::basic_string<Char> format(const CompiledFormat& cf, const Args&... args) {
  basic_memory_buffer<Char> buffer;
  using context = buffer_context<Char>;
  detail::cf::vformat_to<context>(detail::buffer_appender<Char>(buffer), cf,
                                  make_format_args<context>(args...));
  return to_string(buffer);
}

```cpp

这段代码是一个模板函数，名为`format`，它接受一个`S`类型的参数，以及一个可迭代参数`Args`。

函数内部使用了`FMT_ENABLE_IF`为`std::is_compiled_string<S>::value`进行条件编译，如果条件为真，则使用编译期生成的`std::basic_string<typename S::char_type>`类型来格式化字符串。如果不是编译期生成的，则表示字符串长度为2（`str.size() == 2`），并且`str[0] == '{' && str[1] == '}'`，这种情况下返回`fmt::to_string(detail::first(args...))`。

具体实现中，函数首先根据参数的类型，对参数进行归约，如果归约后参数个数为0，则返回一个空字符串。否则，编译并返回一个格式化后的字符串，格式化后的字符串将`Args`中的所有参数作为字符串的一部分进行格式化，最终返回生成的字符串。


```
template <typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_string<S>::value)>
FMT_INLINE std::basic_string<typename S::char_type> format(const S&,
                                                           Args&&... args) {
#ifdef __cpp_if_constexpr
  if constexpr (std::is_same<typename S::char_type, char>::value) {
    constexpr basic_string_view<typename S::char_type> str = S();
    if (str.size() == 2 && str[0] == '{' && str[1] == '}')
      return fmt::to_string(detail::first(args...));
  }
#endif
  constexpr auto compiled = detail::compile<Args...>(S());
  return format(compiled, std::forward<Args>(args)...);
}

```cpp

这段代码定义了两种模板函数，`format_to` 和 `format_to_`.

这两种函数的参数个数不同，第一种有三个参数，第二种有两个参数。它们都有一个共同点，就是它们都接受一个输出迭代器（`OutputIt`）和一个编译格式（`CompiledFormat`）作为第一个参数，以及一个或多个参数作为第二个参数。

在函数体中，它们调用了 `detail::cf::vformat_to` 函数，这个函数将输入的输出迭代器和编译格式作为第一个参数，将第二个参数作为格式化参数。这个函数的实现比较复杂，因为 `detail::is_base_of<detail::basic_compiled_format,
                                       CompiledFormat>::value` 这个条件语句检查了输入的编译格式是否可以被当做基类 `detail::basic_compiled_format` 的子类。如果是，那么这个函数的实现就是第一种情况，否则就是第二种情况。

`format_to` 函数在第一个模板中，如果输入的参数个数是三种，那么它就是 `format_to`；如果输入的参数个数是两种，那么它就是 `format_to_`。它们的实现基本相同，只是第一种实现是在输入参数中自动推导出输出迭代器类型，而第二种实现需要显式地指定。


```
template <typename OutputIt, typename CompiledFormat, typename... Args,
          FMT_ENABLE_IF(std::is_base_of<detail::basic_compiled_format,
                                        CompiledFormat>::value)>
OutputIt format_to(OutputIt out, const CompiledFormat& cf,
                   const Args&... args) {
  using char_type = typename CompiledFormat::char_type;
  using context = format_context_t<OutputIt, char_type>;
  return detail::cf::vformat_to<context>(out, cf,
                                         make_format_args<context>(args...));
}

template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_string<S>::value)>
OutputIt format_to(OutputIt out, const S&, const Args&... args) {
  constexpr auto compiled = detail::compile<Args...>(S());
  return format_to(out, compiled, args...);
}

```cpp

这两段代码定义了一个模板函数名为“format_to_n_result<OutputIt>”，它接受三个模板参数：OutputIt、CompiledFormat 和 Args。

第一个模板参数是“OutputIt”，表示输出的迭代器类型，即被输出的元素类型。第二个模板参数是“CompiledFormat”，表示输出格式所使用的编译格式。它是一个可变参数模板，其中使用了FMT_ENABLE_IF函数，用于在编译时检查可变参数的数量是否为3。第三个模板参数是“Args...”，表示输入参数的类型，其中省略了一些参数，这些参数将在编译时检查。

这两段代码实现了一个将输入参数按照指定的格式进行输出，并返回输出的迭代器和格式信息。通过这两段代码，我们可以创建一个函数，将不同的输入参数按照指定的格式进行输出，例如将字符串s与整数n按照%s%d的格式进行输出，将字符串s与分数n按照%s%f的格式进行输出等。


```
template <
    typename OutputIt, typename CompiledFormat, typename... Args,
    FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value&& std::is_base_of<
                  detail::basic_compiled_format, CompiledFormat>::value)>
format_to_n_result<OutputIt> format_to_n(OutputIt out, size_t n,
                                         const CompiledFormat& cf,
                                         const Args&... args) {
  auto it =
      format_to(detail::truncating_iterator<OutputIt>(out, n), cf, args...);
  return {it.base(), it.count()};
}

template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_compiled_string<S>::value)>
format_to_n_result<OutputIt> format_to_n(OutputIt out, size_t n, const S&,
                                         const Args&... args) {
  constexpr auto compiled = detail::compile<Args...>(S());
  auto it = format_to(detail::truncating_iterator<OutputIt>(out, n), compiled,
                      args...);
  return {it.base(), it.count()};
}

```cpp

这段代码定义了一个模板类，名为`<fmt, typename... Args>`，其中`fmt`是一个已知类型的模板参数，后面的`Args`则是模板的实参类型。

该模板类定义了一个名为`formatted_size`的函数，其参数为`const CompiledFormat&`和`const Args&...`，返回值为`size_t`类型的数据类型。函数内部的代码块通过一个`const`类型的指针`cf`和一些`const`类型的实参`args...`，来获取输入参数的格式信息，并将这些信息传递给`format_to`函数进行格式化，最后返回格式化后的字符数。

具体来说，`formatted_size`函数的作用是接收一个格式化字符串（`const CompiledFormat&`）和一个或多个参数（`const Args&...`），并输出一个对应格式的字符数。这里的`...`表示参数个数的未知，可以让用户自己提供需要的参数类型，从而实现更加灵活的格式化操作。


```
template <typename CompiledFormat, typename... Args>
size_t formatted_size(const CompiledFormat& cf, const Args&... args) {
  return format_to(detail::counting_iterator(), cf, args...).count();
}

FMT_END_NAMESPACE

#endif  // FMT_COMPILE_H_

```