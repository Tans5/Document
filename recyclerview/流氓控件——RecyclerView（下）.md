# 流氓控件——RecyclerView（下） 

在上篇内容中我们谈到了，RecyclerView的measure，layout，draw，ItemDecoration，ItemAnimator。第一次measure过程中如果RecyclerView的宽和高的MeasureMode都是Exactly时，进行简单的测量，就直接进入layout流程，但是如果调用了notifyDataSetChanged方法后，在measure过程中就有可能进入LayoutStep1和LayoutStep2；Layout流程一共分为3步，其主要也是为了动画而服务的，真正的Layout是由今天介绍的LayoutManager实现的，Step1主要是记录上次Layout后子View的位置（前提是必须开启动画，同时这个过程中可能会调用LayoutManager的onLayoutChildren方法），Step2就是进行真实的测量，Step3会对Step2 layout后的子View的位置信息进行测量，类似于Step1，然后把前后为位置信息传递给ItemAnimator，然后执行动画；draw流程就比较简单了，直接调用了ItemDecoration的onDraw方法。如果还有不清楚的地方，请查阅上篇内容。  
  
那么开始本篇的正文内容，本篇的内容中包括LayoutManager和Recycler。  


## LayoutManager

本次LayoutManager的代码分析是采用LinearLayoutManager的代码，最简单的LayoutManager，同时也是最强大的LayoutManager，GridLayoutManager也是LinearLayoutManager的派生类。

开始代码分析前先看个图，有助于对代码的理解。  

![](recyclerview.png)  

上面的图是从Start向End layout的情况，End到Start layout我就不再给图了，自己想象。

先看两个重要的成员变量类，AnchorInfo和LayoutState。

```java

/**
 * 在onLayoutChildren方法中主要是根据pendding或者当前子View的信息初始化该对象，然后通过该对象再初始化LayoutState。
 */
static class AnchorInfo {
		// orientationHelper这个对象非常有用，可以用来拿子View的各种信息，分为Vertical和Horizontal helper
        OrientationHelper mOrientationHelper;
        // 下次Layout需要的Position
        int mPosition;
        // layout的坐标，见上面的图你就明白了。
        int mCoordinate;
        // Layout的方向start to end 或者 end to start
        boolean mLayoutFromEnd;
        // 是否有效
        boolean mValid;
        
        ...
        
        }
        
    /**
     * Helper class that keeps temporary state while {LayoutManager} is filling out the empty
     * space.
     */
    static class LayoutState {

        ...

        /**
         * We may not want to recycle children in some cases (e.g. layout)
         */
         // 是否会进行回收，在layout时不会回收，scroll的时候会回收
        boolean mRecycle = true;

        /**
         * Pixel offset where layout should start
         */
         // 和coordinate一样见上图
        int mOffset;

        /**
         * Number of pixels that we should fill, in the layout direction.
         */
         // 见上图描述。
        int mAvailable;

        /**
         * Current position on the adapter to get the next item.
         */
         // 当前需要layout的position
        int mCurrentPosition;

        /**
         * Defines the direction in which the data adapter is traversed.
         * Should be {@link #ITEM_DIRECTION_HEAD} or {@link #ITEM_DIRECTION_TAIL}
         */
         // 从adapter中取数据的方向。
        int mItemDirection;

        /**
         * Defines the direction in which the layout is filled.
         * Should be {@link #LAYOUT_START} or {@link #LAYOUT_END}
         */
         // Layout的方向。
        int mLayoutDirection;

        /**
         * Used when LayoutState is constructed in a scrolling state.
         * It should be set the amount of scrolling we can make without creating a new view.
         * Settings this is required for efficient view recycling.
         */
         // 这个值只有在滚动的时候有用，到时候再说。
        int mScrollingOffset;

        /**
         * Used if you want to pre-layout items that are not yet visible.
         * The difference with {@link #mAvailable} is that, when recycling, distance laid out for
         * {@link #mExtraFillSpace} is not considered to avoid recycling visible children.
         */
         // 需要额外填充子view的空间，相当于在available加一个offset
        int mExtraFillSpace = 0;

        /**
         * Contains the {@link #calculateExtraLayoutSpace(RecyclerView.State, int[])}  extra layout
         * space} that should be excluded for recycling when cleaning up the tail of the list during
         * a smooth scroll.
         */
         // 不会触发回收的offset
        int mNoRecycleSpace = 0;

        /**
         * Equal to {@link RecyclerView.State#isPreLayout()}. When consuming scrap, if this value
         * is set to true, we skip removed views since they should not be laid out in post layout
         * step.
         */
         // 是否时PreLayout（就是Layout过程中的Step1）
        boolean mIsPreLayout = false;

        /**
         * The most recent {@link #scrollBy(int, RecyclerView.Recycler, RecyclerView.State)}
         * amount.
         */
         // 上次滚动的距离
        int mLastScrollDelta;

        /**
         * When LLM needs to layout particular views, it sets this list in which case, LayoutState
         * will only return views from this list and return null if it cannot find an item.
         */
         // 和Recycler中的scrapList差不多，Recycler中还会看到它，就是上次Layout中的子View会被detached后放到这个list中
        List<RecyclerView.ViewHolder> mScrapList = null;

        /**
         * Used when there is no limit in how many views can be laid out.
         */
         // 在Layout中是否可以无视available中的值无限layout，直到adapter中的item全部layout。
        boolean mInfinite;
        
        ...
        
        }

```




