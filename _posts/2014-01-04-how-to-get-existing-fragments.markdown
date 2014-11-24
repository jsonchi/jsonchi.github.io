---
layout: post
title:  "How to get existing fragments when using FragmentPagerAdapter"
categories: jekyll update
---

* **Few things should noted**:
    * `getItem(int position)` in the `FragmentPagerAdapter` is rather misleading name of what this
    method actually does. It creates new fragments, not returning existing ones. In so meaning, the
    method should be renamed to something like `createItem(int position)` in the Android SDK. So
    this method does not help us getting fragments.
    * Based on explanation in the post [reference to old fragments][refere to old Fragment] you 
    should leave the creation of the fragments to the 
    `FragmentPagerAdapter` and in so meaning you have no reference to the Fragments or their tags. 
    If you have fragment tag though, you can easily retrieve reference to it from the `FragmentManage`
    by calling `findFragmentByTag()`. We need a way to find out tag of a fragment at given page 
    position.  

* **Solution**:

    * Add following helper method in your class to retrieve fragment tag and send it to the
    `findFragmentByTag()` method.
{% highlight java %}
private static String makeFragmentName(int viewId, int position) {
     return "android:switcher:" + viewId + ":" + position;
}
{% endhighlight %}  

NOTE! This is identical method that FragmentPagerAdapter use when creating new fragments. See this link [FragmentPagerAdapter.java][FragmentPagerAdapter]    
  
Refere to: [http://stackoverflow.com/questions/14035090/how-to-get-existing-fragments-when-using-fragmentpageradapter](http://stackoverflow.com/questions/14035090/how-to-get-existing-fragments-when-using-fragmentpageradapter)

[refere to old Fragment]: http://stackoverflow.com/questions/9727173/support-fragmentpageradapter-holds-reference-to-old-fragments
[FragmentPagerAdapter]: http://code.google.com/p/openintents/source/browse/trunk/compatibility/AndroidSupportV2/src/android/support/v2/app/FragmentPagerAdapter.java#104
