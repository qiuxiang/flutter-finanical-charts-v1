# 如何用 flutter 实现体验流畅的 k 线图组件

## 实现效果

https://user-images.githubusercontent.com/1709072/144531944-5b7c9b19-5fd0-4eb4-ab12-defd24c9da45.mp4

- [Web 版在线演示](https://qiuxiang.github.io/flutter-finanical-charts-v1/)
- [下载 app-profile.apk](https://github.com/qiuxiang/flutter-finanical-charts-v1/releases/download/v1/app-profile.apk)

## 体验流畅的关键

- 性能足够好
- 支持无级滑动而不是以单个 k 线宽度为最小单位的滑动
- 支持惯性滑动
- 在范围发生变化时提供动画过渡

能支持前三点就已经足以让人感到体验流畅了，火币、webull 都做到了，但如果能做到第四点就能实现更流畅的体验。
在架构合理的情况下都不难实现。

## 组件架构

首先，毋庸置疑，我们需要使用 canvas 来绘制图表，但这意味着我们要用一个 canvas 绘制出整个 k 线图组件吗？
从布局灵活性考虑显然不合适，更合理的做法应该是将各种图表封装成 widget，再用 flutter widgets 将其组合在一起。
因为是 widget，动画的支持也将变得更容易。

### 提供灵活可定制的接口

以火币为例，完整的行情图由 k 线图 + volume 图 + 指标副图组成，其中指标副图是可以替换的。但这也只是其中一种展现形式而已，
只做一种场景是很简单的，只需要提供一个参数设置指标副图的类型即可。如果我们想要同时显示两个甚至多个的指标副图呢？
各个图表的高度是否可单独设置？整个组件的高度是否可以设置或自适应？

再结合上面的组件架构设计，每种图表都单独封装成 widget 后，我们可以给每种图表提供一个 ChartOptions，其中可以指定高度 height，
或 flex，已满足各种布局需求。

```dart
abstract class ChartOptions {
  final int flex;
  final double height;

  const ChartOptions(this.flex, this.height);
}

class CandlestickChartOptions extends ChartOptions {
  final List<MaOptions> maList;

  final BollOptions boll;

  const CandlestickChartOptions({flex = 1, height, this.maList, this.boll})
      : super(flex, height);
}

class VolumeChartOptions extends ChartOptions {
  const VolumeChartOptions({flex = 1, height}) : super(flex, height);
}

...
```

再给组件提供 charts 参数用于指定渲染哪些图表，以及使用何种参数何种布局。

```dart
final List<ChartOptions> charts;
```

渲染的时候：

```dart
final charts = widget.charts.map((options) {
  Widget child;
  if (options is CandlestickChartOptions) {
    child = CandlestickChart(options);
  }
  if (options is VolumeChartOptions) {
    child = VolumeChart(options);
  }
  if (options is KDJChartOptions) {
    child = KDJChart(options);
  }
  if (options is MACDChartOptions) {
    child = MACDChart(options);
  }
  if (options is RSIChartOptions) {
    child = RSIChart(options);
  }
  if (options.height != null) {
    return SizedBox(height: options.height, child: child);
  }
  return Expanded(flex: options.flex, child: child);
})

Column(children: charts.toList());
```

## 状态管理

由于我们需要将各个图表封装成不同的 widget，这就需要引入状态管理以实现跨组件状态和渲染的同步，不考虑引入第三方库的情况下
SteamBuilder 是可以做到的，只是在状态多的情况下用起来稍有麻烦，我在开发这个项目的时候选择了 provider。但不管用什么，
合理拆分数据，避免不必要的 build，以及 build 性能的最优化都是非常必要的。在结合手势识别的情况下，build 的调用频率与渲染帧率一致，
build 过程中的任何一点性能浪费，都直接导致帧率的下降。

这个项目里就大量使用了 provider 的 Selector，以保证只有在必要的时候才进行 build。

## 手势识别及惯性滑动的实现

手势识别用到 GestureDetector，其中缩放及滑动的识别使用 onScaleUpdate，flutter 的手势识别接口做得很好了，可以直接拿到 scale。
只要计算缩放后 k 线的宽度，canvas 使用缩放后的 k 线宽度重新绘制整个图表即可实现缩放的效果。

onScaleEnd 的时候，ScaleEndDetails 的 velocity.pixelsPerSecond.dx 可作为滑动结束时的速度，
简单根据减速运动公式可以算出惯性滑动的距离及时间，然后让 AnimationController 运行动画。

```dart
void onScaleEnd(ScaleEndDetails event) {
  model.scaleEnd();
  final ratio = MediaQuery.of(context).devicePixelRatio;
  final v = event.velocity.pixelsPerSecond.dx / (ratio == 1 ? 4 : ratio);
  final a = 1;
  final t = (v ~/ a).abs();
  final s = (v * t - a * t * t / 2) / 1000;
  animateTo(model.offset + s, t);
}

void animateTo(double offset, int duration) {
  final tween = Tween(begin: model.offset, end: offset);
  animation = tween.animate(curved);
  animation.removeListener(listener);
  animation.addListener(listener);
  controller.reset();
  controller.duration = Duration(milliseconds: duration);
  controller.forward();
}
```
