# Android中的触摸事件——MotionEvent中的多点触控

## 前言
  
  触摸事件在Android手机中有多重要不言而喻，用户的每次操作都和它有关。不知道大家有没有见过一些自定义的控件有这样的问题：当单手操作的时候，没有问题，但是当第二根手指头放上去的时候，该View中的内容就会“跳动一下”，如果有这样问题的控件，就是没有处理多点触控。该篇内容主要讲解MotionEvent对象中的多点触控信息，以及RecyclerView对于这种多点触控是如何处理的。MotionEvent又是啥呢？用来记录用户在屏幕中的触摸信息的对象。假如你点了一下屏幕（假如你手没抖，而且速度很快），这个时候就会产生两个MotionEvent对象（UP和DOWM）。 
  
  
## 正文
 一次完整的触摸事件流（官方叫gesture）至少包括ACTION\_DOWN和ACTION\_UP两个Event（手指触摸的情况，不考虑使用鼠标的情况，本文后面所描述的所有情况都是手指触摸的情况）。前面提到了Event的ACTION，我们一般有两个方法可以取到ACTION，一个是`getAction()`，还有一个就是`getActionMasked()`。这两个方法有什么区别呢？action中包含了Pointer的Index，而actionMasked中不包含这个Index。那什么又是Pointer呢？这就涉及到该篇内容要重点讨论的多点触控。可以理解成每一个触控点，通俗点讲就是你放在屏幕上的手指头。我们一般都用ActionMasked这个方法来拿这些Action。下面就简单来说明一下这些常见的Event。  
 
 1. ACTION\_DOWN
   
   第一根手指头触摸到屏幕（之前屏幕上没有手指头），一次事件触摸流的开始，很简单，但是很重要，这里也要简单的提一下，在ViewGroup中也是根据这次事件的坐标来决定该次事件流交给谁来处理，直到这次事件流完成（ACTION_UP）。
   
 2. ACTION\_MOVE
  
  就是你的手指头在屏幕上滑动，就会产生这个事件。
  
 3. ACTION\_UP

  最后一根手指头离开屏幕（屏幕上没有手指头了），标志着该次事件流已经完成。
  
 4. ACTION\_CANCEL
 
  这次事件流被取消了，虽然还没有完成，一般是ViewGroup经过某种条件判断会设置这样的ACTION。
  
 5. ACTION\_POINTER\_DOWN
 
  当屏幕上已经有手指头的时候，再按一个手指头下去就会触发这个事件。
 
 6. ACTION\_POINTER\_UP
  
  当手指头离开屏幕，同时屏幕上还有手指头的时候就会触发这个事件。  
  
  
如果你的自定义控件处理好了上面的6种ACTION，那么你的控件对触摸的处理就很好了，因为RecyclerView就只是处理了这6种Event。
 
  
在一个MotionEvent对象中，包含了你在屏幕上所有的触摸点信息，他默认会有一个类似于active的触摸点，可以通过方法`getActionIndex()`拿到这个触摸点的Index，然后再通过方法`getPointerId()`能拿到这个触摸点的Id，Id通过`findPointerIndex()`，能再拿到这个Index。这里需要注意的是在一次事件流中，同一个触摸点的Index是可能发生改变的，但是Id是不会改变的。在方法`getX()`和`getY()`中都可以传一个Index来拿你想要的触摸点的坐标，不止这两个方法可以传入index，其他的读者自己去研究了。下面我们来讨论下不同情况下active的默认触摸点都是哪些点呢？

1. ACTION\_DOWN, ACTION\_MOVE, ACTION\_UP
  
  这些ACTIONs默认的active触摸点Index都是0，也就是说这些事件如果你的初次点击屏幕的手指头没有离开屏幕，那就一直是这个点，如果这个手指头已经离开屏幕，那这个点就变成了第二个点击屏幕的点，依次类推。  
  
  
