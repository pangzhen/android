---
layout: post
title: "低版本CompileSdk报roundIcon错误"
---

Error:(11) No resource identifier found for attribute 'roundIcon' in package 'android'


在Android studio 2.3上，compileSdkVersion用低版本比如19对应的4.4编译工程，会报出找不到'roundIcon'的错误，原因显然是跟资源文件有关，只是删掉`android:roundIcon="@mipmap/ic_launcher_round"`这句，再次sync工程的时候又会自动添加上去。  
解决办法：  
> * AndroidManifest.xml  
> 删除 android:roundIcon="@mipmap/ic_launcher_round"
> 
> * activity_main.xml  
> 布局样式android.support.constraint.ConstraintLayout改成其他旧版本支持的样式，比如RelativeLayout等  
> 删除 xmlns:app="http://schemas.android.com/apk/res-auto"  
> 删除 app:layout_constraint 相关
> 
> * build.gradle  
> 删除 compile 'com.android.support.constraint:constraint-layout:1.0.2'
> 