---
layout: post
title: "Yahoo News Digest Animations"
description: "On-boarding animations of Yahoo News Digest App"
category: android
subheading: On-boarding animations of Yahoo News Digest App
permalink: android/yahoo-animation
---

This week i tried my hands on some cool animations that comes bundled with the [Yahoo News Digest](https://play.google.com/store/apps/details?id=com.yahoo.mobile.client.android.atom) Android app. I have not tried the  the app's content but it has a very impressive UI and a slightly different news-reading experience unlike giving us 10-15  options to choose the category and read the news which most of the apps in the market provide. This experience is enriched by plethora of smooth transitions spread throughout the app.
<br><br>
This project's demo and code is available [here](https://github.com/rahulrj/YahooNewsOnboarding).
<br><br>
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;![](/images/yahoo-news-demo.gif)

## How are the animations designed?

### First screen animation
I did the  animation of the first page by using `ViewPager.PageTransformer`. The structure of this interface is
{% highlight java %}
public interface PageTransformer {
    public void transformPage(View page, float position);
}
{% endhighlight %}
`transformPage(page,pos)` is called for the current page, and the pages adjacent to it at every point of the transition of `ViewPager`. The spheres in the first page all appear to be moving with different speeds on scrolling the `ViewPager`. This can be easily achieved if we assign a different X-value to each of the sphere's position at every point of the scroll.
<br><br>
So lets say the first page is shown on the screen and we drag the `ViewPager` towards the left. So first i restrict that this `transformPage` function should be called by only one page for this animation and not all pages. Because if all pages will call it, the spheres will have X-values. How i do this? Using the below [code](https://github.com/rahulrj/YahooNewsOnboarding/blob/master/app/src/main/java/onboarding/yahoo/com/yahoonewsonboarding/MainActivity.java#L276)
{% highlight java %}
public void transformPage(View page, float position) {
  int pageWidth = page.getWidth();
  if (position < -1) {/*do nothing*/}

  else if (position <= 1) {
      if (!mSecondPageSelected && page.findViewById(R.id.center_box_second) != null) {
              moveTheSpheres(position, pageWidth);
         }
  }
  else {/*do nothing*/}
}
{% endhighlight %}
So this function will be called for the second page only when its `position` changes from 1 to 0 and 0 to 1. Now at each point in the transition,i change the X co-ordinates of the spheres using `setTranslationX(val)`.[(code)](https://github.com/rahulrj/YahooNewsOnboarding/blob/master/app/src/main/java/onboarding/yahoo/com/yahoonewsonboarding/MainActivity.java#L300)
{% highlight java %}
float camcordPos = (float) ((1 - position) * 0.15 * pageWidth);
mCamcordImage.setTranslationX(camcordPos);
{% endhighlight %}
So it appears to be moving as we scroll. If we want a relatively higher speed than the above example, just change the value 0.15 to a higher value.

### Second page animation
Second page is all about drawing,erasing and redrawing on canvas. There are two views in it i.e `SunMoonView`, which is the trajectory for sun/moon and the other is `BookView`, which is the book and its animation.They can be made into a single view of course but just for simplicity, i made two different views.
<br><br>
**How does SunMoonView work ?**
<br>
1. [Create](https://github.com/rahulrj/YahooNewsOnboarding/blob/master/app/src/main/java/onboarding/yahoo/com/yahoonewsonboarding/SunMoonView.java#L91) a circular path using `Path.addArc()` and alternate its direction from clockwise to counter-clockwise when the drag direction changes. Initially,place both the bitmaps on the path by using `canvas.drawBitmap()`. The co-ordinates of the point where we want to place is obtained using `getPosTan()` of `PathMeasure` object mentioned in point 2.
<br><br>
2. For [translating](https://github.com/rahulrj/YahooNewsOnboarding/blob/master/app/src/main/java/onboarding/yahoo/com/yahoonewsonboarding/SunMoonView.java#L112) the bitmap on the path, i used a `PathMeasure` object. In the function `PathMeasure.getPosTan(distance, pos[], tan[])`, we supply the distance( the point on the path where we want to place the object) and it returns the co-ordinates of that position with respect to the screen in `pos` array. Now we can draw the bitmap at that co-ordinates. So when the page is scrolled,i just draw the bitmap at a point on the path. This point is determined according to the `position` argument of `PageTransformer`.
<br><br>
In `BookView`, the book is drawn on the canvas using simple `canvas.drawRect()` and for giving it a fade effect to its lines, i change the alpha value of the `Paint` object over time, which gives it a fade-in effect.

### Third page animation  
In third page, there are three animations. The first one is a simple rotate animation which i did using the same procedure as mentioned for the second screen. Second one is a translate animation for the spheres done using `PageTransformer`.
<br><br>
The third animation occurs when the "Lets Go" button is pressed. That time, all the bitmaps go out for some distance and then they move inwards till they converge at the center of the circle. For this, again i have created 6 paths, one for each bitmap, which originates from the center of the circle and goes a little ahead of the current position of the bitmaps.
<br>
When the button is pressed,the latest position of each of the sphere is obtained from [here](https://github.com/rahulrj/YahooNewsOnboarding/blob/master/app/src/main/java/onboarding/yahoo/com/yahoonewsonboarding/ThirdScreenView.java#L163)
{% highlight java %}
float pos[] = drawCircle(canvas, mSphereOriginalPosArr[0], mSphereStepCountArr[0], mBitmap1);
{% endhighlight %}
This function returns the last drawn position for each bitmap. Now that 'little ahead' distance that i mentioned above is calculated using the simple projection formula.
{% highlight java %}
double len = Math.hypot((pos[0] - 360), (pos[1] - 360));
double dx = (pos[0] - 360) / len;
double dy = (pos[1] - 360) / len;
x = (float) (pos[0] + mTempDistanceX * dx);
y = (float) (pos[1] + mTempDistanceY * dy);
{% endhighlight %}
where (360,360) is center of the circle and (pos[0],pos[1]) are the co-ordinates of the latest point of the sphere. Now for converging the spheres, all i do is translate the spheres on the drawn path. Since this is a closed path, it looks like the sphere are translating first outwards and then inwards.
