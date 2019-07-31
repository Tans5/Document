# 与StatusBar的爱恨情仇
---

## 1.  实现简单的透明状态栏

实现透明的状态栏在项目中很常见，可以通过同时设置windowTranslucentStatus和fitsSystemWindows这两个属性来实现这个效果。

- 通过Theme中的属性设置

  ```XML
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>

        <item name="android:windowTranslucentStatus">true</item>
        <item name="android:fitsSystemWindows">true</item>

    </style>
  ```

- 通过代码中设置  

  ```Kotlin
	window.decorView.systemUiVisibility = WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS
	root_layout.fitsSystemWindows = isRootViewFitSystem   
  ```  
  _Tips: fitsSystemWindows是View的一个属性，也可以在XML中设置。_  

- windowTranslucentStatus  
  如果设置了这个属性为true，系统就不会绘制StatusBar，当没有绘制StatusBar的时候我们的View就会和系统的状态栏重合。 

- fitSystemWindows  
  View的默认实现：当windowTranslucentStatus和fitSystemWindows这个属性为true的时候，view会添加一个paddingTop值，而这个值刚好等于StatusBar的高度，但是会覆盖掉原来的padding值，这点需要注意。  如果windowTranslucentStatus属性为false，那fitSystemWindows无效。

## 2. View和ViewGroup中fitSystemWindows的实现  

处理fitSystemWindows是通过parent view调用child view的 `` dispatchApplyWindowInsets(WindowInsets insets) `` 方法来下发事件，然后子View再对该事件进行处理， Android中运用了很多这样的责任链模式来处理事件，例如touchEvent。我们在编程的时候也是可以借鉴借鉴。我们也从这个方法作为突破点进行分析。 

_源码版本：Sources for Android 28_

- View  

  ```Java
	// Line: 9795
	public WindowInsets dispatchApplyWindowInsets(WindowInsets insets) {
        try {
            mPrivateFlags3 |= PFLAG3_APPLYING_INSETS;
            if (mListenerInfo != null && mListenerInfo.mOnApplyWindowInsetsListener != null) {  // 当设置设置Listener后直接调用Listener的方法
                return mListenerInfo.mOnApplyWindowInsetsListener.onApplyWindowInsets(this, insets);
            } else {
                return onApplyWindowInsets(insets);// 没有设置Listener会调用onApplyWindowInsets方法 
            }
        } finally {
            mPrivateFlags3 &= ~PFLAG3_APPLYING_INSETS;
        }
    }
  ```

  onApplyWindowInsets方法分析： 

  ```Java
	// Line: 9742
	public WindowInsets onApplyWindowInsets(WindowInsets insets) {
        if ((mPrivateFlags3 & PFLAG3_FITTING_SYSTEM_WINDOWS) == 0) {
            // We weren't called from within a direct call to fitSystemWindows,
            // call into it as a fallback in case we're in a class that overrides it
            // and has logic to perform.
            if (fitSystemWindows(insets.getSystemWindowInsets())) {
                return insets.consumeSystemWindowInsets(); //消费insets
            }
        } else {
            // We were called from within a direct call to fitSystemWindows.
            if (fitSystemWindowsInt(insets.getSystemWindowInsets())) {
                return insets.consumeSystemWindowInsets(); //消费insets
            }
        }
        return insets;
    }
  ```
  onApplyWindows方法中如果有设置fitWindowsFlag都会调用fitSystemWindowsInt这个方法，如果这个方法返回true还会消费掉insets。  
