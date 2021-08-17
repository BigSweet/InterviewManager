## motionlayout

motionlayout分为起始动画和结束动画，都是用布局layout文件表示，

俩个文件必须有相同的id，动画会从起始文件布局内容动画到结束布局

如下起始布局为

```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.motion.widget.MotionLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:layoutDescription="@xml/activity_main_scene"
    tools:context=".MainActivity">

    <ImageView
        android:id="@+id/tv1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:scaleType="fitXY"
        android:src="@mipmap/ic_launcher"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <ImageView
        android:id="@+id/tv2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:scaleType="fitXY"
        android:src="@mipmap/ic_launcher"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <ImageView
        android:id="@+id/tv3"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:scaleType="fitXY"
        android:src="@mipmap/ic_launcher"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />


</androidx.constraintlayout.motion.widget.MotionLayout>
```

有三个叠在一起的imageview，

同时motionlayout用layoutDescription指向动画文件

动画文件代码如下

```
<?xml version="1.0" encoding="utf-8"?>
<MotionScene xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:motion="http://schemas.android.com/apk/res-auto">

    <Transition
        motion:constraintSetEnd="@+id/end"
        motion:constraintSetStart="@id/start"
        motion:duration="1000">

<!--        <OnClick motion:clickAction="toggle"-->
<!--            motion:targetId="@+id/textView"/>-->

        <OnSwipe motion:dragDirection="dragDown" motion:touchRegionId="@id/tv1"/>
       <KeyFrameSet>
       </KeyFrameSet>
    </Transition>

    <ConstraintSet android:id="@+id/start">

    </ConstraintSet>

    <ConstraintSet android:id="@+id/end">


        <Constraint
            android:id="@+id/tv1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:scaleType="fitXY"
            android:src="@mipmap/ic_launcher"
            motion:layout_constraintBottom_toBottomOf="parent"
            motion:layout_constraintLeft_toLeftOf="parent"
            motion:layout_constraintRight_toRightOf="parent"
            motion:layout_constraintTop_toTopOf="parent" />

        <Constraint
            android:id="@+id/tv2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:scaleType="fitXY"
            android:src="@mipmap/ic_launcher"
            motion:layout_constraintBottom_toBottomOf="parent"
            motion:layout_constraintLeft_toLeftOf="parent"
            motion:layout_constraintRight_toLeftOf="@+id/tv1"
            motion:layout_constraintTop_toTopOf="parent" />

        <Constraint
            android:id="@+id/tv3"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:scaleType="fitXY"
            android:src="@mipmap/ic_launcher"
            motion:layout_constraintBottom_toBottomOf="parent"
            motion:layout_constraintLeft_toRightOf="@+id/tv1"
            motion:layout_constraintRight_toRightOf="parent"
            motion:layout_constraintTop_toTopOf="parent" />

    </ConstraintSet>
</MotionScene>
```

Transition定义动画文件布局id

onswipe  手势滑动控制这个view来开始做动画

onclick  点击事件控制这个view来开始做动画

结束动画布局文件中也是三个imageview，不过位置不一样，当点击或者滑动我们指定的view的时候，就会触发动画，从开始的布局文件动画到结束的布局文件，而且他们需要有相同的布局id
