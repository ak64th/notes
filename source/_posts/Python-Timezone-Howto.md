---
title: "如何处理Python的datetime时区问题"
date: 2018-03-12T10:24:46+08:00
tags:
- python
- django
keywords:
- python
- datetime
---
`datetime`是python的标准模块，但设计上有一些缺陷，使用中很容易出错。这篇博客记录一些我跳过的坑。
<!-- more -->
## 时区信息

`datetime`是python的标准模块，但设计上有一些缺陷，使用中很容易出错。

比如`datetime.datetime`的实例就存在着timezone aware和unaware两种不同的状态，行为会有不同。`datetime.datetime`类内部保存一个64位整数时间戳，精确到微秒，同时还有一个tzinfo属性保存时区信息，tzinfo不为None的`datetime`实例处于timezone aware状态，反之处于unaware状态。

`datetime.datetime.now()`在没有参数时返回系统本地时间，`datetime.datetime.utcnow()`返回代表当前utc时间的`datetime.datetime`实例，但这个实例的tzinfo为None，这是因为python标准库中并没有包含时区数据库，只是从操作系统获取时间戳。

``` python
import datetime
datetime.datetime.now().tzinfo is None  # True
datetime.datetime.utcnow().tzinfo is None  # True
```

`datetime.datetime`类的构造器和`datetime.datetime.now()`方法都接收一个可选的`tzinfo`参数，当提供这个参数时返回的是该时区的`datetime`实例。问题是，`tzinfo`是什么？ `tzinfo`应该是一个`datetime.tzinfo`子类的实例。

自制一个utc时区并用它生成timezone aware的`datetime.datetime`实例：

``` python
import datetime

class UTC(datetime.tzinfo):
    """UTC"""
    def utcoffset(self, dt):
        return datetime.timedelta(0)

    def tzname(self, dt):
        return "UTC"

    def dst(self, dt):
        return datetime.timedelta(0)

datetime.datetime.now(UTC())
# datetime.datetime(2018, 3, 15, 6, 44, 13, 343971, tzinfo=<__main__.UTC>)
epoch = datetime.datetime(1970, 1, 1, tzinfo=UTC())
# datetime.datetime(1970, 1, 1, 0, 0, tzinfo=<__main__.UTC>)
```

