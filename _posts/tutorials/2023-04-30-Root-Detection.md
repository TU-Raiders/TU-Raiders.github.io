---
title: CTF - Android Root Detection and Bypass
author: 0x41ly
classes: wide
ribbon: MidnightBlue
header:
  teaser: /assets/images/Android-root-detection-and-bypass/root-bypass-final.png
date: 2023-04-30 14:00:00 +0800
categories: Tutorials
# description: "Our first blood Challenge in ICMTC CSC CTF..."
tags: [mobile, root detection, root bypass, android]
toc: true
---

Android devices come with strict access control systems and permissions set up by smartphone manufacturers to protect users from potential security threats and prevent accidental damage. However, some users may find these systems too restrictive if they want to customize their device in ways not allowed by the manufacturer.

Rooting an Android device is the process of gaining complete access to it. This provides various benefits, such as the ability to install apps from third-party app stores, sideload applications, apply custom themes, improve battery life, enhance performance, and modify the behavior of mobile apps on the device.

### What is root detection?
Root detection is a security measure implemented by mobile app developers to prevent their app from running on devices that have been rooted. Rooting a device gives users complete control over the device, which can pose a security threat to the app as well as the user's data. Sensitive apps such as those related to banking, medical, shopping, and government often include root detection checks that prevent them from functioning properly on rooted devices. In some cases, developers may implement strict root detection capabilities that prevent the app from running at all on rooted devices.

Bypassing root detection is possible, although the level of difficulty can vary depending on the app. There are various root detection techniques that can be implemented, ranging from basic checks that can be easily found online to custom detection methods that have never been seen before. Most root detection logic runs directly on the device, making it possible to uncover these techniques through reverse engineering. By using a combination of static and dynamic analysis, it is possible to determine how the root detection is implemented, when it is called, and how to bypass the checks. While the specific steps may vary depending on the app, the overall process to bypass root detection is similar in most cases.

To detect whether an Android device has been rooted, app developers can use various techniques, including:

1.  Checking for the presence of Superuser or SuperSU apps: These apps are typically installed on rooted devices to manage root access permissions, so their presence can indicate that the device has been rooted.
    
2.  Checking for the presence of su binary: The su binary is a program that allows users to execute commands with root privileges. If it is present on the device, it can indicate that the device has been rooted.
    
3.  Checking for modified system files: Rooting often involves modifying system files on the device, so checking for any modifications to these files can indicate that the device has been rooted.
    
4.  Checking for custom ROMs: Rooting often involves installing custom ROMs, which are modified versions of the Android operating system. Checking for the presence of a custom ROM can indicate that the device has been rooted.

