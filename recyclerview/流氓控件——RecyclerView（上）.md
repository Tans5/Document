# 流氓控件——RecyclerView（上）
  
相信大家对于RecyclerView都非常熟悉，这是Google在Android 5.0发布时，一同发布的一个控件。它默认能够实现列表、网格、瀑布流等UI式样。由于它的性能比它的前辈们更加好，可定制性也更加强，现在的开发也基本上放弃使用ListView和GridView等一些年纪比较大的列表控件；在2019年的Google IO大会上推出了ViewPager2，它的实现也是通过RecyclerView，性能无疑也是优于现在的ViewPager，现在的ViewPager被替换掉也只是时间的问题；还有某些知名的库利用RecyclerView对子View懒加载的特性，用来提高UI渲染的性能；甚至还有人用RecyclerView[画龙](https://blog.csdn.net/u011387817/article/details/81875021)；RecyclerView能干很多控件能干的事情，更加恐怖的是，它还能干得更加好，当之无愧的流氓控件。  

*Tips: 源码版本：com.google.android.material:material:1.1.0-alpha10* 

## onMeasure, onLayout, onDraw流程分析

在本阶段主要分析onLayout的过程。
  
### onMeasure

RecyclerView onMeasure方法部分代码： 


```java

@Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        if (mLayout == null) {
            defaultOnMeasure(widthSpec, heightSpec);
            return;
        }
        // LinearLayoutManager 默认是AutoMeasure的，暂时也只考虑这种情况。
        if (mLayout.isAutoMeasureEnabled()) {
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);

            /**
             * This specific call should be considered deprecated and replaced with
             * {@link #defaultOnMeasure(int, int)}. It can't actually be replaced as it could
             * break existing third party code but all documentation directs developers to not
             * override {@link LayoutManager#onMeasure(int, int)} when
             * {@link LayoutManager#isAutoMeasureEnabled()} returns true.
             */
             // 调用layoutManager的onMeasure方法，LinearLayoutManager中默认使用的是RecyclerView中的defaultOnMeasure方法。
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
				
            final boolean measureSpecModeIsExactly =
                    widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
            
            /** 当withMode和heightMode都为EXACTLY时或者mAdapter为空的时候直接返回，
             * 但是当第一次requestLayout的时候withMode和heightModel都为EXACTLY时，后面再次调notify
             * 的方法时这个mode有可能会改变，后面在分析notifyItemRemoved的时候会说到（但是还不清楚为啥）
             */
            if (measureSpecModeIsExactly || mAdapter == null) {
                return;
            }

            if (mState.mLayoutStep == State.STEP_START) {
            	// 进入LayoutStep1逻辑
                dispatchLayoutStep1();
            }
            // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
            // consistency
            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;
            // 进入LayoutStep2逻辑
            dispatchLayoutStep2();

            // now we can get the width and height from the children.
            // 再次通过已经measure的children再次进行measure
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            // if RecyclerView has non-exact width and height and if there is at least one child
            // which also has non-exact width & height, we have to re-measure.
            if (mLayout.shouldMeasureTwice()) {
                mLayout.setMeasureSpecs(
                        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
                mState.mIsMeasuring = true;
                dispatchLayoutStep2();
                // now we can get the width and height from the children.
                mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            }
        } else {
            ...
        }
    }

```

AutoMeasureEnable逻辑：首先会调用LayoutManager的onMeasure方法，在LinearLayoutManger中默认也是调用的defaultOnMeasure方法，这个方法中就只是加上padding值和本身的size作为最后的size。heightMode和widthMode都为EXACTLY时或者adapter为空时会直接结束测量流程（但是某些情况下这个mode会改变）。然后进入LayoutStep1和LayoutStep2，这两个方法后面会讲到。最后通过已经测量好的children再对自己进行测量。 

  
### onDraw

RecyclerView onDraw方法：

```

@Override
    public void onDraw(Canvas c) {
        super.onDraw(c);

        final int count = mItemDecorations.size();
        for (int i = 0; i < count; i++) {
            mItemDecorations.get(i).onDraw(c, this, mState);
        }
    }

``` 

onDraw方法确实有点简单，直接调用了Decorations的onDraw()方法，关于Decoration我们后面再看。  

### onLayout

onLayout方法就相对比较复杂了，其除了真实的layout逻辑（这些逻辑是LayoutManager完成的），主要的代码都是为了处理动画。 
   
onLayout步骤：
  
  - LayoutStep1
    
  	进行PreLayout，也就是将上次Children的位置信息记录下来，然后供动画使用（还有做了一些其他的事情没有看懂了）。 给一个官方的描述：
  	
   > /**
   >  * The first step of a layout where we; <br>
   >  * - process adapter updates <br>
   >  * - decide which animation should run <br>
   >  * - save information about current views <br>
   >  * - If necessary, run predictive layout and save its information <br>
   >  */
  	
  - LayoutStep2
  
  进行真实的Layout逻辑，该逻辑是通过layoutManager的onLayoutChildren方法实现的。 
  
  - LayoutStep3
  
  在LayoutStep2中完成Layout后将当前的children位置信息记录下来然后和Step1中的preLayout位置信息作比较然后执行动画。  
  
  
Layout过程中重要的变量：

  - ViewInfoStore mViewInfoStore
  
  主要保存和执行view相关的动画信息。
  
  ```java
  
  /**
     * View data records for pre-layout
     */
    @VisibleForTesting
    // 这个map中保存了preLayout和layout后的信息等，方便执行动画。
    final ArrayMap<RecyclerView.ViewHolder, InfoRecord> mLayoutHolderMap = new ArrayMap<>();
	
	
	// 这个貌似是变化的ViewHolders，我调查时没有走到相关逻辑，暂时不是太清楚。
    @VisibleForTesting
    final LongSparseArray<RecyclerView.ViewHolder> mOldChangedHolders = new LongSparseArray<>();
  
  ```
  
  InfoRecord中的相关代码： 
  
  ```java
  
  static class InfoRecord {
        // disappearing list
        static final int FLAG_DISAPPEARED = 1;
        // appear in pre layout list
        static final int FLAG_APPEAR = 1 << 1;
        // pre layout, this is necessary to distinguish null item info
        static final int FLAG_PRE = 1 << 2;
        // post layout, this is necessary to distinguish null item info
        static final int FLAG_POST = 1 << 3;
        static final int FLAG_APPEAR_AND_DISAPPEAR = FLAG_APPEAR | FLAG_DISAPPEARED;
        static final int FLAG_PRE_AND_POST = FLAG_PRE | FLAG_POST;
        static final int FLAG_APPEAR_PRE_AND_POST = FLAG_APPEAR | FLAG_PRE | FLAG_POST;
        int flags;
        @Nullable
        RecyclerView.ItemAnimator.ItemHolderInfo preInfo;
        @Nullable
        RecyclerView.ItemAnimator.ItemHolderInfo postInfo;
        static Pools.Pool<InfoRecord> sPool = new Pools.SimplePool<>(20);

        private InfoRecord() {
        }

        static InfoRecord obtain() {
            InfoRecord record = sPool.acquire();
            return record == null ? new InfoRecord() : record;
        }

        static void recycle(InfoRecord record) {
            record.flags = 0;
            record.preInfo = null;
            record.postInfo = null;
            sPool.release(record);
        }

        static void drainCache() {
            //noinspection StatementWithEmptyBody
            while (sPool.acquire() != null);
        }
    }
  
  ```
  这里面的代码记录了prelayout的信息和postlayout的信息，还有flag（用来判断执行什么样的动画）。
  
  - State mState

  这个State在很多地方都可以看到，其中记录了很多的flag和状态，比如说：是否执行动画，item的count等很多信息。在Layout中需要关心的是LayoutStep，分别是STEP\_START, STEP\_LAYOUT, STEP\_ANIMATIONS，分别对应LayoutStep1，LayoutStep2，LayoutStep3.  
  
#### onLayout方法  

```java

@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
        dispatchLayout();
        TraceCompat.endSection();
        mFirstLayoutComplete = true;
    }

``` 

这个方法里面很简单，直接进入dispatchLayout方法。 

dispatchLayout方法代码：

```java

void dispatchLayout() {
        if (mAdapter == null) {
            Log.e(TAG, "No adapter attached; skipping layout");
            // leave the state in START
            return;
        }
        if (mLayout == null) {
            Log.e(TAG, "No layout manager attached; skipping layout");
            // leave the state in START
            return;
        }
        mState.mIsMeasuring = false;
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            // First 2 steps are done in onMeasure but looks like we have to run again due to
            // changed size.
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            // always make sure we sync them (to ensure mode is exact)
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3();
    }

```

这个方法中会根据不同的flag执行Step1，Step2，Step3或者Step2，Step3或者Step3. 

#### onLayoutStep1

```java

/**
     * The first step of a layout where we;
     * - process adapter updates
     * - decide which animation should run
     * - save information about current views
     * - If necessary, run predictive layout and save its information
     */
    private void dispatchLayoutStep1() {
       // 验证State，初始化一些flag
        mState.assertLayoutStep(State.STEP_START);
        fillRemainingScrollValues(mState);
        mState.mIsMeasuring = false;
        startInterceptRequestLayout();
        mViewInfoStore.clear();
        onEnterLayoutOrScroll();
        processAdapterUpdatesAndSetAnimationFlags();
        saveFocusInfo();
        mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
        mItemsAddedOrRemoved = mItemsChanged = false;
        mState.mInPreLayout = mState.mRunPredictiveAnimations;
        mState.mItemCount = mAdapter.getItemCount();
        findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);
		
		 // preLayout逻辑，获取到的View信息只是上次Layout的信息。
        if (mState.mRunSimpleAnimations) {
            // Step 0: Find out where all non-removed items are, pre-layout
            int count = mChildHelper.getChildCount();
            // 遍历所有的子View
            for (int i = 0; i < count; ++i) {
                // 获取到对应子View的ViewHolder
                final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                    continue;
                }
                
                // 生成对应View的animationInfo
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPreLayoutInformation(mState, holder,
                                ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                                holder.getUnmodifiedPayloads());
                // 将信息添加到infoStore，同时打上PRE_LAYOUT的flag
                mViewInfoStore.addToPreLayout(holder, animationInfo);
                
                // 这部分逻辑不知道什么情况下会触发，看后面代码会对changeHolders那个List操作。
                if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                        && !holder.shouldIgnore() && !holder.isInvalid()) {
                    long key = getChangedHolderKey(holder);
                    // This is NOT the only place where a ViewHolder is added to old change holders
                    // list. There is another case where:
                    //    * A VH is currently hidden but not deleted
                    //    * The hidden item is changed in the adapter
                    //    * Layout manager decides to layout the item in the pre-Layout pass (step1)
                    // When this case is detected, RV will un-hide that view and add to the old
                    // change holders list.
                    mViewInfoStore.addToOldChangeHolders(key, holder);
                }
            }
        }
        if (mState.mRunPredictiveAnimations) {
            // Step 1: run prelayout: This will use the old positions of items. The layout manager
            // is expected to layout everything, even removed items (though not to add removed
            // items back to the container). This gives the pre-layout position of APPEARING views
            // which come into existence as part of the real layout.

            // Save old positions so that LayoutManager can run its mapping logic.
            saveOldPositions();
            final boolean didStructureChange = mState.mStructureChanged;
            mState.mStructureChanged = false;
            // temporarily disable flag because we are asking for previous layout
            // 这里会执行Layout逻辑
            mLayout.onLayoutChildren(mRecycler, mState);
            mState.mStructureChanged = didStructureChange;

            for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
                final View child = mChildHelper.getChildAt(i);
                final ViewHolder viewHolder = getChildViewHolderInt(child);
                if (viewHolder.shouldIgnore()) {
                    continue;
                }
                
                // 这段代码不清楚在什么情况下会触发，最终也是对mViewInfoStore进行操作。
                if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                    int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(viewHolder);
                    boolean wasHidden = viewHolder
                            .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                    if (!wasHidden) {
                        flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                    }
                    final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                            mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                    if (wasHidden) {
                        recordAnimationInfoIfBouncedHiddenView(viewHolder, animationInfo);
                    } else {
                        mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                    }
                }
            }
            // we don't process disappearing list because they may re-appear in post layout pass.
            clearOldPositions();
        } else {
            clearOldPositions();
        }
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
        // 进行设置STEP，方便顺利进行下个Step。 
        mState.mLayoutStep = State.STEP_LAYOUT;
    }

```

就和前面提到的一样，这个Step会将上次layout的的View的信息以及Flag保存在mViewInfoStore，上面有两段逻辑不知道什么时候触发，但是也都是对mViewInfoStore的操作。  


#### onLayoutStep2

```java

/**
     * The second layout step where we do the actual layout of the views for the final state.
     * This step might be run multiple times if necessary (e.g. measure).
     */
    private void dispatchLayoutStep2() {
        startInterceptRequestLayout();
        onEnterLayoutOrScroll();
        mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mItemCount = mAdapter.getItemCount();
        mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

        // Step 2: Run layout
        mState.mInPreLayout = false;
        mLayout.onLayoutChildren(mRecycler, mState);

        mState.mStructureChanged = false;
        mPendingSavedState = null;

        // onLayoutChildren may have caused client code to disable item animations; re-check
        mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
        mState.mLayoutStep = State.STEP_ANIMATIONS;
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
    }

```
调用LayoutManager的onLayoutChildren方法进行layout，整体代码相对简单。 

#### LayoutStep3

```java

private void dispatchLayoutStep3() {

        mState.assertLayoutStep(State.STEP_ANIMATIONS);
        startInterceptRequestLayout();
        onEnterLayoutOrScroll();
        mState.mLayoutStep = State.STEP_START;
        
        if (mState.mRunSimpleAnimations) {
            // Step 3: Find out where things are now, and process change animations.
            // traverse list in reverse because we may call animateChange in the loop which may
            // remove the target view holder.
            // 遍历子View，这个时候已经是Layout过后的的数据，和Step1中不一样
            for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
                ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                if (holder.shouldIgnore()) {
                    continue;
                }
                long key = getChangedHolderKey(holder);
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPostLayoutInformation(mState, holder);
                ViewHolder oldChangeViewHolder = mViewInfoStore.getFromOldChangeHolders(key);
                
                // 这部分逻辑不知道什么时候触发，但是应该是和Step1中那部分Change逻辑是对应的，
                if (oldChangeViewHolder != null && !oldChangeViewHolder.shouldIgnore()) {
                    // run a change animation

                    // If an Item is CHANGED but the updated version is disappearing, it creates
                    // a conflicting case.
                    // Since a view that is marked as disappearing is likely to be going out of
                    // bounds, we run a change animation. Both views will be cleaned automatically
                    // once their animations finish.
                    // On the other hand, if it is the same view holder instance, we run a
                    // disappearing animation instead because we are not going to rebind the updated
                    // VH unless it is enforced by the layout manager.
                    final boolean oldDisappearing = mViewInfoStore.isDisappearing(
                            oldChangeViewHolder);
                    final boolean newDisappearing = mViewInfoStore.isDisappearing(holder);
                    if (oldDisappearing && oldChangeViewHolder == holder) {
                        // run disappear animation instead of change
                        mViewInfoStore.addToPostLayout(holder, animationInfo);
                    } else {
                        final ItemHolderInfo preInfo = mViewInfoStore.popFromPreLayout(
                                oldChangeViewHolder);
                        // we add and remove so that any post info is merged.
                        mViewInfoStore.addToPostLayout(holder, animationInfo);
                        ItemHolderInfo postInfo = mViewInfoStore.popFromPostLayout(holder);
                        if (preInfo == null) {
                            handleMissingPreInfoForChangeError(key, holder, oldChangeViewHolder);
                        } else {
                            animateChange(oldChangeViewHolder, holder, preInfo, postInfo,
                                    oldDisappearing, newDisappearing);
                        }
                    }
                } else {
                    // 将animationInfo和对应的ViewHolder添加到mViewInfoStore中，同时打上POST flag。
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                }
            }

            // Step 4: Process view info lists and trigger animations
            // 执行动画
            mViewInfoStore.process(mViewInfoProcessCallback);
        }
			
		// 重置一堆状态。
        mLayout.removeAndRecycleScrapInt(mRecycler);
        mState.mPreviousLayoutItemCount = mState.mItemCount;
        mDataSetHasChangedAfterLayout = false;
        mDispatchItemsChangedEvent = false;
        mState.mRunSimpleAnimations = false;

        mState.mRunPredictiveAnimations = false;
        mLayout.mRequestedSimpleAnimations = false;
        if (mRecycler.mChangedScrap != null) {
            mRecycler.mChangedScrap.clear();
        }
        if (mLayout.mPrefetchMaxObservedInInitialPrefetch) {
            // Initial prefetch has expanded cache, so reset until next prefetch.
            // This prevents initial prefetches from expanding the cache permanently.
            mLayout.mPrefetchMaxCountObserved = 0;
            mLayout.mPrefetchMaxObservedInInitialPrefetch = false;
            mRecycler.updateViewCacheSize();
        }

        mLayout.onLayoutCompleted(mState);
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
        // 清除mViewInfoStore中的数据。
        mViewInfoStore.clear();
        if (didChildRangeChange(mMinMaxLayoutPositions[0], mMinMaxLayoutPositions[1])) {
            dispatchOnScrolled(0, 0);
        }
        recoverFocusFromState();
        resetFocusInfo();
    }

```

在LayoutStep3中首先遍历子View，然后将子view的animationInfo添加到POST中，最后执行动画。但是RecyclerView是怎么判断该执行那种动画呢？在Step1中是将上次Layout的子View添加PRE flag，在Step3中是添加的POST flag。如果在Step1和Step3中都有的view，就会同时添这两种flag（通过与运算），如果只有在Step1有的View则只有PRE flag，反之Step3中就只有POST flag。所以只有PRE的就该执行Disappear动画，只有POST就该执行Add动画，都有的话就可能是move动画或者change动画。 





