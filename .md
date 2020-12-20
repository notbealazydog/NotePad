

# NotePad笔记本应用

------

## 基本功能：添加笔记时间戳和笔记查询功能

### 一、添加笔记时间戳

阅读NotePad原码可以看到NotePadProvider.java中已经定义了时间变量

```
 public void onCreate(SQLiteDatabase db) {
           db.execSQL("CREATE TABLE " + NotePad.Notes.TABLE_NAME + " ("
                   + NotePad.Notes._ID + " INTEGER PRIMARY KEY,"
                   + NotePad.Notes.COLUMN_NAME_TITLE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_NOTE + " TEXT,"
                   + NotePad.Notes.COLUMN_NAME_CREATE_DATE + " INTEGER,"
                   + NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE + " INTEGER"
                   + ");");
       }
```

NoteList.java中使用SimpleCursorAdapter来装配数据，首先查询数据库的内容

```
 Cursor cursor = managedQuery(
         getIntent().getData(),           
         PROJECTION,                      
         null,                             
         null,                             
         NotePad.Notes.DEFAULT_SORT_ORDER);

```

然后通过SimpleCursorAdapter来进行装配

```
 SimpleCursorAdapter adapter
         = new SimpleCursorAdapter(
                   this,                             
                   R.layout.noteslist_item,          
                   cursor,                           
                   dataColumns,
                   viewIDs);

```

这部分就的代码就不需更改

#### 主要步骤

1.添加时间戳的位置在主页面的每个列表项中添加，即在notelist_item.xml布局文件中添加一个 TextView

```
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@android:id/text1"
    android:layout_width="match_parent"
    android:layout_height="?android:attr/listPreferredItemHeight"
    android:textAppearance="?android:attr/textAppearanceLarge"
    android:gravity="center_vertical"
    android:paddingLeft="5dip"
    android:singleLine="true"
```

2.在NoteList类的PROJECTION中添加COLUMN_NAME_MODIFICATION_DATE字段

```
 private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE //日期
    };
```

3.修改适配器内容，增加dataColumns中装配到ListView的内容，所以要同时增加一个ID标识来存放该时间参数，将笔记的修改时间作为笔记本的时间戳。

```
 String[] dataColumns = {
                NotePad.Notes.COLUMN_NAME_TITLE,
                NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE
        } ;

        // The view IDs that will display the cursor columns, initialized to the TextView in
        // noteslist_item.xml
        int[] viewIDs = {
                android.R.id.text1 ,
                R.id.text1_time
        };
```

做完这些，标题下确实会多显示一行数据，但是，这并不是我们平常所见到的时间格式，而是java的默认时间戳，需要对这些数据进行格式转化，使人能看的懂。

4.更改时间格式存入。改动地方分别为**NotePadProvider中的Uri insert方法**和**NoteEditor中的updateNote**方法。前者为创建笔记时产生的时间，后者为修改笔记时产生的时间。

```
Long now = Long.valueOf(System.currentTimeMillis());
        Date date = new Date(now);
        //将时间格式化为常见的年月日表示形式
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yy-MM-dd HH:mm:ss");
        simpleDateFormat.setTimeZone(TimeZone.getTimeZone("GMT+08:00"));
        String dateFormat = simpleDateFormat.format(date);
        // If the values map doesn't contain the creation date, sets the value to the current time.
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_CREATE_DATE) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_CREATE_DATE, dateFormat);
        }

        // If the values map doesn't contain the modification date, sets the value to the current
        // time.
        if (values.containsKey(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE) == false) {
            values.put(NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, dateFormat);
        }
```

#### 效果截图

