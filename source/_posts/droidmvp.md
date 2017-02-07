---
title: 浅谈安卓MVP
date: 2017-01-08 17:30:56
tags: [安卓,MVP]
categories: 理论
---
### 本文核心内容 ###
> **MVP主要目的和实现方式：**
> &#160; &#160; &#160; &#160;<font color="#f40">把业务逻辑抽象成Presenter层接口，把对视图的操作抽象成View接口，Presenter通过操作View来实现对视图的操作，View通过操作Presenter来实现对Model的操作和逻辑调用。</font>
> &#160; &#160; &#160; &#160;<font color="#f40">实际操作中，以一个实体类实现Presenter接口，而Activity或者Fragment实现View接口，在Activity中实例化Presenter对象，并把自己传递给Presenter，然后互相调用，通过接口来代码解耦，并且可以针对接口进行单元测试。</font>

_注：以上内容是本文核心内容，以下内容即是为了说明上面这段话，如果对MVP有一定了解的朋友，可以把省下时间去跟妹纸聊人生聊理想了。_(^_^)

### MVP 是个什么概念？ ###
&#160; &#160; &#160; &#160;最近有朋友问我说："MVP使用那种方式好？"，对于这样的问题，其实我也是被问懵了。因为笔者并不理解为什么会有这样的疑问，在进行了进一步的沟通后，原来他是被网上各种博客和谷歌的参考实现给弄糊涂了，因为实现方式上有一定的差异，导致他不知道该怎么选择自己写代码的方式了。
&#160; &#160; &#160; &#160;对于这样的情况，我不禁会去想一个问题，MVP是个什么样的概念，为什么会让他产生了不知道该选择那种实现方式去写代码的想法。
&#160; &#160; &#160; &#160;其实仔细思索一下不难发现，他的问题不仅仅是因为谷歌的官方参考和别的作者的写法上的差异，而是他其实并没有真正的理解**MVP是个什么样的东西**。
&#160; &#160; &#160; &#160;在具体讨论MVP是什么东西之前，我们先看下百度百科对于 “MVP 模式” 的[定义](http://baike.baidu.com/subview/7294/10754979.htm#viewPageContent)：
> &#160; &#160; &#160; &#160;mvp的全称为Model-View-Presenter，Model提供数据，View负责显示，Controller/Presenter负责逻辑的处理。MVP与MVC有着一个重大的区别：在MVP中View并不直接使用Model，它们之间的通信是通过Presenter (MVC中的Controller)来进行的，所有的交互都发生在Presenter内部，而在MVC中View会直接从Model中读取数据而不是通过 Controller。

