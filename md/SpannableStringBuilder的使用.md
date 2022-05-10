SpannableStringBuilder的使用

![image-20211223180645982](SpannableStringBuilder%E7%9A%84%E4%BD%BF%E7%94%A8.assets/image-20211223180645982.png)

实现这种效果，并且蓝色字体可以实现点击效果。其实只需要一个TextView。

今天我们来介绍一下这个强大的控件，SpannableStringBuilder。首先来看官方给它的定义——This is the class for text whose content and markup can both be changed（这是文本的类，其内容和标记都可以更改。）

这个类最重要的一个方法就是`setSpan (Object what, int start, int end, int flags)`

start： 指定Span的开始位置

end：  指定Span的结束位置，并不包括这个位置。

flags：取值有如下四个

`Spannable. SPAN_INCLUSIVE_EXCLUSIVE`：前面包括，后面不包括，即在文本前插入新的文本会应用该样式，而在文本后插入新文本不会应用该样式

`Spannable. SPAN_INCLUSIVE_INCLUSIVE`：前面包括，后面包括，即在文本前插入新的文本会应用该样式，而在文本后插入新文本也会应用该样式

`Spannable. SPAN_EXCLUSIVE_EXCLUSIVE`：前面不包括，后面不包括

`Spannable. SPAN_EXCLUSIVE_INCLUSIVE`：前面不包括，后面包括

what： 对应的各种Span，不同的Span对应不同的样式。已知的可用类有：

`BackgroundColorSpan` : 文本背景色

`ForegroundColorSpan` : 文本颜色

`MaskFilterSpan` : 修饰效果，如模糊(BlurMaskFilter)浮雕

`RasterizerSpan` : 光栅效果

`StrikethroughSpan` : 删除线

`SuggestionSpan` : 相当于占位符

`UnderlineSpan` : 下划线

`AbsoluteSizeSpan` : 文本字体（绝对大小）

`DynamicDrawableSpan` : 设置图片，基于文本基线或底部对齐。

`ImageSpan` : 图片

`RelativeSizeSpan` : 相对大小（文本字体）

`ScaleXSpan` : 基于x轴缩放

`StyleSpan` : 字体样式：粗体、斜体等

`SubscriptSpan` : 下标（数学公式会用到）

`SuperscriptSpan` : 上标（数学公式会用到）

`TextAppearanceSpan` : 文本外貌（包括字体、大小、样式和颜色）

`TypefaceSpan` : 文本字体

`URLSpan` : 文本超链接

`ClickableSpan` : 点击事件