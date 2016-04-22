---
title: DatePickerDialog的使用
date: 2016-03-28 19:38:09 +08:00
tags: Android Widget
category: Android Dev
---
DatePickerDialog就是那个选时间的控件，控件的使用不难，但是有几处坑要注意一下。
<!-- more -->
先看看基本使用方法：

```
        DatePickerDialog datePickerDialog = new DatePickerDialog(this, new DatePickerDialog.OnDateSetListener() {
            @Override
            public void onDateSet(DatePicker view, int year, int monthOfYear, int dayOfMonth) {
            }
        }, defaultYear, defaultMonth,  defaultDay);
        datePickerDialog.show();

```
设置初始化值，设置回调，很简单。第一个坑来了：

1.设置初始化时间：

构造函数的注释， 没有什么异常的地方：

```
    /**
     * @param context The context the dialog is to run in.
     * @param callBack How the parent is notified that the date is set.
     * @param year The initial year of the dialog.
     * @param monthOfYear The initial month of the dialog.
     * @param dayOfMonth The initial day of the dialog.
     */
         public DatePickerDialog(Context context,
            OnDateSetListener callBack,
            int year,
            int monthOfYear,
            int dayOfMonth) {
        this(context, 0, callBack, year, monthOfYear, dayOfMonth);
    }
```

而实际上OnDateSetListener的注释里却是这样的：

```
    public interface OnDateSetListener {

        /**
         * @param view The view associated with this listener.
         * @param year The year that was set.
         * @param monthOfYear The month that was set (0-11) for compatibility
         *  with {@link java.util.Calendar}.
         * @param dayOfMonth The day of the month that was set.
         */
        void onDateSet(DatePicker view, int year, int monthOfYear, int dayOfMonth);
    }
    
```

所以monthOfYear， 是从0开始的。但是dayOfMonth，却是从1开始的。所以在OnDateSetListener的回调中获取真实月份是要加一的：monthOfYear + 1。

接着说第二个坑：

2. 设置可选的最大时间：

先在DatePickerDialog找到DatePicker吧，然后调用setMaxDate();

如果去搜索“DatePickerDialog设置最大日期”，会找到如下方法：

在show的时机调用：

```

	findDatePicker((ViewGroup) getWindow().getDecorView())

```

findDatePicker的实现：

```

        private DatePicker findDatePicker(ViewGroup group) {
            if (group != null) {
                for (int i = 0, j = group.getChildCount(); i < j; i++) {
                    View child = group.getChildAt(i);
                    if (child instanceof DatePicker) {
                        return (DatePicker) child;
                    } else if (child instanceof ViewGroup) {
                        DatePicker result = findDatePicker((ViewGroup) child);
                        if (result != null)
                            return result;
                    }
                }
            }
            return null;
        }

```


其实不需要这么做，DatePickerDialog有获取DatePicker的方法getDatePicker()。

然后如果想把可设置的最大时间设置成当天，DatePicker＃setMaxDate(System.currentTimeMillis()) ,  实际操作中发现，如果系统时间是2月29号，那么设置上去的最大的时间自动跳到下个月份。
所以在DatePickerDialog的初始化构造函数中执行：

```
            Calendar c = Calendar.getInstance();
            c.set(Calendar.DAY_OF_MONTH, 1); // 不能设置超过28，估计跟二月限制有关
            getDatePicker().setMaxDate(c.getTimeInMillis());

```
 
3. 隐藏日期选择

DatePicker一般都是可以选年／月／日的，而需求要求隐藏日控件。

仔细看了下，DatePicker的源码不复杂，很快能找到一个Layout　date_picker_legacy.xml :

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:layout_gravity="center_horizontal"
    android:orientation="horizontal"
    android:gravity="center">

    <LinearLayout android:id="@+id/pickers"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:orientation="horizontal"
        android:gravity="center">

        <!-- Month -->
        <NumberPicker
            android:id="@+id/month"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="1dip"
            android:layout_marginEnd="1dip"
            android:focusable="true"
            android:focusableInTouchMode="true"
            />

        <!-- Day -->
        <NumberPicker
            android:id="@+id/day"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="1dip"
            android:layout_marginEnd="1dip"
            android:focusable="true"
            android:focusableInTouchMode="true"
            />

        <!-- Year -->
        <NumberPicker
            android:id="@+id/year"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="1dip"
            android:layout_marginEnd="1dip"
            android:focusable="true"
            android:focusableInTouchMode="true"
            />

    </LinearLayout>

    <!-- calendar view -->
    <CalendarView
        android:id="@+id/calendar_view"
        android:layout_width="245dip"
        android:layout_height="280dip"
        android:layout_marginStart="44dip"
        android:layout_weight="1"
        android:focusable="true"
        android:focusableInTouchMode="true"
        android:visibility="gone"
        />

</LinearLayout>

```

找到id为day的NumberPicker控件，隐藏即可。可以通过如下方法：

```
        private ViewGroup findNumberPickerContainer(ViewGroup group) {
            if (group != null) {
                for (int i = 0, j = group.getChildCount(); i < j; i++) {
                    View child = group.getChildAt(i);
                    if (child instanceof NumberPicker) {
                        return group;
                    } else if (child instanceof ViewGroup) {
                        ViewGroup result = findNumberPickerContainer((ViewGroup) child);
                        if (result != null) {
                            return result;
                        }
                    }
                }
            }
            return null;
        }

```

或者更直接一点findViewById，不过这么做都是有坑的。