### Example app 
The apk used can be downloaded from [here](https://github.com/nowsecure/NowSecure-Android-Root-Detection-Test-App/raw/main/root-bypass.apk) 
```zsh
┌──(x41ly㉿0x41ly)-[~/mobile_playground/root_Detection]
└─$ adb devices             
List of devices attached
127.0.0.1:5562	device

                                                                                                                                                                                                            
┌──(x41ly㉿0x41ly)-[~/mobile_playground/root_Detection]
└─$ adb connect 127.0.0.1:5562
already connected to 127.0.0.1:5562
                                                                                                                                                                                                              
┌──(x41ly㉿0x41ly)-[~/mobile_playground/root_Detection]
└─$ adb install root-bypass.apk 
Performing Streamed Install
Success

```

running the app on the genymotion

![img](/assets/images/Android-root-detection-and-bypass/Pasted%20image%2020230428233328.png)

The app detected it is rooted device 

### Static analysis 

```zsh
┌──(x41ly㉿0x41ly)-[~/mobile_playground/root_Detection]
└─$ jadx-gui root-bypass.apk 

```

There is only one activity which is `MainActivity`

#### Activity life cycle in android  
![img](/assets/images/Android-root-detection-and-bypass/Pasted%20image%2020230428234518.png)

So the first method is been called is onCreate

```java
public void onCreate(Bundle bundle) {  
        context = getApplicationContext();  
        super.onCreate(bundle);  
        setContentView(R.layout.activity_main);  
        TextView textView = (TextView) findViewById(R.id.areYouRooted);  
        if (checkForRoot()) {  
            textView.setText("Root Detected!");  
            Toast.makeText(context, "This application appears to be running on a hacked device!", 1).show();  
            return;  
        }  
        textView.setText("No root detected!");  
    }
```

The previous code is just simply checking if it is a rooted device or not by calling  `checkForRoot()` method

```java
private boolean checkForRoot() {  
        boolean doesSuBinaryExist = doesSuBinaryExist();  
        ((TextView) findViewById(R.id.suExistsResults)).setText(Boolean.toString(doesSuBinaryExist));  
        boolean doesWhichSuWork = doesWhichSuWork();  
        ((TextView) findViewById(R.id.whichSuResults)).setText(Boolean.toString(doesWhichSuWork));  
        boolean isRootAppInstalled = isRootAppInstalled();  
        ((TextView) findViewById(R.id.MagiskCheckResults)).setText(Boolean.toString(isRootAppInstalled));  
        return isRootAppInstalled || doesSuBinaryExist || doesWhichSuWork;  
    }
```

There are 3 different techniques used in this app
	- It checks for the `su` binary
	- It cheks for `whichSu` binary
	- it checks for `RootApp` application
Here is the code for that
```java
  
    private boolean isRootAppInstalled() {  
        String[] strArr = {"com.kingroot.kinguser", "com.kingo.root", "com.topjohnwu.magisk"};  
        PackageManager packageManager = context.getPackageManager();  
        for (int i = 0; i < 3; i++) {  
            String str = strArr[i];  
            try {  
                packageManager.getPackageInfo(str, 1);  
                Log.i("ContentValues", "Root package detected: " + str);  
                return true;  
            } catch (PackageManager.NameNotFoundException unused) {  
            }  
        }  
        return false;  
    }  
  
    private boolean doesSuBinaryExist() {  
        String[] strArr = {"/system/xbin/su", "/sbin/su", "/system/su", "/system/bin/su"};  
        for (int i = 0; i < 4; i++) {  
            String str = strArr[i];  
            if (new File(str).exists()) {  
                Log.i("ContentValues", "SU binary detected: " + str);  
                return true;  
            }  
        }  
        return false;  
    }  
  
    private boolean doesWhichSuWork() {  
        Process process = null;  
        try {  
            process = Runtime.getRuntime().exec(new String[]{"which", "su"});  
            boolean z = new BufferedReader(new InputStreamReader(process.getInputStream())).readLine() != null;  
            if (process != null) {  
                process.destroy();  
            }  
            return z;  
        } catch (Throwable unused) {  
            if (process != null) {  
                process.destroy();  
            }  
            return false;  
        }  
    }
```

### Bypass

#### Setting up frida
follow [this guide](https://frida.re/docs/android/) to install frida server on your device


#####  doesSuBinaryExist check bypass

```java
private boolean doesSuBinaryExist() {  
        String[] strArr = {"/system/xbin/su", "/sbin/su", "/system/su", "/system/bin/su"};  
        for (int i = 0; i < 4; i++) {  
            String str = strArr[i];  
            if (new File(str).exists()) {  
                Log.i("ContentValues", "SU binary detected: " + str);  
                return true;  
            }  
        }  
        return false;  
    }  
```

It simply uses the `File` object from `java.io.File` class to creat a file object with a defined path and checks if it exists or no

What we can do?
if we could hook the `File(str).exist()` function so it returns false if the file path ends with `su`

``` js
Java.perform(function(){
   // Su Exists bypass
   const File = Java.use('java.io.File');
   File.exists.implementation = function () {
       const filePath = this.getPath();
       
       if (filePath.endsWith("su")){
           console.log(`Bypassing exists() call to: ${filePath}`);
           return false;
       }
       console.log(`Calling exists() on: ${filePath}`);
       return this.exists();
   };
})
```


![img](/assets/images/Android-root-detection-and-bypass/Pasted%20image%2020230429050246.png)


##### doesWhichSuWork check bypass
```java
    private boolean doesWhichSuWork() {  
        Process process = null;  
        try {  
            process = Runtime.getRuntime().exec(new String[]{"which", "su"});  
            boolean z = new BufferedReader(new InputStreamReader(process.getInputStream())).readLine() != null;  
            if (process != null) {  
                process.destroy();  
            }  
            return z;  
        } catch (Throwable unused) {  
            if (process != null) {  
                process.destroy();  
            }  
            return false;  
        }  
    }
```

 Here it checks for the `su` binary using `which` command and if exist it will retun an output does not equal to null
Again what can we do?
If we could hook the exec function to replace the `su`  with non-existing binary it will return nulll always

```js 
Java.perform(function(){
   // Shell "which su" Bypass
   const Runtime = Java.use('java.lang.Runtime');
   Runtime.exec.overload('[Ljava.lang.String;').implementation = function(commandArray){
       for (var i = 0; i < commandArray.length; i++) {
           if (commandArray[i] == "su") {
               console.log("Bypassing command referencing 'su'!");
               var clonedArray = commandArray.slice();
               clonedArray[i] = "NotARealBinary";
               return this.exec(clonedArray);
           }
       }
       console.log(`Calling exec() on: ${cmd}`)
       return this.exec(cmd);
   }
})
```

the previous code says when ever you find a `su` word replace it with `NotARealBinary`

![img](/assets/images/Android-root-detection-and-bypass/Pasted%20image%2020230429052221.png)


As my emulator does not have a rootApp installed so its detected as false 
but lets find out how it does work

##### isRootAppInstalled check bypass

```java
public class MainActivity extends AppCompatActivity {  
    private static Context context;  
  
    private boolean isRootAppInstalled() {  
        String[] strArr = {"com.kingroot.kinguser", "com.kingo.root", "com.topjohnwu.magisk"};  
        PackageManager packageManager = context.getPackageManager();  
        for (int i = 0; i < 3; i++) {  
            String str = strArr[i];  
            try {  
                packageManager.getPackageInfo(str, 1);  
                Log.i("ContentValues", "Root package detected: " + str);  
                return true;  
            } catch (PackageManager.NameNotFoundException unused) {  
            }  
        }  
        return false;  
    }
```

Here it checks for certen apps installed in the device

Again we can tell the hook function if the packagemanager checks for a certen word base on the root app package name to return not exist 
for example if we are using the `com.kingroot.kinguser` app we will make the hook function looks like this 
```js
Java.perform(function(){
   // Root Application bypass
   const PackageManager = Java.use('android.app.ApplicationPackageManager');
   PackageManager.getPackageInfo.overload('java.lang.String', 'int').implementation = function (packageName, flags) {
       if (packageName.includes("kinguser`")){
           console.log(`Bypassing getPackageInfo() on: ${packageName}`);
           return this.getPackageInfo("this.package.does.not.exist", flags);
       }
       console.log(`Calling getPackageInfo() on: ${packageName} with flags=${flags}`);
       return this.getPackageInfo(packageName, flags);
   };
})
```
