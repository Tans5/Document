# Android中的触摸事件——嵌套滚动


## 前言

  相信绝大部分的Android开发者都遇到过嵌套滑动冲突的问题，后来Google官方也推出了许多解决嵌套滑动冲突的控件。最经典的就是NestedScrollView和RecyclerView。特别是NestedScrollView，他几乎完全取代了ScrollView。Google的这些嵌套滑动控件绝大部分都是通过实现`NestedScrollingChild3`和`NestedScrollingParent3`这两个接口来实现的。当我明白了他们对于嵌套滑动是如何实现的时候，脑袋中就直接冒出两个字——“妙啊”。下面的内容也是以分析NestedScrollView的代码来分析嵌套滑动的处理。  
  
  
## 正文


### NestedScrollingChild3和NestedScorllingParent3

  就像它们的命名一样，NestedScrollingChild3就是那些需要能够滑动同时希望能够支持嵌套滑动的View实现的接口；NestedScrollingParent3就是能够处理Child中有嵌套滑动View的情况。可能描述有点抽象。这里举个例子：假如一个NestedScrollView中有一个RecyclerView，当RecyclerView滑动到底部的时候，再向上滑动，这个时候NestedScrollView就会滑动。在这个例子中RecyclerView就是Child（NestedScrollingChild3的简称，下文也一样），NestedScrollView就是Parent（NestedScrollingParent3的简称，下文也一样）。当Child滑动到底部的时候，再向上滑动就会通知Parent自己还在滑动但是没有办法消费这个滑动距离，然后让Parent来处理这个未消费的滑动距离，当然Parent也可以不处理，可以找它的Parent来处理，依次类推，可以一直向上找处理这个滑动距离的View。所以一个View即可以是Child也可以是Parent，NestedScrollView就同时实现了这两个接口。  
  
  1. NestedScrollingChild3#startNestedScroll

  这个方法是滑动开始的时候会调用的方法，一般是ACTION\_DOWN的时候会调用这个方法，默认情况下，这个方法回去找实现了Parent接口的父View，如果找到了这个View然后就会调用Parent的onStartNestedScroll方法，如果这个方法返回true，表示父View接受了Child的嵌套滑动，Child这时就会把这个Parent保存下来，同时再调用Parent的onNestedScrollAccepted方法，这个方法中会吧Child的滚动方向记录下来。总结一下就是：Child#startNestedScroll -> Parent#onStartNestedScroll -> Parent#onNestedScrollAccepted
  
  2. NestedScrollingChild3#dispatchNestedPreScroll
  
  在Child准备滑动之前会调用这个方法，然后再调用startNestedScroll过程中找到的Parent的onNestedPreScroll方法。这个过程做一个简单的比喻：就好比你想要买一个10000块的电脑，然后你要向老板申请，老板可以拒绝你的申请，也可以只让你买8000的，可能同意你买10000的。在这里你就是Child而老板就是Parent，10000就是Child想要滑动的距离。总结：Child#dispatchNestedPreScroll -> Parent#onNestedPreScroll
  
  3. NestedScrollingChild3#dispatchNestedScroll

  当Child滑动完成后，会调用这个方法，然后调用Parent的onNestedScroll方法。这个过程中有可能Child自己并没有消费完这次的滑动距离，然后把这个没有消费完成的距离交给Parent处理，也就是在上面提到的例子。总结：Child#dispatchNestedScroll -> Parent#onNestedScroll
  
  4. NestedScrollingChild3#dispatchNestedPreFling

  当快速滑动前会调用这个方法，这个方法和dispatchNestedPreScroll方法类似，不同的是：这个方法只会返回两种情况，一种是这次快速滑动被Parent处理，另一种是没有被Parent处理。总结：Child#dispatchNestedPreFling-> Parent#onNestedPreFling
  
  5. NestedScrollingChild3#dispatchNestedFling

  当Child确认处理快速滑动后，调用这个方法。然后Parent能够收到Child的滑动的速度。总结：Child#dispatchNestedFling -> Parent#onNestedFling
  
  
  6. NestedScrollingChild3#stopNestedScroll

  结束滑动时调用这个方法和startNestedScroll方法对应。这个方法会调用Parent中的onStopNestedScroll，然后再将parent清空。总结：Child#startNestedScroll -> Parent#onStopNestedScroll  
  
  
  
