一、概述
----
我们公司目前开发的所有Android APP都是遵循iOS风格设计的，这并不是一个好现象。我决定将Android 5.x控件引入最近开发的项目中，使用RecyclerView取代以往使用的ListView、GridView，使用SwipeRefreshLayout取代pull-to-refresh第三方库，打造更符合Material Design风格的APP。

本篇博客介绍的就是如何使用SwipeRefreshLayout和RecyclerView实现高仿简书Android端的下拉刷新和上拉加载更多的效果。

![这里写图片描述](http://img.blog.csdn.net/20160326214149913)

根据效果图可以发现，本案例实现了如下效果：

 - 第一次进入页面显示SwipeRefreshLayout的下拉刷新效果
 - 当内容铺满屏幕时，向下滑动显示“加载中...”效果并加载更多数据
 - 当SwipeRefreshLayout正在下拉刷新时，将屏蔽加载更多操作
 - 当加载更多数据时，屏蔽有可能的重复的上拉操作
 - 当向上滑动RecyclerView时，隐藏Toolbar以获得更好的用户体验

二、代码实现
------

 - MainActivity
 

```
package com.leohan.refresh;

import android.os.Bundle;
import android.os.Handler;
import android.support.v4.widget.SwipeRefreshLayout;
import android.support.v7.app.AppCompatActivity;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.support.v7.widget.Toolbar;
import android.util.Log;
import android.view.View;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import butterknife.ButterKnife;
import butterknife.InjectView;

/**
 * @author Leo
 */
public class MainActivity extends AppCompatActivity {


    @InjectView(R.id.toolbar)
    Toolbar toolbar;
    @InjectView(R.id.recyclerView)
    RecyclerView recyclerView;
    @InjectView(R.id.SwipeRefreshLayout)
    SwipeRefreshLayout swipeRefreshLayout;

    boolean isLoading;
    private List<Map<String, Object>> data = new ArrayList<>();
    private MyAdapter adapter = new MyAdapter(this, data);
    private Handler handler = new Handler();

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_notice);
        ButterKnife.inject(this);
        initView();
        initData();
    }

    public void initView() {
        setSupportActionBar(toolbar);
        toolbar.setTitle(R.string.notice);
        toolbar.setNavigationOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                finish();
            }
        });

        swipeRefreshLayout.setColorSchemeResources(R.color.blueStatus);
        swipeRefreshLayout.post(new Runnable() {
            @Override
            public void run() {
                swipeRefreshLayout.setRefreshing(true);
            }
        });

        swipeRefreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
            @Override
            public void onRefresh() {
                handler.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        data.clear();
                        getData();
                    }
                }, 2000);
            }
        });
        final LinearLayoutManager layoutManager = new LinearLayoutManager(this);
        recyclerView.setLayoutManager(layoutManager);
        recyclerView.setAdapter(adapter);
        recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                Log.d("test", "StateChanged = " + newState);


            }

            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                Log.d("test", "onScrolled");

                int lastVisibleItemPosition = layoutManager.findLastVisibleItemPosition();
                if (lastVisibleItemPosition + 1 == adapter.getItemCount()) {
                    Log.d("test", "loading executed");

                    boolean isRefreshing = swipeRefreshLayout.isRefreshing();
                    if (isRefreshing) {
                        adapter.notifyItemRemoved(adapter.getItemCount());
                        return;
                    }
                    if (!isLoading) {
                        isLoading = true;
                        handler.postDelayed(new Runnable() {
                            @Override
                            public void run() {
                                getData();
                                Log.d("test", "load more completed");
                                isLoading = false;
                            }
                        }, 1000);
                    }
                }
            }
        });

        //添加点击事件
        adapter.setOnItemClickListener(new MyAdapter.OnItemClickListener() {
            @Override
            public void onItemClick(View view, int position) {
                Log.d("test", "item position = " + position);
            }

            @Override
            public void onItemLongClick(View view, int position) {

            }
        });
    }


    public void initData() {
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                getData();
            }
        }, 1500);

    }

    /**
     * 获取测试数据
     */
    private void getData() {
        for (int i = 0; i < 6; i++) {
            Map<String, Object> map = new HashMap<>();
            data.add(map);
        }
        adapter.notifyDataSetChanged();
        swipeRefreshLayout.setRefreshing(false);
        adapter.notifyItemRemoved(adapter.getItemCount());
    }


}

```

 - RecyclerViewAdapter
 

```
package com.leohan.refresh;

import android.content.Context;
import android.support.v7.widget.RecyclerView.Adapter;
import android.support.v7.widget.RecyclerView.ViewHolder;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import java.util.List;

public class RecyclerViewAdapter extends Adapter<ViewHolder> {

    private static final int TYPE_ITEM = 0;
    private static final int TYPE_FOOTER = 1;
    private Context context;
    private List data;

    public interface OnItemClickListener {
        void onItemClick(View view, int position);

        void onItemLongClick(View view, int position);
    }

    private OnItemClickListener onItemClickListener;

    public void setOnItemClickListener(OnItemClickListener onItemClickListener) {
        this.onItemClickListener = onItemClickListener;
    }

    @Override
    public int getItemCount() {
        return data.size() == 0 ? 0 : data.size() + 1;
    }

    @Override
    public int getItemViewType(int position) {
        if (position + 1 == getItemCount()) {
            return TYPE_FOOTER;
        } else {
            return TYPE_ITEM;
        }
    }

    public RecyclerViewAdapter(Context context, List data) {
        this.context = context;
        this.data = data;
    }

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        if (viewType == TYPE_ITEM) {
            View view = LayoutInflater.from(context).inflate(R.layout.item_notice, parent,
                    false);
            return new ItemViewHolder(view);
        } else if (viewType == TYPE_FOOTER) {
            View view = LayoutInflater.from(context).inflate(R.layout.item_foot, parent,
                    false);
            return new FootViewHolder(view);
        }
        return null;
    }


    @Override
    public void onBindViewHolder(final ViewHolder holder, int position) {
        if (holder instanceof ItemViewHolder) {
            //holder.tv.setText(data.get(position));
            if (onItemClickListener != null) {
                holder.itemView.setOnClickListener(new View.OnClickListener() {
                    @Override
                    public void onClick(View v) {
                        int position = holder.getLayoutPosition();
                        onItemClickListener.onItemClick(holder.itemView, position);
                    }
                });

                holder.itemView.setOnLongClickListener(new View.OnLongClickListener() {
                    @Override
                    public boolean onLongClick(View v) {
                        int position = holder.getLayoutPosition();
                        onItemClickListener.onItemLongClick(holder.itemView, position);
                        return false;
                    }
                });
            }
        }
    }


    static class ItemViewHolder extends ViewHolder {

        TextView tv;

        public ItemViewHolder(View view) {
            super(view);
            tv = (TextView) view.findViewById(R.id.tv_date);
        }
    }

    static class FootViewHolder extends ViewHolder {

        public FootViewHolder(View view) {
            super(view);
        }
    }
}
```

 - item_base.xml
 
```
<?xml version="1.0" encoding="utf-8"?>

<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginLeft="@dimen/margin_10"
    android:layout_marginRight="@dimen/margin_10"
    android:foreground="?android:attr/selectableItemBackgroundBorderless"
    android:layout_marginTop="6dp"
    android:orientation="vertical"
    app:cardBackgroundColor="@color/line"
    app:cardPreventCornerOverlap="true"
    app:cardUseCompatPadding="true"
    app:contentPadding="6dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/tv_date"
            style="@style/NormalTextView"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="2015-12-11 12:00" />

        <android.support.v7.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical"
            app:cardBackgroundColor="@color/white"
            app:cardPreventCornerOverlap="true"
            app:cardUseCompatPadding="true"
            app:contentPadding="10dp">

            <TextView
                android:id="@+id/tv_title"
                style="@style/SmallGreyTextView"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:ellipsize="end"
                android:maxLines="2"
                android:text="视线好转,0729出口开通,0621进口开通。视线好转,0729出口开通,0621进口开通。" />

        </android.support.v7.widget.CardView>
    </LinearLayout>

</android.support.v7.widget.CardView>
```

 - item_foot.xml
 

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="40dp"
    android:gravity="center"
    android:orientation="horizontal"
 >


    <ProgressBar
        android:layout_marginRight="6dp"
        android:id="@+id/progressBar"
        style="?android:attr/progressBarStyleSmall"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center" />

    <TextView
        style="@style/SmallGreyTextView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:text="@string/loading" />


</LinearLayout>

```

三、代码分析
------

 - 上拉加载更多数据通过监听RecyclerView的滚动事件**RecyclerView.OnScrollListener()**实现的，它提供了两个方法：

```
		/**
         * 当RecyclerView的滑动状态改变时触发
         */
        public void onScrollStateChanged(RecyclerView recyclerView, int newState){}

        /**
         * 当RecyclerView滑动时触发
         * 类似点击事件的MotionEvent.ACTION_MOVE
         */
        public void onScrolled(RecyclerView recyclerView, int dx, int dy){}
```
 - RecyclerView的滑动状态有如下三种：
 

```
    /**
     * The RecyclerView is not currently scrolling.
     * 手指离开屏幕
     */
    public static final int SCROLL_STATE_IDLE = 0;

    /**
     * The RecyclerView is currently being dragged by outside input such as user touch input.
     * 手指触摸屏幕
     */
    public static final int SCROLL_STATE_DRAGGING = 1;

    /**
     * The RecyclerView is currently animating to a final position while not under
     * outside control.
     * 手指加速滑动并放开，此时滑动状态伴随SCROLL_STATE_IDLE
     */
    public static final int SCROLL_STATE_SETTLING = 2;
```
 - 由于简书APP的上拉加载更多的是在滑动到最后一个item时自动触发的，与手指是否在屏幕上无关，即与滑动状态无关。因此，实现这种效果只需要在**public void onScrolled(RecyclerView recyclerView, int dx, int dy)** 方法中操作，无需关注当时的滑动状态：

```
  @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                Log.d("test", "onScrolled");

                int lastVisibleItemPosition = layoutManager.findLastVisibleItemPosition();
                if (lastVisibleItemPosition + 1 == adapter.getItemCount()) {
                    Log.d("test", "loading executed");

                    boolean isRefreshing = swipeRefreshLayout.isRefreshing();
                    if (isRefreshing) {
                        adapter.notifyItemRemoved(adapter.getItemCount());
                        return;
                    }
                    if (!isLoading) {
                        isLoading = true;
                        handler.postDelayed(new Runnable() {
                            @Override
                            public void run() {
                                getData();
                                Log.d("test", "load more completed");
                                isLoading = false;
                            }
                        }, 1000);
                    }
                }
            }
