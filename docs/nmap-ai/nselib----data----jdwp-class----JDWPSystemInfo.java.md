# `nmap\nselib\data\jdwp-class\JDWPSystemInfo.java`

```
import java.io.*;
import java.util.Date;
/* This is the JDWPSystemInfo source used for jdwp-info script to get remote
 * system information.
 *
 * Compile simply with:
 * javac JDWPSystemInfo.java (should be in the nselib/data/jdwp-class directory).
 *
 * author = "Aleksandar Nikolic"
 * license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
*/

public class JDWPSystemInfo {
    // 定义一个静态方法，用于获取系统信息并返回字符串形式的结果
    public static String run() {
        // 初始化结果字符串
        String result = "";
        // 获取可用处理器数量并添加到结果字符串
        result += "Available processors: " +  Runtime.getRuntime().availableProcessors() + "\n";
        // 获取空闲内存并添加到结果字符串
        result += "Free memory: " + Runtime.getRuntime().freeMemory() + "\n";
        // 获取文件系统根目录信息并添加到结果字符串
        File[] roots = File.listRoots();
        for (File root : roots) {
            result += "File system root: " + root.getAbsolutePath() + "\n";
            result += "Total space (bytes): " + root.getTotalSpace() + "\n";
            result += "Free space (bytes): " + root.getFreeSpace() + "\n";
        }
        // 获取操作系统名称并添加到结果字符串
        result += "Name of the OS: " + System.getProperty("os.name") + "\n";
        // 获取操作系统版本并添加到结果字符串
        result += "OS Version : " + System.getProperty("os.version") + "\n";
        // 获取操作系统补丁级别并添加到结果字符串
        result += "OS patch level : " + System.getProperty("sun.os.patch.level") + "\n";
        // 获取操作系统架构并添加到结果字符串
        result += "OS Architecture: " + System.getProperty("os.arch") + "\n";
        // 获取Java版本并添加到结果字符串
        result += "Java version: " + System.getProperty("java.version") + "\n";
        // 获取用户名并添加到结果字符串
        result += "Username: " + System.getProperty("user.name") + "\n";
        // 获取用户主目录并添加到结果字符串
        result += "User home: " + System.getProperty("user.home") + "\n";
        // 获取当前系统时间并添加到结果字符串
        Date dateNow = new Date();
        result += "System time: " + dateNow + "\n";

        // 返回系统信息结果字符串
        return result;
    }

    // 主方法，用于执行run方法并将结果打印到控制台
    public static void main(String[] args){
        System.out.println(run());
    }

}
```