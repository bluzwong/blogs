title: Android中 一些维持状态的经验
date: 2016-03-06 17:38:43
tags: Android
---
## 在 Activity 中的情况
### 1. 使用 retained Fragment
大牛鸿洋在[博文](http://blog.csdn.net/lmj623565791/article/details/37936275)中提及一种方法 `如果是大量数据，使用Fragment保持需要恢复的对象`, 用法大概是  

用于维持状态的 Fragment
```java

public class ByFragment extends Fragment {

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 设置 维持实例
        setRetainInstance(true);
    }
    
    // 需要维持状态的数据
    int count = 0;
}

```
需要维持状态的 Activity:
```java

public class ByFragmentActivity extends Activity {

    SwipeRefreshLayout refreshLayout;
    TextView textView;
    ByFragment fragment;
    int count = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_state);

        refreshLayout = (SwipeRefreshLayout) findViewById(R.id.refresh);
        textView = (TextView) findViewById(R.id.tv);
        
        // 获取 activity 的 FragmentManager 中用于维持状态数据的 Fragment
        fragment = (ByFragment) getFragmentManager().findFragmentByTag("ByFragment");
        if (fragment == null) {
            fragment = new ByFragment();
            getFragmentManager().beginTransaction().add(fragment, "ByFragment").commit();
        } else {
            count = fragment.count;
        }
        refreshTv();

        refreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
            @Override
            public void onRefresh() {
                refreshLayout.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        count++;
                        fragment.count = count;
                        refreshTv();
                        refreshLayout.setRefreshing(false);
                    }
                }, 1000);
            }
        });
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        fragment.count = count;
    }

}

```

打开 `开发者选项` 中的 `不保留活动` 以方便测试
![pic1](http://7xr14l.com1.z0.glb.clouddn.com/activitynotkeep.jpg)

进入到 Activity 中，count 是需要维持状态的数据
![pic2](http://7xr14l.com1.z0.glb.clouddn.com/count1.jpg)

旋转屏幕
![pic3](http://7xr14l.com1.z0.glb.clouddn.com/count1_2.jpg)
数据成功维持。

按 Home 键，置于后台，然后再返回 Activity
![pic4](http://7xr14l.com1.z0.glb.clouddn.com/count0.jpg)
数据消失了。

### 2. 使用 savedInstanceState: Bundle
这种方式比较基础，用法就不再多述
```java

public class ByBundleActivity extends Activity {

    SwipeRefreshLayout refreshLayout;
    TextView textView;
    int count = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_state);

        refreshLayout = (SwipeRefreshLayout) findViewById(R.id.refresh);
        textView = (TextView) findViewById(R.id.tv);

        if (savedInstanceState != null) {
            count = savedInstanceState.getInt("KEY_COUNT", 0);
        }
        refreshTv();

        refreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
            @Override
            public void onRefresh() {
                refreshLayout.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        count++;
                        refreshTv();
                        refreshLayout.setRefreshing(false);
                    }
                }, 1000);
            }
        });
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putInt("KEY_COUNT", count);
    }

}
```
进入 Activity 获取数据。
旋转屏幕，数据还在。
按 Home，再回来，数据还在。

### 3. 小结
第一种方式使用 retained Fragment，当配置发生变化时（屏幕旋转就是情况之一），其实例在内存中没有被销毁。优点是能够在 Fragment 中完整维持数据，并且不需要实现任何一种序列化的方式。缺点呢，也很明显，当长时间在后台时，难免 Activity 和 其 Fragment 会被系统所回收（开启不保留活动模拟），所以维持的数据也当然没有了。

第二种方式使用 Bundle。优点是就算被系统回收了，保存的状态也能在下一次启动时 `onCreate(Bundle savedInstanceState)`  Bundle 中获取。缺点是，这些数据都要进行序列化、反序列化，一些复杂对象可能会出现问题，特别是如网络请求等较长时间的任务，是比较难保存的。

综上，数据尽量使用第二种 Bundle 来维持。长时间执行的任务，通过第一种 retained Fragment 来维持，旋转屏幕时，长时间的任务也没有被销毁，破费！

[本文demo](https://github.com/bluzwong/MonkeyKingBar/tree/1b1f8714bdfbe0d62838740eb7ecc186913f3fa4/app/src/main/java/com/github/bluzwong/monkeykingbar/example)
# 很惭愧 只做了一些微小的工作
# 蟹蟹大家