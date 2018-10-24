---
title: Room 是如何做到响应式的？
date: 2018-10-23 17:05:08
tags:
---

#Room 是如何做到响应式的？

##前言
因为我们常用的Rxjava，所以这里会结合RxRoom做分析，所以需要你有Rxjava相关的知识储备。

完整源码参见[googlesamples](https://github.com/googlesamples/android-architecture-components/tree/master/BasicRxJavaSample/app/src/main/java/com/example/android/observability/persistence)，可以自己跑一下，体验一下。

<!--more-->

##简单用法

我在阅读一份代码时最喜欢的入手点就是先看怎么用，从用法去往前推导。

看下面的简单代码,也就是所谓的三大组件。

```java
//首先是实体类对应到数据库也就是一个表
@Entity(tableName = "users")
public class User {

  @PrimaryKey
  @ColumnInfo(name = "userid")
  private String mId;
  
  @ColumnInfo(name = "username")
  private String mUserName;

  ...
}

//数据操作类
@Dao
public interface UserDao {
  @Query("SELECT * FROM users LIMIT 1")
  Flowable<User> getUser();

   @Insert(onConflict = OnConflictStrategy.REPLACE)
  void insertUser(User user);
  ...
}

//Database
@Database(
    entities = {User.class,UserGroup.class},
    version = 1)
public abstract class UsersDatabase extends RoomDatabase {

  private static volatile UsersDatabase INSTANCE;

  public abstract UserDao userDao();

  public static UsersDatabase getInstance(Context context) {
    if (INSTANCE == null) {
      synchronized (UsersDatabase.class) {
        if (INSTANCE == null) {
          INSTANCE =
              Room.databaseBuilder(
                      context.getApplicationContext(), UsersDatabase.class, "Sample.db")
                  .build();
        }
      }
    }
    return INSTANCE;
  }
}
```
定义好上面的3个东西，我们就能直接用来`增删改查`了。

```java

val database = UsersDatabase.getInstance(this)

//插入一条数据
val user = User("kingty")
database.userDao().insertUser(user)

//监听一个查询的实时变化
database.userDao().getUser().subscribe {
	Log.d("TAG", "table changed => " + it.userName + " => " + it.id)
}

```
用法非常简单，做一些注解，它就自动帮你完成了`DAO`中的所有操作，并且还可以监听数据库的变化实时更新数据。这一切看起来比较梦幻。那么问题来了，它是怎么做到这一切的？

## 怎么通过注解就可以操作数据库了？


其实`ORM`的本质就是用一些手段把你的`数据类`和`操作`变成`SQL语句`而已。而这一切无非就是两种手段，编译时做（apt）或者运行时做（reflect）。出于效率考虑现在大部分都是编译时做。编译以下，所以我们来翻一翻`build`目录，看一下Room给我们生成了一些什么东西。


- build
  - generate
     - source
         - apt
             - com.kingty.roomtest
                 - **UserDao_Impl.java**
                 - **UsersDatabase_Impl.java**
                            
就发现在上面这个目录下生成了两个类，这里我们不深究这里是怎么生成这两个类的，说起来可以说两天两夜。其实是我也不懂。有兴趣的同学应该早就知道了，没兴趣的我说不说也无所谓了。总之就是，编译的时候通过我们刚才那些个注解给我们生成了真正的实现类，帮助我们完成了我们想要的操作。


来让我们看一下里面都生成了一些什么？

先看一下`UserDao_Impl.java`这个类做了什么？

```java
x@SuppressWarnings("unchecked")
public final class UserDao_Impl implements UserDao {
  private final RoomDatabase __db;
  private final EntityInsertionAdapter __insertionAdapterOfUser;
  private final SharedSQLiteStatement __preparedStmtOfDeleteAllUsers;

  public UserDao_Impl(RoomDatabase __db) {
    this.__db = __db;
    this.__insertionAdapterOfUser = new EntityInsertionAdapter<User>(__db) {
      @Override
      public String createQuery() {
        return "INSERT OR REPLACE INTO `users`(`userid`,`username`) VALUES (?,?)";
      }
      @Override
      public void bind(SupportSQLiteStatement stmt, User value) {
        if (value.getId() == null) {
          stmt.bindNull(1);
        } else {
          stmt.bindString(1, value.getId());
        }
        if (value.getUserName() == null) {
          stmt.bindNull(2);
        } else {
          stmt.bindString(2, value.getUserName());
        }
      }
    };
    this.__preparedStmtOfDeleteAllUsers = new SharedSQLiteStatement(__db) {
      @Override
      public String createQuery() {
        final String _query = "DELETE FROM Users";
        return _query;
      }
    };
  }

  @Override
  public void insertUser(User user) {
    __db.beginTransaction();
    try {
      __insertionAdapterOfUser.insert(user);
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
    }
  }

  @Override
  public void deleteAllUsers() {
    final SupportSQLiteStatement _stmt = __preparedStmtOfDeleteAllUsers.acquire();
    __db.beginTransaction();
    try {
      _stmt.executeUpdateDelete();
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
      __preparedStmtOfDeleteAllUsers.release(_stmt);
    }
  }

  @Override
  public Flowable<User> getUser() {
    final String _sql = "SELECT * FROM users LIMIT 1";
    final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire(_sql, 0);
    return RxRoom.createFlowable(__db, new String[]{"users"}, new Callable<User>() {
      @Override
      public User call() throws Exception {
        final Cursor _cursor = DBUtil.query(__db, _statement, false);
        try {
          final int _cursorIndexOfMId = _cursor.getColumnIndexOrThrow("userid");
          final int _cursorIndexOfMUserName = _cursor.getColumnIndexOrThrow("username");
          final User _result;
          if(_cursor.moveToFirst()) {
            final String _tmpMId;
            _tmpMId = _cursor.getString(_cursorIndexOfMId);
            final String _tmpMUserName;
            _tmpMUserName = _cursor.getString(_cursorIndexOfMUserName);
            _result = new User(_tmpMId,_tmpMUserName);
          } else {
            _result = null;
          }
          return _result;
        } finally {
          _cursor.close();
        }
      }

      @Override
      protected void finalize() {
        _statement.release();
      }
    });
  }
}

```

首先，在初始化`UserDao_Impl `的时候通过`EntityInsertionAdapter `帮你创建了插入数据的`adapter`，里面有插入数据的sql模板，和绑定数据的方法。通俗一点说就是在这里帮你拼好了插入数据的SQL语句。接下来就是帮你实现了你在`UserDao`中定义的接口。

`增删改`的套路大概类似，都是`__db.beginTransaction();`然后执行拼好的SQL语句，然后`__db.endTransaction();`注意此DB非彼DB，这个DB是封装过的RoomDatabase。后面我们会着重讲一下这个`beginTransaction`和`endTransaction`,他们还是比较重要的一个环节。 **`重要环节一`**。

`查`的套路就不一样了,为什么它不一样，`public Flowable<User> getUser()`这个方法明显看起来大坨一些，这就是它为什么不一样。简单阅读一下，它其实利用RxRoom创建了一个`Flowable`,其中有3个参数我们注意一下，第一个是`__db`也就是database,第二个是`new String[]{"users"}`,是一个表名的数组,第3个就是查询完成之后组装成`User`的回调。特别注意的是第二个参数，这个表名的数组的作用。这是**`重要环节二`**。



再看看`UsersDatabase_Impl.java`这个类的生成了什么？

```java

@SuppressWarnings("unchecked")
public final class UsersDatabase_Impl extends UsersDatabase {
  private volatile UserDao _userDao;

  @Override
  protected SupportSQLiteOpenHelper createOpenHelper(DatabaseConfiguration configuration) {
    final SupportSQLiteOpenHelper.Callback _openCallback = new RoomOpenHelper(configuration, new RoomOpenHelper.Delegate(1) {
      @Override
      public void createAllTables(SupportSQLiteDatabase _db) {
        _db.execSQL("CREATE TABLE IF NOT EXISTS `users` (`userid` TEXT NOT NULL, `username` TEXT, PRIMARY KEY(`userid`))");
        _db.execSQL("CREATE TABLE IF NOT EXISTS `usergroups` (`userGroupId` TEXT NOT NULL, `groupName` TEXT, PRIMARY KEY(`userGroupId`))");
        _db.execSQL("CREATE TABLE IF NOT EXISTS room_master_table (id INTEGER PRIMARY KEY,identity_hash TEXT)");
        _db.execSQL("INSERT OR REPLACE INTO room_master_table (id,identity_hash) VALUES(42, \"8890a9730e4846f27da03382221fc877\")");
      }

      @Override
      public void dropAllTables(SupportSQLiteDatabase _db) {
        _db.execSQL("DROP TABLE IF EXISTS `users`");
        _db.execSQL("DROP TABLE IF EXISTS `usergroups`");
      }

      @Override
      protected void onCreate(SupportSQLiteDatabase _db) {
        if (mCallbacks != null) {
          for (int _i = 0, _size = mCallbacks.size(); _i < _size; _i++) {
            mCallbacks.get(_i).onCreate(_db);
          }
        }
      }

      @Override
      public void onOpen(SupportSQLiteDatabase _db) {
        mDatabase = _db;
        internalInitInvalidationTracker(_db);
        if (mCallbacks != null) {
          for (int _i = 0, _size = mCallbacks.size(); _i < _size; _i++) {
            mCallbacks.get(_i).onOpen(_db);
          }
        }
      }

      @Override
      public void onPreMigrate(SupportSQLiteDatabase _db) {
        DBUtil.dropFtsSyncTriggers(_db);
      }

      @Override
      public void onPostMigrate(SupportSQLiteDatabase _db) {
      }

      @Override
      protected void validateMigration(SupportSQLiteDatabase _db) {
        final HashMap<String, TableInfo.Column> _columnsUsers = new HashMap<String, TableInfo.Column>(2);
        _columnsUsers.put("userid", new TableInfo.Column("userid", "TEXT", true, 1));
        _columnsUsers.put("username", new TableInfo.Column("username", "TEXT", false, 0));
        final HashSet<TableInfo.ForeignKey> _foreignKeysUsers = new HashSet<TableInfo.ForeignKey>(0);
        final HashSet<TableInfo.Index> _indicesUsers = new HashSet<TableInfo.Index>(0);
        final TableInfo _infoUsers = new TableInfo("users", _columnsUsers, _foreignKeysUsers, _indicesUsers);
        final TableInfo _existingUsers = TableInfo.read(_db, "users");
        if (! _infoUsers.equals(_existingUsers)) {
          throw new IllegalStateException("Migration didn't properly handle users(com.kingty.roomtest.User).\n"
                  + " Expected:\n" + _infoUsers + "\n"
                  + " Found:\n" + _existingUsers);
        }
        final HashMap<String, TableInfo.Column> _columnsUsergroups = new HashMap<String, TableInfo.Column>(2);
        _columnsUsergroups.put("userGroupId", new TableInfo.Column("userGroupId", "TEXT", true, 1));
        _columnsUsergroups.put("groupName", new TableInfo.Column("groupName", "TEXT", false, 0));
        final HashSet<TableInfo.ForeignKey> _foreignKeysUsergroups = new HashSet<TableInfo.ForeignKey>(0);
        final HashSet<TableInfo.Index> _indicesUsergroups = new HashSet<TableInfo.Index>(0);
        final TableInfo _infoUsergroups = new TableInfo("usergroups", _columnsUsergroups, _foreignKeysUsergroups, _indicesUsergroups);
        final TableInfo _existingUsergroups = TableInfo.read(_db, "usergroups");
        if (! _infoUsergroups.equals(_existingUsergroups)) {
          throw new IllegalStateException("Migration didn't properly handle usergroups(com.kingty.roomtest.UserGroup).\n"
                  + " Expected:\n" + _infoUsergroups + "\n"
                  + " Found:\n" + _existingUsergroups);
        }
      }
    }, "8890a9730e4846f27da03382221fc877", "1fdb937160bfb054175cfe5daf922b3b");
    final SupportSQLiteOpenHelper.Configuration _sqliteConfig = SupportSQLiteOpenHelper.Configuration.builder(configuration.context)
        .name(configuration.name)
        .callback(_openCallback)
        .build();
    final SupportSQLiteOpenHelper _helper = configuration.sqliteOpenHelperFactory.create(_sqliteConfig);
    return _helper;
  }

  @Override
  protected InvalidationTracker createInvalidationTracker() {
    final HashMap<String, String> _shadowTablesMap = new HashMap<String, String>(0);
    HashMap<String, Set<String>> _viewTables = new HashMap<String, Set<String>>(0);
    return new InvalidationTracker(this, _shadowTablesMap, _viewTables, "users","usergroups");
  }

  @Override
  public void clearAllTables() {
    super.assertNotMainThread();
    final SupportSQLiteDatabase _db = super.getOpenHelper().getWritableDatabase();
    try {
      super.beginTransaction();
      _db.execSQL("DELETE FROM `users`");
      _db.execSQL("DELETE FROM `usergroups`");
      super.setTransactionSuccessful();
    } finally {
      super.endTransaction();
      _db.query("PRAGMA wal_checkpoint(FULL)").close();
      if (!_db.inTransaction()) {
        _db.execSQL("VACUUM");
      }
    }
  }

  @Override
  public UserDao userDao() {
    if (_userDao != null) {
      return _userDao;
    } else {
      synchronized(this) {
        if(_userDao == null) {
          _userDao = new UserDao_Impl(this);
        }
        return _userDao;
      }
    }
  }
}

```
这个类中逻辑比较清晰。首先是创建了一个`SupportSQLiteOpenHelper `来帮你拼了一些必要的SQL语句，比如create table,drop table,open和migrate迁移数据等等。后面还有一个初始化真正的DAO实现类`UserDao_Impl`和删除相关的表数据的方法`clearAllTables `

这些都是一些比较好理解的。然后我们会看到一个我们不好理解的方法`createInvalidationTracker ()`,这个是用来做什么的？我们先把这个疑问留下，叫做  **重要环节三**


##正式初略阅读Room的源码

下面我们提出几个问题：

- 在上面生成的代码中我们看到操作的执行都是通过一个叫`RoomDatabase`的`_db`来做的，那`RoomDatabase`是什么？
- 重要环节一，在执行前后`beginTransaction`和`endTransaction`做了什么?
- 重要环节二，创建`Flowable`的时候做了什么，为什么需要`table names`?
- 重要环节三，`createInvalidationTracker（）`是做什么用的？
- 最后，串联起上面的问题，当一个表发生更改，监听一个查询的实时变化是怎么做到的？

带着这几个问题，我们大体的去阅读一下源码。阅读过程中我们有一个原则，就是先不要特别在意细节，先捋通大概的逻辑流程。如果你对细节感兴趣，再去扣细节。

在项目中加以下引用

```
implementation 'androidx.room:room-runtime:2.1.0-alpha01'
    
annotationProcessor 'androidx.room:room-compiler:2.1.0-alpha01'

implementation 'androidx.room:room-rxjava2:2.1.0-alpha01'
```
编译之后我们可以在`External Libraries`目录下看到以下几个包：

- androidx.room:room-commom
- androidx.room:room-runtime
- androidx.room:room-rxjava
- androidx.sqlite:sqlite
- androidx.sqlite:sqlite-framework


我先简单的介绍下这几个包大概是做什么的。

`androidx.sqlite:sqlite`这个包主要是重新定义了一层SQLite的Support接口。
`androidx.sqlite:sqlite-framework`这个包主要是利用原有的android的`Sqlite`相关的API实现了上面定义的接口。
这两个包主要是对原有的API做了一层代理封装，我的理解是便于扩展。因此我们在看`Room`代码的时候这部分代码大概浏览一下就OK，不必深究。

`androidx.room:room-commom`包中定义了一些公共的属性，和我们用到的所有的注解。
`androidx.room:room-runtime`是我们需要主要阅读的逻辑所在的包，Room的核心逻辑都在这个包中。
`androidx.room:room-rxjava`当我们需要返回一个Rx包装过的结果的时候，需要这个包。里面就是一个重要类`RxRoom.java`用来把Query包装成一个可观察的对象。


下面我们带着上面的问题来看一下代码。

**`RoomDatabase`是做什么的？**

代码太长就不全贴，我们看一下它持有的成员变量

```
protected volatile SupportSQLiteDatabase mDatabase;
private Executor mQueryExecutor;
private SupportSQLiteOpenHelper mOpenHelper;
private final InvalidationTracker mInvalidationTracker;
private boolean mAllowMainThreadQueries;
boolean mWriteAheadLoggingEnabled;
```
它其实是对数据库的进一步封装，利用真正的`SupportSQLiteDatabase `和你`UsersDatabase_Impl`自动生成的`createOpenHelper() `提供的`SupportSQLiteOpenHelper `来操作数据库。进一步封装了`Transcation`并封装了一些其他的逻辑。

**`beginTransaction`和`endTransaction`做了什么?**

实际上这也是上一个问题的一部分，先看看`beginTransaction `的代码

```java
 /**
     * Wrapper for {@link SupportSQLiteDatabase#beginTransaction()}.
     */
    public void beginTransaction() {
        assertNotMainThread();//禁止主线程执行
        SupportSQLiteDatabase database = mOpenHelper.getWritableDatabase();//拿到真正的数据库对象
        mInvalidationTracker.syncTriggers(database);//??
        database.beginTransaction();//开启事务
    }
```
从上面的代码来看其他3句都非常好理解，正常的数据库事务流程，但是在开启事务之前做了一个操作`mInvalidationTracker.syncTriggers(database);`

我们先不忙解释这个是什么意思。我们在看看`endTransaction `

```java
 /**
     * Wrapper for {@link SupportSQLiteDatabase#endTransaction()}.
     */
    public void endTransaction() {
        mOpenHelper.getWritableDatabase().endTransaction();//结束事务
        if (!inTransaction()) {
            // enqueue refresh only if we are NOT in a transaction. Otherwise, wait for the last
            // endTransaction call to do it.
            mInvalidationTracker.refreshVersionsAsync();
        }
    }
```
第一句是正常的结束事务的语句，但是结束之后等待最后一个事务结束，会做一个操作`mInvalidationTracker.refreshVersionsAsync();`

也就是说在开启事务之前，结束事务之后都调用了InvalidationTracker做了一些逻辑，再结合上面的第四个问题，重要环节三`createInvalidationTracker（）是做什么用的？`,一切问题都指向了`InvalidationTracker`。


`InvalidationTracker`这个类是什么作用，我们先从上面的第四个问题看起。






**`createInvalidationTracker（）`是做什么用的？**

下面是在自动生成的`UsersDatabase_Impl.java`类中的方法

```


  @Override
  protected InvalidationTracker createInvalidationTracker() {
    final HashMap<String, String> _shadowTablesMap = new HashMap<String, String>(0);
    HashMap<String, Set<String>> _viewTables = new HashMap<String, Set<String>>(0);
    return new InvalidationTracker(this, _shadowTablesMap, _viewTables, "users","usergroups");
  }
```

在RoomDatabase在被初始化的时候调用这个方法赋值给成员变量

```
/**
     * Creates a RoomDatabase.
     * <p>
     * You cannot create an instance of a database, instead, you should acquire it via
     * {@link Room#databaseBuilder(Context, Class, String)} or
     * {@link Room#inMemoryDatabaseBuilder(Context, Class)}.
     */
    public RoomDatabase() {
        mInvalidationTracker = createInvalidationTracker();
    }
```
因此回答上面的问题就是`createInvalidationTracker（）`给RoomDatabase提供了一个mInvalidationTracker实例。

**mInvalidationTracker起的作用是什么？**

我们来看一下`RoomDatabase.java`中的调用流程。

```
1.初始化

public RoomDatabase() {
        mInvalidationTracker = createInvalidationTracker();
}

2.init中调用mInvalidationTracker.startMultiInstanceInvalidation(configuration.context,configuration.name);

@CallSuper
    public void init(@NonNull DatabaseConfiguration configuration) {
        mOpenHelper = createOpenHelper(configuration);
        boolean wal = false;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
            wal = configuration.journalMode == JournalMode.WRITE_AHEAD_LOGGING;
            mOpenHelper.setWriteAheadLoggingEnabled(wal);
        }
        mCallbacks = configuration.callbacks;
        mQueryExecutor = configuration.queryExecutor;
        mAllowMainThreadQueries = configuration.allowMainThreadQueries;
        mWriteAheadLoggingEnabled = wal;
        if (configuration.multiInstanceInvalidation) {
            mInvalidationTracker.startMultiInstanceInvalidation(configuration.context,
                    configuration.name);
        }
    }
 
 3.每次Transaction 开始之前调用mInvalidationTracker.syncTriggers
 
  /**
     * Wrapper for {@link SupportSQLiteDatabase#beginTransaction()}.
     */
    public void beginTransaction() {
        assertNotMainThread();
        SupportSQLiteDatabase database = mOpenHelper.getWritableDatabase();
        mInvalidationTracker.syncTriggers(database);
        database.beginTransaction();
    }

4. 最后一个Transaction 结束之后调用mInvalidationTracker.refreshVersionsAsync
    /**
     * Wrapper for {@link SupportSQLiteDatabase#endTransaction()}.
     */
    public void endTransaction() {
        mOpenHelper.getWritableDatabase().endTransaction();
        if (!inTransaction()) {
            // enqueue refresh only if we are NOT in a transaction. Otherwise, wait for the last
            // endTransaction call to do it.
            mInvalidationTracker.refreshVersionsAsync();
        }
    }
 
 5，close数据库的时候mInvalidationTracker.stopMultiInstanceInvalidation();与第2步对应。
 
  /**
     * Closes the database if it is already open.
     */
    public void close() {
        if (isOpen()) {
            try {
                mCloseLock.lock();
                mInvalidationTracker.stopMultiInstanceInvalidation();
                mOpenHelper.close();
            } finally {
                mCloseLock.unlock();
            }
        }
    }   

    
```
上面就是InvalidationTracker在RoomDatabase中的整个生命周期中的调用情况。从代码上来看它其实是在track整个数据的更改情况，因为它在每个transcation前后做了一些调用。结合上面最后的一个问题`当一个表发生更改，监听一个查询的实时变化是怎么做到的`。大概可以猜测出来这个类的主要作用是来保证数据发生更改的时候，保证可以通知到这个表上其他的Query。



**怎么样实现的监听？**

我们发现上面还有一个问题我们还没有提到`重要环节二，创建Flowable的时候做了什么，为什么需要table names?`

我们从这里入手讲起。先看下面`RxRoom`中的代码

```
 public static Flowable<Object> createFlowable(final RoomDatabase database,
            final String... tableNames) {
        return Flowable.create(new FlowableOnSubscribe<Object>() {
            @Override
            public void subscribe(final FlowableEmitter<Object> emitter) throws Exception {
                final InvalidationTracker.Observer observer = new InvalidationTracker.Observer(
                        tableNames) {
                    @Override
                    public void onInvalidated(@androidx.annotation.NonNull Set<String> tables) {
                        if (!emitter.isCancelled()) {
                            emitter.onNext(NOTHING);
                        }
                    }
                };
                if (!emitter.isCancelled()) {
                    database.getInvalidationTracker().addObserver(observer);
                    emitter.setDisposable(Disposables.fromAction(new Action() {
                        @Override
                        public void run() throws Exception {
                            database.getInvalidationTracker().removeObserver(observer);
                        }
                    }));
                }

                // emit once to avoid missing any data and also easy chaining
                if (!emitter.isCancelled()) {
                    emitter.onNext(NOTHING);
                }
            }
        }, BackpressureStrategy.LATEST);
    }

    /**
     * Helper method used by generated code to bind a Callable such that it will be run in
     * our disk io thread and will automatically block null values since RxJava2 does not like null.
     *
     * @hide
     */
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static <T> Flowable<T> createFlowable(final RoomDatabase database,
            final String[] tableNames, final Callable<T> callable) {
        Scheduler scheduler = Schedulers.from(database.getQueryExecutor());
        final Maybe<T> maybe = Maybe.fromCallable(callable);
        return createFlowable(database, tableNames)
                .observeOn(scheduler)
                .flatMapMaybe(new Function<Object, MaybeSource<T>>() {
                    @Override
                    public MaybeSource<T> apply(Object o) throws Exception {
                        return maybe;
                    }
                });
    }
```
在上面生成的`UserDao_Impl.java`类中`getUser（）`这个方法中调用的`createFlowable `这个方法，也就是上面的第二个方法，它实际上调用的上面的第一个方法flatmap到这个本次的查询。也就是说只要第一个方法中的Flowable发射一次数据，那么这个查询就会执行一次，并返回结果（也就是执行这个callable）。这里应该就能看出一点端倪，其实第一个方法就是创建出来一个观察这个表变化的观察者`InvalidationTracker.Observer`并把它添加到InvalidationTracker的观察者列表中去，因为一个表肯定不止一个观察者，所有的Query应该都需要观察表的更改。也就是上面的这行代码`database.getInvalidationTracker().addObserver(observer);`到这里`RxRoom`这个类的使命就完成了，他就是这样一个简单的功能，后面你也不需要再关心它。


`InvalidationTracker.Observer`是一个静态类，就注意一下其中的一个方法

```
 /**
         * Called when one of the observed tables is invalidated in the database.
         *
         * @param tables A set of invalidated tables. This is useful when the observer targets
         *               multiple tables and you want to know which table is invalidated. This will
         *               be names of underlying tables when you are observing views.
public abstract void onInvalidated(@NonNull Set<String> tables);
```
从备注上已经写的很清楚了，就是表发生更改状态的时候会调用这个方法，`emitter`就会发射数据，通知`Query`去requery.
我们着重看一下`addObserver `这个方法干了什么？

```
 @WorkerThread
    public void addObserver(@NonNull Observer observer) {
        final String[] tableNames = resolveViews(observer.mTables);
        int[] tableIds = new int[tableNames.length];
        final int size = tableNames.length;
        long[] versions = new long[tableNames.length];

        // TODO sync versions ?
        for (int i = 0; i < size; i++) {
            Integer tableId = mTableIdLookup.get(tableNames[i].toLowerCase(Locale.US));
            if (tableId == null) {
                throw new IllegalArgumentException("There is no table with name " + tableNames[i]);
            }
            tableIds[i] = tableId;
            versions[i] = mMaxVersion;
        }
        ObserverWrapper wrapper = new ObserverWrapper(observer, tableIds, tableNames, versions);
        ObserverWrapper currentObserver;
        synchronized (mObserverMap) {
            currentObserver = mObserverMap.putIfAbsent(observer, wrapper);
        }
        if (currentObserver == null && mObservedTableTracker.onAdded(tableIds)) {
            syncTriggers();
        }
    }

```

首先对`Observer`做了一层包装，主要就是包装了当表发生变化的时候通过各种方式去通知也就是执行`mObserver.onInvalidated(invalidatedTables);`,接下来，把包装后的wrapper放进map里。然后在满足特定条件下会执行` syncTriggers();`这个似曾相识，在上面`RoomDatabase`开始一个事务之前也执行这个方法。我们来仔细看看这个方法做了什么。

```
void syncTriggers(SupportSQLiteDatabase database) {
        if (database.inTransaction()) {
            // we won't run this inside another transaction.
            return;
        }
        try {
            // This method runs in a while loop because while changes are synced to db, another
            // runnable may be skipped. If we cause it to skip, we need to do its work.
            while (true) {
                Lock closeLock = mDatabase.getCloseLock();
                closeLock.lock();
                try {
                    // there is a potential race condition where another mSyncTriggers runnable
                    // can start running right after we get the tables list to sync.
                    final int[] tablesToSync = mObservedTableTracker.getTablesToSync();
                    if (tablesToSync == null) {
                        return;
                    }
                    final int limit = tablesToSync.length;
                    try {
                        database.beginTransaction();
                        for (int tableId = 0; tableId < limit; tableId++) {
                            switch (tablesToSync[tableId]) {
                                case ObservedTableTracker.ADD:
                                    startTrackingTable(database, tableId);
                                    break;
                                case ObservedTableTracker.REMOVE:
                                    stopTrackingTable(database, tableId);
                                    break;
                            }
                        }
                        database.setTransactionSuccessful();
                    } finally {
                        database.endTransaction();
                    }
                    mObservedTableTracker.onSyncCompleted();
                } finally {
                    closeLock.unlock();
                }
            }
        } catch (IllegalStateException | SQLiteException exception) {
            // may happen if db is closed. just log.
            Log.e(Room.LOG_TAG, "Cannot run invalidation tracker. Is the db closed?",
                    exception);
        }
    }
```
这个方法看起来很长，其实是在做一件事.`ObservedTableTracker`维护了一个需要被观察的表的列表，就是发现有新的表需要被观察就执行` startTrackingTable(database, tableId);`,有表不需要被观察了就执行`stopTrackingTable(database, tableId);`。

继续往下看，看看这两个方法做了什么？

```
private void stopTrackingTable(SupportSQLiteDatabase writableDb, int tableId) {
        final String tableName = mShadowTableLookup.get(tableId, mTableNames[tableId]);
        StringBuilder stringBuilder = new StringBuilder();
        for (String trigger : TRIGGERS) {
            stringBuilder.setLength(0);
            stringBuilder.append("DROP TRIGGER IF EXISTS ");
            appendTriggerName(stringBuilder, tableName, trigger);
            writableDb.execSQL(stringBuilder.toString());
        }
    }

    private void startTrackingTable(SupportSQLiteDatabase writableDb, int tableId) {
        final String tableName = mShadowTableLookup.get(tableId, mTableNames[tableId]);
        StringBuilder stringBuilder = new StringBuilder();
        for (String trigger : TRIGGERS) {
            stringBuilder.setLength(0);
            stringBuilder.append("CREATE TEMP TRIGGER IF NOT EXISTS ");
            appendTriggerName(stringBuilder, tableName, trigger);
            stringBuilder.append(" AFTER ")
                    .append(trigger)
                    .append(" ON `")
                    .append(tableName)
                    .append("` BEGIN INSERT OR REPLACE INTO ")
                    .append(UPDATE_TABLE_NAME)
                    .append(" VALUES(null, ")
                    .append(tableId)
                    .append("); END");
            writableDb.execSQL(stringBuilder.toString());
        }
    }