下面是onLayoutChildren的代码，代码比较长，不过我会给注释的，请放心食用。

```java

@Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        // layout algorithm:
        // 1) by checking children and other variables, find an anchor coordinate and an anchor
        //  item position.
        // 2) fill towards start, stacking from bottom
        // 3) fill towards end, stacking from top
        // 4) scroll to fulfill requirements like stack from bottom.
        // create layout state
        if (DEBUG) {
            Log.d(TAG, "is pre layout:" + state.isPreLayout());
        }
        
        /**
          * 我暂时把LayoutChildren分为两部分。第一部分为初始化AnchorInfo，初始化AnchorInfo又有两种方法，一种是通过
          * paddingSavedState，这个State是在LayoutManager意外销毁的时候会通过LayoutState保存起来的，恢复的时候又会恢
          * 复padding这个对象，如果你是个有经验的开发者，这种操作应该见得很多了，个人认为这个东西不太好用。还有一种方法是通
          * 过当前的子View状态来初始化AnchorInfo。第二部分为通过AnchorInfo然后初始化LayoutState，然后执行Layout操作。
          *
          */
          
          // 在某些条件下会回收所有的子View，条件就自己看了。
        if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
            if (state.getItemCount() == 0) {
                removeAndRecycleAllViews(recycler);
                return;
            }
        }
        if (mPendingSavedState != null && mPendingSavedState.hasValidAnchor()) {
            mPendingScrollPosition = mPendingSavedState.mAnchorPosition;
        }
			
		// 初始化LayoutState
        ensureLayoutState();
        mLayoutState.mRecycle = false;
        // resolve layout direction
        // 初始化Layout的方向
        resolveShouldLayoutReverse();
		
		
		// 初始化AnchorInfo，可以通过有焦点的子View或者pendingState或者可见的最后一个子View（如果Layout方向是End）。详细代码后面分析。
        final View focused = getFocusedChild();
        if (!mAnchorInfo.mValid || mPendingScrollPosition != RecyclerView.NO_POSITION
                || mPendingSavedState != null) {
            mAnchorInfo.reset();
            mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
            // calculate anchor position and coordinate
            updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
            mAnchorInfo.mValid = true;
        } else if (focused != null && (mOrientationHelper.getDecoratedStart(focused)
                        >= mOrientationHelper.getEndAfterPadding()
                || mOrientationHelper.getDecoratedEnd(focused)
                <= mOrientationHelper.getStartAfterPadding())) {
            // This case relates to when the anchor child is the focused view and due to layout
            // shrinking the focused view fell outside the viewport, e.g. when soft keyboard shows
            // up after tapping an EditText which shrinks RV causing the focused view (The tapped
            // EditText which is the anchor child) to get kicked out of the screen. Will update the
            // anchor coordinate in order to make sure that the focused view is laid out. Otherwise,
            // the available space in layoutState will be calculated as negative preventing the
            // focused view from being laid out in fill.
            // Note that we won't update the anchor position between layout passes (refer to
            // TestResizingRelayoutWithAutoMeasure), which happens if we were to call
            // updateAnchorInfoForLayout for an anchor that's not the focused view (e.g. a reference
            // child which can change between layout passes).
            mAnchorInfo.assignFromViewAndKeepVisibleRect(focused, getPosition(focused));
        }
        if (DEBUG) {
            Log.d(TAG, "Anchor info:" + mAnchorInfo);
        }

        // LLM may decide to layout items for "extra" pixels to account for scrolling target,
        // caching or predictive animations.

        mLayoutState.mLayoutDirection = mLayoutState.mLastScrollDelta >= 0
                ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
        mReusableIntPair[0] = 0;
        mReusableIntPair[1] = 0;
        // 计算可以额外Layout的extra值，分为Start和End的extra值，你可以重写下面的方法，来返回你想要的值。
        calculateExtraLayoutSpace(state, mReusableIntPair);
        int extraForStart = Math.max(0, mReusableIntPair[0])
                + mOrientationHelper.getStartAfterPadding();
        int extraForEnd = Math.max(0, mReusableIntPair[1])
                + mOrientationHelper.getEndPadding();

        // 下面还会根据不同的状态调整这个extra值，没有深入研究这种情况。
        if (state.isPreLayout() && mPendingScrollPosition != RecyclerView.NO_POSITION
                && mPendingScrollPositionOffset != INVALID_OFFSET) {
            // if the child is visible and we are going to move it around, we should layout
            // extra items in the opposite direction to make sure new items animate nicely
            // instead of just fading in
            final View existing = findViewByPosition(mPendingScrollPosition);
            if (existing != null) {
                final int current;
                final int upcomingOffset;
                if (mShouldReverseLayout) {
                    current = mOrientationHelper.getEndAfterPadding()
                            - mOrientationHelper.getDecoratedEnd(existing);
                    upcomingOffset = current - mPendingScrollPositionOffset;
                } else {
                    current = mOrientationHelper.getDecoratedStart(existing)
                            - mOrientationHelper.getStartAfterPadding();
                    upcomingOffset = mPendingScrollPositionOffset - current;
                }
                if (upcomingOffset > 0) {
                    extraForStart += upcomingOffset;
                } else {
                    extraForEnd -= upcomingOffset;
                }
            }
        }
        int startOffset;
        int endOffset;
        final int firstLayoutDirection;
        if (mAnchorInfo.mLayoutFromEnd) {
            firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL
                    : LayoutState.ITEM_DIRECTION_HEAD;
        } else {
            firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                    : LayoutState.ITEM_DIRECTION_TAIL;
        }
		
		// 这个方法是个空实现，在GridLayoutManager中有实现这个方法，然后我们写的LayoutManager没办法实现这个方法，可见性是internal的。
        onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
        detachAndScrapAttachedViews(recycler);
        mLayoutState.mInfinite = resolveIsInfinite();
        mLayoutState.mIsPreLayout = state.isPreLayout();
        // noRecycleSpace not needed: recycling doesn't happen in below's fill
        // invocations because mScrollingOffset is set to SCROLLING_OFFSET_NaN
        mLayoutState.mNoRecycleSpace = 0;
        if (mAnchorInfo.mLayoutFromEnd) {
            // fill towards start
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForStart;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
            final int firstElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForEnd += mLayoutState.mAvailable;
            }
            // fill towards end
            updateLayoutStateToFillEnd(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForEnd;
            mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;

            if (mLayoutState.mAvailable > 0) {
                // end could not consume all. add more items towards start
                extraForStart = mLayoutState.mAvailable;
                updateLayoutStateToFillStart(firstElement, startOffset);
                mLayoutState.mExtraFillSpace = extraForStart;
                fill(recycler, mLayoutState, state, false);
                startOffset = mLayoutState.mOffset;
            }
        } else {
            // fill towards end
            updateLayoutStateToFillEnd(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForEnd;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
            final int lastElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForStart += mLayoutState.mAvailable;
            }
            // fill towards start
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForStart;
            mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;

            if (mLayoutState.mAvailable > 0) {
                extraForEnd = mLayoutState.mAvailable;
                // start could not consume all it should. add more items towards end
                updateLayoutStateToFillEnd(lastElement, endOffset);
                mLayoutState.mExtraFillSpace = extraForEnd;
                fill(recycler, mLayoutState, state, false);
                endOffset = mLayoutState.mOffset;
            }
        }

        // changes may cause gaps on the UI, try to fix them.
        // TODO we can probably avoid this if neither stackFromEnd/reverseLayout/RTL values have
        // changed
        if (getChildCount() > 0) {
            // because layout from end may be changed by scroll to position
            // we re-calculate it.
            // find which side we should check for gaps.
            if (mShouldReverseLayout ^ mStackFromEnd) {
                int fixOffset = fixLayoutEndGap(endOffset, recycler, state, true);
                startOffset += fixOffset;
                endOffset += fixOffset;
                fixOffset = fixLayoutStartGap(startOffset, recycler, state, false);
                startOffset += fixOffset;
                endOffset += fixOffset;
            } else {
                int fixOffset = fixLayoutStartGap(startOffset, recycler, state, true);
                startOffset += fixOffset;
                endOffset += fixOffset;
                fixOffset = fixLayoutEndGap(endOffset, recycler, state, false);
                startOffset += fixOffset;
                endOffset += fixOffset;
            }
        }
        layoutForPredictiveAnimations(recycler, state, startOffset, endOffset);
        if (!state.isPreLayout()) {
            mOrientationHelper.onLayoutComplete();
        } else {
            mAnchorInfo.reset();
        }
        mLastStackFromEnd = mStackFromEnd;
        if (DEBUG) {
            validateChildOrder();
        }
    }

```