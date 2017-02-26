---
layout:     post
title:      UIView.animate 在 iOS10 里的一个变动
date:       2017-02-16
summary:    UIView.animate 经常会用在转场动画 (UIViewControllerAnimatedTransitioning) 中，这个方法的效果在 iOS10 中和老版本中产生了不同，这时候使用 iOS10 新增的 UIViewPropertyAnimator.runningPropertyAnimator 方法可以达到老版本的效果。
categories: iOS
---

> 在写 OneShare 的时候加了个对 ModalView 可以下拉关闭的功能，在写转场动画的时候，发现取消下拉手势的时候，`UIView.animate` 在 iOS10 里的表现和之前不一样了。最后通过搜索，找到了 iOS10 中新增的 `UIViewPropertyAnimator.runningPropertyAnimator` 可以达到之前想要的效果。所以研究了下这两个方法在手势被取消的时候，有什么不同。

在 iOS9 中利用 `UIViewControllerAnimatedTransitioning` 重写转场动画的时候，会在 `animateTransition(using:)` 方法里面使用 `UIView.animate` 来实现转换效果。比如在 `ModalViewController` 上自定一个下拉的效果，就可以写成：

```swift
let containerView = transitionContext.containerView
guard let fromVC = transitionContext.viewController(forKey: .from),
      let toVC = transitionContext.viewController(forKey: .to)
else {return}
containerView.insertSubview(toVC.view, belowSubview: fromVC.view)

let fromVCFinalFrame = CGRect(origin: CGPoint(x: 0, y: UIScreen.main.bounds.height), 
                              size: UIScreen.main.bound.)

switch style {

case .modal:
    
    UIView.animate(
        withDuration: transitionDuration(using: transitionContext),
        delay: 0,
        options: .curveEaseInOut,
        animations: {
            fromVC.view.alpha = 0.3
            fromVC.view.frame = fromVCFinalFrame
    },
        completion: { _ in
            // 下拉手势被取消
            if transitionContext.transitionWasCancelled {
                toVC.view.removeFromSuperview()
            }
            transitionContext.completeTransition(!transitionContext.transitionWasCancelled)
    })
    
default:
    break

}     
```

这时候没有改变 `fromVCFinalFrame` 的大小，只是改变了坐标，所以不会有什么问题，如果需要改变 `fromVCFinalFrame` 的大小，比如下拉的时候实现 `fromVC` 往中间变小然后消失，这时候 `fromVCFinalFrame` 需要改为：

```swift
let centerPoint = CGPoint(x: UIScreen.main.bounds.width / 2.0 , y: UIScreen.main.bounds.height / 2.0)
let fromVCFinalFrame = CGRect(origin: centerPoint, size: CGSize(width: 0.0, height: 0.0))
```

只这么修改，这在 iOS9 里面是没有问题的，但是在 iOS10 中会发现如果取消下拉的时候，不会恢复到之前的样子，`fromVC.view` 的高宽都会变为 0，所以需要在下拉手势取消的时候重新设置 `fromVC.view` 的 `frame`。

```swift
if transitionContext.transitionWasCancelled {
    toVC.view.removeFromSuperview()
    // iOS10 里面需要重新设置 fromVC.view 的 frame，不然会消失
    // fromVC.view.frame = fromVCOrignFrame
}
```

但是这样做还是会有问题，会出现 `fromVC.view` 一闪而过再恢复到原来的样子，还是不会像 iOS9 里那样。这时候使用 iOS10 中新增的 `UIViewPropertyAnimator.runningPropertyAnimator` 取代 `UIView.animate` 来实现动效到时候，在转场被取消的时候，会达到 `UIView.animate` 在 iOS9 中直接恢复到之前的效果。

```swift
if #available(iOS 10.0, *) {
    UIViewPropertyAnimator.runningPropertyAnimator(
        withDuration: transitionDuration(using: transitionContext),
        delay: 0,
        options: .curveEaseInOut,
        animations:  {
            fromVC.view.alpha = 0.3
            fromVC.view.frame = fromVCFinalFrame
    },
        completion: { _ in
            if transitionContext.transitionWasCancelled {
                toVC.view.removeFromSuperview()
            }
            transitionContext.completeTransition(!transitionContext.transitionWasCancelled)
    })
    break
}
```

在 `ModalViewController` 里面加个居中的 `UILabel` 的会发现下拉取消的时候，`UILabel` 会在左上角上，算坐标会发现就是对 `fromVCFinalFrame` 居中后的坐标。所以推测，iOS9 中的 `UIView.animate` 和 iOS10 中的 `UIViewPropertyAnimator.runningPropertyAnimator` 实现转场动效，在转场被取消的时候，`animation` 参数中的设置会直接恢复到转场之前，但是其他被影响到的界面(如 subview)则不会。而 `UIView.animate` 在 iOS10 遇到同样场景的时候，则不会恢复到之前，需要在转场被取消的时候重新设置来恢复到之前(如此文例子中重新设置 `fromVC.view.frame` 的情况)。

---
参考链接：

[Using custom interactive transitions to dismiss a modal](http://www.thorntech.com/2016/02/ios-tutorial-close-modal-dragging/)