# ViewPager2

## ViewPager预加载机制

默认情况下，每当ViewPager跳转到一个Item时，它会默认加载左右两边的页面，尽管两边的View不可见，这种情况就叫做预加载。ViewPager中的预加载页面数量和`OffscreenPageLimit` 这个值有关。

`OffscreenPageLimit` 是滑动视图中加载的页面数量。比如`OffscreenPageLimit=1` 且水平滑动时，表示ViewPager在当前可见页面外，左右各加载一个页面。

而ViewPager一个让人诟病的点是`OffscreenPageLimit ` 默认设置为-1，所以即使手动调用方法`setOffscreenPageLimit(0) ` ,ViewPager依然会进行预加载。

```java
//默认离屏加载数量为1
private static final int DEFAULT_OFFSCREEN_PAGES = 1;
public void setOffscreenPageLimit(int limit) {
    //如果设置为0
        if (limit < DEFAULT_OFFSCREEN_PAGES) {
            Log.w(TAG, "Requested offscreen page limit " + limit + " too small; defaulting to "
                    + DEFAULT_OFFSCREEN_PAGES);
            limit = DEFAULT_OFFSCREEN_PAGES;
        }
        if (limit != mOffscreenPageLimit) {
            mOffscreenPageLimit = limit;
            populate();//关于适配器Adapter的代码
        }
    }
```

## ViewPager2

相比于ViewPager，ViewPager2有以下改变：

| ViewPager                                      | ViewPager2                                                   |
| ---------------------------------------------- | ------------------------------------------------------------ |
| PagerAdapter                                   | RecyclerView.Adapter                                         |
| FragmentStatePagerAdapter/FragmentPagerAdapter | FragmentStateAdapter                                         |
| 无                                             | 支持垂直滚动`android:orientation="vaertical"` `setOrientation(ViewPager2.ORIENTATION_VERTICAL)` |
| 无                                             | 支持RTL(right to left)`android:layoutDirection="rtl"`        |
| 无                                             | 添加了停止用户输入的功能`(setUserInputEnabled/isUserInputEnabled)` |

>  ViewPager2是对RecycleView的二次封装，这意味着其适配器直接由RecycleView.Adapter替代，而且可以支持RecycleView的缓存机制和预拉取机制，同时支持DiffUtil进行局部刷新；

### Adapter

```java
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {
    String[] colors = {"#ccff99", "#41f1e5", "#8d41f1"};

    @NonNull
    @Override
    public MyViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
            .inflate(R.layout.activity_main, parent, false);
        return new MyViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull MyViewHolder holder, int position) {
        holder.bind(colors[position]);
    }

    @Override
    public int getItemCount() {
        return colors.length;
    }

    class MyViewHolder extends RecyclerView.ViewHolder {
        private TextView mTextView;

        public MyViewHolder(@NonNull View itemView) {
            super(itemView);
            mTextView = itemView.findViewById(R.id.text);
        }

        private void bind(String color) {
            mTextView.setBackgroundColor(Color.parseColor(color));
        }
    }
}
```

### 默认关闭预加载

查看相应`setOffscreenPageLimit()` 方法：

```java
//默认为-1，不进行预加载
public static final int OFFSCREEN_PAGE_LIMIT_DEFAULT = -1;
public void setOffscreenPageLimit(@OffscreenPageLimit int limit) {
        if (limit < 1 && limit != OFFSCREEN_PAGE_LIMIT_DEFAULT) {
            throw new IllegalArgumentException(
                    "Offscreen page limit must be OFFSCREEN_PAGE_LIMIT_DEFAULT or a number > 0");
        }
        mOffscreenPageLimit = limit;
        // Trigger layout so prefetch happens through getExtraLayoutSize()
        mRecyclerView.requestLayout();
    }

```

默认不进行预加载时，ViewPager2会调用RecycleView的缓存机制和预拉取机制。

注：ViewPager2默认在mCachedView中缓存2个remove的item和1个预拉取的item。

刚开始进入页面时没有触发预拉取操作；

![](D:\wuxiang\viewpager1.png)

当从页面0滑动到页面1时，预拉取机制拉取position=2的页面，同时position=0的页面进入二级缓存；

![](D:\wuxiang\viewpager2.png)

当从页面1滑动到页面2时，滑动视图会取出先前预拉取的页面2进行使用，同时开启对position=3的页面进行拉取，页面1进入二级缓存；

![](D:\wuxiang\viewpager3.png)

当从页面2滑动到页面3时，同理回对position=4的页面进行预拉取，这时由于超出了mCachedView的范围，则会将position=0的页面取出来放入四级缓存中的ArrayList容器中，然后position=2才能顺利进入容器中。

### 支持DiffUtil增量更新

传统更新数据方法使用`notifyDataSetChanged()` 它有以下缺陷：

1. 强制调用Adapter对每个ItemView进行重新绘制，从而实现动态刷新；
2. 不会触发RecyclerView的动画（插入，删除，移位）。

DiffUtil可以在刷新RecyclerView时，计算新老数据集的差异，自动调用刷新方法，并且带有动画。

具体操作

#### 计算最小更新数

将新数据传递给Adapter之前，首先调用DiffUtil.calculateDiff()方法，计算新旧数据间的最小更新数据，也就是DiffResult对象。

```java
public static DiffResult calculateDiff(@NonNull Callback cb, boolean detectMoves)
```

第一个参数是一个实现了DiffUtil.Callback接口的对象；

第二个参数是是否检测Item的移动。如果设置为true，DiffUtil会检测数据的移动操作，生成移动操作的更新列表。但是会增加计算量，影响性能。

```java
public abstract static class Callback {
    //旧数据集的大小
        public abstract int getOldListSize();
    
	//新数据集的大小
        public abstract int getNewListSize();
    
	//通过对象下标，判断两个对象是否相同
        public abstract boolean areItemsTheSame(int oldItemPosition, int newItemPosition);
    
	//用来检查两个对象是否有相同的数据
    //只在areItemsTheSame方法返回true的时候
        public abstract boolean areContentsTheSame(int oldItemPosition, int newItemPosition);
    
    //当前两个方法返回true的时候调用
    //标记两个对象不同的位置
    public Object getChangePayload(int oldItemPosition, int newItemPosition) {
            return null;
        }
}
```

如果areItemsTheSame()和areContentsTheSame()这两个方法都返回true，DiffUtil会认为这两个对象相同，不需要进行更新操作。

继承DiffUtil.Callback

```java
private static class MyCallback extends DiffUtil.Callback {
        private List<String> mOldList;
        private List<String> mNewList;

        public MyCallback(List<String> oldList, List<String> newList) {
            mOldList = oldList;
            mNewList = newList;
        }

        @Override
        public int getOldListSize() {
            return mOldList.size();
        }

        @Override
        public int getNewListSize() {
            return mNewList.size();
        }

        @Override
        public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
            return mOldList.get(oldItemPosition).equals(mNewList.get(newItemPosition));
        }

        @Override
        public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
            return mOldList.get(oldItemPosition).equals(mNewList.get(newItemPosition));
        }
    }
```

MainActivity

```java
DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(new MyCallBack(mDatas, newDatas), true);
```

#### 更新列表数据

利用DiffResult.dispatchUpdatesTo(RecyclerView.Adapter adapter)方法，传入adapter，实现数据更新。

```java
MyAdapter mMyAdapter=new MyAdapter();
//更新
diffResult.dispatchUpdatesTo(mMyAdapter);
//将数据传递给Adapter
mMyAdapter.setData(newDatas);
```

