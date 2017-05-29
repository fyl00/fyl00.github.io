---
layout:     post
title:      像 iOS10 音乐播放器一样自定义转场动画
date:       2017-05-23
summary:    OneShare2.0 版的设计里面，编辑界面的转场参考了 iOS10 中的音乐播放器的界面。
categories: iOS
---

> 苹果在 iOS10 里重新设计了音乐播放器（Music），当时在设计圈子还是个不小的事情。对于我个人而言，很喜欢它的「正在播放」界面的转场方式。最近打算对 OneShare 进行最后的升级，正好借鉴了这个转场。发现步骤虽然不难，但还是有点繁琐，有些细节需要注意一下。 

在 iOS 中，自定转场是通过实现协议 `UIViewControllerTransitioningDelegate` 来实现的。协议中提供了 TransitionAnimator（转场动画）、InteractiveAnimator（转场交互）和 PresentationController（视图层级控制）相关的方法。

```objective-c
// TransitionAnimator（转场动画）
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source;
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed;

// InteractiveAnimator（转场交互）
- (id<UIViewControllerInteractiveTransitioning>)interactionControllerForPresentation:(id<UIViewControllerAnimatedTransitioning>)animator;
- (id<UIViewControllerInteractiveTransitioning>)interactionControllerForDismissal:(id<UIViewControllerAnimatedTransitioning>)animator;

// PresentationController（视图层级控制）
- (UIPresentationController *)presentationControllerForPresentedViewController:(UIViewController *)presented presentingViewController:(UIViewController *)presenting sourceViewController:(UIViewController *)source;
```

大多数情况下，我们只会用 TransitionAnimator 那两个方法来实现转场动画。再复杂一点，我们希望实现自定手势交互的时候，就需要重写 InteractiveAnimator 那两个方法了。分析苹果的交互，会发现它还修改了视图层级的关系。所以接下来模仿此交互的步骤大概就是：

1. 实现 `UIViewControllerAnimatedTransitioning` 自定义转场动画；
2. 实现 `UIPresentationController` 自定义视图控制器的层级表现；
3. 实现 `UIViewControllerInteractiveTransitioning` 自定义交互方式。

所以首先在遵守 `UIViewControllerTransitioningDelegate` 协议的 `AnimatorController` 中，直接通过 `UIView` 的 `animateWithDuration` 方法去自定义转场动画即可，这一步并没什么难度。 

接下来实现视图层级控制之前，首先需要将被推出的视图（这里是 `ModalViewController`）的 `modalPresentationStyle` 设置为自定义（`UIModalPresentationCustom`）。这样才会调用协议中的 `presentationControllerForPresentedViewController` 方法。
实现 `UIPresentationController` 时，由于它同时接管了转入（presentation）和转出（dismissal），所以要在步骤 1 的基础上增加一个转出动画。这时可以另外新建一个类来处理转出动画，也可以直接在上面那个 `animateTransition:` 中实现，我使用的时后者。

```objective-c
switch (self.transitionType) {

    case AnimatorDismiss: {
        [UIView animateWithDuration:[self transitionDuration:transitionContext]
                            animations:^{
                            // your animation
                            } completion:^(BOOL finished) {
                                [transitionContext completeTransition:![transitionContext transitionWasCancelled]];
                            }];
        break;
    };

    case AnimatorPresent: {
        [[transitionContext containerView] addSubview:toViewController.view];
        // your variable
        [UIView animateWithDuration:[self transitionDuration:transitionContext] 
                            animations:^{
                            // your animation
                        }completion:^(BOOL finished) {
                            [transitionContext completeTransition:![transitionContext transitionWasCancelled]];
        }];
        break;
    };
}
```

最后就是实现交互了，因为是计划打算使用下拉手势，所以自定义的 `AnimatorInteractive` 是继承的 `UIPercentDrivenInteractiveTransition`，通过百分比来控制视图的动效。这里要注意在重写协议中的 `interactionControllerForDismissal` 方法时，不是通过手势转出的时候，要返回 `nil` ，否则协议不会去调用 `animateTransition:`。

```objective-c
- (id<UIViewControllerInteractiveTransitioning>)interactionControllerForDismissal:(id<UIViewControllerAnimatedTransitioning>)animator {
    return self.dismissInteractor.interactionInProgress ? self.dismissInteractor : nil;
}
```

另外由于在转场中使用了 `CGAffineTransformMakeScale` 转入，所以在转出时，需要设置 `toViewController.view.transform = CGAffineTransformIdentity` 。这时候遇到了个麻烦，在使用下拉手势的时候，父视图在交互过程中，就已经完成整个变化，导致取消下拉的时候，也不会复原。因为没有找到解决办法，仔细观察了下苹果的交互，发现也在下拉的时候父试图也没有动效，所以猜测没有直接解决办法，最后就[改成了音乐播放器这样的实现方式](https://github.com/fyl00/iOSDemo/commit/29bf71ad557b6d2dc83f22f454780055e5622c4a)。

这里提一句在使用 Swift 实现的时候，iOS10 上通过下拉手势关闭试图没成功的时候，`ModalViewController` 可能不会复原，这可能是苹果在 iOS10 改了 `UIView.animateTransition` 实现的原因，用 `UIViewPropertyAnimator` 的 `runningPropertyAnimator` 方法替代就可以了，在[之前的一篇博客](http://blog.slowwalker.me/ios/2017/02/16/iOS-UIViewPropertyAnimator-UIView.animate/)中提到过。

到这儿基本是就完了，还有最后一点修饰工作，就是 StatusBar 的样式。其实设置起来不麻烦，在 `ModalViewController` 里如下设置即可：

```Objective-C
- (void)viewDidLoad {
    [super viewDidLoad];
    self.modalPresentationCapturesStatusBarAppearance = YES;
    // Do any additional setup after loading the view.
    ...
}

- (UIStatusBarStyle)preferredStatusBarStyle {
    return UIStatusBarStyleLightContent;
}
```

但在实际操作中遇到了一点麻烦，因为不知道什么时候把 `Info.plist` 里的 `UIViewControllerBasedStatusBarAppearance` 设置为 `NO` 了，所以一直没有成功，最后看到[这篇文章](http://www.splinter.com.au/2015/12/30/status-bar-colours/)才意识到问题。总结起来决定 StatusBar 样式的优先级依次是:

```
Info.plist > UINavigationController > Modal 中的 preferredStatusBarStyle
```

-----
此文示例代码：[iOS10ModelPresent](https://github.com/fyl00/iOSDemo/tree/master/iOS10ModelPresent)

参考链接：

[Creating Custom UIViewController Transitions](https://www.raywenderlich.com/110536/custom-uiviewcontroller-transitions)

[Status bar colours: Everything there is to know](http://www.splinter.com.au/2015/12/30/status-bar-colours/)