```

 - 如果要实现当且仅当滑动到最后一项并且手指上拉抛出时才执行上拉加载更多效果的话，需要配合**onScrollStateChanged(RecyclerView recyclerView, int newState**的使用，可以将代码改为：

```
 @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                Log.d("test", "StateChanged = " + newState);
                if (newState == RecyclerView.SCROLL_STATE_IDLE && lastVisibleItemPosition + 1 == adapter.getItemCount()) {
                    Log.d("test", "loading executed");

                    boolean isRefreshing = swipeRefreshLayout.isRefreshing();
                    if (isRefreshing) {
                        adapter.notifyItemRemoved(adapter.getItemCount());
                        return;
                    }
                    if (!isLoading) {
                        isLoading = true;
                        handler.postDelayed(new Runnable() {
                            @Override
                            public void run() {
                                getData();
                                Log.d("test", "load more completed");
                                isLoading = false;
                            }
                        }, 1000);
                    }
                }
            }

            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                Log.d("test", "onScrolled");

                lastVisibleItemPosition = layoutManager.findLastVisibleItemPosition();

            }
```

 - 加载更多的效果可以通过item_foot.xml自定义，滑动到最后一项时显示该item并执行加载更多，当加载数据完毕时需要将该item移除掉
 

```
adapter.notifyItemRemoved(adapter.getItemCount());
```

下面的代码就是RecyclerView的多个item布局的实现方法：

```
 @Override
    public int getItemCount() {
        return data.size() == 0 ? 0 : data.size() + 1;
    }

    @Override
    public int getItemViewType(int position) {
        if (position + 1 == getItemCount()) {
            return TYPE_FOOTER;
        } else {
            return TYPE_ITEM;
        }
    }

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        if (viewType == TYPE_ITEM) {
            View view = LayoutInflater.from(context).inflate(R.layout.item_base, parent,
                    false);
            return new ItemViewHolder(view);
        } else if (viewType == TYPE_FOOTER) {
            View view = LayoutInflater.from(context).inflate(R.layout.item_foot, parent,
                    false);
            return new FootViewHolder(view);
        }
        return null;
    }
```
