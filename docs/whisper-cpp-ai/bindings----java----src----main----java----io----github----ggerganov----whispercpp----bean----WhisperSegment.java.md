# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\bean\WhisperSegment.java`

```cpp
/**
 * WhisperSegment 类，表示语音分段
 */
public class WhisperSegment {
  // 分段起始时间
  private long start;
  // 分段结束时间
  private long end;
  // 分段内容
  private String sentence;

  // 无参构造函数
  public WhisperSegment() {
  }

  // 带参构造函数，初始化分段的起始时间、结束时间和内容
  public WhisperSegment(long start, long end, String sentence) {
    this.start = start;
    this.end = end;
    this.sentence = sentence;
  }

  // 获取分段起始时间
  public long getStart() {
    return start;
  }

  // 获取分段结束时间
  public long getEnd() {
    return end;
  }

  // 获取分段内容
  public String getSentence() {
    return sentence;
  }

  // 设置分段起始时间
  public void setStart(long start) {
    this.start = start;
  }

  // 设置分段结束时间
  public void setEnd(long end) {
    this.end = end;
  }

  // 设置分段内容
  public void setSentence(String sentence) {
    this.sentence = sentence;
  }

  // 重写 toString 方法，返回分段的字符串表示
  @Override
  public String toString() {
    return "[" + start + " --> " + end + "]:" + sentence;
  }
}
```