#### 前言

----

最近项目中在横向滑动的PageViewController（WMScrollViewController）里嵌入了Flutter 的 CustomScrollView，原本会觉得一切ok，却出现了一个致命的问题：安卓嵌入之后滑动流程稳定，没有任何问题，iOS 嵌入之后出现Flutter 页面滑动卡顿、不流畅，体现在触发flutter 列表滑动的同时，会触发原生横滑，且必须垂直滑动（无一点左右的滑动偏移）才会稳定触发Flutter 列表的滑动，否则容易触发左右横滑，所以我们习惯了单手用手机的人，根本无法顺畅滑动。如下面的gif（其实我已经很努力的在上下滑了，很明显上下滑动的距离大于水平滑动的距离，用户的用意也应该是要上下滑动，但是却触发了横滑）：

<img src="/Users/ddreader/Desktop/PersionalDEV/HYHKnowledgePoint/自己博客/Flutter/1595228716860927.png" alt="1595228716860927" style="zoom:80%;" />

#### 解决方式

------

首先触摸flutter的页面，原生的UIScrollView的panGesture也会收到触摸事件，导致UIScrollView的手势生效，开始我想着如果能再UIScrollView的手势代理方法中区分出Flutter CustomScrollView的手势，然后再做调整。但是这条路是错误的，因为Flutter的滑动手势并不是采用系统提供的手势，在断点时发现，触发之后，并没有相关flutter 上的 系统手势类。所以我改变了思路用以下去解决这个问题：

1. 如果当前触摸在FlutterView上，我们的子类UIScrollView 手势就不接收touch事件，控制其手势不会生效；
2. 继承FLBFlutterViewContainer父类，重写touches相关方法，手动调整UIScrollView 的contentOffset（注意: 我们混合栈使用的是闲鱼的FlutterBoost， FLBFlutterViewContainer继承自FlutterViewController）

上代码：

1. 自定义继承自UIScrollView的子类，并且实现手势代理方法，控制Touch的传递。如果touch的View是FlutterView 我们的ScrollView就不接收这个touch事件。

   ```objective-c
   @interface WMScrollView()
   
   @end
   
   @implementation WMScrollView
   
   #pragma mark - <UIGestureRecognizerDelegate>
   
   - (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch {
       if ([NSStringFromClass(touch.view.class) isEqualToString:@"FlutterView"]) {
           return NO;
       }
       return YES;
   }
   
   @end
   ```

   

 2. 自定义一个继承自FLBFlutterViewContainer的子类，重写Touches方法，在touchBegan方法的时候，我们让自定的ScrollView手势不生效；在touchMoved的时候，我们干预计算触摸方向的识别，如果在纵滑<5 && 横滑大于2的时候我们默认用户想要横滑，故手动改变ScrollView的contentOffset，否则的话手势继续不生效， flutterView 生效；在touchEnd和touchCancel的情况下，我们判断当前的contentOffset，如果距离大于当前页距离的1/3, 跳转到下一页，如果距离小于上一页距离的2/3, 跳转到前一页；负责调整offset回到当前页面的起始位置。

    ```
    @implementation DDScrollFlutterViewContainer
    
    - (void)viewDidLoad {
        [super viewDidLoad];
     }
    
    - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
        [super touchesBegan:touches withEvent:event];
        [self touchesBegan];
    }
    
    - (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
        [super touchesMoved:touches withEvent:event];
        UITouch *touch = [touches anyObject];
        [self touchMoveWithTouch:touch];
    }
    
    - (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
        [super touchesEnded:touches withEvent:event];
        [self touchEnd];
    }
    
    - (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
        [super touchesCancelled:touches withEvent:event];
        [self touchEnd];
    }
    
    - (void)touchesBegan {
        UIScrollView *superView = (UIScrollView *)self.view.superview;
        superView.panGestureRecognizer.enabled = NO;
    }
    
    - (void)touchMoveWithTouch:(UITouch *)touch {
        CGPoint previoursLocation = [touch previousLocationInView:self.view];
        CGPoint location = [touch locationInView:self.view];
        CGFloat deltax = fabs(location.x - previoursLocation.x);
        CGFloat deltay = fabs(location.y - previoursLocation.y);
        UIScrollView *superView = (UIScrollView *)self.view.superview;
        if (deltax > 2 && deltay <= 5) {
            CGPoint offset = superView.contentOffset;
            superView.panGestureRecognizer.enabled = YES;
            [superView.delegate scrollViewWillBeginDragging:superView];
            [superView setContentOffset:CGPointMake(offset.x - (location.x - previoursLocation.x), offset.y)];
        } else {
            superView.panGestureRecognizer.enabled = NO;
        }
    }
    
    - (void)touchEnd {
        UIScrollView *superView = (UIScrollView *)self.view.superview;
        superView.panGestureRecognizer.enabled = YES;
        CGPoint offset = superView.contentOffset;
        CGFloat offsetX = offset.x;
        if (offsetX > (1 * self.view.width+ self.view.width * 1 / 3)) {
            [superView setContentOffset:CGPointMake(2*self.view.width, offset.y) animated:YES];
        } else if (offsetX <=  self.view.width * 2 / 3) {
            [superView setContentOffset:CGPointMake(0, offset.y) animated:YES];
        } else {
            [superView setContentOffset:CGPointMake(self.view.width, offset.y) animated:YES];
        }
        [superView.delegate scrollViewDidEndDecelerating:superView];
    }
    
    
    @end
    ```

    

处理之后的效果：

<img src="/Users/ddreader/Desktop/PersionalDEV/HYHKnowledgePoint/自己博客/Flutter/屏幕录制2020-07-20 下午4.22.56.png" alt="屏幕录制2020-07-20 下午4.22.56" style="zoom:40%;" />

#### 总结

-----

这个问题也是困扰了许久，因为在安卓上表现良好，而在iphone上表现极其的不舒适，所以最初考虑到这也是Flutter团队对于CustomScrollView 原生渲染适配的问题，但是既然他们适配不好，我们就需要手动的去适配。但是从解决办法的探索中，我们发现Flutter的手势并不是通过iOS原生的手势实现的，而是自己的一套手势识别。而且FlutterViewController 的touches事件是被重写了，因为假如你重写了touches的方法，没有调用父类方法，那么会发现flutter 不能滑动，所以判断flutter的滑动事件Touches方法也是参与其中，所以记得调用父类方法。而我们重写了Touches的事件也是区分了一下用户意图要滑动的方向，相当于做了个手势的方向的识别。