2. ACTION\_POINTER\_DOWN  

 默认active触摸点是新点击到屏幕上的那个点，但是在后续的move中这个默认index又回变成0。这里也可以简单解释下文章开篇中提到的“跳一下”的Bug：因为在第二个手指头点击屏幕的瞬间，active的触摸点为第二个手指头，这个时候默认的坐标也是第二个指头，后续的move事件中默认的active触摸点又会变成第一个手指头，所以会出现跳一下的Bug。


3. ACTION\_POINTER\_UP  

 这个Event和ACTION\_POINTER\_DOWN类似，只是默认的active的Index变成了离开屏幕的那个触摸点。  
 
 
 
下面我们来看看RecyclerView中是如何来处理多点触控的。


```java

    @Override
    public boolean onTouchEvent(MotionEvent e) {
    
    	 final MotionEvent vtev = MotionEvent.obtain(e);
        final int action = e.getActionMasked();
        // 默认active的Index
        final int actionIndex = e.getActionIndex();
        
        
        switch (action) {
            case MotionEvent.ACTION_DOWN: {
            	   // 直接取第一个Pointer为自己要处理的，将它的Id保存到成员变量中
                mScrollPointerId = e.getPointerId(0);
                
                // 记录初始化的坐标
                mInitialTouchX = mLastTouchX = (int) (e.getX() + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY() + 0.5f);
            } break;

            case MotionEvent.ACTION_POINTER_DOWN: {
            	   // 当第二个手指头（也可能是第n个手指头）点击屏幕的时候，就会用这个手指头来接管这次事件流。
                mScrollPointerId = e.getPointerId(actionIndex);
                mInitialTouchX = mLastTouchX = (int) (e.getX(actionIndex) + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY(actionIndex) + 0.5f);
            } break;

            case MotionEvent.ACTION_MOVE: {
                // 通过pointerID拿到需要处理的Index。
                final int index = e.findPointerIndex(mScrollPointerId);
                if (index < 0) {
                    Log.e(TAG, "Error processing scroll; pointer index for id "
                            + mScrollPointerId + " not found. Did any MotionEvents get skipped?");
                    return false;
                }
					
					// 获取需要处理的点的坐标。
                final int x = (int) (e.getX(index) + 0.5f);
                final int y = (int) (e.getY(index) + 0.5f);
                
            } break;

            case MotionEvent.ACTION_POINTER_UP: {
                onPointerUp(e);
            } break;

            case MotionEvent.ACTION_UP: {
                resetTouch();
            } break;

            case MotionEvent.ACTION_CANCEL: {
                cancelTouch();
            } break;
        }
    
    
    }
    
    // 处理pointer up的情况。
    private void onPointerUp(MotionEvent e) {
        // 离开屏幕的触摸点index
        final int actionIndex = e.getActionIndex();
        
        // 如果离开的那个点的id正好是我们接管触摸的那个那个点，那么我们就需要重新再找一个pointer来接管，反之不用管。
        if (e.getPointerId(actionIndex) == mScrollPointerId) {
            // Pick a new pointer to pick up the slack.
            // 如果离开屏幕的点index是0，那就用index 为 1 的点，反之就直接用0。
            final int newIndex = actionIndex == 0 ? 1 : 0;
            
            // 重置id和初始化initX和initY。
            mScrollPointerId = e.getPointerId(newIndex);
            mInitialTouchX = mLastTouchX = (int) (e.getX(newIndex) + 0.5f);
            mInitialTouchY = mLastTouchY = (int) (e.getY(newIndex) + 0.5f);
        }
    }

```  

这里简单说明下：一个手指到滑动就不说了，当屏幕上有新的手指加入时（之前屏幕上已经有手指头了），这个新加入的手指就会接管RecyclerView的滑动。当有手指头离开屏幕时（屏幕上还有其他的手指头），这个时候如果离开的手指头刚好时接管滑动的那个手指头，这个时候就会找index为0或着1的手指头重新接管滑动；如果离开的不是接管滑动的手指头，就不用管。


<br>
<br>
<br>


到这里多点触控相关的内容就没了，如果有错误的地方，欢迎大家指出来。后面可能还会有触摸事件的发送和滚动相关内容。  