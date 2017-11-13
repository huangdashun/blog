---
title: getColor兼容引起的崩溃
date: 2017-11-13 16:40:46
tags:
---
> 今天将项目跑在系统为5.0的魅族的手机上,哇擦类,崩溃!打开Log,提示调用android的getColor()方法出现 java.lang.NoSuchMethodError: android.content.res.Resources.getColor.

# 怎么会这样
**原来:** context.getColor(res)方法是API23才添加的,context.getDrawable(res)是API21才添加的

# 解决办法
使用ContextCompat兼容类.

```
   @ColorInt
    public static final int getColor(Context context, @ColorRes int id) {
        final int version = Build.VERSION.SDK_INT;
        if (version >= 23) {
            return ContextCompatApi23.getColor(context, id);
        } else {
            return context.getResources().getColor(id);
        }
    }
```


```
 public static final Drawable getDrawable(Context context, @DrawableRes int id) {
        final int version = Build.VERSION.SDK_INT;
        if (version >= 21) {
            return ContextCompatApi21.getDrawable(context, id);
        } else if (version >= 16) {
            return context.getResources().getDrawable(id);
        } else {
            // Prior to JELLY_BEAN, Resources.getDrawable() would not correctly
            // retrieve the final configuration density when the resource ID
            // is a reference another Drawable resource. As a workaround, try
            // to resolve the drawable reference manually.
            final int resolvedId;
            synchronized (sLock) {
                if (sTempValue == null) {
                    sTempValue = new TypedValue();
                }
                context.getResources().getValue(id, sTempValue, true);
                resolvedId = sTempValue.resourceId;
            }
            return context.getResources().getDrawable(resolvedId);
        }
    }
```