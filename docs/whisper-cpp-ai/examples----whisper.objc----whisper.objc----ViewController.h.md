# `whisper.cpp\examples\whisper.objc\whisper.objc\ViewController.h`

```cpp
//
//  ViewController.h
//  whisper.objc
//
//  Created by Georgi Gerganov on 23.10.22.
//

// 导入 UIKit 框架
#import <UIKit/UIKit.h>

// 导入音频相关框架
#import <AVFoundation/AVFoundation.h>
#import <AudioToolbox/AudioQueue.h>

// 定义常量：缓冲区数量、最大音频时长、采样率
#define NUM_BUFFERS 3
#define MAX_AUDIO_SEC 30
#define SAMPLE_RATE 16000

// 定义 whisper_context 结构体
struct whisper_context;

// 定义 StateInp 结构体
typedef struct
{
    int ggwaveId;
    bool isCapturing;
    bool isTranscribing;
    bool isRealtime;
    UILabel * labelReceived;

    AudioQueueRef queue;
    AudioStreamBasicDescription dataFormat;
    AudioQueueBufferRef buffers[NUM_BUFFERS];

    int n_samples;
    int16_t * audioBufferI16;
    float   * audioBufferF32;

    struct whisper_context * ctx;

    void * vc;
} StateInp;

// 定义 ViewController 类
@interface ViewController : UIViewController
{
    StateInp stateInp;
}

@end
```