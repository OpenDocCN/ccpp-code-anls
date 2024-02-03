# `whisper.cpp\examples\whisper.objc\whisper.objc\SceneDelegate.h`

```cpp
//
//  SceneDelegate.h
//  whisper.objc
//  该文件是 SceneDelegate 类的头文件，用于声明 SceneDelegate 类
//
//  Created by Georgi Gerganov on 23.10.22.
//  由 Georgi Gerganov 在 2023 年 10 月 23 日创建
//

#import <UIKit/UIKit.h>
// 导入 UIKit 框架

@interface SceneDelegate : UIResponder <UIWindowSceneDelegate>
// 声明 SceneDelegate 类，继承自 UIResponder 类，遵循 UIWindowSceneDelegate 协议

@property (strong, nonatomic) UIWindow * window;
// 声明 window 属性，类型为 UIWindow 类型的指针，使用 strong 修饰，支持自动引用计数

@end
// 结束 SceneDelegate 类的声明
```