```
插曲： InvalidationTracker自己维护了一个叫`room_table_modification_log`的表，有两个字段，一个是`version`它是自增的，还有一个是table_id，是被观察的表的标识。

其实就是当需要去观察一个表的时候`startTrackingTable ()`就在数据库上创建了三个数据库的 `Trigger` 。关于`Trigger`是什么这是数据库基础知识，请自备。也就是说，只要在这个表上发生了插入修改或者删除，就会往`room_table_modification_log`表里面插入一条数据`INSERT OR REPLACE INTO room_table_modification_log VALUES(null, table_id)`。

当不需要观察一个表的时候，就通过`stopTrackingTable `把这三个`Trigger`删除掉。

以上就是我们在创建一个Query做的事情。
我们先对创建一个Query的流程做一个小的总结：

- 通过自动生成的代码创建一个Flowable
- RxRoom会根据这个Flowable创建一个InvalidationTracker.Observer
- InvalidationTracker把这个Observer加到自己的观察列表中
- 如果之前没有人观察过这个表，会去创建这个表上修改的Trigger



到这里，我们似乎应该有一点头绪了，既然每次有数据更新的时候就会往这个表中插入一条数据，那在每一个`Trascation`结束之后去查这个表就应该可以知道哪些表上的Query可以更新。所以我们回到上面的`RoomDatabase`中看看`endTrasction`之后的`mInvalidationTracker.refreshVersionsAsync();`到底做了什么？

```
/**
     * Enqueues a task to refresh the list of updated tables.
     * <p>
     * This method is automatically called when {@link RoomDatabase#endTransaction()} is called but
     * if you have another connection to the database or directly use {@link
     * SupportSQLiteDatabase}, you may need to call this manually.
     */
    @SuppressWarnings("WeakerAccess")
    public void refreshVersionsAsync() {
        // TODO we should consider doing this sync instead of async.
        if (mPendingRefresh.compareAndSet(false, true)) {
            mDatabase.getQueryExecutor().execute(mRefreshRunnable);
        }
    }
    
    
     @VisibleForTesting
    Runnable mRefreshRunnable = new Runnable() {
        @Override
        public void run() {
            final Lock closeLock = mDatabase.getCloseLock();
            boolean hasUpdatedTable = false;
            try {
                closeLock.lock();

                if (!ensureInitialization()) {
                    return;
                }

                if (!mPendingRefresh.compareAndSet(true, false)) {
                    // no pending refresh
                    return;
                }

                if (mDatabase.inTransaction()) {
                    // current thread is in a transaction. when it ends, it will invoke
                    // refreshRunnable again. mPendingRefresh is left as false on purpose
                    // so that the last transaction can flip it on again.
                    return;
                }

                mCleanupStatement.executeUpdateDelete();
                mQueryArgs[0] = mMaxVersion;
                if (mDatabase.mWriteAheadLoggingEnabled) {
                    // This transaction has to be on the underlying DB rather than the RoomDatabase
                    // in order to avoid a recursive loop after endTransaction.
                    SupportSQLiteDatabase db = mDatabase.getOpenHelper().getWritableDatabase();
                    try {
                        db.beginTransaction();
                        hasUpdatedTable = checkUpdatedTable();
                        db.setTransactionSuccessful();
                    } finally {
                        db.endTransaction();
                    }
                } else {
                    hasUpdatedTable = checkUpdatedTable();
                }
            } catch (IllegalStateException | SQLiteException exception) {
                // may happen if db is closed. just log.
                Log.e(Room.LOG_TAG, "Cannot run invalidation tracker. Is the db closed?",
                        exception);
            } finally {
                closeLock.unlock();
            }
            if (hasUpdatedTable) {
                synchronized (mObserverMap) {
                    for (Map.Entry<Observer, ObserverWrapper> entry : mObserverMap) {
                        entry.getValue().notifyByTableVersions(mTableVersions);
                    }
                }
            }
        }

        private boolean checkUpdatedTable() {
            boolean hasUpdatedTable = false;
            Cursor cursor = mDatabase.query(SELECT_UPDATED_TABLES_SQL, mQueryArgs);
            //noinspection TryFinallyCanBeTryWithResources
            try {
                while (cursor.moveToNext()) {
                    final long version = cursor.getLong(0);
                    final int tableId = cursor.getInt(1);

                    mTableVersions[tableId] = version;
                    hasUpdatedTable = true;
                    // result is ordered so we can safely do this assignment
                    mMaxVersion = version;
                }
            } finally {
                cursor.close();
            }
            return hasUpdatedTable;
        }
    }
    
    
    
    static final String SELECT_UPDATED_TABLES_SQL = "SELECT * FROM " + UPDATE_TABLE_NAME
            + " WHERE " + VERSION_COLUMN_NAME
            + "  > ? ORDER BY " + VERSION_COLUMN_NAME + " ASC;";
