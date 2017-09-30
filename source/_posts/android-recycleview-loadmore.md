---
title: recycleview上拉刷新下拉加载更多封装
date: 2017-09-30 14:42:25
categories: android
tags: 
    android
    java
---
### 效果图   
![dd.gif](http://upload-images.jianshu.io/upload_images/2898841-86a94254fbe1f685.gif?imageMogr2/auto-orient/strip)
### 说明
1.本demo使用的数据,均由gank.io提供   
2.下拉刷新使用的是SwipeRefreshLayout   
3.上拉加载更多使用的是BRVAH提供的BaseRecyclerViewAdapterHelper:2.1.3

### 抽取接口
```java
public interface IPagingService<T extends List> {
    /**
     * 加载分页数据
     * @param page 加载第几页
     * @param limit 1页加载多少条
     */
    void getData(int page,int limit, Observer<T> observer);
}
```


### 编写基类(实现分页逻辑)
```java
public class BasePagingActivity<T> extends AppCompatActivity implements SwipeRefreshLayout.OnRefreshListener, BaseQuickAdapter.RequestLoadMoreListener {
    private static final int PAGE_SIZE = 20;
    private RecyclerView mRecyclerView;
    private BaseQuickAdapter mQuickAdapter;
    private IPagingService<List<T>> mPagingService;
    private SwipeRefreshLayout mSwipeRefreshLayout;
    private int currentPage;
    private int lastPage;

    private void setSwipeRefreshLayout(SwipeRefreshLayout swipeRefreshLayout) {
        if (swipeRefreshLayout != null) {
            mSwipeRefreshLayout = swipeRefreshLayout;
            mSwipeRefreshLayout.setProgressBackgroundColorSchemeResource(android.R.color.white);
            mSwipeRefreshLayout.setProgressViewOffset(false, 0, (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 24, getResources().getDisplayMetrics()));
            mSwipeRefreshLayout.setColorSchemeResources(android.R.color.holo_blue_light, android.R.color.holo_red_light, android.R.color.holo_orange_light, android.R.color.holo_green_light);
            swipeRefreshLayout.setOnRefreshListener(this);
        } else {
            throw new NullPointerException("swipeRefreshLayout not null");
        }
    }

    private void setRecyclerView(RecyclerView recyclerView) {
        mRecyclerView = recyclerView;
        if (mRecyclerView.getLayoutManager() == null) {
            mRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        }
    }

    private void setQuickAdapter(BaseQuickAdapter quickAdapter) {
        if (quickAdapter != null) {
            mQuickAdapter = quickAdapter;
            mQuickAdapter.openLoadAnimation();
            mQuickAdapter.openLoadMore(PAGE_SIZE);
            mQuickAdapter.setOnLoadMoreListener(this);
            mRecyclerView.setAdapter(quickAdapter);
        } else {
            throw new NullPointerException("swipeRefreshLayout not null");
        }
    }

    /**
    * 开始获取数据,提供给子类调用
    */
    protected void startGetData(RecyclerView recyclerView,SwipeRefreshLayout swipeRefreshLayout,BaseQuickAdapter quickAdapter, IPagingService<List<T>> pagingService){
        mPagingService = pagingService;
        setRecyclerView(recyclerView);
        setSwipeRefreshLayout(swipeRefreshLayout);
        setQuickAdapter(quickAdapter);
        onLoadFirstData();
    }

    @Override
    public void onRefresh() {
        currentPage = 1;
        mPagingService.getData(currentPage, PAGE_SIZE, new Observer<List<T>>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {
                Toast.makeText(BasePagingActivity.this,e.getMessage(),Toast.LENGTH_SHORT).show();
                mSwipeRefreshLayout.setRefreshing(false);
                currentPage = lastPage;
            }

            @Override
            public void onNext(List<T> list) {
                if (list == null) return;
                mQuickAdapter.getData().clear();
                mQuickAdapter.addData(list);
                mQuickAdapter.notifyDataSetChanged();
                mSwipeRefreshLayout.setRefreshing(false);
            }
        });
    }

    @Override
    public void onLoadMoreRequested() {
        lastPage = currentPage;
        currentPage++;
        mPagingService.getData(currentPage, PAGE_SIZE, new Observer<List<T>>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {
                Toast.makeText(BasePagingActivity.this,e.getMessage(),Toast.LENGTH_SHORT).show();
                currentPage = lastPage;
            }

            @Override
            public void onNext(List<T> list) {
                if ((list != null && list.isEmpty())) {
                    Toast.makeText(BasePagingActivity.this,"没有更多数据了",Toast.LENGTH_SHORT).show();
                    mQuickAdapter.addData(list);
                    mQuickAdapter.loadComplete();
                } else {
                    mQuickAdapter.addData(list);
                }
                lastPage = currentPage;
            }
        });
    }

    public void onLoadFirstData(){
        lastPage = currentPage = 1;
        mSwipeRefreshLayout.setRefreshing(true);
        mPagingService.getData(currentPage, PAGE_SIZE, new Observer<List<T>>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {
                Toast.makeText(BasePagingActivity.this,e.getMessage(),Toast.LENGTH_SHORT).show();
                mSwipeRefreshLayout.setRefreshing(false);
            }

            @Override
            public void onNext(List<T> list) {
                if (list == null) return;
                mQuickAdapter.addData(list);
                mQuickAdapter.notifyDataSetChanged();
                mSwipeRefreshLayout.setRefreshing(false);
            }
        });
    }
}
```
`fragment同样可以这样做`
### 使用   
#### 1.实现IPagingService

```java
    public class WelfareServer implements IPagingService<List<WelfareEntity>>{
        @Override
        public void getData(int page, int limit, Observer<List<WelfareEntity>> observer) {
            RetrofitManager.getInstance().createReq(GankIo.class)
                    .getWelfareImg(limit, page)
                    .subscribeOn(Schedulers.io())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(observer);
        }
    }
```

### 2.继承BasePagingActivity并调用startGetData方法
```java
public class MainActivity extends BasePagingActivity<WelfareEntity> {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        SwipeRefreshLayout mSwipeRefreshLayout = (SwipeRefreshLayout) findViewById(R.id.refresh_layout);
        RecyclerView mRecyclerView = (RecyclerView) findViewById(R.id.list);
        startGetData(mRecyclerView, mSwipeRefreshLayout, new BaseQuickAdapter<WelfareEntity>(R.layout.item_welfare,new ArrayList()){
            @Override
            protected void convert(BaseViewHolder baseViewHolder, WelfareEntity welfareEntity) {
                Glide.with(MainActivity.this)
                        .load(welfareEntity.getUrl())
                        .placeholder(R.mipmap.load_image_bg)
                        .into((ImageView) baseViewHolder.getView(R.id.iv));
            }
        },new WelfareServer());

    }
}
```

最后附上demo地址
github : https://github.com/aii1991/LoadDataDemo