[![rdrCfx.png](https://s3.ax1x.com/2020/12/21/rdrCfx.png)](https://imgchr.com/i/rdrCfx)

### 二、笔记查询功能

#### 主要步骤

1.要添加笔记查询功能，就要在应用中增加一个搜索的入口。找到list_options_menu.xml文件，添加一个搜索的item，搜索图标用安卓自带的图标，设为总是显示

```
<item
    android:id="@+id/menu_search"
    android:title="@string/menu_search"
    android:icon="@android:drawable/ic_search_category_default"
    android:showAsAction="always">
</item>
```

2.在NoteList中找到onOptionsItemSelected方法，在switch中添加笔记搜索跳转的case语句:

```
case R.id.menu_search:
                //查找功能
                //startActivity(new Intent(Intent.ACTION_SEARCH, getIntent().getData()));
                Intent intent = new Intent(this, NoteSearch.class);
                this.startActivity(intent);
                return true;
```

3.新建一个名为NoteSearch的activity。由于搜索出来的也是笔记列表，所以可以模仿NoteList的activity继承ListActivity。在安卓中有个用于搜索控件：SearchView，可以把**SearchView跟ListView相结合**，**动态地显示搜索结果**。先布局搜索页面，在layout中新建布局文件note_search_list.xml：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".NoteSearch">
    <SearchView
        android:id="@+id/search_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:iconifiedByDefault="false"
    ></SearchView>
    <ListView
        android:id="@+id/list_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        ></ListView>


</LinearLayout>
```

4.NoteSearch实现

要动态地显示搜索结果，就要对SearchView文本变化设置监听，NoteSearch除了要继承ListView外还要实现SearchView.OnQueryTextListener接口：

```
public class NoteSearch extends Activity implements SearchView.OnQueryTextListener
{
    ListView listView;
    SQLiteDatabase sqLiteDatabase;
    /**
     * The columns needed by the cursor adapter
     */
    private static final String[] PROJECTION = new String[]{
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE//时间
    };

    public boolean onQueryTextSubmit(String query) {
        Toast.makeText(this, "您选择的是："+query, Toast.LENGTH_SHORT).show();
        return false;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.note_search);
        SearchView searchView = findViewById(R.id.search_view);
        Intent intent = getIntent();
        if (intent.getData() == null) {
            intent.setData(NotePad.Notes.CONTENT_URI);
        }
        listView = findViewById(R.id.list_view);
        sqLiteDatabase = new NotePadProvider.DatabaseHelper(this).getReadableDatabase();
        //设置该SearchView显示搜索按钮
        searchView.setSubmitButtonEnabled(true);

        //设置该SearchView内默认显示的提示文本
        searchView.setQueryHint("查找");
        searchView.setOnQueryTextListener(this);

    }
```

对于数据显示部分直接沿用NoteList中的代码即可

```
public boolean onQueryTextChange(String string) {
        String selection1 = NotePad.Notes.COLUMN_NAME_TITLE+" like ? or "+NotePad.Notes.COLUMN_NAME_NOTE+" like ?";
        String[] selection2 = {"%"+string+"%","%"+string+"%"};
        Cursor cursor = sqLiteDatabase.query(
                NotePad.Notes.TABLE_NAME,
                PROJECTION, // The columns to return from the query
                selection1, // The columns for the where clause
                selection2, // The values for the where clause
                null,          // don't group the rows
                null,          // don't filter by row groups
                NotePad.Notes.DEFAULT_SORT_ORDER // The sort order
        );
        // The names of the cursor columns to display in the view, initialized to the title column
        String[] dataColumns = {
                NotePad.Notes.COLUMN_NAME_TITLE,//标题
                NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE //时间
        } ;
        // The view IDs that will display the cursor columns, initialized to the TextView in
        // noteslist_item.xml
        int[] viewIDs = {
                android.R.id.text1,
             R.id.text1_time
        };
        // Creates the backing adapter for the ListView.
        SimpleCursorAdapter adapter
                = new SimpleCursorAdapter(
                this,                             // The Context for the ListView
                R.layout.noteslist_item,         // Points to the XML for a list item
                cursor,                           // The cursor to get items from
                dataColumns,
                viewIDs
        );
        // Sets the ListView's adapter to be the cursor adapter that was just created.
        listView.setAdapter(adapter);
        return true;
    }
```

5.最后要在AndroidManifest.xml注册NoteSearch

```
 <activity
        android:name="NoteSearch"
        android:label="@string/title_notes_search">
    </activity>
```

#### 截图效果

[![rdrip6.png](https://s3.ax1x.com/2020/12/21/rdrip6.png)](https://imgchr.com/i/rdrip6)

------



## 附加功能

#### 一、添加背景颜色

在数据库中添加一个颜色的字段，在这之前在NotePad类中添加：

```
public static final String COLUMN_NAME_BACK_COLOR = "color";


public static final int DEFAULT_COLOR = 0; //白
public static final int YELLOW_COLOR = 1; //黄
public static final int BLUE_COLOR = 2; //蓝
public static final int GREEN_COLOR = 3; //绿
public static final int RED_COLOR = 4; //红
```

创建数据库表地方添加颜色的字段：

```
+ NotePad.Notes.COLUMN_NAME_BACK_COLOR + " INTEGER" //颜色
```

自定义一个CursorAdapter继承SimpleCursorAdapter，既能完成cursor读取的数据库内容填充到item，又能将颜色填充：

```
public class CursorAdapter extends SimpleCursorAdapter {
    public MyCursorAdapter(Context context, int layout, Cursor c,
                           String[] from, int[] to) {
        super(context, layout, c, from, to);
    }
    @Override
    public void bindView(View view, Context context, Cursor cursor){
        super.bindView(view, context, cursor);
        //从数据库中读取的cursor中获取笔记列表对应的颜色数据，并设置笔记颜色
        int x = cursor.getInt(cursor.getColumnIndex(NotePad.Notes.COLUMN_NAME_BACK_COLOR));
     
        switch (x){
            case NotePad.Notes.DEFAULT_COLOR:
                view.setBackgroundColor(Color.rgb(255, 255, 255));
                break;
            case NotePad.Notes.YELLOW_COLOR:
                view.setBackgroundColor(Color.rgb(247, 216, 133));
                break;
            case NotePad.Notes.BLUE_COLOR:
                view.setBackgroundColor(Color.rgb(165, 202, 237));
                break;
            case NotePad.Notes.GREEN_COLOR:
                view.setBackgroundColor(Color.rgb(161, 214, 174));
                break;
            case NotePad.Notes.RED_COLOR:
                view.setBackgroundColor(Color.rgb(244, 149, 133));
                break;
            default:
                view.setBackgroundColor(Color.rgb(255, 255, 255));
                break;
        }
    }
}
```

NoteList中的PROJECTION添加颜色项：

```
private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_TITLE, // 1
      
            NotePad.Notes.COLUMN_NAME_MODIFICATION_DATE, // 2
            NotePad.Notes.COLUMN_NAME_BACK_COLOR, //颜色
    };
```

NoteList中添加adapter

```
adapter = new MyCursorAdapter(
        this,
        R.layout.noteslist_item,
        cursor,
        dataColumns,
        viewIDs
    );
```

添加一个更改背景的选项

```
<item android:id="@+id/menu_color"
        android:title="@string/menu_color"
        android:icon="@drawable/ic_menu_color"
        android:showAsAction="always"/>
```

选择颜色界面进行布局

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal" android:layout_width="match_parent"
    android:layout_height="match_parent">
    <ImageButton
        android:id="@+id/color_white"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorWhite"
        android:onClick="white"/>
    <ImageButton
        android:id="@+id/color_yellow"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorYellow"
        android:onClick="yellow"/>
    <ImageButton
        android:id="@+id/color_blue"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorBlue"
        android:onClick="blue"/>
    <ImageButton
        android:id="@+id/color_green"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorGreen"
        android:onClick="green"/>
    <ImageButton
        android:id="@+id/color_red"
        android:layout_width="0dp"
        android:layout_height="50dp"
        android:layout_weight="1"
        android:background="@color/colorRed"
        android:onClick="red"/>
</LinearLayout>
```

颜色选择代码

```
public class NoteColor extends Activity {
    private Cursor mCursor;
    private Uri mUri;
    private int color;
    private static final int COLUMN_INDEX_TITLE = 1;
    private static final String[] PROJECTION = new String[] {
            NotePad.Notes._ID, // 0
            NotePad.Notes.COLUMN_NAME_BACK_COLOR,
    };
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.note_color);
        mUri = getIntent().getData();
        mCursor = managedQuery(
                mUri,        // The URI for the note that is to be retrieved.
                PROJECTION,  // The columns to retrieve
                null,        // No selection criteria are used, so no where columns are needed.
                null,        // No where columns are used, so no where values are needed.
                null         // No sort order is needed.
        );
    }
    @Override
    protected void onResume(){
    //执行顺序在onCreate之后
        if (mCursor != null) {
            mCursor.moveToFirst();
            color = mCursor.getInt(COLUMN_INDEX_TITLE);
        }
        super.onResume();
    }
    @Override
    protected void onPause() {
    //执行顺序在finish()之后，将选择的颜色存入数据库
        super.onPause();
        ContentValues values = new ContentValues();
        values.put(NotePad.Notes.COLUMN_NAME_BACK_COLOR, color);
        getContentResolver().update(mUri, values, null, null);
    }
    public void white(View view){
        color = NotePad.Notes.DEFAULT_COLOR;
        finish();
    }
    public void yellow(View view){
        color = NotePad.Notes.YELLOW_COLOR;
        finish();
    }
    public void blue(View view){
        color = NotePad.Notes.BLUE_COLOR;
        finish();
    }
    public void green(View view){
        color = NotePad.Notes.GREEN_COLOR;
        finish();
    }
    public void red(View view){
        color = NotePad.Notes.RED_COLOR;
        finish();
    }

}
```

Manifest注册

```
  <activity android:name="NoteColor"
            android:theme="@android:style/Theme.Holo.Light.Dialog"
            android:label="ChangeColor"
            android:windowSoftInputMode="stateVisible">
```



#### 效果截图

[![rdDz79.png](https://s3.ax1x.com/2020/12/21/rdDz79.png)](https://imgchr.com/i/rdDz79)

[![rdrpkR.png](https://s3.ax1x.com/2020/12/21/rdrpkR.png)](https://imgchr.com/i/rdrpkR)