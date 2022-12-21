创建PackageParser解析下发的apk

```java
Class<?> packageParserClass = Class.forName("android.content.pm.PackageParser");
Method parsePackageMethod = packageParserClass.getDeclaredMethod("parsePackage",
        File.class, int.class);
parsePackageMethod.setAccessible(true);
Object packageParser = packageParserClass.newInstance();
```

解析成package信息

```java
Object packageObj = parsePackageMethod.invoke(packageParser, apkFile,
        PackageManager.GET_RECEIVERS);
packageObj.hashCode();
```

获取package中的receivers

```java
Field receiversField = packageObj.getClass().getDeclaredField("receivers");

List receivers = (List) receiversField.get(packageObj);
```



进行注册

```java
        Class<?> componentClass = Class.forName("android.content.pm.PackageParser$Component");
        Field intentsField = componentClass.getDeclaredField("intents");
        for (Object receiverObject : receivers) {
            String name = (String) receiverObject.getClass().getField("className")
                    .get(receiverObject);
//                class  --->对象
            try {
                BroadcastReceiver broadcastReceiver = (BroadcastReceiver) dexClassLoader.loadClass(name).newInstance();
                List<? extends IntentFilter> filters = (List<? extends IntentFilter>)
                        intentsField.get(receiverObject);
                for (IntentFilter filter : filters) {
                    context.registerReceiver(broadcastReceiver, filter);
                }
            } catch (Exception e) {
            }

        }
```



## 分析packageInstall



传入File后

进行startActivity



PackageIntsllActivity

startInstall

拷贝到data/app

mPm.安装文件到PMS中



