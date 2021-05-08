2021-04-29更新，最近在复习热修复，发现我的老代码不行了，所以把这篇文章更新一下
本篇是gradle接入
tinker的github网址为[tinker](https://github.com/Tencent/tinker)
我使用的最新版本为TINKER_VERSION=1.9.14.12
下面开始更新新版的代码

在peoject的build中配置如下
```

buildscript {
    repositories {
        mavenLocal()
        jcenter()
        google()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        classpath("com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}") {
            changing = TINKER_VERSION?.endsWith("-SNAPSHOT")
            exclude group: 'com.android.tools.build', module: 'gradle'
        }
    }
    configurations.all {
        it.resolutionStrategy.cacheDynamicVersionsFor(5, 'minutes')
        it.resolutionStrategy.cacheChangingModulesFor(0, 'seconds')
    }
}

allprojects {
    repositories {
        mavenLocal()
        jcenter()
        google()
    }
}
```
TINKER_VERSION需要在gradle.properties中进行配置我的gradle.properties文件内容为
```
TINKER_VERSION=1.9.14.12
android.useAndroidX=true
android.enableJetifier=true
android.enableR8 = false
```
我在用老的项目测试最新版本的tinker的时候，发现R8打补丁包的时候总是报错，所以我把R8静止掉了

接下来是配置app的build，官方文档推荐我们使用他们demo里面的配置
[build.gradle](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/build.gradle)
于是我全部copy了过来，不过里面的几个地方还是需要修改的
首先先贴上全部的
我会在里面写上注释

```
apply plugin: 'com.android.application'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    testImplementation 'junit:junit:4.12'
    implementation "androidx.appcompat:appcompat:1.1.0"
    api("com.tencent.tinker:tinker-android-lib:${TINKER_VERSION}") { changing = true }

    // Maven local cannot handle transist dependencies.
    implementation("com.tencent.tinker:tinker-android-loader:${TINKER_VERSION}") { changing = true }

    annotationProcessor("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    compileOnly("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    compileOnly("com.tencent.tinker:tinker-android-anno-support:${TINKER_VERSION}") { changing = true }

    implementation "androidx.multidex:multidex:2.0.1"

}

def gitSha() {
    try {
        String gitRev = 'git rev-parse --short HEAD'.execute(null, project.rootDir).text.trim()
        if (gitRev == null) {
            throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
        }
        return gitRev
    } catch (Exception e) {
        throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
    }
}

def javaVersion = JavaVersion.VERSION_1_7

android {
    compileSdkVersion 28
    buildToolsVersion '28.0.3'

    compileOptions {
        sourceCompatibility javaVersion
        targetCompatibility javaVersion
    }
    //recommend
    dexOptions {
        jumboMode = true
    }

    signingConfigs {
        debug {
            keyAlias "key0"
            keyPassword "123456"
            storeFile file('robustsign2')
            storePassword "123456"
        }
        release {
            keyAlias "key0"
            keyPassword "123456"
            storeFile file('robustsign2')
            storePassword "123456"
        }
    }

    defaultConfig {
        applicationId "tinker.sample.android"
        minSdkVersion 14
        targetSdkVersion 26
        versionCode 1
        versionName "1.0.0"
        /**
         * you can use multiDex and install it in your ApplicationLifeCycle implement
         */
        multiDexEnabled true
        multiDexKeepProguard file("tinker_multidexkeep.pro")
        /**
         * buildConfig can change during patch!
         * we can use the newly value when patch
         */
        buildConfigField "String", "MESSAGE", "\"I am the base apk\""
//        buildConfigField "String", "MESSAGE", "\"I am the patch apk\""
        /**
         * client version would update with patch
         * so we can get the newly git version easily!
         */
        buildConfigField "String", "TINKER_ID", "\"${getTinkerIdValue()}\""
        buildConfigField "String", "PLATFORM", "\"all\""
    }

//    aaptOptions{
//        additionalParameters "--emit-ids", "${project.file('public.txt')}"
//        cruncherEnabled false
//    }

//    //use to test flavors support
//    productFlavors {
//        flavor1 {
//            applicationId 'tinker.sample.android.flavor1'
//        }
//
//        flavor2 {
//            applicationId 'tinker.sample.android.flavor2'
//        }
//    }

    buildTypes {
        release {
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), project.file('proguard-rules.pro')
        }
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
        }
    }
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }

    packagingOptions {
        exclude "/META-INF/**"
    }
}

def bakPath = file("${buildDir}/bakApk/")

/**
 * you can use assembleRelease to build you base apk
 * use tinkerPatchRelease -POLD_APK=  -PAPPLY_MAPPING=  -PAPPLY_RESOURCE= to build patch
 * add apk from the build/bakApk
 */
ext {
    //for some reason, you may want to ignore tinkerBuild, such as instant run debug build?
    tinkerEnabled = true

    //for normal build
    //old apk file to build patch apk

    tinkerOldApkPath = "${bakPath}/app-release-0429-15-58-52.apk"
    //proguard mapping file to build patch apk
    tinkerApplyMappingPath = "${bakPath}/app-release-0429-15-58-52-mapping.txt"
    //resource R.txt to build patch apk, must input if there is resource changed
    tinkerApplyResourcePath = "${bakPath}/app-release-0429-15-58-52-R.txt"

    //only use for build all flavor, if not, just ignore this field
    tinkerBuildFlavorDirectory = "${bakPath}/app-1018-17-32-47"
}


def getOldApkPath() {
    return hasProperty("OLD_APK") ? OLD_APK : ext.tinkerOldApkPath
}

def getApplyMappingPath() {
    return hasProperty("APPLY_MAPPING") ? APPLY_MAPPING : ext.tinkerApplyMappingPath
}

def getApplyResourceMappingPath() {
    return hasProperty("APPLY_RESOURCE") ? APPLY_RESOURCE : ext.tinkerApplyResourcePath
}

def getTinkerIdValue() {
    return hasProperty("TINKER_ID") ? TINKER_ID : gitSha()
}

def buildWithTinker() {
    return hasProperty("TINKER_ENABLE") ? Boolean.parseBoolean(TINKER_ENABLE) : ext.tinkerEnabled
}

def getTinkerBuildFlavorDirectory() {
    return ext.tinkerBuildFlavorDirectory
}

if (buildWithTinker()) {
    apply plugin: 'com.tencent.tinker.patch'

    tinkerPatch {
        /**
         * necessary，default 'null'
         * the old apk path, use to diff with the new apk to build
         * add apk from the build/bakApk
         */
        oldApk = getOldApkPath()
        /**
         * optional，default 'false'
         * there are some cases we may get some warnings
         * if ignoreWarning is true, we would just assert the patch process
         * case 1: minSdkVersion is below 14, but you are using dexMode with raw.
         *         it must be crash when load.
         * case 2: newly added Android Component in AndroidManifest.xml,
         *         it must be crash when load.
         * case 3: loader classes in dex.loader{} are not keep in the main dex,
         *         it must be let tinker not work.
         * case 4: loader classes in dex.loader{} changes,
         *         loader classes is ues to load patch dex. it is useless to change them.
         *         it won't crash, but these changes can't effect. you may ignore it
         * case 5: resources.arsc has changed, but we don't use applyResourceMapping to build
         */
        ignoreWarning = false

        /**
         * optional，default 'true'
         * whether sign the patch file
         * if not, you must do yourself. otherwise it can't check success during the patch loading
         * we will use the sign config with your build type
         */
        useSign = true

        /**
         * optional，default 'true'
         * whether use tinker to build
         */
        tinkerEnable = buildWithTinker()

        /**
         * Warning, applyMapping will affect the normal android build!
         */
        buildConfig {
            /**
             * optional，default 'null'
             * if we use tinkerPatch to build the patch apk, you'd better to apply the old
             * apk mapping file if minifyEnabled is enable!
             * Warning:
             * you must be careful that it will affect the normal assemble build!
             */
            applyMapping = getApplyMappingPath()
            /**
             * optional，default 'null'
             * It is nice to keep the resource id from R.txt file to reduce java changes
             */
            applyResourceMapping = getApplyResourceMappingPath()

            /**
             * necessary，default 'null'
             * because we don't want to check the base apk with md5 in the runtime(it is slow)
             * tinkerId is use to identify the unique base apk when the patch is tried to apply.
             * we can use git rev, svn rev or simply versionCode.
             * we will gen the tinkerId in your manifest automatic
             */
            tinkerId = "1.0"

            /**
             * if keepDexApply is true, class in which dex refer to the old apk.
             * open this can reduce the dex diff file size.
             */
            keepDexApply = false

            /**
             * optional, default 'false'
             * Whether tinker should treat the base apk as the one being protected by app
             * protection tools.
             * If this attribute is true, the generated patch package will contain a
             * dex including all changed classes instead of any dexdiff patch-info files.
             */
            isProtectedApp = false

            /**
             * optional, default 'false'
             * Whether tinker should support component hotplug (add new component dynamically).
             * If this attribute is true, the component added in new apk will be available after
             * patch is successfully loaded. Otherwise an error would be announced when generating patch
             * on compile-time.
             *
             * <b>Notice that currently this feature is incubating and only support NON-EXPORTED Activity</b>
             */
            supportHotplugComponent = false
        }

        dex {
            /**
             * optional，default 'jar'
             * only can be 'raw' or 'jar'. for raw, we would keep its original format
             * for jar, we would repack dexes with zip format.
             * if you want to support below 14, you must use jar
             * or you want to save rom or check quicker, you can use raw mode also
             */
            dexMode = "jar"

            /**
             * necessary，default '[]'
             * what dexes in apk are expected to deal with tinkerPatch
             * it support * or ? pattern.
             */
            pattern = ["classes*.dex",
                       "assets/secondary-dex-?.jar"]
            /**
             * necessary，default '[]'
             * Warning, it is very very important, loader classes can't change with patch.
             * thus, they will be removed from patch dexes.
             * you must put the following class into main dex.
             * Simply, you should add your own application {@code tinker.sample.android.SampleApplication}
             * own tinkerLoader, and the classes you use in them
             *
             */
            loader = [
                    //use sample, let BaseBuildInfo unchangeable with tinker
                    "tinker.sample.android.app.BaseBuildInfo"
            ]
        }

        lib {
            /**
             * optional，default '[]'
             * what library in apk are expected to deal with tinkerPatch
             * it support * or ? pattern.
             * for library in assets, we would just recover them in the patch directory
             * you can get them in TinkerLoadResult with Tinker
             */
            pattern = ["lib/*/*.so"]
        }

        res {
            /**
             * optional，default '[]'
             * what resource in apk are expected to deal with tinkerPatch
             * it support * or ? pattern.
             * you must include all your resources in apk here,
             * otherwise, they won't repack in the new apk resources.
             */
            pattern = ["res/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]

            /**
             * optional，default '[]'
             * the resource file exclude patterns, ignore add, delete or modify resource change
             * it support * or ? pattern.
             * Warning, we can only use for files no relative with resources.arsc
             */
            ignoreChange = ["assets/sample_meta.txt"]

            /**
             * default 100kb
             * for modify resource, if it is larger than 'largeModSize'
             * we would like to use bsdiff algorithm to reduce patch file size
             */
            largeModSize = 100
        }

        packageConfig {
            /**
             * optional，default 'TINKER_ID, TINKER_ID_VALUE' 'NEW_TINKER_ID, NEW_TINKER_ID_VALUE'
             * package meta file gen. path is assets/package_meta.txt in patch file
             * you can use securityCheck.getPackageProperties() in your ownPackageCheck method
             * or TinkerLoadResult.getPackageConfigByName
             * we will get the TINKER_ID from the old apk manifest for you automatic,
             * other config files (such as patchMessage below)is not necessary
             */
            configField("patchMessage", "tinker is sample to use")
            /**
             * just a sample case, you can use such as sdkVersion, brand, channel...
             * you can parse it in the SamplePatchListener.
             * Then you can use patch conditional!
             */
            configField("platform", "all")
            /**
             * patch version via packageConfig
             */
            configField("patchVersion", "1.0")
        }
        //or you can add config filed outside, or get meta value from old apk
        //project.tinkerPatch.packageConfig.configField("test1", project.tinkerPatch.packageConfig.getMetaDataFromOldApk("Test"))
        //project.tinkerPatch.packageConfig.configField("test2", "sample")

        /**
         * if you don't use zipArtifact or path, we just use 7za to try
         */
        sevenZip {
            /**
             * optional，default '7za'
             * the 7zip artifact path, it will use the right 7za with your platform
             */
            zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
            /**
             * optional，default '7za'
             * you can specify the 7za path yourself, it will overwrite the zipArtifact value
             */
//        path = "/usr/local/bin/7za"
        }
    }

    List<String> flavors = new ArrayList<>();
    project.android.productFlavors.each { flavor ->
        flavors.add(flavor.name)
    }
    boolean hasFlavors = flavors.size() > 0
    def date = new Date().format("MMdd-HH-mm-ss")

    /**
     * bak apk and mapping
     */
    android.applicationVariants.all { variant ->
        /**
         * task type, you want to bak
         */
        def taskName = variant.name

        tasks.all {
            if ("assemble${taskName.capitalize()}".equalsIgnoreCase(it.name)) {

                it.doLast {
                    copy {
                        def fileNamePrefix = "${project.name}-${variant.baseName}"
                        def newFileNamePrefix = hasFlavors ? "${fileNamePrefix}" : "${fileNamePrefix}-${date}"

                        def destPath = hasFlavors ? file("${bakPath}/${project.name}-${date}/${variant.flavorName}") : bakPath

                        if (variant.metaClass.hasProperty(variant, 'packageApplicationProvider')) {
                            def packageAndroidArtifact = variant.packageApplicationProvider.get()
                            if (packageAndroidArtifact != null) {
                                try {
                                    from new File(packageAndroidArtifact.outputDirectory.getAsFile().get(), variant.outputs.first().apkData.outputFileName)
                                } catch (Exception e) {
                                    from new File(packageAndroidArtifact.outputDirectory, variant.outputs.first().apkData.outputFileName)
                                }
                            } else {
                                from variant.outputs.first().mainOutputFile.outputFile
                            }
                        } else {
                            from variant.outputs.first().outputFile
                        }

                        into destPath
                        rename { String fileName ->
                            fileName.replace("${fileNamePrefix}.apk", "${newFileNamePrefix}.apk")
                        }

                        def dirName = variant.dirName
                        if (hasFlavors) {
                            dirName = taskName
                        }
                        from "${buildDir}/outputs/mapping/${dirName}/mapping.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("mapping.txt", "${newFileNamePrefix}-mapping.txt")
                        }

                        from "${buildDir}/intermediates/symbols/${dirName}/R.txt"
                        from "${buildDir}/intermediates/symbol_list/${dirName}/R.txt"
                        from "${buildDir}/intermediates/runtime_symbol_list/${dirName}/R.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("R.txt", "${newFileNamePrefix}-R.txt")
                        }
                    }
                }
            }
        }
    }
    project.afterEvaluate {
        //sample use for build all flavor for one time
        if (hasFlavors) {
            task(tinkerPatchAllFlavorRelease) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Release")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}ReleaseManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 15)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-R.txt"

                    }

                }
            }

            task(tinkerPatchAllFlavorDebug) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Debug")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}DebugManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 13)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-R.txt"
                    }

                }
            }
        }
    }
}


task sortPublicTxt() {
    doLast {
        File originalFile = project.file("public.txt")
        File sortedFile = project.file("public_sort.txt")
        List<String> sortedLines = new ArrayList<>()
        originalFile.eachLine {
            sortedLines.add(it)
        }
        Collections.sort(sortedLines)
        sortedFile.delete()
        sortedLines.each {
            sortedFile.append("${it}\n")
        }
    }
}

```
主要修改了下面几个地方
```
def getTinkerIdValue() {
 hasProperty("TINKER_ID") ? TINKER_ID : gitSha()
}
```
官方文档默认是将git提交的记录作为think_id记录下来，这里我不需要，所以我改成下图
```
def getTinkerIdValue() {
    return "1.0"
}
```

接下来修改自己的签名文件,我这里配置的文件路径在APP的目录下面账号密码如下，是我在写robustdemo的时候生成的
```
   //配置自己的签名文件，签名文件的生成和导入可以去百度，本篇不讲解
    signingConfigs {
        debug {
            keyAlias "key0"
            keyPassword "123456"
            storeFile file('robustsign2')
            storePassword "123456"
        }
        release {
            keyAlias "key0"
            keyPassword "123456"
            storeFile file('robustsign2')
            storePassword "123456"
        }
    }
```
```
   buildTypes {
        release {
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), project.file('proguard-rules.pro')
        }
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
        }
    }
```
配置完上面之后，build就配置好了，接下来配置application
先上代码
```
package com.anlaiye.swt.gradletest;

import android.annotation.TargetApi;
import android.app.Application;
import android.content.Context;
import android.content.Intent;
import android.os.Build;

import androidx.multidex.MultiDex;

import com.tencent.tinker.anno.DefaultLifeCycle;
import com.tencent.tinker.entry.ApplicationLike;
import com.tencent.tinker.lib.tinker.TinkerInstaller;
import com.tencent.tinker.loader.shareutil.ShareConstants;

@DefaultLifeCycle(application = "tinker.sample.android.app.SampleApplication",
        flags = ShareConstants.TINKER_ENABLE_ALL,
        loadVerifyFlag = false)
public class SimpleTinkerInApplicationLike extends ApplicationLike {

    public SimpleTinkerInApplicationLike(Application application, int tinkerFlags, boolean tinkerLoadVerifyFlag, long applicationStartElapsedTime, long applicationStartMillisTime, Intent tinkerResultIntent) {
        super(application, tinkerFlags, tinkerLoadVerifyFlag, applicationStartElapsedTime, applicationStartMillisTime, tinkerResultIntent);
    }

    @Override
    public void onBaseContextAttached(Context base) {
        super.onBaseContextAttached(base);
        MultiDex.install(base);
        TinkerInstaller.install(this);

    }

    @TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
    public void registerActivityLifecycleCallbacks(Application.ActivityLifecycleCallbacks callback) {
        getApplication().registerActivityLifecycleCallbacks(callback);
    }
}
```
这个application官网有提供的，也可以copy我的，SimpleTinkerInApplicationLike 并不是一个application真正的application是@DefaultLifeCycle里面的tinker.sample.android.app.SampleApplication
所以在androidmanifest中配置application的name
```
   android:name="tinker.sample.android.app.SampleApplication"

```
打开SampleApplication可以看到其实会定位到我们的application
```
public class SampleApplication extends TinkerApplication {

    public SampleApplication() {
        super(15, "com.anlaiye.swt.gradletest.SimpleTinkerInApplicationLike", "com.tencent.tinker.loader.TinkerLoader", false, false);
    }

}
```
同时配置读取sd卡的权限

接下来测试
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.anlaiye.swt.gradletest.MainActivity">

    <TextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="22222"/>


    <Button
        android:layout_below="@+id/tv"
        android:onClick="loadPath"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="更新"/>
</RelativeLayout>


```
第一次我设置textview的值是22222
然后设置按钮调用onclick
```
package com.anlaiye.swt.gradletest;

import android.Manifest;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.Bundle;
import android.os.Environment;
import android.util.Log;
import android.view.View;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import com.tencent.tinker.lib.tinker.TinkerInstaller;

import java.io.File;

public class MainActivity extends AppCompatActivity {

    public static final int MY_PERMISSIONS_REQUEST_READ_CONTACTS = 1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if (!hasRequiredPermissions()) {
            ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.READ_EXTERNAL_STORAGE}, 0);
        }
    }

    private boolean hasRequiredPermissions() {
        if (Build.VERSION.SDK_INT >= 16) {
            final int res = ContextCompat.checkSelfPermission(this.getApplicationContext(), Manifest.permission.READ_EXTERNAL_STORAGE);
            return res == PackageManager.PERMISSION_GRANTED;
        } else {
            // When SDK_INT is below 16, READ_EXTERNAL_STORAGE will also be granted if WRITE_EXTERNAL_STORAGE is granted.
            final int res = ContextCompat.checkSelfPermission(this.getApplicationContext(), Manifest.permission.WRITE_EXTERNAL_STORAGE);
            return res == PackageManager.PERMISSION_GRANTED;
        }
    }

    //加载补丁
    public void loadPath(View view) {
        requestPermi();
    }

    public void loadApk() {
        String path = Environment.getExternalStorageDirectory().getAbsolutePath() + "/app-release-patch_signed_7zip.apk";
        File file = new File(path);
        if (file.exists()) {
            Toast.makeText(this, "补丁已经存在", Toast.LENGTH_SHORT).show();
            TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), path);
            Log.d("swy", path);
        } else {
            Toast.makeText(this, "补丁已经不存在", Toast.LENGTH_SHORT).show();
            Log.d("swy", path);
        }
    }

    private void requestPermi() {
        if (ContextCompat.checkSelfPermission(MainActivity.this,
                Manifest.permission.READ_EXTERNAL_STORAGE)
                != PackageManager.PERMISSION_GRANTED) {
            // Should we show an explanation?
            if (ActivityCompat.shouldShowRequestPermissionRationale(MainActivity.this,
                    Manifest.permission.READ_EXTERNAL_STORAGE)) {
                // Show an expanation to the user *asynchronously* -- don't block
                // this thread waiting for the user's response! After the user
                // sees the explanation, try again to request the permission.

            } else {
                // No explanation needed, we can request the permission.
                ActivityCompat.requestPermissions(MainActivity.this,
                        new String[]{Manifest.permission.READ_EXTERNAL_STORAGE},
                        MY_PERMISSIONS_REQUEST_READ_CONTACTS);
                Log.d("swt", "requestPermi: ");
                // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
                // app-defined int constant. The callback method gets the
                // result of the request.
            }
        } else {
            //有权限直接加载apk
            loadApk();
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode,
                                           String permissions[], int[] grantResults) {
        switch (requestCode) {
            case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
                // If request is cancelled, the result arrays are empty.
                if (grantResults.length > 0
                        && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    // permission was granted, yay! Do the
                    // contacts-related task you need to do.
                    //权限申请成功加载apk
                    loadApk();
                    Log.d("swt", "permissionsuss: ");
                } else {

                    // permission denied, boo! Disable the
                    // functionality that depends on this permission.
                }
                return;
            }
        }
    }
}
```
我是直接运行的release包把这个修改了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210429162833689.png)

然后我运行APP
会生成如下的目录结构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210429162913263.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)

同时界面效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210429162951374.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)

接下来打开app的build找到ext模块
做如下修改将里面的名字和我们生成apk的名字对应
```

