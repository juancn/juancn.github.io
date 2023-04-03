---
layout: post
title:  "Intersecting a QuadCurve2D (Part II)"
date:   2005-07-20 10:52:33 -0300
categories: post
---
(continued from [Part One](post/2005/07/06/intersecting-quadcurve2d-part-i.html))

So far we have the parametric equation of quadratic bezier curve:

$$Q(t) = P_1 (1 - t)^2 + P_2 (2t(1-t)) + P_3 (t^2)$$

All we have to do now is devise an efficient way to check the intersection with a rectangle. We have three cases of intersection:

 - If one of the endpoints of the curve is contained in the rectangle, the curve intersects the rectangle
 - If the curve intersects one of the line segments that delimit the rectangle, the curve intersects the rectangle
 - If neither of these is true, there is no intersection.

The first case is the easiest to check, we just have to check if the rectangle contains either one of the endpoints ($$P_1$$ and $$P_3$$).
The second one is a bit trickier. First, let me make one observation: the sides of a rectangle are always parallel to one of the coordinate axis (I know this sounds obvious in this context, but bear with me).
What this means is that the top and bottom edges of the rectangle are contained by a line of the form:

$$y = c$$

with `c = rect.getY()` for the top edge and `c = rect.getY() + rect.getHeight()` for the bottom edge.

Analogously, the left and right edges:

$$x = c$$

with `c = rect.getX()` for the left edge and `c = rect.getX()+rect.getWidth()` for the right edge.

Let's use the top edge as an example, to check if the curve cuts that edge, we must do two checks:
That exists a value of $$t$$, such as $$0 <= t <= 1$$ and `c = Qy(t)`
And that `rect.getX() <= Qx(t) <= rect.getX() + rect.getWidth()` is true for that value of $$t$$

Where `Qx(t)` and `Qy(t)` are the parametric equations of the curve for each coordinate:

$$Qx(t) = Px1 (1 - t)^2 + Px2 (2t(1-t)) + Px3 (t^2)$$

$$Qy(t) = Py1 (1 - t)^2 + Py2 (2t(1-t)) + Py3 (t^2)$$

To perform the aforementioned checks, we must solve $$t$$ for:

$$c = Q(t)$$

(since $$Qx(t)$$ and $$Qy(t)$$ are analogous, I won't specify which one I'm referring , since the result is the same).

That yields (after some more mathemagic passes):

$$(P_1 - 2 P_2 + P_3)t^2 + (2 P_2-2 P_1) t + (P_1 - c) = 0$$

Which is a simple quadratic equation. As you know, quadratic equations have (potentially) two roots. So we must check both values (if present) to see if any of those yields the expected result.

To solve it in code, we can use the handy `QuadCurve2D.solveQuadratic()` method.

