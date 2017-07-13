---
title:浅谈 Android ValueAnimator
---

Android 3.0之后动画氛围两种
1. View Animation
  - Tween Animation

    可以实现简单的 旋转(rotate) 平移(translate) 缩放(scale) 透明度变化(alpha)
  - Frame Animation

    补间动画，其实就是把一张张图片播放出来

2. Property Animation (3.0 level11 以后安卓官方支持)
  - ObjectAnimation 作用于控件的某一个属性（本篇略）
  - ValueAnimator

#### ValueAnimator

ValueAnimation本身并不会与任何一个属性发生作用，它本身也不提供任何动画。简单来说，ValueAnimation就是一个
数值发生器。它可以生产你想要的各种数值。其实，在Android属性动画中，如何产生每一步具体实现动画，都是通过
ValueAnimator计算出来的。

比如我们现在要实现一个从0~100的位移动画，ValueAnimator会根据动画持续的总时间产生一个0~1时间因子，
有了这样一个时间因子。通过相应的变幻，就可以根据你的startValue和endValue来生成相应的值。
同时，通过插值器的使用，我们还可以进一步控制每一个时间因子它产生值的一个变化速率。
如果我们使用线性插值器，那么它生成数值的时候，就会形成一个线性变化，只要时间相同，它的增量也相同。
如果我们使用一个加速度的插值器，那么它的增量变化就会呈现一个二次曲线图。增长率会越来越快。
由于我们的ValueAnimator并不响应任何一个动画，也不能控制任何一个属性，所以它并没有ObjectAnimator使用的那么广泛。
我们还是来看一下如何在程序中使用ValueAnimator吧。

下面我们利用ValueAnimator做一个水平方向初速度的抛物线
首先我们把抛物线分解成一个竖直方向（加速度恒定的直线运动）的运动和一个水平方向（匀速直线运动）的运动
1. 水平方向的运动
```
ValueAnimator animator = ValueAnimator.ofFloat(0,1);
        animator.setDuration(1000).start();
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                myView.setTranslationX((Float) animation.getAnimatedValue()*(屏幕款或者高));
            }
        });
```
以上代码其实就是，通过ValueAnimator产生了一个从0到1的不断均匀变化的值。然后我们设置一个监听在监听方法的会调离处理数据实现动画便ok了

2. 抛物线
```
public void paowuxian(View view){
        ValueAnimator valueAnimator = new ValueAnimator();
        valueAnimator.setDuration(1000);
        valueAnimator.setObjectValues(new PointF(0,0));
        valueAnimator.setInterpolator(new LinearInterpolator());

        valueAnimator.setEvaluator(new TypeEvaluator() {
            @Override
            public Object evaluate(float fraction, Object startValue, Object endValue) {
                Log.e("PaowuxianAc", fraction * 3 + "");
                // x方向200px/s ，则y方向0.5 * 10 * t
                PointF point = new PointF();
                point.x = 200 * fraction * 3;
                point.y = 0.5f * 200 * (fraction * 3) * (fraction * 3);
                return point;
            }
        });

        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                PointF pointF = (PointF) animation.getAnimatedValue();
                mBlueBall.setTranslationY(pointF.y);
                mBlueBall.setTranslationX(pointF.x);
            }
        });
        valueAnimator.start();
    }
```
这里有两个方法：
- setEvaluator
  我们需要自定义的数值，那么久需要new出TypeEvaluator，通过重写evaluate()这个方法，可以返回各种各样我们想要的值，
  它的第一个参数fraction就是前面说到的时间因子，它是一个从0-1之间变化的值，还有startValue参数和endValue参数，
  这些参数结合起来通过各种各样的计算方式，就可以产生所有我们想要产生的值。这里不光能产生普通的数据结构，
  通过泛型同样可以定义更加复杂的数据结构。
- setInterpolator
  这个方法是给ValueAnimator设置一个差值器，说简单点就是控制从0-1的变化规律，在说的详细一点就是控制从0-1这条线每个点的斜率
  上面用到了LinearInterpolator()，其实就是一条直线（斜率是一定的）。其他插值器还有：
  - AccelerateInterpolator
  - AnticipateInterpolator
  - DecelerateInterpolator
  - BounceInterpolator
  - OvershootInterpolator
根据不同的需求我们可以采用不同插值器，上面的每个插值器对应怎样的变换，可以稍微google一下。

总结
1. 动画常用的一些属性：
   translationX\translationY  水平和垂直方向偏移

   rotation、rotationX\rotationY  rotation指3D翻转，rotationX\rotationY指水平和竖直方向的一个旋转动画

   scaleX\scaleY  X轴方向，和Y轴方向缩放的一个动画

   X\Y  具体会移动到某一个点

   alpha  透明度动画

2. 常用的方法、类
   ValueAnimator  数值发生器

   ObjectAnimator  是ValueAnimator的一个子类，它封装了ValueAnimator，可以更轻松的使用属性动画

   AnimatorUpdateListener

   AnimatorListenerAdapter  这2个是用来做监听事件的

   PropertyValuesHolder

   AnimatorSet  这2个用来控制集合动画的一个显示效果、顺序

   TypeEvaluators  值计算器

   Interpolators  插值器，用来控制具体数值的变化规律