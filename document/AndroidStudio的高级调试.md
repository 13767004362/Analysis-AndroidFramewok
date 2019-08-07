

androidStudio除了简单的单行调试外，还有许多的调试高级用法。


#### **Evaluate Expression弹窗**

debug过程中,在Studio-->Run-->Evaluate Expression弹窗,可以用于求值环境。

**常用的场景**：
- 修改变量的值去执行,查看执行结果。
- 调用变量对象的方法去执行，用于查看执行方法的结果。

通过一个案例进一步了解。

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E5%8E%9F%E6%9C%AC%E7%9A%84%E5%8F%98%E9%87%8F%E5%80%BC.png)

例如 , 弹窗中表达式框内输入`result="哈哈"`,点击evaluate按钮,接着继续走调试流程,则最后调试结果如下：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E4%BF%AE%E6%94%B9%E5%8F%98%E9%87%8F%E7%9A%84%E5%80%BC.png)

----

#### **条件断点**

对着代码鼠标左键会打上红色的断点，对着断点右键点击会出现一个弹窗。

该弹窗内有一个执行的条件，可以过滤掉其他的调试流程。

例如,在一个for循环中,只需要调试一个种情况，例如输入`s.equals("3")`,则直接跳到该条件下的调试，如下图所示：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E6%9D%A1%E4%BB%B6%E8%B0%83%E8%AF%95.png)

----

#### **日志断点**

对着代码鼠标左键会打上红色的断点，对着断点右键点击会出现一个弹窗。

当行代码,未添加日志，但又想知道输入出日志

取消勾选suspend选项，出现如下弹窗：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E6%97%A5%E5%BF%97%E8%B0%83%E8%AF%95%E5%89%8D.png)

勾选`Log message to console`和`Evaluate and log`,输入日志信息`"s的值："+s`,如下图所示：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E6%89%93%E5%8D%B0%E6%97%A5%E5%BF%97%E8%B0%83%E8%AF%95.png)

代码调试到这里，**不会停在断点，但会输出日志**，如下：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E6%97%A5%E5%BF%97%E7%BB%93%E6%9E%9C.png)

----

#### **方法断点**

若是想了解某个方法的参数或者返回值，可以使用方法断点,进行调试。
在感兴趣的方法头那一行左键打上断点，如下所示：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E6%96%B9%E6%B3%95%E8%B0%83%E8%AF%95.png)



如果经常跳出函数或者只对某个函数的参数感兴趣,这种类型的断点非常实用。

----

#### **异常断点**

在调试整个项目过程中，需要排除某个特殊的异常，可考虑异常断点。

异常断点，会对特定异常发生时候，会让程序断点停下来的情况。

具体做法是,run-->View BreakPoints --> 绿色的+号，如下所示：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E5%BC%82%E5%B8%B8%E8%B0%83%E8%AF%951.jpg)

例如,这里除数为0,会抛出ArithmeticException。点击 java exception breakpoints选项,在输入框填写ArithmeticException,点击ok完成。
![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E5%BC%82%E5%B8%B8%E8%B0%83%E8%AF%952.png)

运行调试,当捕捉到指定异常时候，会自动停下来,如下图所示：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E5%BC%82%E5%B8%B8%E8%B0%83%E8%AF%95%E7%9A%84%E6%8D%95%E6%8D%89.png)

----

#### **Field WatchPoint**

在多线程情况下,修改共享对象是很常见的，但有时候想了解对象的属性，通过日志查看修改状态是较为复杂.

在了解修改属性的值时候，就可以使用Filed WatchPoint。
当属性每次被修改，程序都会停下来，可以查看修改前的值和修改后的值。

操作方式直接在变量左键，进入Filed WatchPoint。

通过一个小案例，进一步体验。

第一次修改的查看，如下：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E7%AC%AC%E4%B8%80%E6%AC%A1%E4%BF%AE%E6%94%B9.png)

查看上次的变量的值：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E4%BF%AE%E6%94%B9%E5%89%8D%E7%9A%84%E5%80%BC.jpg)


第二次修改的查看，如下：

![image](https://github.com/13767004362/Analysis-AndroidFramewok/blob/master/picture/studioDebug/%E5%B1%9E%E6%80%A7%E7%9A%84%E7%AC%AC%E4%BA%8C%E6%AC%A1%E4%BF%AE%E6%94%B9.png)

**参考资料**：
- [Android Studio你不知道的调试技巧](http://weishu.me/2015/12/21/android-studio-debug-tips-you-may-not-know/)