```java
  public boolean curveIntersects(QuadCurve2D curve, double rx, double ry, double rw, double rh)
  {
      /**
       * A quadratic bezier curve has the following parametric equation:
       *
       * Q(t) = P0(1-t)^2 + P1(2t(1-t)) + P2(t^2)
       *
       * Where 0 <= t <= 1 and P0 is the starting point, P1 is the control point and
       * P2 is the end point.
       *
       * Therefore, the equations for the x and y coordinates are:
       *
       * Qx(t) = Px0(1-t)^2 + Px1(2t(1-t)) + Px2(t^2)
       * Qy(t) = Py0(1-t)^2 + Py1(2t(1-t)) + Py2(t^2)
       *
       *  0 <= t <= 1
       *
       * A bezier curve intersects a rectangle if:
       *
       *  1 - Either one of its endpoints is inside of the rectangle
       *  2 - The curve intersects one of the rectangles sides (top, bottom, left or right)
       *
       * The equation for a horizontal line is:
       *
       *   y = c
       *
       * The line intersects the bezier if:
       *
       *   Qy(t) = c and 0 <= t <= 1
       *
       * We can rewrite this as:
       *
       *   -c + Py0 + (-2Py0 + 2Py1)t + (Py0 - 2Py1 + Py2) t^2 == 0
       *   and 0 <= t <= 1
       *
       * We can use the valid roots of the quadratic, to evaluate Qx(t), and see if the value
       * falls withing the rectangle bounds.
       *
       * The case for vertical lines is analogous to this one.
       * (juancn)
       */
      double y1 = curve.getY1();
      double y2 = curve.getY2();
      double x1 = curve.getX1();
      double x2 = curve.getX2();

      //If the rectangle contains one of the endpoints, it intersects the curve
      if(rectangleContains(x1, y1, rx,ry,rw,rh)  rectangleContains(x2, y2,rx,ry,rw,rh)) {
          return true;
      }

      double eqn[] = new double[3];
      double ctrlY = curve.getCtrlY();
      double ctrlX = curve.getCtrlX();

      return intersectsLine(eqn, y1, ctrlY, y2, ry, x1, ctrlX, x2, rx, rx+rw) //Top
       intersectsLine(eqn, y1, ctrlY, y2, ry + rh, x1, ctrlX, x2,rx, rx+rw) //Bottom
       intersectsLine(eqn, x1, ctrlX, x2, rx, y1, ctrlY, y2, ry, ry+rh) //Left
       intersectsLine(eqn, x1, ctrlX, x2, rx+rw, y1, ctrlY, y2, ry, ry+rh); //Right
  }

  private boolean rectangleContains(double x, double y, double rx, double ry, double rw, double rh)
  {
      return (x >= rx &&
          y >= ry &&
          x < rx + rw &&
          y < ry + rh);
  }

  /**
   * Returns true if a line segment parallel to one of the axis intersects the specified curve.
   * This function works fine if you reverse the axes.
   *
   * @param eqn a double[] of lenght 3 used to hold the quadratic equation coeficients
   * @param p0 starting point of the curve at the desired axis (i.e.: curve.getX1())
   * @param p1 control point of the curve at the desired axis  (i.e.: curve.getCtrlX())
   * @param p2 end point of the curve at the desired axis (i.e.: curve.getX2())
   * @param c where is the line segment (i.e.: in X axis)
   * @param pb0 starting point of the curve at the other axis (i.e.: curve.getY1())
   * @param pb1 control point of the curve at the other axis  (i.e.: curve.getCtrlY())
   * @param pb2 end point of the curve at the other axis (i.e.: curve.getY2())
   * @param from starting point of the line segment (i.e.: in Y axis)
   * @param to end point of the line segment (i.e.: in Y axis)
   * @return
   */
  private static boolean intersectsLine(double[] eqn, double p0, double p1, double p2, double c,
                                 double pb0, double pb1, double pb2,
                                 double from, double to)
  {
      /**
       * First we check if a line parallel to the axis we are evaluating intersects
       * the curve (the line is at c).
       *
       * Then we check if any of the intersection points is between 'from' and 'to' in the other
       * axis (wether it belongs to the rectangle)
       */

      //Fill the coefficients of the equation
      eqn[2] = p0 - 2*p1 + p2;
      eqn[1] = 2*p1-2*p0;
      eqn[0] = p0 - c;

      int nRoots = QuadCurve2D.solveQuadratic(eqn);
      boolean result;
      switch(nRoots) {
      case 1:
          result = (eqn[0] >= 0) && (eqn[0] <= 1);
          if(result) {
              double intersection = evalQuadraticCurve(pb0,pb1,pb2,eqn[0]);
              result = (intersection >= from) && (intersection <= to);
          }
          break;
      case 2:
          result = (eqn[0] >= 0) && (eqn[0] <= 1);
          if(result) {
              double intersection = evalQuadraticCurve(pb0,pb1,pb2,eqn[0]);
              result = (intersection >= from) && (intersection <= to);
          }

          //If the first root is not a valid intersection, try the other one
          if(!result) {
              result = (eqn[1] >= 0) && (eqn[1] <= 1);
              if(result) {
                  double intersection = evalQuadraticCurve(pb0,pb1,pb2,eqn[1]);
                  result = (intersection >= from) && (intersection <= to);
              }
          }

          break;
      default:
          result = false;
      }
      return result;
  }

  public static double evalQuadraticCurve(double c1, double ctrl, double c2, double t)
  {
      double u = 1 - t;
      double res = c1 * u * u + 2 * ctrl * t * u + c2 * t * t;

      return res;
  }
```