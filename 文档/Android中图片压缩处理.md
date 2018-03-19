#   Android中图片压缩原理

想了解如何将图片进行压缩,首先就清楚Android中是如何决定一张图片分配的内存大小的.
1.  图片的像素点个数.通过宽高计算出
2.  图片每个像素点占用字节数. Android中4中模式
    *   ALPHA_8:每个像素点占一个字节
    *   RGB_565:每个像素点占2个字节
    *   ARGB_4444:每个像素点2个字节
    *   ARGB_8888:每个像素点占4个字节

一张图片在Android中所占的内存大小其实就是图片像素点个数*每个像素点所占的内存.
知道了这个原理之后,我们就知道可以从上述两个方面的进行图片压缩.

```
1.  对图片的宽高进行压缩:
    通过BitmapFactory.Options中的inSampleSize可以控制缩放比例.该值必须是2的幂.
    同时还可以通过Options的inJustDecodeBounds获取原始宽高信息.以此来计算实际需要缩放的比例.

        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(context.getResources(),resId,options);
        int outHeight = options.outHeight;
        int outWidth = options.outWidth;
        Log.d("fxj","原始:    outHeight: "+outHeight+",outWidth: "+outWidth);
        options.inSampleSize = 2;
        options.inJustDecodeBounds = false;
        bitmap = BitmapFactory.decodeResource(context.getResources(),resId,options);

    另外:我们知道Andorid中图片资源保存有几个不同的drawable目录,在图片加载过程中,系统会自动帮我们进行一次缩放.
    mdpi:代表 densityDpi: 160
    xhdpi:  代表densityDpi:   320
    xxhdpi: 代表densityDpi:   480
    xxxhdpi:    代表densityDpi:   640

    假设我们手机本身的densityDpi是为440时.那么图片在哪个文件目录下.图片都会做相应的440/目录所表示densityDpi的倍数变换.
    而且我们上面所设置的inSampleSize也是在系统做倍数转变之后再进行缩放.

2.  对图片的像素点表示做变换.
    同样也是通过BitmapFactory.Options控制.是用的参数是inPreferredConfig. 可以使用的值有4个:ALPHA_8,RAG_565,ARGB4444,ARGB8888.
    默认的是ARGB8888.
    如果对图片像素展示要求不高.可以通过该参数.对图片所占内存进行压缩.
```
