# `xmrig\src\3rdparty\hwloc\src\static-components.h`

```
# 声明外部可见的 hwloc_noos_component 结构体
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_noos_component;
# 声明外部可见的 hwloc_xml_component 结构体
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_xml_component;
# 声明外部可见的 hwloc_synthetic_component 结构体
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_synthetic_component;
# 声明外部可见的 hwloc_xml_nolibxml_component 结构体
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_xml_nolibxml_component;
# 声明外部可见的 hwloc_windows_component 结构体
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_windows_component;
# 声明外部可见的 hwloc_x86_component 结构体
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_x86_component;
# 静态数组，包含指向各种组件结构体的指针
static const struct hwloc_component * hwloc_static_components[] = {
  &hwloc_noos_component,
  &hwloc_xml_component,
  &hwloc_synthetic_component,
  &hwloc_xml_nolibxml_component,
  &hwloc_windows_component,
  &hwloc_x86_component,
  NULL
};
```