## navigation

基本用法如下

定义基础跳转fragment.NavHostFragment

在首页的layout中

如我的首页是homefragment

```
 <fragment
            android:id="@+id/my_nav_host_fragment"
            android:name="androidx.navigation.fragment.NavHostFragment"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:defaultNavHost="true"
            app:navGraph="@navigation/mobile_navigation" />
```



name为android:name="androidx.navigation.fragment.NavHostFragment"

navGraph为mobile_navigation

mobile_navigation中定义了所有跳转的界面



### mobile_navigation

已navigation为根节点

里面配置了fragment

```
  
  <navigation xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:app="http://schemas.android.com/apk/res-auto"
            xmlns:tools="http://schemas.android.com/tools"
    app:startDestination="@+id/home_dest">
    <fragment
        android:id="@+id/home_dest"
        android:name="com.example.android.codelabs.navigation.HomeFragment"
        android:label="@string/home"
        tools:layout="@layout/home_fragment">

        <action
            android:id="@+id/next_action"
            app:destination="@+id/flow_step_one_dest"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left"
            app:popEnterAnim="@anim/slide_in_left"
            app:popExitAnim="@anim/slide_out_right" />

    </fragment>

    <fragment
        android:id="@+id/flow_step_one_dest"
        android:name="com.example.android.codelabs.navigation.FlowStepFragment"
        tools:layout="@layout/flow_step_one_fragment">
        	//safe传参
        <argument
            android:name="flowStepNumber"
            app:argType="integer"
            android:defaultValue="2"/>

        <action
            android:id="@+id/next_action"
            app:destination="@+id/flow_step_two_dest">
        </action>
    </fragment>

    <fragment
        android:id="@+id/flow_step_two_dest"
        android:name="com.example.android.codelabs.navigation.FlowStepFragment"
        tools:layout="@layout/flow_step_two_fragment">

				//safe传参
        <argument
            android:name="flowStepNumber"
            app:argType="integer"
            android:defaultValue="2"/>

        <action
            android:id="@+id/next_action"
            app:popUpTo="@id/home_dest">
        </action>
    </fragment>
    </navigation>
```

代码中调用一下语句可以触发该fragment的action标签id为R.id.next_action

```
Navigation.createNavigateOnClickListener(R.id.next_action)
```

destination为action标签中要跳转的下一个界面

如

```
<action
   android:id="@+id/next_action"
   app:destination="@+id/flow_step_one_dest">
</action>
        
id为flow_step_one_dest的是下一个界面的id对应的fragment为FlowStepFragment

 <fragment
        android:id="@+id/flow_step_one_dest"
        android:name="com.example.android.codelabs.navigation.FlowStepFragment"
        tools:layout="@layout/flow_step_one_fragment">
```

或者直接通过代码跳转

```
findNavController(this).navigate(R.id.flow_step_one_dest, null, options)
```

options为跳转动画

```
val options = navOptions {
            anim {
                enter = R.anim.slide_in_right
                exit = R.anim.slide_out_left
                popEnter = R.anim.slide_in_left
                popExit = R.anim.slide_out_right
            }
        }
```

### popupto

action中的popupto方法

```
<action
    android:id="@+id/next_action"
    app:popUpTo="@id/settings_dest"
    app:popUpToInclusive="true"
    app:destination="@+id/flow_step_two_dest">
</action>
```

fragmentA-fragmentB-fragmentC-fragmentSettingsDest

当fragmentSettingsDest跳转到fragmentA的时候

fragmentB-fragmentC-会弹出

同时如果加了popUpToInclusive这个属性的话

那么回退栈只有一个A，否则会有俩个A



## 参数传递

mobile_navigation某个item配置如下

```
 <fragment
        android:id="@+id/flow_step_two_dest"
        android:name="com.example.android.codelabs.navigation.FlowStepFragment"
        tools:layout="@layout/flow_step_two_fragment">

        <argument
            android:name="flowStepNumber"
            app:argType="integer"
            android:defaultValue="2"/>

        <action
            android:id="@+id/next_action"
            app:popUpTo="@id/settings_dest">
        </action>
    </fragment>
```



 <argument标签为参数传递标签

传参方法如下A-B

A代码点击事件

```
 view.findViewById<Button>(R.id.navigate_action_button)?.setOnClickListener {
            val flowStepNumberArg = 1
            val action = HomeFragmentDirections.nextAction(flowStepNumberArg)
            findNavController(this).navigate(action)
        }
```

B代码接收在oncreate中

```
 val safeArgs: FlowStepFragmentArgs by navArgs()
 val flowStepNumber = safeArgs.flowStepNumber
 flowStepNumber参数为1
```



剩余问题，

传统的startactivity可以startactivityResult接收requestcode

用navigation怎么实现？

传统的startactivity可以跳转一些系统界面用navigation怎么实现