&#160; &#160; &#160; &#160;做过开发的同学应该对MVC的Model View Controller这些概念有一定的了解，这里就不再赘述了，那我们这里来说说MVP与MVC的不同，简单来说就是**使用Presenter来进行View和Model间的通信**。那么这里我们先抛开其他的问题，先记住上面加黑字体的这句话。
&#160; &#160; &#160; &#160;我们做移动应用开发从创建一个项目，到设计界面，然后写代码直至项目运行起来的过程中，基本上都是将视图设计这一块挪到了代码外部放到一个单独的布局文件中这就是我们要讲的View(_包含不仅限于此_)，而我们在与服务器端程序通信的过程中，传递的数据对象，这里就是Model，那么我们的controller呢？在不使用任何设计模式的情况下，我们的Activity对象就是进行着Controller的工作。具体实现方式请看下面的例子。
```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/textView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

</RelativeLayout>
```
```java
public class SomeService {
    public String doSomeThing() {
        return "success";
    }
}
public class MainActivity extends Activity {
    private SomeService someService;
    private TextView mTextView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        someService = new SomeService();
        mTextView = (TextView) findViewById(R.id.textView);
        mTextView.setText(someService.doSomeThing());
    }
}
```
&#160; &#160; &#160; &#160;到这里是不是很熟悉，基本上是我们使用IDE创建一个项目的时候自动生成的代码，但是又稍微有点不同的是，我们把需要设置到TextView上的内容，封装成了对象SomeService的一个方法，这样就更符合我们写项目时候的一些行为，因为我们在具体项目中，有些数据是从后台获取，然后展示到界面上。这个是基本mvc的使用方式，也就是安卓平台给我们提供的mvc。
&#160; &#160; &#160; &#160;接下来我们看下谷歌官方提供的[MVP参考实现](https://github.com/googlesamples/android-architecture)，官方实现中包含几种不同的版本(todo‑mvp|todo‑mvp‑rxjava|等等)，我们这里以最基本实现方式(不使用其他库的情况下实现)进行分析。在这里我们只讨论与MVP相关的内容。先上代码，等下解释，可以在看代码的同时，运行官方示例对照查看。
![谷歌MVP参考实现包结构](/assets/res/20170108/mvp_google_code_structure.png)
&#160; &#160; &#160; &#160;简单说明下这些包的内容，data中是对实体类Model、获取Model的抽象层的封装与实现，与J2EE中DAO层的概念完全一致。谷歌这里是使用TasksDataSource将Task的CRUD等操作进行了一层抽象，TasksLocalDataSource提供了本地Sqlite的一个实现，TasksRemoteDataSource提供了一个内存LinkedHashMap的方式模拟服务器数据获取。通过TasksRepository持有TasksLocalDataSource和TasksRemoteDataSource，进行缓存控制，有兴趣的同学可以看下源码，具体奥妙之处还是真正看源码才能体会的到，这里就不展开讨论了。
&#160; &#160; &#160; &#160;util就是工具类的包，addedittask、statistics、taskdetail、tasks这些包，都是具体每个显示到用户面前界面的模块，每个包下面包含Activity、Fragment和具体Presenter的实现，这里针对tasks这个包下的内容来详细探讨，其他包中的内容大同小异，就不详细去讲了，具体可以看下源码。
```java
//这是 所有Presenter的基本接口
package com.example.android.architecture.blueprints.todoapp;
public interface BasePresenter {
    void start();
}

//这是 所有View的基本接口
package com.example.android.architecture.blueprints.todoapp;
public interface BaseView<T> {
    // 设置Presenter引用给View
    void setPresenter(T presenter);
}

//这是实体类对象，也就是我们说的Model
package com.example.android.architecture.blueprints.todoapp.data;
public final class Task {
    private final String mId;
    private final String mTitle;
    private final String mDescription;
    private final boolean mCompleted;
}

package com.example.android.architecture.blueprints.todoapp.tasks;
public interface TasksContract {
    // 这里是TaskActivity针对视图操作的抽象接口
    interface View extends BaseView<Presenter> {
        // 显示/隐藏 SwipeRefreshLayout 的刷新动画
        void setLoadingIndicator(boolean active);
        // 显示 给定的Task
        void showTasks(List<Task> tasks);
        // 显示 添加Task 控件
        void showAddTask();
        // 显示 Task详情UI
        void showTaskDetailsUi(String taskId);
        // 标记Task已经完成
        void showTaskMarkedComplete();
        //标记 Task 活跃
        void showTaskMarkedActive();
        // 清除所有已经完成的Task
        void showCompletedTasksCleared();
        // 显示 加载 Task 错误
        void showLoadingTasksError();
        // 显示 未加载 到 Task
        void showNoTasks();
        //显示 活跃的 过滤器 标签
        void showActiveFilterLabel();
        //显示 已完成 过滤器 标签
        void showCompletedFilterLabel();
        //显示 所有 过滤器 标签
        void showAllFilterLabel();
        //显示 无活跃的Task
        void showNoActiveTasks();
        //显示 无已完成Task
        void showNoCompletedTasks();
        // 显示保存成功信息
        void showSuccessfullySavedMessage();
        //当前 Fragment的 active状态
        boolean isActive();
        // 显示 过滤器 PopupMenu
        void showFilteringPopUpMenu();
    }
    // 这里是TaskActivity针对Model操作的抽象接口
    interface Presenter extends BasePresenter {
        // Activity Result 回调方法
        void result(int requestCode, int resultCode);
        // 加载Task
        void loadTasks(boolean forceUpdate);
        // 添加Task
        void addNewTask();
        // 显示Task详情
        void openTaskDetails(Task requestedTask);
        // 将Task设置为完成状态
        void completeTask(Task completedTask);
        // 将Task设置为活跃状态
        void activateTask(Task activeTask);
        // 清空所有已经完成的Task
        void clearCompletedTasks();
        //设置当前过滤器类型
        void setFiltering(TasksFilterType requestType);
        //获取当前过滤器类型
        TasksFilterType getFiltering();
    }
}
```
&#160; &#160; &#160; &#160;其实看到这里同学们应该发现了，我们先抛开所有的MVP等等概念，仅看这几个类的方法和名称，我们会发现几个特点：
- TasksContract.View 接口中封装的都是对视图(VIEW)的操作方法
- TasksContract.Presenter 接口中封装的都是对Model和逻辑的操作方法
- BaseView中封装了setPresenter方法，使VIEW获取Presenter引用
<span id = "a1"></span>