[`pytz`](https://pypi.org/project/pytz/)是python世界时区数据库的事实标准，接下来使用pytz的时区。


## 获取时间戳

如果只需要获取当前的UNIX Timestamp，最快的方法是使用`time.time`函数。但是如何获取某一个`datetime.datetime`实例对应的时间戳呢。

如果使用python3，这个问题可以用`datetime.datetime.timestamp()`方法轻松解决，但在python 2下需要格外小心。

python2中可以手动计算timedelta的秒数。

``` python
import datetime
import pytz
epoch = datetime.datetime(1970, 1, 1, tzinfo=pytz.UTC)
now = datetime.datetime.now(pytz.UTC)
(now - epoch).total_seconds()
# 1521096253.343971
```

需要注意，timezone aware和unaware的`datetime.datetime`实例之间不能进行差值运算。

另外一种方法，`time`模块的`mktime`函数也可以按timetuple产生timestamp，但`mktime`函数预期接受的是本地时间，所以需要先转换到和操作系统统一时区进行计算才能得到正确的结果。[`tzlocal`](https://pypi.org/project/tzlocal/)是一个获取操作系统当前时区的工具库。

``` python
import time
import tzlocal
current_tz = tzlocal.get_localzone()
# <DstTzInfo 'Asia/Shanghai' LMT+8:06:00 STD>
local_now = now.astimezone(current_tz)
# datetime.datetime(2018, 3, 15, 14, 44, 13, 343971, tzinfo=<DstTzInfo 'Asia/Shanghai' CST+8:00:00 STD>)
time.mktime(now.astimezone(current_tz).timetuple())
# 1521096253.0
```

实测使用`timedelta.total_seconds()`的方法速度更快，使用上也更方便。


# Django处理时区的方式

[Django](https://pypi.org/project/Django/)是目前最重要的python web框架，[它对于时区的支持方式](https://docs.djangoproject.com/en/1.11/topics/i18n/timezones/)非常有参考性。

`Django`对时区的支持由[`USE_TZ`](https://docs.djangoproject.com/en/1.11/ref/settings/#std:setting-USE_TZ)选项决定。在`USE_TZ`为`False`时，`Django`只使用timezone unaware datetime。当`USE_TZ`为`True`，日期数据会转成到utc时区再保存进数据库。以前`pytz`是`Django`的可选依赖，从1.11开始，`pytz`成为`Django`的必须依赖。并且Django官方文档建议无论是否需要使用国际化和本地化功能，都应该开启`USE_TZ`选项。

对于`datetime`模块的缺陷，Django采取的处理方式是所有`datetime.datetime`实例都使用utc时区，只在需要输出人类可读的信息时转换到当前时区。

如果需要在模板中输出日期，可以使用`date`filter。

``` html
    <div class="info-field">
      <div class="info-field-name">到期时间</div>
      <div class="info-field-value">{{ device.period_end_time|date:"Y-m-d H:i:s" }}</div>
    </div>
```

如果一个filter预期接受带有本地时区信息的`datetime`对象，需要在注册的时候传入`expects_localtime=True`

``` python
@register.filter(expects_localtime=True, is_safe=False)
def date(value, arg=None):
    """Formats a date according to the given format."""
```

django在输出模板时会使用`django.utils.timezone.localtime`函数把日期转换到本地时区。

在向数据库存储日期信息时，如果使用的是postgresql，会在数据库中保存时区信息，`USE_TZ`选项可以切换。
如果使用其他数据库，数据库内会存储utc时区的时间。如果改变了`USE_TZ`选项，需要手动修改数据库内数据到正确的时间。

在按日期进行group by操作时，有可能需要调用数据库的时区转换函数，数据库系统可能需要额外配置才能进行时区转换。
比如mysql需要[建立时区表](https://dev.mysql.com/doc/refman/5.7/en/mysql-tzinfo-to-sql.html)才能使用CONVERT_TIMEZONE函数。需要注意系统自带的时区表和pytz的可能不一致

# DST

在有夏令时制度(Daylight Saving Time)的地区，一年内某一个小时会重复两次，用`datetime.datetime`的数据格式无法表示其中的区别。这也是使用UTC时区的另一个原因。

在使用`astimezone`转换时区时如果发生了跨域DST时区的状况，结果可能会不准确。使用`pytz.timezone.normalize`方法可以修正这一问题。

*为什么本地时区的偏差值不是整数小时？*

某一地区的时区并不是固定的。比如在北美州的大部分地区，在夏时制以外的时间处于比UTC晚六个小时的`CST -6`时区，在夏时制期间则处于比UTC晚五个小时的`CDT -5`时区。时区可以看作一个签名为`fn (region, date)`的函数，所以当我们运行`pytz.timezone('Asia/Shanghai')`时，pytz无法得知应该返回哪个时间的时区信息，默认返回数据库内最早的记录，也就是`Local Mean Timezone`的信息。

具体可以看`pytz.tzfile.build_tzinfo()`函数。

来看看时区信息具体有哪些：

```python
import pytz

sh = pytz.timezone('Asia/Shanghai')

sh._utc_transition_times
'''
[datetime.datetime(1, 1, 1, 0, 0),
 datetime.datetime(1901, 12, 13, 20, 45, 52),
 datetime.datetime(1940, 5, 31, 16, 0),
 datetime.datetime(1940, 10, 12, 15, 0),
 datetime.datetime(1941, 3, 14, 16, 0),
 datetime.datetime(1941, 11, 1, 15, 0),
 datetime.datetime(1942, 1, 30, 16, 0),
 datetime.datetime(1945, 9, 1, 15, 0),
 datetime.datetime(1946, 5, 14, 16, 0),
 datetime.datetime(1946, 9, 30, 15, 0),
 datetime.datetime(1947, 4, 14, 16, 0),
 datetime.datetime(1947, 10, 31, 15, 0),
 datetime.datetime(1948, 4, 30, 16, 0),
 datetime.datetime(1948, 9, 30, 15, 0),
 datetime.datetime(1949, 4, 30, 16, 0),
 datetime.datetime(1949, 5, 27, 15, 0),
 datetime.datetime(1986, 5, 3, 18, 0),
 datetime.datetime(1986, 9, 13, 17, 0),
 datetime.datetime(1987, 4, 11, 18, 0),
 datetime.datetime(1987, 9, 12, 17, 0),
 datetime.datetime(1988, 4, 16, 18, 0),
 datetime.datetime(1988, 9, 10, 17, 0),
 datetime.datetime(1989, 4, 15, 18, 0),
 datetime.datetime(1989, 9, 16, 17, 0),
 datetime.datetime(1990, 4, 14, 18, 0),
 datetime.datetime(1990, 9, 15, 17, 0),
 datetime.datetime(1991, 4, 13, 18, 0),
 datetime.datetime(1991, 9, 14, 17, 0)]
'''
sh._transition_info
'''
[(datetime.timedelta(0, 29160), datetime.timedelta(0), 'LMT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST'),
 (datetime.timedelta(0, 32400), datetime.timedelta(0, 3600), 'CDT'),
 (datetime.timedelta(0, 28800), datetime.timedelta(0), 'CST')]
'''
```

使用`now`或`astimezone`等方法时pytz会把时区调整到当前使用的标准时区。如果需要处理带有这些遗留时区信息的datetime对象，可以用`normalize`方法：

``` python
lmt_now = now.replace(tzinfo=current_tz)
# datetime.datetime(2018, 3, 15, 6, 44, 13, 343971, tzinfo=<DstTzInfo 'Asia/Shanghai' LMT+8:06:00 STD>)
lmt_now = current_tz.normalize(lmt_now)
# datetime.datetime(2018, 3, 15, 6, 38, 13, 343971, tzinfo=<DstTzInfo 'Asia/Shanghai' CST+8:00:00 STD>)
```

另外宽泛的说，`datetime.datetime.replace()`方法非常容易引入bug，应该尽量避免使用这个方法。如果需要构造一个带有时区信息的datetime实例，使用`datetime.astimezone()`或者`pytz.timezone.localize()`：

```python
import pytz

sh = pytz.timezone('Asia/Shanghai')

# Don't do this
datetime.datetime(2019, 12, 31, tzinfo=sh)
# Nor this
datetime.datetime(2019, 12, 31).replace(tzinfo=sh)
# datetime.datetime(2019, 12, 31, 0, 0, tzinfo=<DstTzInfo 'Asia/Shanghai' LMT+8:06:00 STD>)

# Do this
sh.localize(datetime.datetime(2010, 12, 31))
# Or
datetime.datetime(2019, 12, 31).astimezone(sh)
# datetime.datetime(2019, 12, 31, 0, 0, tzinfo=<DstTzInfo 'Asia/Shanghai' CST+8:00:00 STD>)
```

# Reference

[从字符串获取时间戳](https://aboutsimon.com/blog/2013/06/06/Datetime-hell-Time-zone-aware-to-UNIX-timestamp.html)
[设置MySql的默认时区](https://www.pending.io/django-mysql-time-zones-and-how-to-fix-it/)
