---
title: "Android AntiDebug and Signature Verify Example"
date: 2016-01-09T21:44:08+08:00
categories: ['Android']
tags: ['AntiDebug', 'Signature Verify']
---

Reference:[grafx](http://blog.csdn.net/grafx/article/details/40403577), [IT Dreamer](http://burningcodes.net/%E7%94%A8jni%E5%AE%9E%E7%8E%B0apk%E7%9A%84%E5%8F%8D%E8%B0%83%E8%AF%95/)

## Introduction

We can use lots of tools like apktool, baksmali, dex2jar to convert a android app to java source code easily. So now most of our developers will put the core-function in the native level(with C/C++ code) because the arm-assembly language or C pseudo code are more difficult to read than java.
But others can still read the native code if they really want, they can use IDA to read .so files and debug our apps dynamicly. In order to make our apps difficultly to debug dynamicly, we may will need to add some methods that can provide antidebug functions for us.

This aritcle will provide a simple example about how to antidebug and the sinature-verify.

*PS: Recently I am helping my teacher to prepare a Android-Security Course, so I feel it is time to record something, and* **Sorry for my poor English. (: but I think write article in English is the fast way to help me to improve my English.**

## Sinature Verify

### Java Level

When we publish our apps to the app-store, we will sign apps with our private key, and put the public key into our apps, if others modify our apps, he will re-sign the apps and the public key will be changed, so we can add some code into our apps to check the sinature, if it is not same with ourselves public keys, we can let the apps to exit to prevent the modified apps do something that is harmful to users.
Below is the code in java level:
```Java
public class CheckSign {
    Context context;
    PackageManager packageManager;
    String strPackagename;
    byte[] byteSign;
    String strSign;
    Signature[] signatureArray;

    CheckSign(Context context){
        this.context = context;
    }

    private native void checkSignNative(Context context);
    //The arguement is your public key's value that is deal with md5 and base64
    boolean checkSign(String strOriSign){
        checkSignNative(this.context);
        try{
            packageManager = this.context.getPackageManager();
            strPackagename = this.context.getPackageName();
            signatureArray = this.packageManager.getPackageInfo(strPackagename, PackageManager.GET_SIGNATURES).signatures;

            /*Improper validation of app signatures could lead to
             *issues where a malicious app submits itself to the
             * Play Store with both its real certificate and a
             * fake certificate and gains access to functionality
             * or information it shouldn't have due to another
             * application only checking for the fake certificate
             * and ignoring the rest. Please make sure to validate
             * all signatures returned by this method.
             * */
            for (int i = 0; i < signatureArray.length; i++) {
                Log.v("Verbose: ", "Check Signatures");
                byteSign = signatureArray[i].toByteArray();
                byteSign = CertificateFactory.getInstance("X509").generateCertificate(new ByteArrayInputStream(byteSign)).getEncoded();
                strSign = new String(Base64.encode(MessageDigest.getInstance("md5").digest(byteSign), 19));
                //strSign = Base64.encodeToString(MessageDigest.getInstance("md5").digest(byteSign), 19));
                if (!strOriSign.equals(strSign)) {
                    Log.e("Error: ", "Fake Signature");
                    return false;
                }
                else
                    return true;
            }
        }catch (Exception e){
            Log.e("Error: ", "CheckSign Failed" + e.toString());
            System.exit(-1);
        }
        return false;
    }
}
```

The java code above will do the sinature verify in java level. As what we have metioned, it is easy to be pathed by modify the smali file. So the more safy way is to do it in a .so file, which is so-called "native level".

### Sinature Verify via C

To do it, we should add some code in our java code, and be careful, what this example do is simple add a native funciton that will only do the sign-check, this can be patched easily by modify the smali, too. So in the real env we should merge the code to our core-function, that means if we patched the native method call in java level, the app won't work.
Below code will return the signature, you can do the verify after you get the sinature easily:
```CPP
//Return the String, which is generated via base64(md5(signature));
char *getSignature(JNIEnv* env, jobject obj)
{
    jstring jstringSign;
    jbyte *byteSign;
    char *strSign;
    // 获得Context类
    jclass cls = (*env)->GetObjectClass(env, obj);
    // 得到getPackageManager方法的ID
    jmethodID mid = (*env)->GetMethodID(env, cls, "getPackageManager", "()Landroid/content/pm/PackageManager;");

    // 获得应用包的管理器
    jobject pm = (*env)->CallObjectMethod(env, obj, mid);

    // 得到getPackageName方法的ID
    mid = (*env)->GetMethodID(env, cls, "getPackageName", "()Ljava/lang/String;");
    // 获得当前应用包名
    jstring packageName = (jstring)(*env)->CallObjectMethod(env, obj, mid);

    // 获得PackageManager类
    cls = (*env)->GetObjectClass(env, pm);
    // 得到getPackageInfo方法的ID
    mid  = (*env)->GetMethodID(env, cls, "getPackageInfo", "(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;");
    // 获得应用包的信息
    jobject packageInfo = (*env)->CallObjectMethod(env, pm, mid, packageName, 0x40); //GET_SIGNATURES = 64;
    // 获得PackageInfo 类
    cls = (*env)->GetObjectClass(env, packageInfo);
    // 获得签名数组属性的ID
    jfieldID fid = (*env)->GetFieldID(env, cls, "signatures", "[Landroid/content/pm/Signature;");
    // 得到签名数组
    jobjectArray signatures = (jobjectArray)(*env)->GetObjectField(env, packageInfo, fid);
    // 得到签名
    jobject sign = (*env)->GetObjectArrayElement(env, signatures, 0);


    // 获得Signature类
    cls = (*env)->GetObjectClass(env, sign);
    //得到ByteSign
    mid = (*env)->GetMethodID(env, cls, "toByteArray", "()[B");
    byteSign = (*env)->CallObjectMethod(env, sign, mid);

    //根据Byte生成md5消息摘要
    cls  = (*env)->FindClass(env, "java/security/MessageDigest");
    mid = (*env)->GetStaticMethodID(env, cls, "getInstance", "(Ljava/lang/String;)Ljava/security/MessageDigest;");
    jobject jmd5 = (*env)->CallStaticObjectMethod(env, cls, mid, (*env)->NewStringUTF(env, "md5"));
    mid = (*env)->GetMethodID(env, cls, "update", "([B)V");
    (*env)->CallVoidMethod(env, jmd5, mid, byteSign);
    mid = (*env)->GetMethodID(env, cls, "digest", "()[B");
    //md5散列后的byte[]
    byteSign = (*env)->CallObjectMethod(env, jmd5, mid);

    //生成Base64后的String
    cls = (*env)->FindClass(env, "android/util/Base64");
    mid = (*env)->GetStaticMethodID(env, cls, "encodeToString", "([BI)Ljava/lang/String;");
    //得到base64后的String
    jstringSign = (*env)->CallStaticObjectMethod(env, cls, mid, byteSign, 19);
    strSign = (*env)->GetStringUTFChars(env, jstringSign, NULL);
    //LOGW("%s", strSign);

    return strSign;
}
```

The code above will call the java function via relfcetion, and the second arguement is a Context object that can be passed by declare a java method like
```Java
public native [Return Type] FunctionName(Context, ...);
```

## Antidebug

Simplly, we can start a new process via the fork(), the new process will check the parent's **/proc/[PID]/status** file to see whether the **TracerPid** is 0, if not, kill the parent and then exit. Also, we should prevent others to debug the child process via dynamic debuging, we can let the child process [**ptrace**](https://en.wikipedia.org/wiki/Ptrace) itself.

The code below will provide antidebug function:
```CPP

void antiDebug()
{
    const int bufsize = 1024;
    char filename[bufsize];
    char line[bufsize];
    int pid = getpid();
    sprintf(filename, "/proc/%d/status", pid);
    FILE* fd;
    //Child Proc check the parent's status
    if (fork() == 0) {
        int pt;
        pt = ptrace(PTRACE_TRACEME, 0, 0, 0); //Child Proc anti ptrace
        while (1) {
            fd = fopen(filename, "r");
            if (fd != NULL) {
                while (fgets(line, bufsize, fd)) {
                    if (strncmp(line, "TracerPid", 9) == 0) {
                        int statue = atoi(&line[10]);
                        if (statue != 0) {
                            fclose(fd);
                            int ret = kill(pid, SIGKILL);
                            exit(-1);
                        }
                        break;
                    }
                }
                fclose(fd);
            }
            else {
				;
            }
            sleep(100);
        }
    }
}
```

To do something more, you can also create a marco which will do
```CPP
	fd = fopen(filename, "r");
    if (fd != NULL) {
    	while (fgets(line, bufsize, fd)) {
        	if (strncmp(line, "TracerPid", 9) == 0) {
            	int statue = atoi(&line[10]);
            	if (statue != 0) {
            		fclose(fd);
            		int ret = kill(pid, SIGKILL);
           		 	exit(-1);
           		}
            	break;
            }
		}
    }
	fclose(fd);
```
and put it in a lot of places in your c source code, to add the difficulty of reversing apps.

## Conclusion

In this article we provide a simple example about antidebug and signCheck, it can not prevent others to reverse our apps completely, but it can add the difficulty in reversing. Hope this article can help someone. :)