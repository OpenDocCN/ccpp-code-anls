# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\whispercpp\java\whisper\CpuInfo.java`

```cpp
package com.whispercpp.java.whisper;

import android.os.Build;
import android.util.Log;

import androidx.annotation.RequiresApi;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class CpuInfo {
  private static final String LOG_TAG = "WhisperCpuConfig";

  private List<String> lines;

  public CpuInfo(List<String> lines) {
    this.lines = lines;
  }

  // 获取高性能 CPU 数量
  @RequiresApi(api = Build.VERSION_CODES.N)
  public int getHighPerfCpuCount0() {
    try {
      // 通过 CPU 频率获取高性能 CPU 数量
      return getHighPerfCpuCountByFrequencies();
    } catch (Exception e) {
      // 如果读取 CPU 频率失败，则通过 CPU 变体获取高性能 CPU 数量
      Log.d(LOG_TAG, "Couldn't read CPU frequencies", e);
      return getHighPerfCpuCountByVariant();
    }
  }

  // 通过 CPU 频率获取高性能 CPU 数量
  @RequiresApi(api = Build.VERSION_CODES.N)
  private int getHighPerfCpuCountByFrequencies() {
    // 获取 CPU 频率列表
    List<Integer> frequencies = getCpuValues("processor", line -> {
        try {
          // 获取最大 CPU 频率
          return getMaxCpuFrequency(Integer.parseInt(line.trim()));
        } catch (IOException e) {
          e.printStackTrace();
        }
        return 0;
      }
    );
    // 打印 CPU 频率分布
    Log.d(LOG_TAG, "Binned cpu frequencies (frequency, count): " + binnedValues(frequencies));
    // 计算去除最小值后的 CPU 数量
    return countDroppingMin(frequencies);
  }

  // 通过 CPU 变体获取高性能 CPU 数量
  @RequiresApi(api = Build.VERSION_CODES.N)
  private int getHighPerfCpuCountByVariant() {
    // 获取 CPU 变体列表
    List<Integer> variants = getCpuValues("CPU variant", line -> Integer.parseInt(line.trim().substring(line.indexOf("0x") + 2), 16));
    // 打印 CPU 变体分布
    Log.d(LOG_TAG, "Binned cpu variants (variant, count): " + binnedValues(variants));
    // 计算保留最小值后的 CPU 数量
    return countKeepingMin(variants);
  }

  // 统计数值分布
  @RequiresApi(api = Build.VERSION_CODES.N)
  private Map<Integer, Integer> binnedValues(List<Integer> values) {
    // 创建数值计数映射
    Map<Integer, Integer> countMap = new HashMap<>();
    // 遍历数值列表，统计每个数值出现的次数
    for (int value : values) {
      countMap.put(value, countMap.getOrDefault(value, 0) + 1);
    }
  // 返回计数映射
  return countMap;
}

@RequiresApi(api = Build.VERSION_CODES.N)
private List<Integer> getCpuValues(String property, Mapper mapper) {
  // 创建整数列表
  List<Integer> values = new ArrayList<>();
  // 遍历每行
  for (String line : lines) {
    // 如果行以指定属性开头
    if (line.startsWith(property)) {
      // 将映射后的值添加到列表中
      values.add(mapper.map(line.substring(line.indexOf(':') + 1)));
    }
  }
  // 对值列表进行排序
  values.sort(Integer::compareTo);
  // 返回值列表
  return values;
}

@RequiresApi(api = Build.VERSION_CODES.N)
private int countDroppingMin(List<Integer> values) {
  // 获取最小值
  int min = values.stream().mapToInt(i -> i).min().orElse(Integer.MAX_VALUE);
  // 返回大于最小值的元素数量
  return (int) values.stream().filter(value -> value > min).count();
}

@RequiresApi(api = Build.VERSION_CODES.N)
private int countKeepingMin(List<Integer> values) {
  // 获取最小值
  int min = values.stream().mapToInt(i -> i).min().orElse(Integer.MAX_VALUE);
  // 返回等于最小值的元素数量
  return (int) values.stream().filter(value -> value.equals(min)).count();
}

@RequiresApi(api = Build.VERSION_CODES.N)
public static int getHighPerfCpuCount() {
  try {
    // 读取 CPU 信息并返回高性能 CPU 数量
    return readCpuInfo().getHighPerfCpuCount0();
  } catch (Exception e) {
    // 捕获异常并返回可用处理器数量减去4的最大值
    Log.d(LOG_TAG, "Couldn't read CPU info", e);
    return Math.max(Runtime.getRuntime().availableProcessors() - 4, 0);
  }
}

private static CpuInfo readCpuInfo() throws IOException {
  // 读取 CPU 信息并返回 CpuInfo 对象
  try (BufferedReader reader = new BufferedReader(new FileReader("/proc/cpuinfo"))) {
    List<String> lines = new ArrayList<>();
    String line;
    // 逐行读取 CPU 信息
    while ((line = reader.readLine()) != null) {
      lines.add(line);
    }
    // 返回 CpuInfo 对象
    return new CpuInfo(lines);
  }
}

private static int getMaxCpuFrequency(int cpuIndex) throws IOException {
  // 构建 CPU 频率文件路径
  String path = "/sys/devices/system/cpu/cpu" + cpuIndex + "/cpufreq/cpuinfo_max_freq";
  // 读取最大 CPU 频率并返回
  try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
    return Integer.parseInt(reader.readLine());
  }
}

private interface Mapper {
  // 定义映射接口
  int map(String line);
}
# 闭合之前的代码块
```