<br />
  fitSystemWindowsInt方法分析：

  ```Java
	// Line: 9698
	private boolean fitSystemWindowsInt(Rect insets) {
        if ((mViewFlags & FITS_SYSTEM_WINDOWS) == FITS_SYSTEM_WINDOWS) { //判断一下Flag，然后设置padding值，同时返回是否消耗insets。
            mUserPaddingStart = UNDEFINED_PADDING;
            mUserPaddingEnd = UNDEFINED_PADDING;
            Rect localInsets = sThreadLocal.get();
            if (localInsets == null) {
                localInsets = new Rect();
                sThreadLocal.set(localInsets);
            }
            boolean res = computeFitSystemWindows(insets, localInsets);
            mUserPaddingLeftInitial = localInsets.left;
            mUserPaddingRightInitial = localInsets.right;
            internalSetPadding(localInsets.left, localInsets.top,
                    localInsets.right, localInsets.bottom); // 设置padding值
            return res;
        }
        return false;
    }
  ```

  总结：View的dispatchApplyWindowInsets方法被调用后会先判断有没有相关的Listener，如果有就交给Listener处理，反之调用自己的onApplyWindowInsets方法来处理。（这里和onTouchEvent事件的下发类似，有兴趣的同学可以去分析下。） 在onApplyWindowInsets方法中会判断是否有fitwindowflag。如果有该flag就会调用fitSystemWindowsInt方法，同时消费掉该insets。在fitSystemWindowsInt中会根据insets设置padding值。整个过程还是比较简单。  

- ViewGroup  

  在ViewGroup中重写了View中的``dispatchApplyWindowInsets``方法，我们直接来看代码。

  ```java
	// Line: 7050
	@Override
    public WindowInsets dispatchApplyWindowInsets(WindowInsets insets) {
        insets = super.dispatchApplyWindowInsets(insets); // 首先调用View的dispatchApplyWindowInsets方法
        if (!insets.isConsumed()) { // 如果自己没有消耗就依次下发给子View
            final int count = getChildCount();
            for (int i = 0; i < count; i++) {
                insets = getChildAt(i).dispatchApplyWindowInsets(insets);
                if (insets.isConsumed()) {
                    break; // 当有子View消耗时，结束循环
                }
            }
        }
        return insets;
    }
  ```
  ViewGroup中首先调用自己的dispatchApplyWindowInsets方法，如果自己消费了该insets就不会下发给子view，反之下发给子view。这里和View的触摸事件不太一样，触摸事件是当没有子view处理的时候父view才会自己处理。  
  
  
# 3. 刁民CoordinatorLayout和他的一些熊孩子View对fitSystemWindows的特殊处理  
本以为分析完View和ViewGroup就对fitSystemWindows这个属性就完了。然而被CoordinatorLayout狠狠的扇了一巴掌。  

_源码版本：com.google.android.material:material:1.1.0-alpha06_