```java
//TasksActivity 
package com.example.android.architecture.blueprints.todoapp.tasks;
public class TasksActivity extends AppCompatActivity {
    // 具体Task界面使用到的Presenter
    // ① 同时请大家思考下，为什么这里使用的是TasksPresenter对象，而不是BaseView接口
    private TasksPresenter mTasksPresenter;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.tasks_act);
        // Set up the toolbar.
        //略过代码中的视图设置
        // Set up the navigation drawer.

        // 从布局获取 Fragment，并强转为TasksFragment
        TasksFragment tasksFragment = 
            (TasksFragment) getSupportFragmentManager()
            .findFragmentById(R.id.contentFrame);
        if (tasksFragment == null) {
            // 创建TasksFragment
            tasksFragment = TasksFragment.newInstance();
            //吧TasksFragment 添加到当前的界面上
            ActivityUtils.addFragmentToActivity(
                    getSupportFragmentManager(), tasksFragment, 
                    R.id.contentFrame);
        }

        // 创建 presenter
        mTasksPresenter = new TasksPresenter(
                // 这里就是刚才说到的Task的CRUD封装对象
                Injection.provideTasksRepository(getApplicationContext()), 
                tasksFragment);

        // Load previously saved state, if available.
        if (savedInstanceState != null) {
            TasksFilterType currentFiltering = 
                (TasksFilterType) savedInstanceState
                .getSerializable(CURRENT_FILTERING_KEY);
            // 这里是调用了 TasksPresenter 的设置 过滤器 方法
            mTasksPresenter.setFiltering(currentFiltering);
        }
    }
}
```
&#160; &#160; &#160; &#160;TasksActivity 做的事情比较简单：
- 创建视图
- 创建 TasksFragment，TasksFragment实现TasksContract.View接口
- 实例化 TasksPresenter ，TasksPresenter实现TasksContract.Presenter接口
- 调用 TasksPresenter 的 setFiltering方法，这是目前看到一次对Presenter的使用
<span id = "a2"></span>
```java
package com.example.android.architecture.blueprints.todoapp.tasks;
// 实现了TasksContract.View 接口的对象，在这里可以看到具体对Presenter的操作
public class TasksFragment extends Fragment implements TasksContract.View {
    // ② 请大家思考下，这里为什么使用的是 TasksContract.Presenter 而不是 TasksPresenter？
    private TasksContract.Presenter mPresenter;
    
    @Override
    public void onResume() {
        super.onResume();
        // 调用 TasksPresenter的start方法   1
        mPresenter.start();
    }

    @Override
    public void setPresenter(@NonNull TasksContract.Presenter presenter) {
        // 设置 TasksPresenter对象   2
        mPresenter = checkNotNull(presenter);
    }

    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        // 调用 TasksPresenter的result方法   3
        mPresenter.result(requestCode, resultCode);
    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // 省略具体创建 UI 视图的过程

        // Set up floating action button
        FloatingActionButton fab =
                (FloatingActionButton) getActivity().findViewById(R.id.fab_add_task);

        fab.setImageResource(R.drawable.ic_add);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // 调用 TasksPresenter的addNewTask方法   4
                mPresenter.addNewTask();
            }
        });
        swipeRefreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
            @Override
            public void onRefresh() {
                // 调用 TasksPresenter的 loadTasks 方法   5
                mPresenter.loadTasks(false);
            }
        });
        return root;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case R.id.menu_clear:
                // 调用 TasksPresenter的 clearCompletedTasks 方法   6
                mPresenter.clearCompletedTasks();
                break;
            case R.id.menu_filter:
                showFilteringPopUpMenu();
                break;
            case R.id.menu_refresh:
                // 调用 TasksPresenter的 loadTasks 方法   7
                mPresenter.loadTasks(true);
                break;
        }
        return true;
    }
    
    @Override
    public void showFilteringPopUpMenu() {
        PopupMenu popup = new PopupMenu(getContext(), getActivity().findViewById(R.id.menu_filter));
        popup.getMenuInflater().inflate(R.menu.filter_tasks, popup.getMenu());

        popup.setOnMenuItemClickListener(new PopupMenu.OnMenuItemClickListener() {
            public boolean onMenuItemClick(MenuItem item) {
                switch (item.getItemId()) {
                    case R.id.active:
                        // 调用 TasksPresenter的 loadTasks 方法   8
                        mPresenter.setFiltering(TasksFilterType.ACTIVE_TASKS);
                        break;
                    case R.id.completed:
                        // 调用 TasksPresenter的 loadTasks 方法   9
                        mPresenter.setFiltering(TasksFilterType.COMPLETED_TASKS);
                        break;
                    default:
                        // 调用 TasksPresenter的 loadTasks 方法   10
                        mPresenter.setFiltering(TasksFilterType.ALL_TASKS);
                        break;
                }
                // 调用 TasksPresenter的 loadTasks 方法   11
                mPresenter.loadTasks(false);
                return true;
            }
        });

        popup.show();
    }

    /**
     * Listener for clicks on tasks in the ListView.
     */
    TaskItemListener mItemListener = new TaskItemListener() {
        @Override
        public void onTaskClick(Task clickedTask) {
            // 调用 TasksPresenter的 loadTasks 方法   12
            mPresenter.openTaskDetails(clickedTask);
        }

        @Override
        public void onCompleteTaskClick(Task completedTask) {
            // 调用 TasksPresenter的 loadTasks 方法   13
            mPresenter.completeTask(completedTask);
        }

        @Override
        public void onActivateTaskClick(Task activatedTask) {
            // 调用 TasksPresenter的 loadTasks 方法   14
            mPresenter.activateTask(activatedTask);
        }
    };
}
```
&#160; &#160; &#160; &#160;上面是 TasksFragment 一部分相关内容，我们简单看下就可以发现：
- 生成基本视图
- 对 TasksPresenter 的调用相对比较多
- 把 TasksPresenter 当(就)作(是)一个普通对象来操作
```java
package com.example.android.architecture.blueprints.todoapp.tasks;
public class TasksPresenter implements TasksContract.Presenter {
    // 数据CRUD 操作
    private final TasksRepository mTasksRepository;
    // View 对象
    private final TasksContract.View mTasksView;

    public TasksPresenter(@NonNull TasksRepository tasksRepository, @NonNull TasksContract.View tasksView) {
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null");
        // 保存View的引用
        mTasksView = checkNotNull(tasksView, "tasksView cannot be null!");
        // 把自己的引用设置给View
        mTasksView.setPresenter(this);
    }
    @Override
    public void result(int requestCode, int resultCode) {
        // If a task was successfully added, show snackbar
        if (AddEditTaskActivity.REQUEST_ADD_TASK == requestCode && Activity.RESULT_OK == resultCode) {
            // 操作View
            mTasksView.showSuccessfullySavedMessage();
        }
    }
    
    private void loadTasks(boolean forceUpdate, final boolean showLoadingUI) {
        if (showLoadingUI) {
            // 操作View
            mTasksView.setLoadingIndicator(true);
        }
        // ... 省略其他无关代码
        mTasksRepository.getTasks(new TasksDataSource.LoadTasksCallback() {
            @Override
            public void onTasksLoaded(List<Task> tasks) {
                // ... 省略其他无关代码
                if (showLoadingUI) {
                    // 操作View
                    mTasksView.setLoadingIndicator(false);
                }
                // ... 省略其他无关代码
            }

            @Override
            public void onDataNotAvailable() {
                // The view may not be able to handle UI updates anymore
                // 操作View
                if (!mTasksView.isActive()) {
                    return;
                }
                // 操作View
                mTasksView.showLoadingTasksError();
            }
        });
    }

    private void processTasks(List<Task> tasks) {
        if (tasks.isEmpty()) {
            // Show a message indicating there are no tasks for that filter type.
            processEmptyTasks();
        } else {
            // 操作View
            mTasksView.showTasks(tasks);
            // Set the filter label's text.
            showFilterLabel();
        }
    }

    private void showFilterLabel() {
        switch (mCurrentFiltering) {
            case ACTIVE_TASKS:
                // 操作View
                mTasksView.showActiveFilterLabel();
                break;
            case COMPLETED_TASKS:
                // 操作View
                mTasksView.showCompletedFilterLabel();
                break;
            default:
                // 操作View
                mTasksView.showAllFilterLabel();
                break;
        }
    }

    private void processEmptyTasks() {
        switch (mCurrentFiltering) {
            case ACTIVE_TASKS:
                // 操作View
                mTasksView.showNoActiveTasks();
                break;
            case COMPLETED_TASKS:
                // 操作View
                mTasksView.showNoCompletedTasks();
                break;
            default:
                // 操作View
                mTasksView.showNoTasks();
                break;
        }
    }

    @Override
    public void addNewTask() {
        // 操作View
        mTasksView.showAddTask();
    }

    @Override
    public void openTaskDetails(@NonNull Task requestedTask) {
        checkNotNull(requestedTask, "requestedTask cannot be null!");
        // 操作View
        mTasksView.showTaskDetailsUi(requestedTask.getId());
    }

    @Override
    public void completeTask(@NonNull Task completedTask) {
        checkNotNull(completedTask, "completedTask cannot be null!");
        mTasksRepository.completeTask(completedTask);
        // 操作View
        mTasksView.showTaskMarkedComplete();
        loadTasks(false, false);
    }

    @Override
    public void activateTask(@NonNull Task activeTask) {
        checkNotNull(activeTask, "activeTask cannot be null!");
        mTasksRepository.activateTask(activeTask);
        // 操作View
        mTasksView.showTaskMarkedActive();
        loadTasks(false, false);
    }

    @Override
    public void clearCompletedTasks() {
        mTasksRepository.clearCompletedTasks();
        // 操作View
        mTasksView.showCompletedTasksCleared();
        loadTasks(false, false);
    }
}
```
上面是 TasksPresenter 的部分相关内容，我们简单看下就可以发现：
- 内部保留了TasksContract.View的引用，用来操作视图
- 对视图的操作，都是直接调用View的引用
- 操作视图时候，并不考虑在哪个线程中操作(**这里就是比较重要的，那么我们后台线程如果操作UI线程怎么办呢？实际上，这里并不需要后台线程去考虑这个问题，因为真正去操作UI的还是View，所以应该由View去控制**)

### 强化对于MVP的理解 ###
&#160; &#160; &#160; &#160;其实，MVP总体来说就是这个思想，毕竟MVP是个模式，你如何使用是具体的实现方式。我们再来强化下对于MVP的理解：
- MVP是个模式，并不是某个具体实现，其本质是面向接口编程
- 业务逻辑方法抽象成接口Presenter
- 视图操作方法抽象成接口View
- View和Presenter互相保存引用，在View中通过调用Presenter方法操作业务逻辑，在Presenter中调用View方法操作视图

### <font color="#f40">对于①②解释：</font> ###
- 对于[①](#a1)，这里是把TasksContract.Presenter进行实例化的地方，其实是已经很明确知道需要使用到的Presenter对象的类型了，所以直接使用实体类就可以了。因为当你切换到不同的Presenter时候，依然是需要修改这里的代码的。
- 对于[②](#a2)，这里是视图对逻辑引用的保存，但是这里其实是并不清楚具体逻辑的实现，针对接口编程当然最好啦，当你替换下不同的实现，并不需要修改这里的代码。        


