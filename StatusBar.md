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

_源码版本：com.google.android.material:material:1.1.0-alpha01_

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
   在测量子View的时候会根据insets来限制View的最大高度和宽度，具体参考注释。在实际测量的时候会调用Behavior来测量，如果Behavior没有测量就会自己测量，目前只有两个Behavior重写了测量子View的这个方法，一个是AppbarLayout的默认Behavior，等分析AppbarLayout的时候我们再看，另一个暂时不讨论。  
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
	
	onWindowInsetChanged方法中，如果设置了fitWindows Flag，就会吧insets保存在成员变量lastInsets中，然后requestLayout，CoordinatorLayout也干了这两件事，也没有消耗这个insets，不同的是，在AppbarLayout中并没有用这个insets去计算和测量。可以通过``getTopInset()``方法获取到这个topInset值，也就是StatusBar的高度。
   
   
   
   





