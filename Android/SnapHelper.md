[让你明明白白的使用RecyclerView-SnapHelper详解](https://www.jianshu.com/p/e54db232df62)
定义一定的对齐规则，用于辅助`RecyclerView`在滚动时将`Item`对齐到某个位置。
这个类是一个抽象类，可以使用子类`LinearSnapHelper`， 使用方式也很简单，只需要创建对象之后调用`attachToRecyclerView()`附着到对应的RecyclerView对象上就可以了。
```java
new LinearSnapHelper().attachToRecyclerView(mRecyclerView);
//或者
new PagerSnapHelper().attachToRecyclerView(mRecyclerView);
```
# 原理剖析
## Fling
手指在屏幕上滑动`RecyclerView`然后松手，`RecyclerView`中的内容会顺着惯性继续往手指滑动的方向继续滚动直到停止。这个过程从手指离开屏幕被瞬间触发，在滚动停止时结束。
## 三个抽象方法
```java
public abstract View findSnapView(RecyclerView.LayoutManager layoutManager);
```
该方法会找到当前`LayoutManager`上最接近对齐位置的那个`view`，该`view`称为`SanpView`，对应的`position`称为`SnapPosition`。如果返回`Null`，就表示没有需要对齐的`View`，也就不会做滚动对齐调整。
```java
public abstract int findTargetSnapPosition(RecyclerView.LayoutManager layoutManager,  int velocityX,  
        int velocityY);
```
参数分别为`Fling`操作的速率。寻找到`RecyclerView`需要滚动到的位置。
```java
public abstract int[] calculateDistanceToFinalSnap(@NonNull RecyclerView.LayoutManager layoutManager,  
        @NonNull View targetView);
```
计算目标`View`相对于需要对齐的坐标之间的距离。
## 其他方法
### attachToRecyclerView()
```java
public void attachToRecyclerView(@Nullable RecyclerView recyclerView)  
        throws IllegalStateException {  

//如果已经附着，就不用进行任何操作
    if (mRecyclerView == recyclerView) {  
        return; // nothing to do  
    }  

//如果之前附着的RecyclerView和现在的不一致，就清理回调
    if (mRecyclerView != null) {  
        destroyCallbacks();  
    }  

//更新对象引用
    mRecyclerView = recyclerView;  
    if (mRecyclerView != null) {  

		//设置当前回调
        setupCallbacks();  
        mGravityScroller = new Scroller(mRecyclerView.getContext(),  
                new DecelerateInterpolator());  

		//实现滚动对齐处理
        snapToTargetExistingView();  
    }  
}
```
### snapToTargetExistingView()
```java
void snapToTargetExistingView() {  
    if (mRecyclerView == null) {  
        return;  
    }  
    RecyclerView.LayoutManager layoutManager = mRecyclerView.getLayoutManager();  
    if (layoutManager == null) {  
        return;  
    }  

//找到snapView
    View snapView = findSnapView(layoutManager);  
    if (snapView == null) {  
        return;  
    }  

//计算需要滚动的距离
    int[] snapDistance = calculateDistanceToFinalSnap(layoutManager, snapView);  

//执行
    if (snapDistance[0] != 0 || snapDistance[1] != 0) {  
        mRecyclerView.smoothScrollBy(snapDistance[0], snapDistance[1]);  
    }  
}
```
### setupCallbacks()和destroyCallbacks()
```java
private void setupCallbacks() throws IllegalStateException {  
    if (mRecyclerView.getOnFlingListener() != null) {  
        throw new IllegalStateException("An instance of OnFlingListener already set.");  
    }  
    mRecyclerView.addOnScrollListener(mScrollListener);  
    mRecyclerView.setOnFlingListener(this);  
}

private void destroyCallbacks() {  
    mRecyclerView.removeOnScrollListener(mScrollListener);  
    mRecyclerView.setOnFlingListener(null);  
}
```
#### `mScrollListener`
```java
private final RecyclerView.OnScrollListener mScrollListener =  
        new RecyclerView.OnScrollListener() {  
            boolean mScrolled = false;  
  
            @Override  
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {  
                super.onScrollStateChanged(recyclerView, newState);  
//滑动结束 && 之前进行过滚动
                if (newState == RecyclerView.SCROLL_STATE_IDLE && mScrolled) {  
                    mScrolled = false;  

					//滚动对齐处理
                    snapToTargetExistingView();  
                }  
            }  
  
            @Override  
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {  
                if (dx != 0 || dy != 0) {  
                    mScrolled = true;  
                }  
            }  
        };
```
#### `OnFlingListener`
接口
```java
public abstract static class OnFlingListener {  
  public abstract boolean onFling(int velocityX, int velocityY);  
}
```
具体实现
```java
public boolean onFling(int velocityX, int velocityY) {  
    RecyclerView.LayoutManager layoutManager = mRecyclerView.getLayoutManager();  
    if (layoutManager == null) {  
        return false;  
    }  
    RecyclerView.Adapter adapter = mRecyclerView.getAdapter();  
    if (adapter == null) {  
        return false;  
    }  

//获取最小速率
    int minFlingVelocity = mRecyclerView.getMinFlingVelocity();  
    return (Math.abs(velocityY) > minFlingVelocity || Math.abs(velocityX) > minFlingVelocity)  
    //该方法实现平滑滚动并使得在滚动停止的时候itemView对齐到目的地
            && snapFromFling(layoutManager, velocityX, velocityY);  
}
```
#### `snapFromFling()`
```java
private boolean snapFromFling(@NonNull RecyclerView.LayoutManager layoutManager, int velocityX,  
        int velocityY) {  

//必须实现接口
    if (!(layoutManager instanceof RecyclerView.SmoothScroller.ScrollVectorProvider)) {  
        return false;  
    }  

//创建平滑滚动器
    RecyclerView.SmoothScroller smoothScroller = createScroller(layoutManager);  
    if (smoothScroller == null) {  
        return false;  
    }  
  
    int targetPosition = findTargetSnapPosition(layoutManager, velocityX, velocityY);  
    if (targetPosition == RecyclerView.NO_POSITION) {  
        return false;  
    }  

//设置目标滚动地址
    smoothScroller.setTargetPosition(targetPosition);  
//开始滚动
    layoutManager.startSmoothScroll(smoothScroller);  
    return true;}
```
#### `createScroller()`
```java
    @Nullable
    protected LinearSmoothScroller createSnapScroller(LayoutManager layoutManager) {
    

//同样，这里也是先判断layoutManager是否实现了ScrollVectorProvider这个接口，
//没有实现该接口就不创建SmoothScroller
        if (!(layoutManager instanceof ScrollVectorProvider)) {
            return null;
        }
      
//这里创建一个LinearSmoothScroller对象，然后返回给调用函数，
//也就是说，最终创建出来的平滑滚动器就是这个LinearSmoothScroller
        return new LinearSmoothScroller(mRecyclerView.getContext()) {
          //该方法会在targetSnapView被layout出来的时候调用。
          //这个方法有三个参数：
          //第一个参数targetView，就是本文所讲的targetSnapView
          //第二个参数RecyclerView.State这里没用到，先不管它
          //第三个参数Action，这个是什么东西呢？它是SmoothScroller的一个静态内部类,
          //保存着SmoothScroller在平滑滚动过程中一些信息，比如滚动时间，滚动距离，差值器等
            @Override
            protected void onTargetFound(View targetView, RecyclerView.State state, Action action) {
             //calculateDistanceToFinalSnap（）方法上面解释过，
             //得到targetSnapView当前坐标到目的坐标之间的距离
                int[] snapDistances = calculateDistanceToFinalSnap(mRecyclerView.getLayoutManager(),
                        targetView);
                final int dx = snapDistances[0];
                final int dy = snapDistances[1];
              //通过calculateTimeForDeceleration（）方法得到做减速滚动所需的时间
                final int time = calculateTimeForDeceleration(Math.max(Math.abs(dx), Math.abs(dy)));
                if (time > 0) {
                  //调用Action的update()方法，更新SmoothScroller的滚动速率，使其减速滚动到停止
                  //这里的这样做的效果是，此SmoothScroller用time这么长的时间以mDecelerateInterpolator这个差值器的滚动变化率滚动dx或者dy这么长的距离
                    action.update(dx, dy, time, mDecelerateInterpolator);
                }
            }

//该方法是计算滚动速率的，返回值代表滚动速率，该值会影响刚刚上面提到的
//calculateTimeForDeceleration（）的方法的返回返回值，
//MILLISECONDS_PER_INCH的值是100，也就是说该方法的返回值代表着每dpi的距离要滚动100毫秒
            @Override
            protected float calculateSpeedPerPixel(DisplayMetrics displayMetrics) {
                return MILLISECONDS_PER_INCH / displayMetrics.densityDpi;
            }
        };
    }
```