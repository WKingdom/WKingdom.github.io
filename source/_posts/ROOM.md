---
title: ROOM
date: 2020-03-20 00:19:28
categories: 
- Android
- JetPack
tag: JetPack
---
不使用AndroidX的依赖

```
def room_version = "1.1.1"

    implementation "android.arch.persistence.room:runtime:$room_version"
    kapt "android.arch.persistence.room:compiler:$room_version" // use kapt for Kotlin
```

```

@Entity
data class User(
        @PrimaryKey val uid: Int,
        @ColumnInfo(name = "first_name") val firstName: String?,
        @ColumnInfo(name = "last_name") val lastName: String?)
```


```
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```


```
@Dao
interface UserDao {
    @Query("SELECT * FROM user")
    fun getAll(): List<User>

    @Query("SELECT * FROM user WHERE uid IN (:userIds)")
    fun loadAllByIds(userIds: IntArray): List<User>

    @Query("SELECT * FROM user WHERE first_name LIKE :first AND " +
            "last_name LIKE :last LIMIT 1")
    fun findByName(first: String, last: String): User

    @Insert
    fun insertAll(vararg users: User)

    @Delete
    fun delete(user: User)

    @Query("select max(uid) from user ")
    fun queryMaxId(): Int
}
```


```
 val userDao: UserDao by lazy {
            Room.databaseBuilder(
                    appContext,
                    AppDatabase::class.java, "database-name"
            ).build().userDao()
        }
```
#### 监控数据变化
使用RoomDatabase.getInvalidationTracker获取InvalidationTracker对象来监听表数据的改变。一般推荐直接在DAO方法中返回LiveData或者Observable对象。

```
  @Query("SELECT * FROM ConsumeInfo where timeStr LIKE :month ORDER BY time ASC")
    fun getDataWithMonth(month: String): LiveData<List<ConsumeInfo>>
    
    ViewModel中监听，也可以放在界面上监听，下面的代码只能在Model中监听
    
     val liveLists = DaoUtil.instance.getConsumeInfoList(month)
            // 添加数据监听，当数据变化时自动更新数据
            liveLists.observe(owner, Observer<List<ConsumeInfo>> {
                Logger.e("getHistoryConsume observe change")
                getAverageConsume(it)
            })
```



```
 private val users: MutableLiveData<List<User>> by lazy {
        MutableLiveData<List<User>>().also {
            uiScope.launch {
                val rst = async(Dispatchers.IO) { App.userDao.getAll() }
                it.value = rst.await()
            }
        }
    }

    fun getUsers(): LiveData<List<User>> {
        uiScope.launch {
            val rst = async(Dispatchers.IO) { App.userDao.getAll() }
            users.value = rst.await()
        }
        return users
    }


    fun setUsers() {
        uiScope.launch {
            val rst = async(Dispatchers.IO) {
                var count = App.userDao.queryMaxId()
                App.userDao.insertAll(User(++count, "f$count", "l$count"))
                App.userDao.getAll()
            }
            users.value = rst.await()
        }
    }
```
