---
layout:     post
title:      像 iOS10 音乐播放器一样自定义转场动画
date:       2017-05-23
summary:    OneShare2.0 版的设计里面，编辑界面的转场参考了 iOS10 中的音乐播放器的界面。
categories: iOS
---

在 iOS 中自定转场是通过实现协议 `UIViewControllerTransitioningDelegate` 来实现的。如果只是简单地实现一个转场，而不涉及到模态交互的修改（`modalPresentationStyle`），那只要重写协议中 `animationControllerForPresentedController:presentingController:sourceController:` 和 `animationControllerForDismissedController:` 方法就行了。但如果想做一个像 iOS10 音乐播放器里正在播放页面的交互，那么需要重写协议中的 `presentationControllerForPresentedViewController:presentingViewController:sourceViewController:` 方法来自定义视图控制器的层级结构，以及重写协议中 `interactionControllerForPresentation:` 和 `interactionControllerForDismissal:` 这两个方法来自定义交互方式了。概括起来就是一下三个步骤：

> 1. 实现 `UIViewControllerAnimatedTransitioning` 自定义转场动画；
> 2. 实现 `UIPresentationController` 自定义视图控制器的层级表现；
> 3. 实现 `UIViewControllerInteractiveTransitioning` 自定义交互方式。

在步骤 1 中，在重写遵守协议 `UIViewControllerTransitioningDelegate` 的类里面 `animateTransition:` 方法时，直接通过 `UIView` 的 `animateWithDuration:animations:completion:` 就能实现。

进入步骤 2 后，首先需要将被 push 出的 `UIViewController` 的 `modalPresentationStyle` 设置为 custom，这样才会调用 `UIViewControllerTransitioningDelegate` 中的 `presentationControllerForPresentedViewController` 方法。在自定义的 `UIPresentationController` 子类中，重写在调用 delegate 时会被使用 `presentationTransitionDidEnd` 和 `dismissalTransitionDidEnd`，同时在由于完全接管了 presentation 和 dismissal，步骤 1 中的转场动画还要增加 dismissal 动画，可以另外新建一个遵守 `UIViewControllerAnimatedTransitioning` 的类，也可以直接在上面那个 `animateTransition:` 中实现，我使用的时后者。

```objectivec
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

接下来就是自定义 `UIViewControllerInteractiveTransitioning` 子类实现步骤 3 中的滑动交互。由于这里使用的是常见下拉手势 `UIPanGestureRecognizer` 来关闭当前视图，所以是继承 InteractiveTransitioning 子类 `UIPercentDrivenInteractiveTransition`。实现 InteractiveTransitioning 之后再去重写 `interactionControllerForDismissal` 时，要注意在通过按钮而不是手势交互的时候，返回 `nil` ，不然步骤 1 中提到的 delegate 不会去调用 `animateTransition:`。

```objectivec
- (id<UIViewControllerInteractiveTransitioning>)interactionControllerForDismissal:(id<UIViewControllerAnimatedTransitioning>)animator {
    return self.dismissInteractor.interactionInProgress ? self.dismissInteractor : nil;
}
```

另外由于在转场中使用了 `CGAffineTransformMakeScale` 转入，所以在转出时，需要设置 `toViewController.view.transform = CGAffineTransformIdentity` 。这时候遇到了个麻烦，在使用下拉手势的时候，父视图在交互过程中，就已经完成整个变化，导致取消下拉的时候，也不会复原。因为没有找到解决办法，而且 iOS10 音乐播放器也在下拉的时候父试图也没有动效，所以猜测没有直接对解决办法，最后就[改成了音乐播放器这样的实现方式]（https://github.com/fyl00/iOSDemo/commit/29bf71ad557b6d2dc83f22f454780055e5622c4a）。


-----
此文示例源代码：[iOS10ModelPresent](https://github.com/fyl00/iOSDemo/tree/master/iOS10ModelPresent)

参考链接：[Creating Custom UIViewController Transitions](https://www.raywenderlich.com/110536/custom-uiviewcontroller-transitions)