# flutter知识

几乎所有的ui和功能都是widget

flutter是单线程

程序入口为

```
void main() {
  runApp(myApp1());
}
class MyApp3 extends StatelessWidget
```

需要添加padding可以使用Container

```
Widget titleSection = new Container(
      padding: const EdgeInsets.all(32.0)
```

布局分为行和列

横向布局用Row

垂直布局用Column

```
new Row(
  children: [
    new Expanded(
      child: new Column(
```

添加paddding

```
EdgeInsets.all(32.0),
EdgeInsets.only(bottom: 8.0),
```

添加text

```
new Text('41'),
```

添加icon

```
new Icon(
```

添加image网络图片和本地图片

```
new Image.network(
url或者本地图片路径,
  height: 240.0,
  fit: BoxFit.cover,
),
```

添加点击事件

```
new GestureDetector
```

页面跳转

```
Navigator.push(
  context,
  new MaterialPageRoute(builder: (context) => new MyApp3()),
);
```

context必须为StatefulWidget的context

添加一个listview

```
body: new ListView(
  children: [
    new Image.asset(
      'images/lake.jpg',
      height: 240.0,
      fit: BoxFit.cover,
    ),
    titleSection,
    buttonSection,
    textSection,
    // ...
  ],
),
```

网络请求

导包

```
pubspec.yaml
http: '>=0.11.3+12'

```

```
import 'package:http/http.dart' as http;
import 'dart:convert';
loadData() async {
  String dataURL = "https://jsonplaceholder.typicode.com/posts";
  http.Response response = await http.get(dataURL);
  print(json.decode(response.body));
}
```

数据库

SQLite for Flutter

和原生通信flutter调用android

获取手机电量

```
Future<void> _getBatteryLevel() async {
  String batteryLevel;
  try {
    final int result = await methodChannel.invokeMethod('getBatteryLevel');
    batteryLevel = 'Battery level: $result%.';
  } on PlatformException {
    batteryLevel = 'Failed to get battery level.';
  }
  print(batteryLevel);
  // Fluttertoast.showToast(msg: batteryLevel);
}

android项目汇中
override fun configureFlutterEngine(@NonNull flutterEngine: FlutterEngine) {
MethodChannel(
            flutterEngine.getDartExecutor(),
            BATTERY_CHANNEL
        ).setMethodCallHandler { call, result ->
            if (call.method.equals("getBatteryLevel")) {
                val batteryLevel = getBatteryLevel()
                if (batteryLevel != -1) {
                    result.success(batteryLevel)
                } else {
                    result.error("UNAVAILABLE", "Battery level not available.", null)
                }
            } else {
                result.notImplemented()
            }
        }
    }
 
 然后flutter中的result会收到回调
```

android 调用flutter

```
android
private fun native2Dart() {
    val eventChannel = MethodChannel(flutterEngine?.dartExecutor, EVENT_CHANNEL)
    eventChannel.invokeMethod("nihao", "android调用flutter", object : MethodChannel.Result {
        override fun success(result: Any?) {
            Log.e("plateform_channel", "success")
        }

        override fun error(errorCode: String?, errorMessage: String?, errorDetails: Any?) {
        }

        override fun notImplemented() {
        }

    })
}
dart

class homestate extends State<content> {
  static const MethodChannel event =
      MethodChannel('samples.flutter.io/receive');

  @override
  void initState() {
    super.initState();
    event.setMethodCallHandler((call) async {
      print(call.arguments);
    });
  }
```

修改程序入口

flutter run -t lib/main2.dart   //运行

 flutter build apk -t lib/main2.dart  //打包
