---
title: view或布局转成Bitmap
date: 2017-11-01 13:14:35
tags: View
---
**需求**:大家工作的时候肯定会碰到将页面截图分享,最近在做项目中,需要将几块view拼接起来转成Bitmap,所以记录一下过程.
# 获取view的Bitmap对象.
---

主要涉及了三个方法,setDrawingCacheEnabled、buildDrawingCache和getDrawingCache,大部分view如果没有设置setDrawingCacheEnabled(true),来启用绘制缓存功能,那默认不开启.

---
## 获取最新DrawingCache的两种方式

### 方式一:
```
view.setDrawingCacheEnabled(true);
Bitmap drawingCache = view.getDrawingCache();
```

### 方式二
```
view.buildDrawingCache();
Bitmap drawingCache = view.getDrawingCache();
```

*注意事项*:在调用setDrawingCacheEnabled(true);以后就不要再调用buildDrawingCache方法了,在源码中buildDrawingCache会调用destroyDrawingCache方法对之前的DrawingCache回收

# 拼接Bitmap
## 创建Canvas
```
Bitmap bitmap = Bitmap.createBitmap(width, height, Config.ARGB_8888);        
Canvas canvas = new Canvas(bitmap);
```
*注意事项*:创建canvas的时候必须传入Bitmap,因为只有Bitmap才能保存像素,用来承载画的内容.

### 拼接
直接上代码

```
//根据总高度创建一个父Bitmap(即需要分享的)
        Bitmap parentBitmap = Bitmap.createBitmap(dataView.getWidth(), dataView.getHeight() + mapBitmap.getHeight() + heartChart.getHeight(), Bitmap.Config.ARGB_8888);
        Canvas parentCanvas = new Canvas(parentBitmap);
        Paint paint = new Paint();
        //地图map
        parentCanvas.drawBitmap(mapBitmap, 0, 0, paint);
        //将数据view转成bitmap
        Bitmap dataBitmap = convertViewToBitmap(dataView);
        parentCanvas.drawBitmap(dataBitmap, 0, mapBitmap.getHeight(), paint);
        //心率图表
        Bitmap heartBitmap = convertViewToBitmap(heartChart);
        parentCanvas.drawBitmap(heartBitmap, 0, dataBitmap.getHeight() + mapBitmap.getHeight(), paint);
		//释放资源
        dataBitmap.recycle();
        heartBitmap.recycle();
        mapBitmap.recycle();
    /**
     * 将view转换成bitmap
     *
     * @param view
     * @return
     */
    private static Bitmap convertViewToBitmap(View view) {
        view.buildDrawingCache();
        return view.getDrawingCache();
    }
```
用canvas去drawBitmap的时候,按照UI给的view顺序去设置left,top就好.

# 最后

当我拼接完分享后,发现有的view分享出去变成了黑色,只需要给view添加一层背景颜色即可.

例如:将上文中的heartBitmap原始的view的根布局加一层background即可.