```

它实际上是执行了`mRefreshRunnable`的,这个runnerable的逻辑非常清晰，先做一些边界检测，然后去`checkUpdatedTable `,看有没有用表在变化，怎么检测。看上面的sql语句，就是去查`room_table_modification_log`中相同的table_id的version，如果有大于之前保存的maxversion的数据，说明有新的修改。然后调用ObserverWrapper 中的`notifyByTableVersions `去通知表上的观察者。


这也就回到了上面最后一个问题`当一个表发生更改，监听一个查询的实时变化是怎么做到的？`。




**MultiInstanceInvalidation**

到这里我们还漏了一点没有讲到。那就是刚才说InvalidationTracker在RoomDatabase中的整个生命周期中的调用情况的时候还有初始化的时候和关闭数据库的时候执行了

```
mInvalidationTracker.startMultiInstanceInvalidation(configuration.context,configuration.name);
和
mInvalidationTracker.stopMultiInstanceInvalidation();
```
因为我们在引用中不可能永远是单标上的查询。也就是说我们一个查询可能是连表的查询，那么这个查询的更新就会依赖于多个表的观察操作。这就引出了框架中的一个经典的CS结构的两个类`MultiInstanceInvalidationClient`, `MultiInstanceInvalidationService`

在初始化`RoomDatabase`的时候我们会开启一个Client也就是`startMultiInstanceInvalidation `,其实就是创建了有一个Client

```
void startMultiInstanceInvalidation(Context context, String name) {
        mMultiInstanceInvalidationClient = new MultiInstanceInvalidationClient(context, name, this,
                mDatabase.getQueryExecutor());
    }
    
    
