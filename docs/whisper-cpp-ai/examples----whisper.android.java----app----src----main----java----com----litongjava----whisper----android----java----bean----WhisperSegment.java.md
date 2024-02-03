# `whisper.cpp\examples\whisper.android.java\app\src\main\java\com\litongjava\whisper\android\java\bean\WhisperSegment.java`

```cpp
/**
 * WhisperSegment类用于表示一个语音片段，包含起始时间、结束时间和语句内容
 */
package com.litongjava.whisper.android.java.bean;

/**
 * WhisperSegment类的构造函数，用于创建一个空的WhisperSegment对象
 */
public class WhisperSegment {
  private long start, end;
  private String sentence;

  /**
   * WhisperSegment类的构造函数，用于创建一个包含起始时间、结束时间和语句内容的WhisperSegment对象
   * @param start 起始时间
   * @param end 结束时间
   * @param sentence 语句内容
   */
  public WhisperSegment(long start, long end, String sentence) {
    this.start = start;
    this.end = end;
    this.sentence = sentence;
  }

  /**
   * 获取起始时间
   * @return 起始时间
   */
  public long getStart() {
    return start;
  }

  /**
   * 获取结束时间
   * @return 结束时间
   */
  public long getEnd() {
    return end;
  }

  /**
   * 获取语句内容
   * @return 语句内容
   */
  public String getSentence() {
    return sentence;
  }

  /**
   * 设置起始时间
   * @param start 起始时间
   */
  public void setStart(long start) {
    this.start = start;
  }

  /**
   * 设置结束时间
   * @param end 结束时间
   */
  public void setEnd(long end) {
    this.end = end;
  }

  /**
   * 设置语句内容
   * @param sentence 语句内容
   */
  public void setSentence(String sentence) {
    this.sentence = sentence;
  }

  /**
   * 重写toString方法，返回WhisperSegment对象的字符串表示形式
   * @return WhisperSegment对象的字符串表示形式
   */
  @Override
  public String toString() {
    return "["+start+" --> "+end+"]:"+sentence;
  }
}
```