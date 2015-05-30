---
layout: post
title: [译]顺畅的签字（上）
date: 2015-05-30 16:22:05
categories: android 签字 翻译
excerpt: android
---

这是一篇英语翻译，这是文章中的上篇，原文地址是：![https://corner.squareup.com/2010/07/smooth-signatures.html]({{ https://corner.squareup.com/2010/07/smooth-signatures.html }});

在信用卡支付过程中获取一个签名提高安全性并且降低处理到费用。当你使用Square(注：一个信用卡处理和商业解决方案商)，你用手签名在屏幕上而不是用笔签一张收据：
  ![图片]({{ site.url }}/after.png);

这个签名会显示在电子邮件账单上，帮助Square检测和防止骗局。

当实现这android客户端时，我们开始用能够完成工作最简单的方式：一个自定义的View获取触摸事件然后把这些点加到path中：

 {% highlight java %}

 public class SignatureView extends View {
 private Paint paint = new Paint();
 private Path path = new Path();

 public SignatureView(Context context, AttributeSet attrs) {
   super(context, attrs);

   paint.setAntiAlias(true);
   paint.setColor(Color.BLACK);
   paint.setStyle(Paint.Style.STROKE);
   paint.setStrokeJoin(Paint.Join.ROUND);
   paint.setStrokeWidth(5f);
 }

 @Override
 protected void onDraw(Canvas canvas) {
   canvas.drawPath(path, paint);
 }

@Override
 public boolean onTouchEvent(MotionEvent event) {
   float eventX = event.getX();
   float eventY = event.getY();

   switch (event.getAction()) {
     case MotionEvent.ACTION_DOWN:
       path.moveTo(eventX, eventY);
       return true;
     case MotionEvent.ACTION_MOVE:
     case MotionEvent.ACTION_UP:
       path.lineTo(eventX, eventY);
       break;
     default:
       return false;
   }

   // Schedules a repaint.
   invalidate();
   return true;
 }
}
 {% endhighlight %}

 然而简单的实现，这个方法留了很多需要实现的，这个签名是成坨的而且用户体验不好。
 ![图片]({{ site.url }}/before.png);

我们用了两个不同的方式解决这些问题：
  <h3>消失的事件</h3>
  我们自定义的view跟不上我的手指，开始，我们担心：
    <ul>
      <li>android采样触摸屏的频率太低，或者</li>
      <li>绘图阻塞了触摸屏的采样</li>
    </ul>
幸运的事，这两个担心都不是真的。我们很快发现了android批触摸事件。每一个传递到onTouchEvent()的MotionEvent包含上次onThuchEvent()被调用时获取到的几个坐标。为了绘制更加顺滑地绘制签名，我们需要包含这些点。

下面的MotionEvent的方法暴露了这些点的数组：
  <ul>
    <li>getHistorySize()</li>
    <li>getHistoricalX()</li>
    <li>getHistoricalY()</li>
  </ul>

让我们更新一下SignatureView来合并这些中间点：

 {% highlight java %}
 public class SignatureView extends View {
  public boolean onTouchEvent(MotionEvent event) {
    ...
    switch (event.getAction()) {
      case MotionEvent.ACTION_MOVE:
      case MotionEvent.ACTION_UP:

        // When the hardware tracks events faster than they are delivered,
        // the event will contain a history of those skipped points.
        int historySize = event.getHistorySize();
        for (int i = 0; i < historySize; i++) {
          float historicalX = event.getHistoricalX(i);
          float historicalY = event.getHistoricalY(i);
          path.lineTo(historicalX, historicalY);
        }

        // After replaying history, connect the line to the touch point.
        path.lineTo(eventX, eventY);
        break;
    ...
  }
}
 {% endhighlight %}

 这个简单的改变产生了签名样子很大提高，但是反应以上很慢。

 <h3>小心翼翼地更新</h3>

 每次调用onTouchEvent(),SignatureView绘制触摸点之间的线片段然后更新整个view。SignatureView强制要求Android去重绘整个视图尽管只有一个部分的像素改变了。

 重绘整个试图很慢而且没有必要，使用View.invalidate(Rect)去选择性地更新最近被加入的线片段的矩形区域大大的提高的绘制效率。

 算法差不多就像这样：
  <ul>
    <li>创建一个矩形代表脏（修改过的）区域的矩形</li>
    <li>在ACTION_DOWN事件中设置四个角的x，y坐标</li>
    <li>在ACTION_MOVE和ACTION_UP中扩展矩形区域使它包含新的点（不要忘记以前的坐标）</li>
    <li>只把脏矩形传递给invalidate()，android不会重绘其他区域</li>
  </ul>
改变之后响应速度立刻显著提高了。

<h3>翅</h3>

利用这些中间事件让签名看起来更加顺滑和更加真实。通过避免不必要的工作提高绘制效率提高了重绘率和让签名感觉响应更快。
下面就是结果：

![图片]({{ site.url }}/after_final.png);

然后，这是最终的代码，去掉了一些辅助特性，如抖动检测：
{% highlight java %}
public class SignatureView extends View {

private static final float STROKE_WIDTH = 5f;

private static final float HALF_STROKE_WIDTH = STROKE_WIDTH / 2;

private Paint paint = new Paint();
private Path path = new Path();


private float lastTouchX;
private float lastTouchY;
private final RectF dirtyRect = new RectF();

public SignatureView(Context context, AttributeSet attrs) {
  super(context, attrs);

  paint.setAntiAlias(true);
  paint.setColor(Color.BLACK);
  paint.setStyle(Paint.Style.STROKE);
  paint.setStrokeJoin(Paint.Join.ROUND);
  paint.setStrokeWidth(STROKE_WIDTH);
}


public void clear() {
  path.reset();

  // Repaints the entire view.
  invalidate();
}

@Override
protected void onDraw(Canvas canvas) {
  canvas.drawPath(path, paint);
}

@Override
public boolean onTouchEvent(MotionEvent event) {
  float eventX = event.getX();
  float eventY = event.getY();

  switch (event.getAction()) {
    case MotionEvent.ACTION_DOWN:
      path.moveTo(eventX, eventY);
      lastTouchX = eventX;
      lastTouchY = eventY;
      // There is no end point yet, so don't waste cycles invalidating.
      return true;

    case MotionEvent.ACTION_MOVE:
    case MotionEvent.ACTION_UP:
      // Start tracking the dirty region.
      resetDirtyRect(eventX, eventY);

      // When the hardware tracks events faster than they are delivered, the
      // event will contain a history of those skipped points.
      int historySize = event.getHistorySize();
      for (int i = 0; i < historySize; i++) {
        float historicalX = event.getHistoricalX(i);
        float historicalY = event.getHistoricalY(i);
        expandDirtyRect(historicalX, historicalY);
        path.lineTo(historicalX, historicalY);
      }

      // After replaying history, connect the line to the touch point.
      path.lineTo(eventX, eventY);
      break;

    default:
      debug("Ignored touch event: " + event.toString());
      return false;
  }

  // Include half the stroke width to avoid clipping.
  invalidate(
      (int) (dirtyRect.left - HALF_STROKE_WIDTH),
      (int) (dirtyRect.top - HALF_STROKE_WIDTH),
      (int) (dirtyRect.right + HALF_STROKE_WIDTH),
      (int) (dirtyRect.bottom + HALF_STROKE_WIDTH));

  lastTouchX = eventX;
  lastTouchY = eventY;

  return true;
}


private void expandDirtyRect(float historicalX, float historicalY) {
  if (historicalX < dirtyRect.left) {
    dirtyRect.left = historicalX;
  } else if (historicalX > dirtyRect.right) {
    dirtyRect.right = historicalX;
  }
  if (historicalY < dirtyRect.top) {
    dirtyRect.top = historicalY;
  } else if (historicalY > dirtyRect.bottom) {
    dirtyRect.bottom = historicalY;
  }
}


private void resetDirtyRect(float eventX, float eventY) {

  // The lastTouchX and lastTouchY were set when the ACTION_DOWN
  // motion event occurred.
  dirtyRect.left = Math.min(lastTouchX, eventX);
  dirtyRect.right = Math.max(lastTouchX, eventX);
  dirtyRect.top = Math.min(lastTouchY, eventY);
  dirtyRect.bottom = Math.max(lastTouchY, eventY);
}
}

{% endhighlight %}