```

看一下Client初始化的过程

```
MultiInstanceInvalidationClient(Context context, String name,
            InvalidationTracker invalidationTracker, Executor executor) {
        mContext = context.getApplicationContext();
        mName = name;
        mInvalidationTracker = invalidationTracker;
        mExecutor = executor;
        mObserver = new InvalidationTracker.Observer(invalidationTracker.mTableNames) {
            @Override
            public void onInvalidated(@NonNull Set<String> tables) {
                if (mStopped.get()) {
                    return;
                }
                try {
                    mService.broadcastInvalidation(mClientId,
                            tables.toArray(new String[0]));
                } catch (RemoteException e) {
                    Log.w(Room.LOG_TAG, "Cannot broadcast invalidation", e);
                }
            }

            @Override
            boolean isRemote() {
                return true;
            }
        };
        Intent intent = new Intent(mContext, MultiInstanceInvalidationService.class);
        mContext.bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
    }
```
从上面来看，其实在创建`RoomDatabase`的时候创建Client的时候，我们就也创建了一个`InvalidationTracker.Observer`，并且添加进`InvalidationTracker`的观察列表，当这个表发生更新的时候会通过服务端Service `broadcastInvalidation`方法去通知客户端Client。

```
 @SuppressWarnings("WeakerAccess")
    final Runnable mSetUpRunnable = new Runnable() {
        @Override
        public void run() {
            try {
                final IMultiInstanceInvalidationService service = mService;
                if (service != null) {
                    mClientId = service.registerCallback(mCallback, mName);
                    mInvalidationTracker.addObserver(mObserver);
                }
            } catch (RemoteException e) {
                Log.w(Room.LOG_TAG, "Cannot register multi-instance invalidation callback", e);
            }
        }
    };
   
```
而每个Client在Setup的时候会去`service.registerCallback`

```
final IMultiInstanceInvalidationCallback mCallback =
            new IMultiInstanceInvalidationCallback.Stub() {
                @Override
                public void onInvalidation(final String[] tables) {
                    mExecutor.execute(new Runnable() {
                        @Override
                        public void run() {
                            mInvalidationTracker.notifyObserversByTableNames(tables);
                        }
                    });
                }
            };
```
这个`callback`就是是说收到`broadcastInvalidation `的信息的时候会去执行。

这个流程就是在多个`RoomDatabase`之间是如何沟通的，也就是说在其他的`RoomDatabase`也修改了你这个表，那是如何通知到你发生改变的。


##小结

到这里我们在整体上把这个Room是如何做到响应式的做了一个框架的解析。基本上也已经浏览了整个Room的核心代码。当然其中还有很多的细节，如果感兴趣可以自己去好好读一下。因为可能我也不太清楚。我也是初略的读了一下做了一些自己的分析。肯定有理解不对的地方。大家阅读过程中请辩证看待，多多指正。






