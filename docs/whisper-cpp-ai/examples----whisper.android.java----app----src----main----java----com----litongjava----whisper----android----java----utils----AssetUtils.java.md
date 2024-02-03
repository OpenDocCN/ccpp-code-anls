# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\utils\AssetUtils.java`

```cpp
package com.litongjava.whisper.android.java.utils;

import android.content.Context;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
importjava.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;

public class AssetUtils {
  private static Logger log = LoggerFactory.getLogger(AssetUtils.class);

  // 检查目标文件是否存在，如果不存在则从 assets 目录复制文件到目标目录
  public static File copyFileIfNotExists(Context context, File distDir, String filename) {
    // 创建目标文件对象
    File dstFile = new File(distDir, filename);
    // 如果目标文件已经存在，则直接返回
    if (dstFile.exists()) {
      return dstFile;
    } else {
      // 获取目标文件的父目录
      File parentFile = dstFile.getParentFile();
      log.info("parentFile:{}", parentFile);
      // 如果父目录不存在，则创建
      if (!parentFile.exists()) {
        parentFile.mkdirs();
      }
      // 从 assets 目录复制文件到目标目录
      AssetUtils.copyFileFromAssets(context, filename, dstFile);
    }
    return dstFile;
  }

  // 从 assets 目录复制整个目录到目标目录
  public static void copyDirectoryFromAssets(Context appCtx, String srcDir, String dstDir) {
    // 如果源目录或目标目录为空，则直接返回
    if (srcDir.isEmpty() || dstDir.isEmpty()) {
      return;
    }
    try {
      // 如果目标目录不存在，则创建
      if (!new File(dstDir).exists()) {
        new File(dstDir).mkdirs();
      }
      // 遍历源目录下的文件和子目录
      for (String fileName : appCtx.getAssets().list(srcDir)) {
        String srcSubPath = srcDir + File.separator + fileName;
        String dstSubPath = dstDir + File.separator + fileName;
        // 如果是子目录，则递归复制子目录
        if (new File(srcSubPath).isDirectory()) {
          copyDirectoryFromAssets(appCtx, srcSubPath, dstSubPath);
        } else {
          // 如果是文件，则复制文件
          copyFileFromAssets(appCtx, srcSubPath, dstSubPath);
        }
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  // 从 assets 目录复制单个文件到目标文件
  public static void copyFileFromAssets(Context appCtx, String srcPath, String dstPath) {
    File dstFile = new File(dstPath);
    copyFileFromAssets(appCtx, srcPath, dstFile);
  }

  // 从 assets 目录复制单个文件到目标文件
  public static void copyFileFromAssets(Context appCtx, String srcPath, File dstFile) {
    // 如果源文件路径为空，则直接返回
    if (srcPath.isEmpty()) {
      return;
    }
    // 声明输入流和输出流变量，并初始化为null
    InputStream is = null;
    OutputStream os = null;
    try {
      // 从应用程序上下文获取资产文件的输入流，并使用缓冲输入流包装
      is = new BufferedInputStream(appCtx.getAssets().open(srcPath));

      // 创建目标文件的输出流，并使用缓冲输出流包装
      os = new BufferedOutputStream(new FileOutputStream(dstFile));
      // 创建一个字节数组作为缓冲区
      byte[] buffer = new byte[1024];
      int length = 0;
      // 从输入流中读取数据到缓冲区，然后写入输出流中
      while ((length = is.read(buffer)) != -1) {
        os.write(buffer, 0, length);
      }
    } catch (FileNotFoundException e) {
      // 捕获文件未找到异常并打印堆栈跟踪
      e.printStackTrace();
    } catch (IOException e) {
      // 捕获IO异常并打印堆栈跟踪
      e.printStackTrace();
    } finally {
      try {
        // 关闭输出流和输入流
        os.close();
        is.close();
      } catch (IOException e) {
        // 捕获IO异常并打印堆栈跟踪
        e.printStackTrace();
      }
    }

  }
# 闭合大括号，表示代码块的结束
```