- CoordinatorLayout  
  
   在CoordinatorLayout的构造函数中调用了``setupForInsets()``方法，我们先从这个方法开始分析。  
   
   ```java
	// Line: 3266
	private void setupForInsets() { // 放弃对Android 5.0 以下机型的支持
        if (Build.VERSION.SDK_INT < 21) {
            return;
        }

        if (ViewCompat.getFitsSystemWindows(this)) { // 如果设置了fitWindow的Flag
            if (mApplyWindowInsetsListener == null) {
                mApplyWindowInsetsListener =
                        new androidx.core.view.OnApplyWindowInsetsListener() {
                            @Override
                            public WindowInsetsCompat onApplyWindowInsets(View v,
                                    WindowInsetsCompat insets) {
                                return setWindowInsets(insets); // 当有insets事件来的时候调用setWindowInsets方法
                            }
                        };
            }
            // First apply the insets listener
            ViewCompat.setOnApplyWindowInsetsListener(this, mApplyWindowInsetsListener); // 手动添加insets的Listener

            // Now set the sys ui flags to enable us to lay out in the window insets
            setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                    | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN); // 设置Layout_FULLSCREEN的flag
        } else {
            ViewCompat.setOnApplyWindowInsetsListener(this, null);
        }
    }
   
   ```
   setupForInsets方法中首先判断是否有flag，如果有先添加一个insets的Listener，然后设置SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN UI Flag。在View的分析我我们讲到，但有Listener的时候就不会再调用onApplyWindowInsets方法了，也就修改了默认View对fitSystemWindow这个Flag的处理，忘了的同学可以看前面View的分析。在这个Listener中调用了setWindowInsets方法。  
   
   setWindowInsets方法分析：  
   
   ```java 
	// Line: 366
	final WindowInsetsCompat setWindowInsets(WindowInsetsCompat insets) {
        if (!ObjectsCompat.equals(mLastInsets, insets)) { // 判断是否改变
            mLastInsets = insets; // 保存到成员变量中
            mDrawStatusBarBackground = insets != null && insets.getSystemWindowInsetTop() > 0;
            setWillNotDraw(!mDrawStatusBarBackground && getBackground() == null); // 判断是否绘制background

            // Now dispatch to the Behaviors
            insets = dispatchApplyWindowInsetsToBehaviors(insets); // 下发给子View的Behavior
            requestLayout(); //重新布局
        }
        return insets;
    }
   
   ```
   这个方法主要干了三件事：将新的insets保存到成员变量中，下发该事件给子View的Behavior，重新布局。开始我以为下发给子View的Behavior是新的突破点，然后在众多的Behavior都没有管这个insets，所以这个方法可以先不看。重点就放在了成员变量mLastInsets上。这里还有一个非常重要的事情需要注意：就是这个insets从头到尾都没有被消费。也就是说CoordinatorLayout的子View还可以收到这个事件。 
   
   
   ```java
	// Line: 1172
	private void layoutChild(View child, int layoutDirection) {
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        final Rect parent = acquireTempRect();
        parent.set(getPaddingLeft() + lp.leftMargin,
                getPaddingTop() + lp.topMargin,
                getWidth() - getPaddingRight() - lp.rightMargin,
                getHeight() - getPaddingBottom() - lp.bottomMargin);
		
	    // 在Layout子View的时候，加入了insets值的计算，当parent设置fitsSystemWindows Flag同时子View没有这个Flag，子View就没有办法Layout在这个insets内，反之就可以（也就是说可以和父View的StatusBar那个范围重合）。
        if (mLastInsets != null && ViewCompat.getFitsSystemWindows(this)
                && !ViewCompat.getFitsSystemWindows(child)) {
            // If we're set to handle insets but this child isn't, then it has been measured as
            // if there are no insets. We need to lay it out to match.
            parent.left += mLastInsets.getSystemWindowInsetLeft();
            parent.top += mLastInsets.getSystemWindowInsetTop();
            parent.right -= mLastInsets.getSystemWindowInsetRight();
            parent.bottom -= mLastInsets.getSystemWindowInsetBottom();
        }

        final Rect out = acquireTempRect();
        GravityCompat.apply(resolveGravity(lp.gravity), child.getMeasuredWidth(),
                child.getMeasuredHeight(), parent, out, layoutDirection);
        child.layout(out.left, out.top, out.right, out.bottom);

        releaseTempRect(parent);
        releaseTempRect(out);
    }
   
   ```
   在Layout的Child的时候都加入了insets的计算，具体处理方案参考注释中的解释。  
   
   接下来我们分析下在onMeasure方法中测量子View的代码片段。  
   
   ```java
			// Line: 796
			int childWidthMeasureSpec = widthMeasureSpec;
			int childHeightMeasureSpec = heightMeasureSpec;
			// 当父View中有fitSystemFlag而子View中没有的时候，测量子view的最大宽度和高度会减去insets中的值，反之就不会。
			if (applyInsets && !ViewCompat.getFitsSystemWindows(child)) {
				// We're set to handle insets but this child isn't, so we will measure the
				// child as if there are no insets
				final int horizInsets = mLastInsets.getSystemWindowInsetLeft()
                        + mLastInsets.getSystemWindowInsetRight();
				final int vertInsets = mLastInsets.getSystemWindowInsetTop()
                        + mLastInsets.getSystemWindowInsetBottom();

				childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        widthSize - horizInsets, widthMode);
				childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        heightSize - vertInsets, heightMode);
			}
			
			// 调用子View的Behavior去测量子View，如果Behavior没有处理就自己来测量。
			final Behavior b = lp.getBehavior();
			if (b == null || !b.onMeasureChild(this, child, childWidthMeasureSpec, keylineWidthUsed,
                    childHeightMeasureSpec, 0)) {
				onMeasureChild(child, childWidthMeasureSpec, keylineWidthUsed,
                        childHeightMeasureSpec, 0);
			}
   ```
   在测量子View的时候会根据insets来限制View的最大高度和宽度，具体参考注释。在实际测量的时候会调用Behavior来测量，如果Behavior没有测量就会自己测量，目前只有两个Behavior重写了测量子View的这个方法，一个是AppbarLayout的默认Behavior，等分析AppbarLayout的时候我们再看，另一个暂时不讨论。(在onLayout方法中也会调用Behavior的onLayoutChild方法，在AppbarLayout的Behavior中还是会调用上面分析到的layoutChild方法进行layout，除此之外，还会做一些其他的操作，我也没太看懂)  
   到目前为止，感觉变得有点复杂，其中涉及到view的测量和layout，还有Behavior，感到不适的同学可以多几次前面的内容。  
   
