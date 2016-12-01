---
layout: post
title: Work Note
subtitle:   "工作学习随笔记"
date: 2016-11-22 18:34:39
catalog:    true
author:     "Jam"
header-img: "post-bg-2015.jpg"
tags:
  - 工作笔记
  - 学习笔记
---
### Timer Task hanlder 案例应用
> 使用timerTask制定定时任务

``` java
//TimerTas 类
private class LongPressTask extends TimerTask {
    int mSelection;
    public LongPressTask(int selection) {
        mSelection = selection;
    }
    @Override
    public void run() {
        longClicked = true;
        Message msg = new Message();
        msg.what = mSelection;
        mTempHanlder.sendMessage(msg);
    }
};
//初始化
private LongPressTask lpTask;
private Timer lpTimer;
private Handler mTempHanlder = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        if(msg.what == FASTFORWARD)
          ;
        else if(msg.what == REVERSE)
          ;  
    }
};
//启用task
lpTask = new LongPressTask(FASTFORWARD);
lpTimer =new Timer();
lpTimer.schedule(lpTask,300,500);
//停用Task
lpTimer.cancel();
```

### Android窗口打开动画
> 控制进入界面的动画

``` xml
<!-- Theme中应用 -->
<item name="android:windowAnimationStyle">@style/music_windowAnimationStyle</item>
<!-- sytle -->
<style name="music_windowAnimationStyle" parent="android:style/Animation.Translucent">
    <item name="android:windowEnterAnimation">@null</item>
    <item name="android:windowExitAnimation">@null</item>
    <item name="android:activityOpenEnterAnimation">@null</item>
    <item name="android:activityOpenExitAnimation">@null</item>
</style>
```

### 根据当前文件获取当前目录
``` java
String[] filePath = mFileUri.toString().split("/");
String fileDirectoryString = "";
for (int i = 0; i < filePath.length - 1; i ++) {
    fileDirectoryString += filePath[i] + "/";
}
```

### 查询当前目录下的所有视频不包括子目录
> query();  @arg1 uri, @arg2 要查的列，@arg3 where 语句，@arg4 where语句参数，@arg5排序。
sqlit3语句不太会用，应该有更好的方法

``` java
private boolean getFilePathVideoList(String filePath) {
    String[] cols = new String[] {MediaStore.Audio.Media._ID, MediaStore.Video.Media.TITLE,
            MediaStore.Video.Media.MIME_TYPE, MediaStore.Video.Media.DATE_TAKEN, MediaStore.Video.Media.DATA};
    String where = MediaStore.Video.Media.DATA +" like '%" + filePath  + "%'";
    ContentResolver resolver = getContentResolver();
    if (resolver == null) {
        Log.d(TAG, "getFilePathVideoList: resolver is null");
        return false;
    } else {
        Cursor cursor = null;
        try {
            cursor = resolver.query(MediaStore.Video.Media.EXTERNAL_CONTENT_URI, cols,
                    where, null, null);//查询当前目录下所有的视频
            if (cursor != null && cursor.moveToFirst()) {
                do {
                    String[] pathArray= cursor.getString(4).split(filePath);
                    if (!pathArray[1].contains("/")) { //过滤子目录的视频
                        HashMap<String, Object> map = new HashMap<String, Object>();
                        map.put(MediaStore.Audio.Media.TITLE,
                                cursor.getString(cursor.getColumnIndexOrThrow(MediaStore.Audio.Media.TITLE)));
                        map.put(MediaStore.Audio.Media.MIME_TYPE,
                                cursor.getString(cursor.getColumnIndexOrThrow(MediaStore.Video.Media.MIME_TYPE)));
                        map.put(KEY_EXTERNAL_CONTENT_URI, ContentUris.withAppendedId(MediaStore.Video.Media.EXTERNAL_CONTENT_URI,
                                cursor.getInt(cursor.getColumnIndexOrThrow(MediaStore.Audio.Media._ID))));
                        map.put("filedate",
                                cursor.getLong(cursor.getColumnIndexOrThrow(MediaStore.Video.Media.DATE_TAKEN)));
                        fileManagerVideoList.add(map);
                    }
                } while (cursor.moveToNext());
            }
        } catch (Exception e) {
            Log.d(TAG, "getFilePathVideoList: fail");
            return false;
        } finally {
            Log.d(TAG, "getVideoList: fileManagerVideoListSize::" + fileManagerVideoList.size());
            if (cursor != null)
                cursor.close();
        }
    }
    return true;
}
```
### 调节视频播放速度接口
``` java
//f （0.5～2.0）
mMediaPlayer.setPlaybackParams(mMediaPlayer.getPlaybackParams().setSpeed(f));
```
### SharedPreferences使用
``` java
//存入
SharedPreferences mSharedPreferences = PreferenceManager.getDefaultSharedPreferences(context);
mSharedPreferences.edit().putInt("search_position", 1).commit();
//取出
SharedPreferences mSharedPreferences = PreferenceManager.getDefaultSharedPreferences(context);
int serachIndex = mSharedPreferences.getInt("serach_area_tip", 0);

//跨应用取值
//写入com.test.app
SharedPreferences preferences = mContext.getSharedPreferences("other_app_tag", Context.MODE_WORLD_READABLE|Context.MODE_WORLD_WRITEABLE|Context.MODE_MULTI_PROCESS);
Editor editor = preferences.edit();
editor.putLong("keyword", value);
editor.clear().commit();
//取值
try {
		Context contextFromGallery = null;
		contextFromApp = createPackageContext(
				"com.test.app", MODE_WORLD_READABLE | MODE_WORLD_WRITEABLE | MODE_MULTI_PROCESS);
		SharedPreferences sp = contextFromApp
				.getSharedPreferences("other_app_tag",
						MODE_WORLD_READABLE | MODE_WORLD_WRITEABLE | MODE_MULTI_PROCESS);
		vaule = sp.getLong("keyword", 0);
		Log.d(TAG, getUri + " value = " + vaule);
} catch (Exception e) {
		Log.d(TAG, "com.test.app Package not found!");
}
```

### 格式化文件大小
``` java
private static String formateFileSize(long fileSize) {
DecimalFormat dFormat = new DecimalFormat("#.00");
String fileSizeString = "";
String wrongString = "0 B";
if (fileSize == 0) {
  return wrongString;
}
if (fileSize < 1024) {
  fileSizeString = dFormat.format((double)fileSize) +" B";
} else if (fileSize < 1048576) {
  fileSizeString = dFormat.format((double)fileSize/1024) + " KB";
} else if (fileSize < 1073741824) {
  fileSizeString = dFormat.format((double)fileSize/1024/1024) + " MB";
} else {
  fileSizeString = dFormat.format((double)fileSize/1024/1024/1024) + " GB";
}
return fileSizeString;
}
```
### 格式化时间
``` java
public static String formatDuration( int duration) {
     int h = duration / 3600;
     int m = (duration - h * 3600) / 60;
     int s = duration - (h * 3600 + m * 60);
     String durationValue;
     String wrongString = "00:00:00";
     if (duration == 0) {
   return wrongString;
 }
     if (h == 0) {
         if (m == 0 && s == 0) s = 1;//[BUGFIX]-Modify by TCTNJ,su.jiang, 2016-03-09,PR1759891
         durationValue = String.format("00:%02d:%02d", m, s);
     } else {
         durationValue = String.format("%d:%02d:%02d", h, m, s);
     }
     return durationValue;
 }
```
