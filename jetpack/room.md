room数据库使用

## roombase

建库@database 继承room的database

```
@Database(
    entities = [Repo::class, RemoteKeys::class],
    version = 1,
    exportSchema = false
)
abstract class RepoDatabase : RoomDatabase() {

    abstract fun reposDao(): RepoDao
    abstract fun remoteKeysDao(): RemoteKeysDao

    companion object {

        @Volatile
        private var INSTANCE: RepoDatabase? = null

        fun getInstance(context: Context): RepoDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE
                    ?: buildDatabase(context).also { INSTANCE = it }
            }

        private fun buildDatabase(context: Context) =
            Room.databaseBuilder(
                context.applicationContext,
                RepoDatabase::class.java, "Github.db"
            )
                .build()
    }
}
```

创建roomdatabase数据库，该类需要标记注解Database，entities（表），version（版本号）

exportschema查看注释可以知道

```
the database schema will be exported into the given folder
```

如果设置为true的话，数据库将会导入到指定的文件夹中



设置好database之后就可以写sql语句了定义一个类

```
@Dao
interface RepoDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(repos: List<Repo>)

    @Query(
        "SELECT * FROM repos WHERE " +
            "name LIKE :queryString OR description LIKE :queryString " +
            "ORDER BY stars DESC, name ASC"
    )
    fun reposByName(queryString: String): PagingSource<Int, Repo>

    @Query("DELETE FROM repos")
    suspend fun clearRepos()
}
```

一般就是增删改查

同时model类也需要加上数据库的注解

```
@Entity(tableName = "repos")
data class Repo(
    @PrimaryKey @field:SerializedName("id") val id: Long,
    @field:SerializedName("name") val name: String,
    @field:SerializedName("full_name") val fullName: String,
    @field:SerializedName("description") val description: String?,
    @field:SerializedName("html_url") val url: String,
    @field:SerializedName("stargazers_count") val stars: Int,
    @field:SerializedName("forks_count") val forks: Int,
    @field:SerializedName("language") val language: String?
)
```

这样就可以开始使用了

```
RepoDatabase.getInstance(context).reposDao().insertAll(repos)
```

最后增删改查可以返回很多种类型

rxjava

```
 @Insert(onConflict = OnConflictStrategy.REPLACE)
 fun insertAll(repos: List<Repo>):Maybe<List<CycleCode>>
```

用法

```
RepoDatabase.getInstance(context).reposDao().insertAll(repos).subscribeOn(Schedulers.io()).toObservable()
.observeOn(AndroidSchedulers.mainThread().subscribe {
}
```

livedata

```
  @Insert(onConflict = OnConflictStrategy.REPLACE)
  fun insertAll(): LiveData<List<CycleCode>>
```

用法

```
RepoDatabase.getInstance(context).reposDao().insertAll(repos).observe(this){
                    
}
```

flow

```
@Query("SELECT * FROM User")
fun getAllUsers(): Flow<List<User>>
```

用法

```
RepoDatabase.getInstance(context).reposDao().getAllUsers(repos).collect{

}
```



协程

```
@Insert(onConflict = OnConflictStrategy.REPLACE)
suspend fun insertUsers(vararg users: User)
```

用法

```
在viewmodel中
viewModelScope.launch(Dispatchers.Main) {
     val data = RepoDatabase.getInstance(context).reposDao().insertUsers(repos)
}
```



多张表查询通过主表的主键和外表的外键相关联

数据库升级需要addMigrations

```
val dao = Room.databaseBuilder(applicationContext, StudentDataBase::class.java, "test")
                .addMigrations()
                .build()
```