- AppBarLayout  
  
    AppBarLayout和CoordinatorLayout一样也是直接在构造函数中设置了insets的Listener，覆盖了默认的onApplyWindowInsets方法，当有insets事件来的时候，会调用方法``onWindowInsetChanged``，我们也从这个方法开始分析。  
	
	```java 
	WindowInsetsCompat onWindowInsetChanged(final WindowInsetsCompat insets) {
		WindowInsetsCompat newInsets = null;

		if (ViewCompat.getFitsSystemWindows(this)) {
			// If we're set to fit system windows, keep the insets
			newInsets = insets;
		}

		// If our insets have changed, keep them and trigger a layout...
		if (!ObjectsCompat.equals(lastInsets, newInsets)) {
			lastInsets = newInsets;
			requestLayout();
		}

		return insets;
	}
	```
	
	onWindowInsetChanged方法中，如果设置了fitWindows Flag，就会吧insets保存在成员变量lastInsets中，然后requestLayout，CoordinatorLayout也干了这两件事，也没有消耗这个insets。可以通过``getTopInset()``方法获取到这个topInset值，也就是StatusBar的高度。  
	现在我们接着分析上面提到的AppbarLayout默认Behavior中的onMeasureChild()方法：  
	
	```java
	// Line: 1456
	@Override
    public boolean onMeasureChild(
        CoordinatorLayout parent,
        T child,
        int parentWidthMeasureSpec,
        int widthUsed,
        int parentHeightMeasureSpec,
        int heightUsed) {
      final CoordinatorLayout.LayoutParams lp =
          (CoordinatorLayout.LayoutParams) child.getLayoutParams();
      if (lp.height == CoordinatorLayout.LayoutParams.WRAP_CONTENT) {
        // If the view is set to wrap on it's height, CoordinatorLayout by default will
        // cap the view at the CoL's height. Since the AppBarLayout can scroll, this isn't
        // what we actually want, so we measure it ourselves with an unspecified spec to
        // allow the child to be larger than it's parent
        parent.onMeasureChild(
            child,
            parentWidthMeasureSpec,
            widthUsed,
            MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED),
            heightUsed);
        return true;
      }

      // Let the parent handle it as normal
      return super.onMeasureChild(
          parent, child, parentWidthMeasureSpec, widthUsed, parentHeightMeasureSpec, heightUsed);
    }
	```  
	当AppbarLayout的高度是WRAP LAYOUT的时候，就会调用CooradinatorLayout的onMeasureChild方法，同时将WRAP LAYOUT对应的MeasureSpec设置成UNSPECIFIED。其他情况就会调用通用的测量（主要是没有设置成UNSPECIFIED）。  
	在CooradinatorLayout的onMeasureChild方法中最后也会调用子View的measure方法，现在我们来分析一下AppbarLayout的onMeasure方法。  
	
	```java
	// Line: 418
	@Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    // If we're set to handle system windows but our first child is not, we need to add some
    // height to ourselves to pad the first child down below the status bar
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    if (heightMode != MeasureSpec.EXACTLY
        && ViewCompat.getFitsSystemWindows(this)
        && shouldOffsetFirstChild()) {
      int newHeight = getMeasuredHeight();
      switch (heightMode) {
        case MeasureSpec.AT_MOST:
          // For AT_MOST, we need to clamp our desired height with the max height
          newHeight =
              MathUtils.clamp(
                  getMeasuredHeight() + getTopInset(), 0, MeasureSpec.getSize(heightMeasureSpec));
          break;
        case MeasureSpec.UNSPECIFIED:
          // For UNSPECIFIED we can use any height so just add the top inset
          newHeight += getTopInset(); // 添加topInsets
          break;
        default: // fall out
      }
      setMeasuredDimension(getMeasuredWidth(), newHeight);
    }

    invalidateScrollRanges();
  }
	
	```
	在onMeasure方法中对应MeasureSpec的UNSPECIFIED时会在高度额外添加topInset，也对应了上面讲到的，在AppbarLayout的Behavior的onMeasureChild方法中会将WRAP CONTENT的MeasureSpec设置成UNSPECIFIED。  
	
	onLayout方法代码片段: 
	
	```java
	//Line：452
	// 当有fit window flag 并且满足shouldOffsetFirstChild条件时会给第一个子View添加一个topInset的offset。
	if (ViewCompat.getFitsSystemWindows(this) && shouldOffsetFirstChild()) {
      // If we need to offset the first child, we need to offset all of them to make space
      final int topInset = getTopInset();
      for (int z = getChildCount() - 1; z >= 0; z--) {
        ViewCompat.offsetTopAndBottom(getChildAt(z), topInset);
      }
    }
	
	```
	判断flag，如果满足条件会给child添加一个topInset大小的offset值。shoulOffsetFirstChild方法中判断了第一个子view可见性不为gone同时没有设置fitsSystemWindows Flag。``firstChild.getVisibility() != GONE && !ViewCompat.getFitsSystemWindows(firstChild);``  
	小结一下：在设置了windowTranslucentStatus为true的条件下，当CoordinatorLayout的fitSystemWindow的flag为true其子View为false的时候，measure Child的时候会限制子View的最大尺寸不会超过本身的尺寸再减去insets，在layout child的时候不允许出现在insets范围内，其他的情况不会有这两个限制。当AppbarLayout中有fitSystemWindow
	flag时，会在测量的时候会在高度额外添加一个topInset。   
	
	
