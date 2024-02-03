# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\model\WhisperTokenData.java`

```cpp
/**
 * 表示令牌数据的结构体。
 */
public class WhisperTokenData extends Structure {

    /** 令牌 ID。 */
    public int id;

    /** 强制时间戳令牌 ID。 */
    public int tid;

    /** 令牌的概率。 */
    public float p;

    /** 令牌的对数概率。 */
    public float plog;

    /** 时间戳令牌的概率。 */
    public float pt;

    /** 所有时间戳令牌概率的总和。 */
    public float ptsum;

    /**
     * 令牌的开始时间（令牌级别时间戳数据）。
     * 如果您没有计算令牌级别时间戳，请勿使用。
     */
    public long t0;

    /**
     * 令牌的结束时间（令牌级别时间戳数据）。
     * 如果您没有计算令牌级别时间戳，请勿使用。
     */
    public long t1;

    /** 令牌的语音长度。 */
    public float vlen;

    @Override
    protected List<String> getFieldOrder() {
        return Arrays.asList("id", "tid", "p", "plog", "pt", "ptsum", "t0", "t1", "vlen");
    }
}
```