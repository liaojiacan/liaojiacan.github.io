title: 谈谈多语言设计（二）之Spring多语言
author: Jiacan Liao
tags:
  - i18n
  - 多语言
categories:
  - 架构总结
date: 2018-03-08 19:07:00
---
&emsp;&emsp;现在大部分成熟的web框架默认就支持多语言，如果业务比较简单，使用框架自身的多语言支持就可以了。本文将以SpringMvc为例介绍一下JavaWeb的多语言中的一些关键类。
## 关键字
- i18n
- Locale
- LocaleContext
- MessageSource

## I18N
&emsp;&emsp; i18n(其来源是英文单词 internationalization的首末字符i和n，18为中间的字符数),除了i18n还有L10n、g11n、m17n。

## Locale
&emsp;&emsp;Locale是区域信息，通常locale信息包含应该语言信息和区域标识符，如zh_CN:zh为中文,CN为中国的国家代码。常见的Locale代码可以上网获取。[ Locale Codes](https://www.science.co.il/language/Locale-codes.php)。java中的Locale.java 也定义了常见的区域信息。我们应该尽量使用Locale类型来表达地域信息变量，不应该使用String类型

```
 static public final Locale ENGLISH = createConstant("en", "");

    /** Useful constant for language.
     */
    static public final Locale FRENCH = createConstant("fr", "");

    /** Useful constant for language.
     */
    static public final Locale GERMAN = createConstant("de", "");

    /** Useful constant for language.
     */
    static public final Locale ITALIAN = createConstant("it", "");

    /** Useful constant for language.
     */
    static public final Locale JAPANESE = createConstant("ja", "");

    /** Useful constant for language.
     */
    static public final Locale KOREAN = createConstant("ko", "");

    /** Useful constant for language.
     */
    static public final Locale CHINESE = createConstant("zh", "");
```


```
 //从Locale的构造方法中，也看出Locale中可包含语言（language）,国家(country),变体（variant）
 public Locale(String language, String country, String variant) {
        if (language== null || country == null || variant == null) {
            throw new NullPointerException();
        }
        baseLocale = BaseLocale.getInstance(convertOldISOCodes(language), "", country, variant);
        localeExtensions = getCompatibilityExtensions(language, "", country, variant);
}
```
## LocaleContext
&emsp;&emsp; Spring中存储用户地域信息的上下文，LocaleContext中采用的是ThreadLocal变量来存储信息，线程隔离，这样就可以让locale参数从各层的方法参数中移除。我们在写业务代码时也尽量不要将Locale参数通过方法参数中传入，利用LocaleContext可以让代码显得更优雅点。此外Spring也提供一些LocaleChangeInterceptor 的实现，并不需要我们自己维护这些信息。
- AcceptHeaderLocaleResolver  通过Accept-Language加载locale信息
- CookieLocaleResolver 通过Cookie加载locale信息
- FixedLocaleResolver  全局静态的，返回一个默认的Locale
- SessionLocaleResolver 通过SessionLocaleResolver加载locale信息

## MessageSource
&emsp;&emsp; 对于多语言的翻译，无非就是先定义一些key，然后根据给这些key配置各种locale对应的文本。而Spring中的MessageSource就是维护这些配置信息的组件，Spring中有ResourceBundleMessageSource 和 ReloadableResourceBundleMessageSource的实现，可以将多语言的配置在${basename}_${locale}.properties文件中。ResourceBundleMessageSource 每次修改配置需要重启服务才能生效，而ReloadableResourceBundleMessageSource可以热更新。

&emsp;&emsp; 当然我们也可以自己实现一个MessageSource从DB或者从其他存储源加载配置，后面我将单独写一篇文章介绍如何自定义MessageSource进行拓展的。


## 多语言的处理流程

![upload successful](/images/pasted-1.png)