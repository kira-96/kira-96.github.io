---
title: "GDI+绘制旋转的椭圆"
date: 2021-01-21T16:23:42+08:00
draft: true
isCJKLanguage: true
---

GDI+绘制椭圆时只支持输入一个矩形范围，无法绘制倾斜的椭圆。

绘制椭圆的 API：
``` csharp
// 摘要: 绘制边界 System.Drawing.RectangleF 定义的椭圆。
// 参数:
//   pen: System.Drawing.Pen，它确定曲线的颜色、宽度和样式。
//   rect: System.Drawing.RectangleF 结构，它定义椭圆的边界。
// 异常:
//   T:System.ArgumentNullException: pen 为 null。
public void DrawEllipse(Pen pen, RectangleF rect);
//
// 摘要: 绘制一个由边框（该边框由一对坐标、高度和宽度指定）定义的椭圆。
// 参数:
//   pen: System.Drawing.Pen，它确定曲线的颜色、宽度和样式。
//   x: 定义椭圆的边框的左上角的 X 坐标。
//   y: 定义椭圆的边框的左上角的 Y 坐标。
//   width: 定义椭圆的边框的宽度。
//   height: 定义椭圆的边框的高度。
// 异常:
//   T:System.ArgumentNullException: pen 为 null。
public void DrawEllipse(Pen pen, float x, float y, float width, float height);
//
// 摘要: 绘制边界 System.Drawing.Rectangle 结构指定的椭圆。
// 参数:
//   pen: System.Drawing.Pen，它确定曲线的颜色、宽度和样式。
//   rect: System.Drawing.Rectangle 结构，它定义椭圆的边界。
// 异常:
//   T:System.ArgumentNullException: pen 为 null。
public void DrawEllipse(Pen pen, Rectangle rect);
//
// 摘要: 绘制一个由边框定义的椭圆，该边框由矩形的左上角坐标、高度和宽度指定。
// 参数:
//   pen: System.Drawing.Pen，它确定曲线的颜色、宽度和样式。
//   x: 定义椭圆的边框的左上角的 X 坐标。
//   y: 定义椭圆的边框的左上角的 Y 坐标。
//   width: 定义椭圆的边框的宽度。
//   height: 定义椭圆的边框的高度。
// 异常:
//   T:System.ArgumentNullException: pen 为 null。
public void DrawEllipse(Pen pen, int x, int y, int width, int height);
```

可以看到，绘制椭圆的API，都只支持输入一个矩形区域，长和宽都只能是水平或者竖直的，因此，绘制出椭圆的长轴和短轴也是水平或竖直的，而无法绘制一个旋转过的椭圆，如下图：

![rotated-ellipse.png](https://i.loli.net/2021/01/22/A7P8l6HpJ3rnbMv.png)

给定椭圆的4个顶点，绘制出椭圆。虽然不能直接调用API绘制椭圆，但是可以通过绘制4条连续的贝塞尔曲线来闭合成一个椭圆。

``` csharp
//
// 摘要: 用 System.Drawing.PointF 结构数组绘制一系列贝塞尔样条。
// 参数:
//   pen: System.Drawing.Pen，它确定曲线的颜色、宽度和样式。
//   points: System.Drawing.PointF 结构的数组，这些结构表示确定曲线的点。
//   此数组中的点数应为 3 的倍数加 1，如 4、7 或 10。
//
// 异常:
//   T:System.ArgumentNullException: pen 为 null。- 或 -points 为 null。
public void DrawBeziers(Pen pen, PointF[] points);
```

使用这种方法实际上只需要知道椭圆的中心点，两个轴的长度以及旋转角度即可，而这些都可以通过4个顶点计算得到。

根据以上的信息，依次计算出连续贝塞尔曲线的13个控制点，然后调用`DrawBeziers`绘制椭圆。

各个点的位置如图：

![bezier-draw-ellipse.png](https://i.loli.net/2021/01/22/XTIPG5rk2tUZo9z.png)

``` csharp
/**
 * C#
 */
// MAGICAL CONSTANT to map ellipse to beziers
// 2/3*(sqrt(2)-1)
const double Ellipse2Beziers = 0.2761423749154;

// GDI Bitmap
using var bitmap = new System.Drawing.Bitmap(width, height);
using var graphics = System.Drawing.Graphics.FromImage(bitmap);

// 椭圆的4个顶点
PointF[] ellipse = new PointF[4] { point1, point2, point3, point4 };

// 两个轴的长度
double r1 = ellipse[2].DistanceTo(ellipse[3]);
double r2 = ellipse[0].DistanceTo(ellipse[1]);
// 旋转角度
double angle = -SysMath.Atan2(ellipse[2].Y - ellipse[3].Y, ellipse[2].X - ellipse[3].X);
double sin = SysMath.Sin(angle);
double cos = SysMath.Cos(angle);
// 贝塞尔曲线控制点相对于中心点的偏移长度
SizeF offset = new SizeF((float)(r1 * Ellipse2Beziers), (float)(r2 * Ellipse2Beziers));
// 椭圆中心点
PointF center = new PointF((ellipse[0].X + ellipse[1].X) / 2f, (ellipse[0].Y + ellipse[1].Y) / 2f);
// 贝塞尔曲线的控制点
PointF[] beziers = new PointF[13]
{
    new PointF((float)(center.X - r1 / 2.0), center.Y),
    new PointF((float)(center.X - r1 / 2.0), center.Y - offset.Height),
    new PointF(center.X - offset.Width, (float)(center.Y - r2 / 2.0)),
    new PointF(center.X, (float)(center.Y - r2 / 2.0)),
    new PointF(center.X + offset.Width, (float)(center.Y - r2 / 2.0)),
    new PointF((float)(center.X + r1 / 2.0), center.Y - offset.Height),
    new PointF((float)(center.X + r1 / 2.0), center.Y),
    new PointF((float)(center.X + r1 / 2.0), center.Y + offset.Height),
    new PointF(center.X + offset.Width, (float)(center.Y + r2 / 2.0)),
    new PointF(center.X, (float)(center.Y + r2 / 2.0)),
    new PointF(center.X - offset.Width, (float)(center.Y + r2 / 2.0)),
    new PointF((float)(center.X - r1 / 2.0), center.Y + offset.Height),
    new PointF((float)(center.X - r1 / 2.0), center.Y)
};
// 旋转变换
double offsetX = center.X - center.X * cos - center.Y * sin;
double offsetY = center.Y + center.X * sin - center.Y * cos;
for (int j = 0; j < beziers.Length; j++)
{
    beziers[j] = new PointF(
        (float)(beziers[j].X * cos + beziers[j].Y * sin + offsetX),
        (float)(beziers[j].Y * cos - beziers[j].X * sin + offsetY));
}
// 绘制曲线
graphics.DrawBeziers(new Pen(Brushes.White, 1f)/* Pen */, beziers);
```

通过上面代码就可以完美的绘制出一个旋转任意角度的椭圆了。

**参考**

[MFC上如何绘制一个可以旋转的椭圆](https://blog.csdn.net/liuyunhuanying/article/details/11645965)

[Drawing Rotated and Skewed Ellipses](https://www.codeguru.com/cpp/g-m/gdi/article.php/c131/Drawing-Rotated-and-Skewed-Ellipses.htm)
