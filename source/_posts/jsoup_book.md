# 采用jsoup爬取妹子图(www.mzitu.com)网站图片

## 效果图

![xiaogutu.gif](https://camo.githubusercontent.com/d8642ea7fcf7ad7fc138855a69016782b4d59653/68747470733a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f323839383834312d343663313164636562643233646665632e6769663f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970)

## 设计

![sheji.png](https://camo.githubusercontent.com/26bd867666eba972902fa322e9eb9e74702bca14/68747470733a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f323839383834312d373330616631353338396132303262392e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

## 优缺点

* 优点:无需自己搭建服务器
* 缺点:网页数据,无用数据过多,耗流量

## 优化

* 对html数据进行缓存,在一定时间内若缓存数据中存在则从缓存中获取

## 代码

1. 配置

```java
  public class MyApplication extends BaseApplication{
    public static final String BASE_URL = "http://www.mzitu.com/";

    @Override
    public void onCreate() {
        super.onCreate();
        BFConfig.INSTANCE.init(new BaseConfig.Builder()
                .setBaseUrl(BASE_URL)
                .setPageSize(24)
                .setApiQueryCacheMode(BaseConfig.CacheMode.CACHE_ELSE_NETWORK) //缓存策略,先从缓存再网络
                .setRspCacheTime(1000 * 60 * 60 * 24) //缓存时间
                .setConverter(DocumentConverter.create())
                .setDebug(true)
                .build()
        );
    }

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
    }
}

```

2. 视图对象

```java
/**
 *  图片类型,如: 最新 性感等
 */
public class Type {
    private String name; //类型名称
    private String url; //页面地址

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        Type type = (Type) o;

        if (name != null ? !name.equals(type.name) : type.name != null) return false;
        return url != null ? url.equals(type.url) : type.url == null;
    }

    @Override
    public int hashCode() {
        int result = name != null ? name.hashCode() : 0;
        result = 31 * result + (url != null ? url.hashCode() : 0);
        return result;
    }
}

```

```java
 private String title; //图册标题
    private String cover; //封面地址
    private String typeName;//所属类型名
    private String detailUrl;// 图册页面地址

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getCover() {
        return cover;
    }

    public void setCover(String cover) {
        this.cover = cover;
    }

    public String getTypeName() {
        return typeName;
    }

    public void setTypeName(String typeName) {
        this.typeName = typeName;
    }

    public String getDetailUrl() {
        return detailUrl;
    }

    public void setDetailUrl(String detailUrl) {
        this.detailUrl = detailUrl;
    }

```

```java
public class Image {
    private String url; //图片地址
    private String desc; //图片描述

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getDesc() {
        return desc;
    }

    public void setDesc(String desc) {
        this.desc = desc;
    }
}

```

3. 数据仓库(使用jsoup将html数据解析成vo对象交由viewmodel处理)

```java
public class ImageRepository {
    private ImageRepository(){}

    public static ImageRepository create(){
        return new ImageRepository();
    }

    private static final String DEFAULT_URL = "";
    private static List<Type> mTypeList;
    private static Map<String,String> mTypeMap;
    private static String mCurrentUrl;
    private Map<String,Integer> mTypeMaxPageMap;

    @BindingAdapter(value = {
            "loadWelfareImg"
    })
    public static void loadWelfareImg(ImageView v,String url){
        if (url == null) return;

        Map<String,String> headers = new HashMap<>();
        headers.put("referer",BFConfig.INSTANCE.getConfig().getBaseUrl() + mCurrentUrl); //由于妹子图采用了referer防止图片盗链,所以此处必须加入referer否则无法加载图片
        headers.put("user-Agent","Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.140 Safari/537.36");
        ImageLoaderManager.getInstance().loadImg(GlideApp.with(v.getContext()),url,headers,null,null)
                .into(v);
    }

    /**
     * 获取图片所属类型 如：最新 性感
     * @return
     */
    public Observable<List<Type>> getTypes(){
        return getHtmlData(DEFAULT_URL)
                .map(document -> {
                    if (mTypeList == null){
                        mTypeList = parseTypes(document);
                    }
                    return mTypeList;
                });
    }

    public Observable<List<String>> getDetailPageUrls(String url){
        mCurrentUrl = url;
        return getHtmlData(url)
                .map(this::parseDetailUrl);
    }

    /**
     * 获取详情大图 图片地址
     * @param pageUrl
     * @return
     */
    public Observable<Image> getImage(String pageUrl){
        mCurrentUrl = pageUrl;
        return getHtmlData(pageUrl)
                .map(this::parseImageUrl);
    }

    /**
     * 通过对应类型的url获取图册对象
     * @param url
     * @param currentPage
     * @return
     */
    public Observable<List<Atlas>> getAtlas(String url, int currentPage){
        mCurrentUrl = url;
        String value = currentPage  <= 1 ? url : url + "page/" + currentPage;
        return getHtmlData(value)
                .map(document -> parseAtlasData(document,currentPage));
    }

    /**
     * 获取html数据
     * @param url
     * @return
     */
    private Observable<Document> getHtmlData(String url) {
        return RetrofitManager
                .getInstance()
                .createReq(WelfareApi.class)
                .getData(url)
                .compose(TransformerHelper.observableToMainThreadTransformer());
    }

    /**
     * 解析对应类型的图册对象
     * @param document
     * @param currentPage
     * @return
     */
    private List<Atlas> parseAtlasData(Document document, int currentPage) {
        List<Atlas> atlasList = new ArrayList<>();
        if (mTypeList == null){
            mTypeList = parseTypes(document);
        }
        if (!hasPage(document,currentPage)){
            return atlasList;
        }
        Elements elements = document.select(".postlist > ul > li");
        for (Element element : elements){
            if (element.children().size() < 1){
                continue;
            }
            Element aElement = element.child(0);
            Element spanElement = element.child(1);
            Atlas atlas = new Atlas();
            atlas.setTypeName(mTypeMap.get(mCurrentUrl));
            atlas.setDetailUrl(parseUrl(spanElement.child(0).attr("href")) + "/");
            Element imgElement = aElement.child(0);
            atlas.setCover(imgElement.attr("data-original"));
            atlas.setTitle(imgElement.attr("alt"));
            atlasList.add(atlas);
        }
        return atlasList;
    }

    /**
     * 解析详情详情页图片对象
     * @param document
     * @return
     */
    private Image parseImageUrl(Document document) {
        Image image = new Image();
        Elements elements = document.select(".main-image > p > a > img");
        if (elements.size() > 0){
            Element element = elements.get(0);
            image.setUrl(element.attr("src"));
            image.setDesc(element.attr("alt"));
        }
        return image;
    }

    /**
     * 获取对应图册所有图片所在页面的Url地址
     * @param document
     * @return
     */
    private List<String> parseDetailUrl(Document document) {
        List<String> list = new ArrayList<>();
        int maxPage = 0;
        Elements elements = document.select(".pagenavi > a > span");
        for (Element element : elements){
            String strPage = element.text();
            if (strPage.matches("^[0-9]*$")){
                int page = Integer.valueOf(strPage);
                if (page > maxPage){
                    maxPage = page;
                }
            }
        }
        for (int i=1; i<=maxPage; i++){
            list.add(mCurrentUrl + i);
        }
        return list;
    }

    /**
     *  判断当前页数是否小于等于最大页数
     * @param document
     * @param currentPage
     * @return
     */
    private boolean hasPage(Document document,int currentPage) {
        if (mTypeMaxPageMap == null){
            mTypeMaxPageMap = new HashMap<>();
        }
        if (mTypeMaxPageMap.get(mCurrentUrl) != null){
            return currentPage <= mTypeMaxPageMap.get(mCurrentUrl);
        }

        Elements elements = document.select("nav .nav-links .page-numbers");
        int maxPage = -1;
        for (Element element : elements){
            if (element.hasClass("dots")){
                continue;
            }
            String text = element.text();
            try {
                int page = Integer.parseInt(text);
                if (page > maxPage){
                    maxPage = page;
                }
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        mTypeMaxPageMap.put(mCurrentUrl,maxPage);
        return currentPage <= maxPage;
    }

    private List<Type> parseTypes(Document document) {
        Elements elements = document.select(".mainnav > ul > li > a");
        List<Type> typeList = new ArrayList<>();
        mTypeMap = new HashMap<>();
        Elements subElements = document.select(".main > .main-content > .subnav > a");
        if (subElements != null && !subElements.isEmpty()){
            addType(subElements,typeList);
        }
        addType(elements, typeList);
        return typeList;
    }

    private void addType(Elements elements, List<Type> typeList) {
        for (Element element : elements){
            Type type = new Type();
            String text = element.text();
            String value = element.attr("href").replaceAll(BASE_URL,"");
            if ("首页".equals(text) || "zipai/".equals(value) || "all/".equals(value) || "best/".equals(value) || "zhuanti/".equals(value)){
                continue;
            }
            type.setName(text);
            type.setUrl(value);
            typeList.add(type);
            mTypeMap.put(value,text);
        }
    }

    private String parseUrl(String url) {
        return url.substring(url.lastIndexOf("/") + 1);
    }
}

```

4. viewmodel

```java
public class MainVm extends BaseVm{
    private ImageRepository mRepository;
    private List<Type> typeList = new ArrayList<>();

    public MainVm(BaseActivity activity){
        super(activity);
        mRepository = ImageRepository.create();
    }

    @BindingAdapter(value = {
            "activity","data","tab_layout_id"
    })
    public static void bindPagerAdapterToTabLayout(ViewPager viewPager, BaseActivity activity, List<Type> datas, @IdRes int id){
        if (datas == null || activity == null) return;
        PagerAdapter pagerAdapter = ImagePagerAdapter.createPageAdapter(activity.getSupportFragmentManager(),datas);
        viewPager.setAdapter(pagerAdapter);
        TabLayout tabLayout = activity.findViewById(id);
        tabLayout.setupWithViewPager(viewPager);
    }

    public void startGetData(Consumer<List<Type>> loadComplete){
        showProgressDialog(LOADING_MSG);
        mRepository.getTypes()
                .subscribe(types -> {
                    typeList.clear();
                    typeList.addAll(types);
                    dismissProgressDialog();
                    if (loadComplete != null){
                        loadComplete.accept(typeList);
                    }
                },throwable -> {
                    dismissProgressDialog();
                    showShortToast(throwable.getMessage());
                });
    }

    public List<Type> getTypeList() {
        return typeList;
    }
}

```

5. activity

```java
public class MainActivity extends BaseActivity {
    MainVm mainVm;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);
        mainVm = new MainVm(this);
        mainVm.startGetData(types -> binding.setMainVm(mainVm));
    }
}

```

6. xml

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">
    <data>
        <variable
            name="mainVm"
            type="com.boiledcoffee.welfare.vm.MainVm" />
    </data>

    <android.support.design.widget.CoordinatorLayout
        android:id="@+id/main_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <android.support.design.widget.AppBarLayout
            android:id="@+id/appbar"
            android:orientation="vertical"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">
            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                app:title="@string/title_index"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                android:background="?attr/colorPrimary"
                app:layout_scrollFlags="scroll|enterAlways"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light"/>

            <android.support.design.widget.TabLayout
                android:id="@+id/tab_layout"
                android:layout_width="match_parent"
                app:tabMode="scrollable"
                android:layout_height="50dp" />

        </android.support.design.widget.AppBarLayout>

        <android.support.v4.view.ViewPager
            android:id="@+id/view_page"
            app:activity="@{mainVm.getActivity()}"
            app:data="@{mainVm.getTypeList()}"
            app:tab_layout_id="@{@id/tab_layout}"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:layout_behavior="@string/appbar_scrolling_view_behavior" />

    </android.support.design.widget.CoordinatorLayout>

</layout>

```

## github完整代码: https://github.com/boildcoffee/welfare