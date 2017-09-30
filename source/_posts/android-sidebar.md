---
title: a-z导航栏
date: 2017-09-30 14:33:20
categories: android
tags: 
    android
    java
---
先上张demo的效果图
![sidebarDemoImg.png](http://upload-images.jianshu.io/upload_images/2898841-667d52dcdfff7360.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ截图20160909164412.png](http://upload-images.jianshu.io/upload_images/2898841-b77bab62aca7a099.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图上可以看到该布局由2部分组成,ListView和右边的sidebar组成,那么我们要实现自己的字母导航就需要知道:   
1.如何自定义Sidebar绘制出a-z   
2.如何将sidebar与Listview结合 实现字母导航


### 自定义Sidebar
#### 绘制UI
```java
public class Sidebar extends View{
    public static String[] alphabets = new String[]{ "A", "B", "C", "D", "E", "F", "G", "H", "I",
            "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z", "#" };
    private int selectedPosition; //选中字母的位置
    private Paint mPaint;
    private int cellHeight; //每一个字母的高度
    private int alphabetDefaultColor;
    private float textSize;

    public Sidebar(Context context) {
        super(context);
        init();
    }

    public Sidebar(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public Sidebar(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        textSize = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 12, getResources().getDisplayMetrics());
        mPaint = new Paint();
        mPaint.setAntiAlias(true);
        mPaint.setTextSize(textSize);
        alphabetDefaultColor = Color.GRAY;
        alphabetSelectedColor = Color.RED;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        cellHeight = getHeight() / alphabets.length;
        for (int i=0; i< alphabets.length; i++){
            drawAlphabet(canvas,i);
        }

    }

    private void drawAlphabet(Canvas canvas,int positon) {
        String alphabet = alphabets[positon];
        mPaint.setColor(alphabetDefaultColor);

        int baseLine = (positon+1) * cellHeight; //position是从0开始的所以需要+1
        canvas.drawText(alphabet, (getWidth() - mPaint.measureText(alphabet)) / 2, baseLine, mPaint);
    }


    public float getTextSize() {
        if (mPaint == null) return 0;
        return mPaint.getTextSize();
    }

    public void setTextSize(float textSize) {
        if (mPaint == null) return ;
        mPaint.setTextSize(textSize);
    }
}
```
drawAlphabet方法主要实现a-z从上到下的绘制工作,其中需要注意的是canvas.drawText的第3个参数Y指的是基线(参考文章:https://zh.wikipedia.org/wiki/%E5%9F%BA%E7%B7%9A).
![410px-Typography_Line_Terms.svg.png](http://upload-images.jianshu.io/upload_images/2898841-084599a04c5ccc77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)   

![QQ截图20160909144330.png](http://upload-images.jianshu.io/upload_images/2898841-b9e9c2ca42581e94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从上图我们能知道`D基线=(4/3)C基线=2B基线=4A基线`,因此每一个字母的基线就等于它所处位置*字母的高度。
我们要让字母水平居中显示所以canvas.drawText 的第二个参数水平方向的位置就应该是 (view的总宽度-绘制字母的长度)/2。
完成以上步骤后我们就能成功绘制出a-z。


#### 监听onTouch事件,计算出被选中字母的位置并提供回调函数
```java
public class Sidebar extends View{
    ...
    private void drawAlphabet(Canvas canvas,int positon) {
        String alphabet = alphabets[positon];

        mPaint.setColor(alphabetDefaultColor);
        if (isPressed()){
            if (positon == selectedPosition){
                mPaint.setColor(alphabetSelectedColor);
                if (onAlphabetChangeListener != null){
                    onAlphabetChangeListener.alphabetChangeListener(this,alphabet,positon);
                }
            }
        }

        int baseLine = (positon+1) * cellHeight;
        canvas.drawText(alphabet, (getWidth() - mPaint.measureText(alphabet)) / 2, baseLine, mPaint);

    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()){
            case MotionEvent.ACTION_DOWN:
                setPressed(true);
                break;
            case MotionEvent.ACTION_MOVE:
                float y = event.getY();
                selectedPosition = (int)(Math.ceil((y / cellHeight)) - 1); //postion是从0开始的,所以需要-1
                break;
            case MotionEvent.ACTION_UP:
                setPressed(false);
                break;
        }
        invalidate();
        return true;
    }

    public interface OnAlphabetChangeListener{
        void alphabetChangeListener(View v,String alphabet,int position);
    }

    public OnAlphabetChangeListener getOnAlphabetChangeListener() {
        return onAlphabetChangeListener;
    }

    public void setOnAlphabetChangeListener(OnAlphabetChangeListener onAlphabetChangeListener) {
        this.onAlphabetChangeListener = onAlphabetChangeListener;
    }

    ...
}
```
这里主要说明下Math.ceil()函数的作用是向上取整即:1.1 = 2,1.5=2。通过Math.ceil((y / cellHeight)我们就可以计算出当前手指选中的是那个字母的位置。为了能让用户知道他当前选中的是哪个字母,我们可以在onTouchEvent return前调用invalidate,
调用invalidate后会重新绘制页面,onDraw方法会被调用,所以我们可以在drawAlphabet中加上当前要绘制的字母是否被用户选中,是则采用其他颜色绘制。最后就是提供回调函数alphabetChangeListener,该回调函数是用于与listview实现字母导航。

### 与ListView实现字母导航
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.jason.sidebardemo.MainActivity">

    <ListView
        android:id="@+id/list"
        android:divider="@null"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

    <com.jason.library.widget.Sidebar
        android:id="@+id/sidebar"
        android:layout_alignParentRight="true"
        android:layout_width="30dp"
        android:layout_height="match_parent" />

</RelativeLayout>
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/tv_header"
        android:visibility="gone"
        android:layout_width="match_parent"
        android:layout_height="30dp"
        android:gravity="center_vertical"
        android:paddingLeft="5dp"
        android:background="@android:color/darker_gray" />


    <TextView
        android:id="@+id/tv_name"
        android:layout_width="match_parent"
        android:layout_height="30dp"
        android:gravity="center_vertical"
        android:paddingLeft="10dp" />

</LinearLayout>
```
上面为activity和listview item的布局文件.   
要使用Listview主要就是设置adapter,那么我们就先看adapter的代码

```java
public class MyAdapter extends BaseAdapter implements SectionIndexer{
    private Context mContext;
    private List<Contact> mContacts;
    public MyAdapter(Context context,List<Contact> contacts){
        mContext = context;
        mContacts = contacts;
    }

    @Override
    public int getCount() {
        return mContacts.size();
    }

    @Override
    public Object getItem(int position) {
        return mContacts.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder viewHolder;
        if (convertView == null){
            viewHolder = new ViewHolder();
            convertView = LayoutInflater.from(mContext).inflate(R.layout.item_contact, null);
            viewHolder.tvHeader = (TextView) convertView.findViewById(R.id.tv_header);
            viewHolder.tvName = (TextView) convertView.findViewById(R.id.tv_name);
            convertView.setTag(viewHolder);
        }else {
            viewHolder = (ViewHolder) convertView.getTag();
        }

        if (position == 0){ //第1个需要显示首字母
            viewHolder.tvHeader.setVisibility(View.VISIBLE);
        }else if (mContacts.get(position).getFirstAlphabet().charAt(0) != mContacts.get(position - 1).getFirstAlphabet().charAt(0)){
            //前后2个首字母不相同,需要显示首字母
            viewHolder.tvHeader.setVisibility(View.VISIBLE);
        }else {
            viewHolder.tvHeader.setVisibility(View.GONE);
        }

        viewHolder.tvHeader.setText(mContacts.get(position).getFirstAlphabet());
        viewHolder.tvName.setText(mContacts.get(position).getName());

        return convertView;
    }

    @Override
    public Object[] getSections() {
        return Arrays.copyOf(Sidebar.alphabets,Sidebar.alphabets.length);
    }

    @Override
    public int getPositionForSection(int sectionIndex) {
        for (int i=0; i<getCount(); i++){
            if (((String)getSections()[sectionIndex]).charAt(0) == mContacts.get(i).getFirstAlphabet().charAt(0)){
                return i;
            }
        }
        return 0;
    }

    @Override
    public int getSectionForPosition(int position) {
        return 0;
    }

    class ViewHolder{
        public TextView tvHeader;
        public TextView tvName;
    }
}
```

```java
public class Contact implements Comparable{
    private String firstAlphabet; //名字的第一个字的首字母
    private String name;

    public Contact(String name) {
        setName(name);
    }

    public String getFirstAlphabet() {
        return firstAlphabet;
    }


    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
        firstAlphabet = PinYin4jUtil.getFirstAlphabet(name);  //获取第一个字符的字母,若为中文则使用pinyin4j获取第一个字的拼音的第一个字符,若为英文字母获取第一个字母,否则返回#
    }

    @Override
    public int compareTo(Object another) {
        Contact compareContact = (Contact) another;
        if (compareContact.getFirstAlphabet().equals("#")){
            return 1;
        }else if (getFirstAlphabet().equals("#")){
            return -1;
        }else {
            return getFirstAlphabet().compareTo(((Contact) another).getFirstAlphabet());
        }
    }
}
```

在这里有2点需要说明下的：   
1.a-z字母导航的数据源必须经过a-z排序,Contact类通过实现Comparable提供对象排序算法.`(A>B return 1,A=B return 0,A<B return -1)`   
2.Listview的adapter需要实现SectionIndexer接口,SectionIndexer接口需要实现3个方法`getSections(),getPositionForSection(int sectionIndex),getSectionForPosition(int position)`
,getSections返回的值为章节数组即`(a-z字符数组)`，getPositionForSection通过章节位置`(Sidebar中a-z的位置)`获取position的起始位置,getSectionForPosition通过位置获取对应的章节.
其中需要说明下section和position的关系,其实就和书的章节与页数一样,第一章有100页,那么`0-100(postion)`就对应`第一章(section)`.


```java
public class MainActivity extends AppCompatActivity {
    ListView mListView;
    Sidebar mSidebar;
    MyAdapter mAdapter;



    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();
    }

    private void initView() {
        mListView = (ListView) findViewById(R.id.list);
        mSidebar = (Sidebar) findViewById(R.id.sidebar);
        mSidebar.setOnAlphabetChangeListener(new Sidebar.OnAlphabetChangeListener() {
            @Override
            public void alphabetChangeListener(View v, String alphabet, int position) {
                mListView.setSelection(mAdapter.getPositionForSection(position));
            }
        });
        mAdapter = new MyAdapter(this, getDatas());
        mListView.setAdapter(mAdapter);
    }

    private List<Contact> getDatas() {
        List<Contact> datas = new ArrayList<>();
        datas.add(new Contact("小熊"));
        datas.add(new Contact("小明"));
        datas.add(new Contact("老王"));
        datas.add(new Contact("老宋"));
        datas.add(new Contact("李死"));
        datas.add(new Contact("小张"));
        datas.add(new Contact("王五"));
        datas.add(new Contact("jason"));
        datas.add(new Contact("java"));
        datas.add(new Contact("python"));
        datas.add(new Contact("c"));
        datas.add(new Contact("c#"));
        datas.add(new Contact("c++"));
        datas.add(new Contact("盲僧"));
        datas.add(new Contact("蛮王"));
        datas.add(new Contact("剑圣"));
        datas.add(new Contact("赵兴"));
        datas.add(new Contact("女警"));
        datas.add(new Contact("亚索"));
        datas.add(new Contact("狗熊"));
        datas.add(new Contact("刀妹"));
        datas.add(new Contact("吸血鬼"));
        datas.add(new Contact("卡萨丁"));
        datas.add(new Contact("火女"));
        datas.add(new Contact("女枪"));
        datas.add(new Contact("奥巴马"));
        Collections.sort(datas);  //排序数据
        return datas;
    }
}
```
最后就是通过alphabetChangeListener回调与listview实现导航功能,通过mAdapter.getPositionForSection()获取到选中字母的第一个条目的位置,再通过mListView.setSelection()让ListView定位到该条目上   
  
附上demo地址:https://github.com/aii1991/SidebarDemo