### Scroller简单介绍

  有的人可能不熟悉Scroller，但是你肯定熟悉ValueAnimator。它们的实现都是大同小异，都是将用来计算一个值到另一个值的变化过程。不同是Scroller是用来计算滑动，ValueAnimator是用来计算动画。在很多的可以滑动的View中都可以见到它的身影。注意它本身并不实际参与View的滑动，View的滑动是View调用`scrollTo`和`scrollBy`这两个方法来滑动的，这两个方法相信大家都很熟悉。  
  这里以快速滑动这个场景举一个例子。快速滑动会调用Scroller的fling方法，这个方法中可以传入开始滑动的位置，滑动的速度，最大和最小滑动的位置。当调用fling方法后然后再调用`postInvalidateOnAnimation`方法，这个方法会申请View的重绘，在重绘的过程中会调用View的`computeScroll`方法，这个方法默认是空实现，需要我们重写，在这个方法中我们可以调用scroller的`computeScrollOffset`来完成这一帧的滑动位置的计算，然后调用View的`scrollTo`或者`scrollBy`来完成实际View滑动，这时如果scroller的计算还没有完成，就再次调用`postInvalidateOnAnimation`方法申请继续重新绘制View。在文章后面的NestedScrollView的代码分析中能够看到这部分的具体代码。  
  
  
