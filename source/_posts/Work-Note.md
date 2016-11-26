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
