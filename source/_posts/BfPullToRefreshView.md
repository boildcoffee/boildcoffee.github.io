---
title: 自定义上拉刷新下拉加载更多控件
date: 2019-03-12 15:41:22
categories: android
tags: 
    android
    java
---
# BfPullToRefreshView

## 效果图

![xiaoguo.gif](https://upload-images.jianshu.io/upload_images/2898841-fbe933f94c96d60f.gif?imageMogr2/auto-orient/strip)

## 使用

布局文件

```xml
<com.boildcoffee.library.widget.BfPullToRefreshView
        android:id="@+id/bfprv"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</com.boildcoffee.library.widget.BfPullToRefreshView>
```

Activity代码

```java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_recycle_view);

        mPullToRefreshView = findViewById(R.id.bfprv);
        mRecyclerView = findViewById(R.id.rv);

        mRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        mBaseQuickAdapter = new BaseQuickAdapter<String,BaseViewHolder>(R.layout.item,DataUtils.generateData(0,20)) {
            @Override
            protected void convert(BaseViewHolder helper, String item) {
                helper.setText(R.id.tv,item);
            }
        };
        mRecyclerView.setAdapter(mBaseQuickAdapter);
        mPullToRefreshView.setRefreshListener(new OnRefreshListener() { //下拉刷新
            @Override
            public void refreshListener(View view) {
                mPullToRefreshView.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        mBaseQuickAdapter.setNewData(DataUtils.generateData(0,20));
                        mPullToRefreshView.setRefreshComplete();
                    }
                },3000);
            }
        });
        mPullToRefreshView.setLoadMoreListener(new OnLoadMoreListener() { //上拉加载更多
            @Override
            public void loadMoreListener(View v) {
                mPullToRefreshView.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        mBaseQuickAdapter.addData(DataUtils.generateData(20,40));
                        mPullToRefreshView.setLoadMoreComplete();
                    }
                },3000);
            }
        });
    }

```

## 实现原理

![yuanli.png](https://upload-images.jianshu.io/upload_images/2898841-01dc7abbc8220a66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`实现原理比较简单,BfPullToRefreshView由三部分组成,屏幕外的BfRefreshView,屏幕内的ListView/RecycleView等,屏幕外的BfLoadMoreView。当ListView/RecycleView滑动到顶部或者底部时拦截事件交由BfPullToRefreshView的处理,通过滚动BfPullToRefreshView来显示BfRefreshView/BfLoadMoreView`

## 实现代码

### 定义接口

- 要实现下拉刷新上拉加载更多必然要提供2个回调,让外部处理下拉刷新和上拉加载更多。所以我们可以先定义这2个接口,如下:

```java
  public interface OnRefreshListener {
      void refreshListener(View view);
  }

  public interface OnLoadMoreListener {
    void loadMoreListener(View v);
  }
```

- 我们要将上拉刷新/下拉加载更多的各个状态提供给BfRefreshView/BfLoadMoreView做出相应的操作,所以我们应提供以下接口

```java
public interface IRefreshView {

    void releaseToRefresh();

    void refreshing();

    void refreshComplete();

    /**
     *  rate = 下拉距离/RealView.height
     * @param rate
     */
    void getPercentRage(float rate);

    /**
     * 获取真正的刷新view
     * @return
     */
    View getRealView();
}

```

```java
  public interface ILoadMoreView {
      void preToLoadMore();

      void releaseToLoadMore();

      void loading();

      void loadComplete();

      /**
      *  rate = 下拉距离/RealView.height
      * @param rate
      */
      void getPercentage(float rate);

      /**
      * 获取真正的刷新view
      * @return
      */
      View getRealView();
  }
```

## 基类BasePullToRefreshView主要代码

- 此处需要了解View的事件分发机制,这里简单介绍下,dispatchTouchEvent负责事件分发,当有touch事件到来时,首先调用dispatchTouchEvent进行事件分发,然后由onInterceptTouchEvent判断是否拦截事件,若拦截后续的事件将交由onTouchEvent处理,若不拦截则事件将会继续向下传递

```java
  private final static float SCROLL_RATIO = 0.6f; //阻尼系数

  private final static int STATE_INIT = -1;
  private final static int STATE_PULL_UP = 0; //上拉状态
  private final static int STATE_PULL_DOWN = 1; //下拉状态

  private final static int STATE_PRE_TO_REFRESH = 2; //准备开始刷新状态
  private final static int STATE_RELEASE_TO_REFRESH = 3; //释放刷新
  private final static int STATE_REFRESHING = 4; //正在刷新中

  private final static int STATE_PRE_TO_LOAD_MORE = 5; //准备开始加载更多
  private final static int STATE_RELEASE_TO_LOAD_MORE = 6; //释放加载更多
  private final static int STATE_LOADING = 7; //正在加载更多

  protected T mContentView; //中间的内容view ListView/RecycleView等
  private IRefreshView mRefreshView;
  private View mRealRefreshView;
  private View mRealLoadMoreView;
  private ILoadMoreView mLoadMoreView;
  private OnLoadMoreListener mLoadMoreListener;
  private OnRefreshListener mRefreshListener;

  private int mPullState = STATE_INIT;
  private int mRefreshState = STATE_INIT;
  private int mLoadMoreState = STATE_INIT;
  private boolean openLoadMore;
  private boolean openPullToRefresh;

  private int mLastY;

  private Scroller mScroller;

  /**
  * 设置刷新View
  * @param refreshView
  */
  public void setRefreshView(IRefreshView refreshView){
        mRefreshView = refreshView;
        mRealRefreshView = mRefreshView.getRealView();
        openPullToRefresh = true;
        initContentView();

        initLayoutParams(mRealRefreshView);

        addView(mRealRefreshView,0);
        post(new Runnable() {
            @Override
            public void run() {
               hideRefreshView();//设置margin-top=-mRealRefreshView.height让其移除屏幕
            }
        });
    }

    /**
     * 设置加载更多View
     * @param loadMoreView
     */
    public void setLoadMoreView(ILoadMoreView loadMoreView){
        mLoadMoreView = loadMoreView;
        mRealLoadMoreView = loadMoreView.getRealView();
        openLoadMore = true;

        initContentView();

        initLayoutParams(mRealLoadMoreView);

        addView(mRealLoadMoreView);
    }

     /**
     * 设置刷新完成
     */
    public void setRefreshComplete(){
        resetState();
        resetScrollPosition();
        mRefreshView.refreshComplete();
    }

    /**
     * 设置加载完成
     */
    public void setLoadMoreComplete(){
        resetState();
        resetScrollPosition();
        mLoadMoreView.loadComplete();
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mPullState == STATE_REFRESHING || mLoadMoreState == STATE_LOADING){ //当处于正在刷新或者正在加载更多时,不对事件进行分发。
            return true;
        }
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        switch (ev.getAction()){
            case MotionEvent.ACTION_DOWN:
                mLastY = (int) ev.getY();
                break;
            case MotionEvent.ACTION_MOVE:
                // deltaY > 0 是向下运动,< 0是向上运动
                int deltaY = (int) (ev.getY() - mLastY);
                Log.d(TAG,"deltaY=>"+deltaY);
                if (deltaY >= -20 && deltaY <= 20){ //滚动距离过小，不拦截
                    return false;
                }
                setPullState(deltaY); //设置当前处于上拉还是下拉状态
                if (isScrollToTopOrBottom()){ //判断当前是否滑动到顶部或者底部,是则拦截
                    return true;
                }
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                break;
        }
        return super.onInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()){
            case MotionEvent.ACTION_DOWN:
                mLastY = (int) event.getY();
                break;
            case MotionEvent.ACTION_MOVE:
                int deltaY = (int) (event.getY() - mLastY);
                deltaY = (int)(deltaY * SCROLL_RATIO);
                int scrollY = getScrollY();
                Log.d(TAG,"getY()="+getScrollY());
                if (openPullToRefresh && mPullState == STATE_PULL_DOWN){
                    prepareToRefresh(deltaY,scrollY,event); //准备刷新
                }
                if (openLoadMore && mPullState == STATE_PULL_UP){
                    prepareToLoadMore(deltaY,scrollY,event); //准备加载更多
                }
                mLastY = (int) event.getY();
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                if ((openPullToRefresh && mRefreshState == STATE_PRE_TO_REFRESH) || (openLoadMore && mLoadMoreState == STATE_PRE_TO_LOAD_MORE)){
                    //刷新View/加载更多View未完全显示出来
                    resetScrollPosition(); //复位
                }
                if (openPullToRefresh && mPullState == STATE_PULL_DOWN && mRefreshState == STATE_RELEASE_TO_REFRESH){
                    handleRefreshing(); // 处理正在刷新
                }
                if (openLoadMore && mPullState == STATE_PULL_UP && mLoadMoreState == STATE_RELEASE_TO_LOAD_MORE){
                    handleLoading(); // 处理正在加载更多
                }
                break;
        }
        return super.onTouchEvent(event);
    }

    private void prepareToRefresh(int deltaY,int scrollY) {
        if (scrollY > 0){ //scrollY>0证明刷新view出现了并且是在向上滑动
            mRefreshState = STATE_PRE_TO_REFRESH;
            mContentView.onTouchEvent(event);
            return;
        }else {
            scrollY = -scrollY;
        }
        final int height = mRealRefreshView.getHeight();
        if (scrollY >= height){
            Log.d(TAG,"STATE_RELEASE_TO_REFRESH");
            //刷新View完全出现
            mRefreshState = STATE_RELEASE_TO_REFRESH;
            mRefreshView.releaseToRefresh();
            mRefreshView.getPercentRage(1.0f);
        }else {
            mRefreshState = STATE_PRE_TO_REFRESH;
            float rate = scrollY / (height * 1.0f);
            mRefreshView.getPercentRage(rate);
        }
        scrollBy(0,-deltaY); //滚动内容
    }

    private void prepareToLoadMore(int deltaY,int scrollY) {
        if (scrollY < 0){ //scrollY>0证明当前是在像下滑动
            mLoadMoreState = STATE_PRE_TO_LOAD_MORE;
            return;
        }
        final int height = mRealLoadMoreView.getHeight();
        if (scrollY >= height){ //加载View完全出现
            Log.d(TAG,"STATE_RELEASE_TO_LOAD_MORE");
            mLoadMoreState = STATE_RELEASE_TO_LOAD_MORE;
            mLoadMoreView.releaseToLoadMore();
            mLoadMoreView.getPercentage(1.0f);
        }else {
            mLoadMoreState = STATE_PRE_TO_LOAD_MORE;
            float rate = scrollY / (height * 1.0f);
            mLoadMoreView.getPercentage(rate);
            mLoadMoreView.preToLoadMore();
        }
        scrollBy(0,-deltaY); //滚动内容
    }

    private void resetScrollPosition(){
        int sy = getScrollY();
        Log.d(TAG,"sy=>"+sy);
        mScroller.startScroll(0,sy,0,-sy,800);
        invalidate();
    }

    private void handleRefreshing() {
        Log.d(TAG,"refreshing");
        mRefreshState = STATE_REFRESHING;
        scrollToRefreshingPosition(); //使用scroller实现弹性回滚
        mRefreshView.refreshing();
        if (mRefreshListener != null){
            mRefreshListener.refreshListener(this);
        }
    }

    private void handleLoadMore() {
        Log.d(TAG,"Loading");
        mLoadMoreState = STATE_LOADING;
        scrollToLoadingPosition();
        mLoadMoreView.loading();
        if (mLoadMoreListener != null){
            mLoadMoreListener.loadMoreListener(this);
        }
    }

```

## BfPullToRefreshView代码

```java
  public class BfPullToRefreshView extends BasePullToRefreshView<View>{
    public BfPullToRefreshView(Context context) {
        super(context);
    }

    public BfPullToRefreshView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public BfPullToRefreshView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        BfRefreshView bfRefreshView = new BfRefreshView(getContext());
        bfRefreshView.setBackgroundColor(Color.BLUE);
        bfRefreshView.setLayoutParams(new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                (int)TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP,60,getResources().getDisplayMetrics())));
        setRefreshView(bfRefreshView);

        BfLoadMoreView loadMoreView = new BfLoadMoreView(getContext());
        loadMoreView.setLayoutParams(new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                (int)TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP,60,getResources().getDisplayMetrics())));
        setLoadMoreView(loadMoreView);
    }

    @Override
    protected View createContentView() {
        return getChildAt(0);
    }

  }

```

BfRefreshView和BfLoadMoreView的代码在这里就不贴出来了,有兴趣的话可以自行查看[github demo](https://github.com/aii1991/BfPullToRefreshView)