### NestedScrollView嵌套滑动的实现  

  经过前面Scroller和嵌套滑动接口知识的铺垫，还有前面两篇系列文章（多点触控，触摸事件下发）的介绍，现在就可以开始对NestedScrollView源码进行分析了。源码分析可能会迟到，但永远不会缺席。  
  
  首先分析的是`onInterceptTouchEvent`方法，这个方法在前面的文章介绍过，用来控制是否拦截子View的触摸事件。  
  
  ```java
  
  
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        /*
         * This method JUST determines whether we want to intercept the motion.
         * If we return true, onMotionEvent will be called and we do the actual
         * scrolling there.
         */

        /*
        * Shortcut the most recurring case: the user is in the dragging
        * state and he is moving his finger.  We want to intercept this
        * motion.
        */
        final int action = ev.getAction();
        
        // 如果是MOVE事件同时已经被被拖拽住，直接拦截。
        if ((action == MotionEvent.ACTION_MOVE) && mIsBeingDragged) {
            return true;
        }

        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_MOVE: {
                /*
                 * mIsBeingDragged == false, otherwise the shortcut would have caught it. Check
                 * whether the user has moved far enough from his original down touch.
                 */

                /*
                * Locally do absolute value. mLastMotionY is set to the y value
                * of the down event.
                */
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    // If we don't have a valid id, the touch down wasn't on content.
                    break;
                }

                final int pointerIndex = ev.findPointerIndex(activePointerId);
                if (pointerIndex == -1) {
                    Log.e(TAG, "Invalid pointerId=" + activePointerId
                            + " in onInterceptTouchEvent");
                    break;
                }

                final int y = (int) ev.getY(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                
                // 当滑动的距离大于touchSlop同时子View嵌套滑动方向不是垂直方向时，该次事件流就会被
                // NestedScrollView接管。这里很重要，我举一个例子：当Child中有一个Button，用户用手按住Button
                // 这个时候NestedScrollView会把这次事件交给Button处理，但是在这个过程中你的手又向上滑动了，那么
                // 这次事件将被NSV拦截掉，然后不再给Button处理，NSV自己处理。
                if (yDiff > mTouchSlop
                        && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
                    
                    // 确认被拖拽
                    mIsBeingDragged = true;
                    mLastMotionY = y;
                    
                    // 初始化velocityTracker，用来计算用户滑动的速度
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                    mNestedYOffset = 0;
                    
                    
                    final ViewParent parent = getParent();
                    // 不允许parent拦截，前面事件下发分析文章有讲过。
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }
                break;
            }

            case MotionEvent.ACTION_DOWN: {
                final int y = (int) ev.getY();
                // 如果点击不再Child内，停止拖拽。
                if (!inChild((int) ev.getX(), y)) {
                    mIsBeingDragged = false;
                    recycleVelocityTracker();
                    break;
                }

                /*
                 * Remember location of down touch.
                 * ACTION_DOWN always refers to pointer index 0.
                 */
                mLastMotionY = y;
                mActivePointerId = ev.getPointerId(0);

                initOrResetVelocityTracker();
                mVelocityTracker.addMovement(ev);
                /*
                 * If being flinged and user touches the screen, initiate drag;
                 * otherwise don't. mScroller.isFinished should be false when
                 * being flinged. We need to call computeScrollOffset() first so that
                 * isFinished() is correct.
                */
                
                // 判断scroller的滑动是否完成，如果没有完成则标记为拖拽。
                mScroller.computeScrollOffset();
                mIsBeingDragged = !mScroller.isFinished();
                
                // 开始嵌套滑动，等下分析下这个方法里面做的事情。
                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
                break;
            }

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                /* Release the drag */
                mIsBeingDragged = false;
                mActivePointerId = INVALID_POINTER;
                recycleVelocityTracker();
                
                // 滑动到边界的时候，实现弹簧效果（NestedScrollView中没有边界的弹簧效果，后面这个代
                // 码中我就不分析它了），这段代码现目前应该是没有用的。
                if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0, getScrollRange())) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
                // 结束嵌套滑动
                stopNestedScroll(ViewCompat.TYPE_TOUCH);
                break;
            case MotionEvent.ACTION_POINTER_UP:
            	   
            	   // 多点触控的处理，在前面的文章中有分析，后面在onTouchEvent中再简单分析下。
                onSecondaryPointerUp(ev);
                break;
        }

        /*
        * The only time we want to intercept motion events is if we are in the
        * drag mode.
        */
        // 如果已经被拖拽，就拦截。
        return mIsBeingDragged;
    }
  
  
  ``` 
  
  上面代码中主要是判断拖拽，如果已经拖拽就拦截。这里简单总结下拦截的情况：1. DOWN：当点击的位置不在Child内时，停止拦截。2. DOWN：如果当前滑动未结束，拦截。3. MOVE: 如果Child没有在垂直方向上的滑动，则拦截滑动。
  
  
  下面再分析下`startNestedScroll`和`stopNestedScroll`中的代码：  
  
  
  ```java
  
  	
  	 // 直接调用的Helper中的startNestedScroll方法，这个helper是NestedScrollingChildHelper，还有
  	 // parentHelper，类型是NestedScrollingParentHelper。在初始化这两个helper的时候记得开启嵌套滑动。  
    @Override
    public boolean startNestedScroll(int axes, int type) {
        return mChildHelper.startNestedScroll(axes, type);
    }
  
  
    // NestedScrollingChildHelper#startNestedScroll
    /**
     * Start a new nested scroll for this view.
     *
     * <p>This is a delegate method. Call it from your {@link android.view.View View} subclass
     * method/{@link androidx.core.view.NestedScrollingChild2} interface method with the same
     * signature to implement the standard policy.</p>
     *
     * @param axes Supported nested scroll axes.
     *             See {@link androidx.core.view.NestedScrollingChild2#startNestedScroll(int,
     *             int)}.
     * @return true if a cooperating parent view was found and nested scrolling started successfully
     */
    public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
        // axes: 滑动的方向，水平和垂直。  
        // type：这里的type一共有两种，TOUCH和NONE TOUCH。分别对应通过Touch滑动和非Touch滑动（例如惯性滑动），
        // 后面很多方法中都有这个参数就不赘述了。
        
        // 如果已经有parent处理了就直接返回。
        if (hasNestedScrollingParent(type)) {
            // Already in progress
            return true;
        }
        
        // 支持嵌套滑动时
        if (isNestedScrollingEnabled()) {
            ViewParent p = mView.getParent();
            View child = mView;
            // 遍历child的parent
            while (p != null) {
            		// 调用Parent的onStartNestedScroll方法，当返回为true时表示，这个parent接收了这次嵌套滑动。 
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                		
                
                		// 这个方法会将这个parent和对应的type保存下来，方便后面的一系列操作。  
                    setNestedScrollingParentForType(type, p);
                    
                    // 调用parent的Accepted方法。
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
    
    // NestedScrollView#onStartNestedScroll方法
    // 判断滑动方向是不是水平的，如果是就接受这次嵌套滑动。  
    @Override
    public boolean onStartNestedScroll(@NonNull View child, @NonNull View target, int axes,
            int type) {
        return (axes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
    }
    
    // NestedScrollView#onNestedScrollAccepted
    
    // 这里会调用parentHelper的onNestedScrollAccepted方法，然后又调用了startNestedScroll方法（也就是再找
    // 它的parent来处理这次嵌套滑动）。
    @Override
    public void onNestedScrollAccepted(@NonNull View child, @NonNull View target, int axes,
            int type) {
        mParentHelper.onNestedScrollAccepted(child, target, axes, type);
        startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, type);
    }
    
    // NestedScrollingParentHelper#onNestedScrollAccepted
    
    /**
     * Called when a nested scrolling operation initiated by a descendant view is accepted
     * by this ViewGroup.
     *
     * <p>This is a delegate method. Call it from your {@link android.view.ViewGroup ViewGroup}
     * subclass method/{@link androidx.core.view.NestedScrollingParent2} interface method with
     * the same signature to implement the standard policy.</p>
     */
    // 将接受的子View的滑动type保存下来。  
    public void onNestedScrollAccepted(@NonNull View child, @NonNull View target,
            @ScrollAxis int axes, @NestedScrollType int type) {
        if (type == ViewCompat.TYPE_NON_TOUCH) {
            mNestedScrollAxesNonTouch = axes;
        } else {
            mNestedScrollAxesTouch = axes;
        }
    }
  
  ```
  
  这里简单总结下start的过程，Child调用startNestedScroll，这里会寻找一个Parent来处理这次嵌套滑动，Parent会通过调用onStartNestedScroll方法来确认是否（是否是垂直方向）接受这次滑动，如果接受这次滑动，Parent还会调用onNestedScrollAccepted方法同时又调用了自己的startNestedScroll方法（这时的Parent又作为一个Child去寻找它的parent来处理这次嵌套滑动，通俗点来说就是“套娃”，有很多这样的“套娃”操作，后面就不详说明了）。