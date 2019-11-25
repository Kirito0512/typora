在项目里做各种输入框时，经常会用到EditText，一般产品都会对输入内容有各种要求，这里记录一下一些心得

有一个通用的办法，就是使用InputFilter

EditText有一个方法，setFilters，可以传入一个InputFilter的数组。

```
InputFilter[] filters = {new InputFilter.LengthFilter(8), KTVUIUtility.getNameFilter()};
mNickNameEt.setFilters(filters);
```

使用InputFilter,可以实现以下需求:

#### 1. 限制输入长度

> 这个是最简单的，因为InputFilter类内置了对应的静态类，LengthFilter，按照下面调用就行

`new InputFilter.LengthFilter(8)`

#### 2. 限制输入的内容

> 这个根据我们限制的内容的不同，我们需要了解对应的，正则表达式。一些常见的正则表达式，可以参考文末的链接。

#### 2.1过滤各种特殊字符

> 下面举一个，限制只能输入 汉字，英文，数字的例子

```
    public static InputFilter getNameFilter() {
        return new InputFilter() {
            @Override
            public CharSequence filter(CharSequence source, int start, int end, Spanned dest, int dstart, int dend) {
            		// 注意重点就是，compile中的内容，根据需要替换这里的，正则表达式即可
                Pattern p = Pattern.compile("^[\\u4E00-\\u9FA5A-Za-z0-9]+$");
                Matcher m = p.matcher(source.toString());
                if (!m.matches()) return "";
                return null;
            }
        };
    }
```

#### 2.2 过滤换行

> 当然也不是一定要用正则表达式

```
    // 过滤换行
    public static InputFilter getEnterKeyFilter() {
        return new InputFilter() {
            @Override
            public CharSequence filter(CharSequence charSequence, int i, int i1, Spanned spanned, int i2, int i3) {
                return charSequence.toString().replace("\n","");
            }
        };
    }
```



#### 参考

[正则表达式在线测试 | 菜鸟工具](https://c.runoob.com/front-end/854)

