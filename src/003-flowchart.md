# 使用 plantuml，用代码画流程图

plantuml类似一门简单语言，用代码逻辑来绘制流程图。本文从流程图元素、控制语句、布局、样式四个维度学习。为后续分析源码时绘制流程图做好准备。

## 流程图元素
- 开始
```plantuml
start
```

- 结束
```plantuml
end
```

- 语句
```plantuml
 :打开扫一扫;
```

- 子流程
```plantuml
:扫码登录|
```

- note
 ```plantuml
 :打水;
 note right
 注意烫伤
 endnote
 ```


## 控制语句

- 条件
    - if else
    ```plantuml
    if (年龄大于18) then (yes)
      :可办理成人身份证;
    elseif(年龄大于3) then(yes)
      :父母代表儿童身份证;
    else (no)
      :不
      符合条件;
    endif
    ```

    - switch

    ```plantuml
    switch (第一字节头两位)
        case ( 00 )
          :整数，占用1字节;
        case ( 01 ) 
          :整数，占用2字节;
        case ( 10 )
          :整数，占用4字节;
        case ( 11 )
          :字符串;
          switch (第二字节头两位)
            case (00)
                :字符串，占用1字节;
            case (01)
                :字符串，占用2字节;
          endswitch
    endswitch
    ```

- while
```plantuml 
while (活着)
    :吃饭;
    :睡觉;
    :打豆豆;
endwhile
:结束生命;
```

## 布局

- fork
```plantuml
fork
    : 唱歌;
fork again
    : 吃饭;
end fork
```

- split
```plantuml
split
    : 唱歌;
split again
    : 吃饭;
end split
```


- group
```plantuml
group 初始化分组 
    :read config file;
    :init internal variable;
end group
group 运行分组
    :wait for user interaction;
    :print information;
end group
```

 - rectangle
```plantuml
rectangle {
    :吃饭;
    :睡觉;
}
```

- swimlanes
```plantuml
|Swimlane1|
:foo1;
|Swimlane2|
:foo2;
:foo3;
|Swimlane1|
:foo4;
```

## 样式
- arrow

```plantuml
: a ;
-[dotted]->
: b ;
-[dashed]->
: c ;
```

- color  
    颜色： 色块优先选择：红、绿、黄、蓝，实在不够用了才会选择：橙、青、紫。白色背景，黑色线条和文字，默认字体。  
    红： #EC5D57 绿： #70BF41 蓝： #51A7F9 黄： #F5D328 紫： #B36AE2 橙： #F39019 青： #00E5F9

    - 箭头颜色
    ```plantuml
    : a ;
    -[#blue,dotted]->
    : b ;
    -[dashed]->
    : c ;
    ```
    - 语句颜色
    ```plantuml
    #blue: a ;
    ```
    - 流程控制颜色
    ```plantuml
    if (ok) then (<color:red> yes)
        -[#blue,dotted]->
        :yes;
    else (no)
        -[dashed]->
        :no;
    endif
    ```
    - group 颜色
    ```plantuml
    group #red test {
    :a;
    }
    ```
    - 泳道颜色
    ```plantuml
    |#red|a|
    : aaaa;
    |#blue|b|
    : bbb;
    |c|
    : ccc;
    ```


## 参考
- [ plantuml ](https://plantuml.com/zh/activity-diagram-beta)

## 总结
本文不是全部语法的枚举。而是从学编程的角度从语句、流程控制、布局、样式学起。够用即可，当有需求不能实现时，请参考上文的参考部分。本文无过组合，比如 `if` 里调用 `case` 再调用 `while`。但 plantuml 是支持的。