- Toolbar  

	在Toolbar中并没有对fitSystemWindows作出特殊的处理，是默认ViewGroup的默认处理方式。但是Toolbar中有一些特殊的逻辑会影响到fitSystemWindows的表现。  
	
	1. 当Toolbar的高度小于最小的建议高度时会设置这个建议的最小高度  
		
		Toolbar onMeasure方法片段分析：  
		
		```java
		// Line: 1808
		// Measurement already took padding into account for available space for the children,
        // add it in for the final size.
        width += getPaddingLeft() + getPaddingRight();
        height += getPaddingTop() + getPaddingBottom();

        final int measuredWidth = View.resolveSizeAndState(
                Math.max(width, getSuggestedMinimumWidth()),
                widthMeasureSpec, childState & View.MEASURED_STATE_MASK);
        final int measuredHeight = View.resolveSizeAndState(
                Math.max(height, getSuggestedMinimumHeight()), //当测量的高度小于建议的高度时，会使用建议的高度。  
                heightMeasureSpec, childState << View.MEASURED_HEIGHT_STATE_SHIFT);

        setMeasuredDimension(measuredWidth, shouldCollapse() ? 0 : measuredHeight);
		
		```  
		
		View.reolveSizeAndState方法分析： 
		
		```java
		//Line: 23328
		public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
			final int specMode = MeasureSpec.getMode(measureSpec);
			final int specSize = MeasureSpec.getSize(measureSpec);
			final int result;
			switch (specMode) {
				case MeasureSpec.AT_MOST: // wrap content
					if (specSize < size) {
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    result = size;
                }
                break;
				case MeasureSpec.EXACTLY: // 对应match parent和指定尺寸
					result = specSize;
					break;
				case MeasureSpec.UNSPECIFIED: // 某些特殊的view会制定这个Spec，子View需要多大尺寸，就给多大尺寸。
				default:
					result = size;
			}
			return result | (childMeasuredState & MEASURED_STATE_MASK);
		}
		```
		
		
		
		上面的height和width如果是自定的wrap content就会通过测量子view的高度和宽度依次叠加，然后和建议的尺寸来进行比较，如果小于建议尺寸就会使用建议的尺寸。建议的高度大概是56dp。如果是match parent和指定尺寸，就不会有这个建议尺寸的限制，具体看resolveSizeAndState方法中的代码和注释。介于Toolbar的这个特性就会对fitSystemWindows产生影响，在前面讲过默认的View处理fitSystemWindows就是给当前View添加一个Status高度的paddingTop，因为在Toolbar中的wrap content模式时使用了一个建议固定高度，当添加padding后其子View的显示空间就减少了StatusBar高度，就会导致子view的显示位置靠下。    
	
	2. 无法通过xml给Toolbar的子View设置fitSystemWindows Flag  
	
		当在Activity中的Theme中设置了fitSystemWindowsFlag时，如果没有手动指定，该Activity中的View也会使用这个Theme，在这种情况下如果Toolbar的fitSystemWindowsFlag手动设置为false，这时他的子View会去消耗这个insets，会导致子View设置一个padding值从而内容显示靠下。 