//老版本的文件所在的位置，大家也可以动态配置，不用每次都在这里修改
ext {
    //for some reason, you may want to ignore tinkerBuild, such as instant run debug build?
    tinkerEnabled = true

    //for normal build
    //old apk file to build patch apk

    tinkerOldApkPath = "${bakPath}/app-release-0429-16-28-46.apk"
    //proguard mapping file to build patch apk
    tinkerApplyMappingPath = "${bakPath}/app-release-0429-16-28-46-mapping.txt"
    //resource R.txt to build patch apk, must input if there is resource changed
    tinkerApplyResourcePath = "${bakPath}/app-release-0429-16-28-46-R.txt"

    //only use for build all flavor, if not, just ignore this field
    tinkerBuildFlavorDirectory = "${bakPath}/app-1018-17-32-47"
}
```

接下来修改mainactivity的xml文件把textview改成33333，用来和第一次作区别

注意接下来的这个操作是避免这个错误
```
app\build\intermediates\tinker_intermediates\values_backup
```

接下来把bakapk整个文件夹移出来,移到app的目录下面这样
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021042916342798.png)
然后再clean一下,在到app的目录下面生成build文件夹
把bakapk文件夹移进去

然后点击tinker的path命令生成渠道包
![这里写图片描述](https://img-blog.csdnimg.cn/img_convert/079d08672c49187174fd054c8e94b499.png)

生成目录如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210429165227268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5,size_16,color_FFFFFF,t_70)

 注意箭头所指的文件
 我们在onclick中执行的文件名就是这个，这个文件就是我们需要的最终的文件
 然后将这个文件导入SD卡的最外层目录
 ```
  adb push /Users/yanzhe/AndroidStudioProjects/TinkerGradleDemo/app/build/outputs/apk/tinkerPatch/release/app-release-patch_signed_7zip.apk /sdcard
 ```
 导入成功后
 点击更新按钮
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210429165609130.gif)
textview由22222变成33333
但是应用会闪退一下
好像是和这个issue有关
[issue](https://github.com/Tencent/tinker/issues/1195)

本篇只是简单的演示了整个流程，整理了一下容易踩得坑，可以优化的地方还有很多，大家可以在这个基础上自行发挥。

总结一下自己碰到的坑，
1,首先没有在gradle.properties中配置tinker_id，会提示错误tinker_id not set!!!
2,debug没有配置  proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
导入生成的apk一直没有mapping文件，找了半天才发现

最后说下热修复的大概流程
最终我们需要的是一个patch.apk文件，这个文件是通过老apk和新apk对比生成的,
是通过腾讯自研的dex差异算法得出
算法相关的地址在这里
[dex算法](https://www.zybuluo.com/dodola/note/554061)

新APK和老APK的mapping必须是一致的，所以我们需要把老APK的mapping保存起来，方便新APK与他对比。
这样就能得到最终的patch，然后调用sdk的TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), path);
这样就实现了热修复的目的
下篇讲解命令行接入
本篇下载地址：
csdn下载地址
http://download.csdn.net/download/qq_15527709/10017530 这是老地址
https://download.csdn.net/download/qq_15527709/18243488 这是新的地址
考虑到现在csdn改版下载代码都需要分了，所以我重新上传了一份到了github
https://github.com/BigSweet/TinkerGradleDemohttps://github.com/Tencent/tinker)