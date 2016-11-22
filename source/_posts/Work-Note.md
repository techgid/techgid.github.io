---
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