- CollapsingToolbarLayout

	CollapsingToolbarLayout和AppbarLayout，CoordinateLayout一样，也在构造函数中添加了onApplyWindows这个Listener，处理的方法是onWindowInsetChanged。  
	
	
	onWindowInsetChanged方法解析  
	
	```java
	// Line: 278
	WindowInsetsCompat onWindowInsetChanged(final WindowInsetsCompat insets) {
    WindowInsetsCompat newInsets = null;

    if (ViewCompat.getFitsSystemWindows(this)) {
      // If we're set to fit system windows, keep the insets
      newInsets = insets;
    }

    // If our insets have changed, keep them and invalidate the scroll ranges...
    if (!ObjectsCompat.equals(lastInsets, newInsets)) {
      lastInsets = newInsets;
      requestLayout();
    }

    // Consume the insets. This is done so that child views with fitSystemWindows=true do not
    // get the default padding functionality from View
    return insets.consumeSystemWindowInsets(); // 消费了这个insets，这是和AppbarLayout不同的地方。  
  }
	
	```
		
	都是熟悉的套路，不过多讲解了，和AppbarLayout不同的是消费这个insets。  


	onMeasure方法分析：  
	
	```java
	
	// Line: 418
	@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		ensureToolbar();
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);

		final int mode = MeasureSpec.getMode(heightMeasureSpec);
		final int topInset = lastInsets != null ? lastInsets.getSystemWindowInsetTop() : 0;
		// 当height设置为wrap content，同时topInset大于0的时候会给当前测量的高度再加一个topInset，同时MeasureSpec为EXACTLY
		if (mode == MeasureSpec.UNSPECIFIED && topInset > 0) {
		// If we have a top inset and we're set to wrap_content height we need to make sure
		// we add the top inset to our height, therefore we re-measure
		heightMeasureSpec =
			MeasureSpec.makeMeasureSpec(getMeasuredHeight() + topInset, MeasureSpec.EXACTLY);
		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
		}
	}
	
	```
	
	onLayout方法片段分析： 
	
	```java
	
	// Line：435
	super.onLayout(changed, left, top, right, bottom); //先调用FramLayout的onLayout方法对子View Layout

	if (lastInsets != null) {
      // Shift down any views which are not set to fit system windows
      final int insetTop = lastInsets.getSystemWindowInsetTop();
      for (int i = 0, z = getChildCount(); i < z; i++) {
        final View child = getChildAt(i);
		// 当子view没有fit System Window flag 同时子View的top小于insetsTop时就会给子View添加一个insetTop大小的offset。
        if (!ViewCompat.getFitsSystemWindows(child)) {
          if (child.getTop() < insetTop) {
            // If the child isn't set to fit system windows but is drawing within
            // the inset offset it down
            ViewCompat.offsetTopAndBottom(child, insetTop);
          }
        }
      }
    }
	
	
	```
	
	小结：CollapsingToolbarLayout如果height是wrap content同时有fitwindow flag时会在测量的时候额外添加一个topInsets的高度。在layout 子View的时候，如果子View没有fitsSystemWindows flag 同时top值小于当前的topInsets，就会给子View在垂直方向上添加一个topInsets大小的offset。 CollapsingToolbarLayout是会消费insets事件的，在子view中是收不到这个事件的。  
		
	
   
   
   
   





