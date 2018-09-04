# iOS12适配及问题记录

## 版本信息

> Xcode: Version 10.0 beta (10L176w)
> macOS: 10.14 Beta (18A293u)
> iOS: 12.0(16A5288q)


## 问题及解决过程

### 1,StatusBar内部结构改变
> 现象：crash
> 
> crash log：
> 
> -[_UIStatusBarIdentifier isEqualToString:]: unrecognized selector sent to instance 0x283452820
> 
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[_UIStatusBarIdentifier isEqualToString:]: unrecognized selector sent to instance 0x283452820'

—————————————————————————————————————————————————————

> 问题代码和解决方法

```

+ (NSString *)getIphoneXNetWorkStates {    
    UIApplication *app = [UIApplication sharedApplication];
    id statusBar = [[app valueForKeyPath:@"statusBar"] valueForKeyPath:@"statusBar"];
    id one = [statusBar valueForKeyPath:@"regions"];
    id two = [one valueForKeyPath:@"trailing"];
    NSArray *three = [two valueForKeyPath:@"displayItems"];
    NSString *state = @"无网络";
    for (UIView *view in three) {
        //alert: iOS12.0 情况下identifier的变成了类"_UIStatusBarIdentifier"而不是NSString，所以会在调用“isEqualToString”方法时发生crash，
        //修改前
//        NSString *identifier = [view valueForKeyPath:@"identifier"];
        //修改后
        NSString *identifier = [[view valueForKeyPath:@"identifier"] description];
        if ([identifier isEqualToString:@"_UIStatusBarWifiItem.signalStrengthDisplayIdentifier"]) {
            id item = [view valueForKeyPath:@"_item"];
            
            //alert: 这个问题和上边一样itemId是_UIStatusBarIdentifier 类型，不是string
            NSString *itemId = [[item valueForKeyPath:@"identifier"] description];
            if ([itemId isEqualToString:@"_UIStatusBarWifiItem"]) {
                state = @"WIFI";
            }
            state = @"不确定";
            
        } else if ([identifier isEqualToString:@"_UIStatusBarCellularItem.typeDisplayIdentifier"]) {
            UIView *statusBarStringView = [view valueForKeyPath:@"_view"];
            // 4G/3G/E
            state = [statusBarStringView valueForKeyPath:@"text"];
        }

    }
    
    return state;
}
```

### 2,[UIImage imageNamed:]不能正常加载Assets中的图片

解决：
将图片放到bundle中
使用一下方式加载即可
```    
NSString *path = [[NSBundle mainBundle] pathForResource:@"bg_login" ofType:@"png"];
_backgroundView = [[UIImageView alloc] initWithImage:[UIImage imageWithContentsOfFile:path]];
```
这个不能正常加载的情况只出现在个别的地方，目前找到的共性是加载的图片偏大，其他并没有头绪，感觉像是测试版本的Bug，google也没有人解答此类问题，后续会继续关注。

