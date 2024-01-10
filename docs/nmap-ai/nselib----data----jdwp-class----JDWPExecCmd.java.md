# `nmap\nselib\data\jdwp-class\JDWPExecCmd.java`

```
// 导入 java.io 包
import java.io.*;

/* This is the JDWPExecCmd source used for jdwp-exec script to execute
 * a command on the remote system.
 *
 * It just executes the shell command passed as string argument to
 * run() function and returns its output.
 *
 * Compile simply with:
 * javac JDWPExecCmd.java (should be in the nselib/data/ directory).
 *
 * author = "Aleksandar Nikolic"
 * license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
*/

// 定义 JDWPExecCmd 类
public class JDWPExecCmd {
    // 定义 run 方法，用于执行远程系统上的命令
    public static String run(String cmd) {
        // 初始化结果字符串
        String result = cmd + " output:\n";
        try{
            // 执行传入的 shell 命令
            Process p = Runtime.getRuntime().exec(cmd);
            // 读取命令执行的输出流
            BufferedReader in = new BufferedReader(new InputStreamReader(p.getInputStream()));
            // 初始化行变量
            String line = null;
            // 逐行读取输出流内容并添加到结果字符串中
            while ((line = in.readLine()) != null) {
                result += line.trim()+"\n";
            }
            // 添加换行符
            result += "\n";
        }catch(Exception ex){
            // 捕获异常
        }
        // 返回结果字符串
        return result;
    }
}
```