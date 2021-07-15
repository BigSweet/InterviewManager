jetpack相关

## room数据库

建库@database 继承room的database

```
@Database(entities = [Student::class], version = 1)
abstract class StudentDataBase : RoomDatabase() {
    abstract fun studentDao(): StudentDao
}
```

建表@enetry

```
@Entity
class Student() {

    @PrimaryKey
    var id: Int = 0


    @ColumnInfo(name = "name")
    var name: String = ""

    @ColumnInfo(name = "age")
    var age: Int = 0

    override fun toString(): String {
        return "Student(id=$id, name='$name', age='$age')"
    }

}
```

建操作层dao 里面有insert delete update等操作

```
@Dao
interface StudentDao {

    @Insert
    fun insert(vararg student: Student)


    @Delete
    fun delete(student: Student)

    @Query("select * from Student")
    fun getAll():List<Student>
}
```

最后通过线程操作数据库

```
Thread {
            val dao = Room.databaseBuilder(applicationContext, StudentDataBase::class.java, "test").build()
            dao.studentDao().insert(student)
            Log.d("swt", dao.studentDao().getAll().toString())
        }.start()
```



多张表查询通过主表的主键和外表的外键相关联

数据库升级需要addMigrations

```
val dao = Room.databaseBuilder(applicationContext, StudentDataBase::class.java, "test")
                .addMigrations()
                .build()
```

