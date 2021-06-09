---
title: 代号：宝可梦（二）
date: 2021-06-07 10:34:17
tags:
 - unity
 - 开发技术

categories:
 - 代号：宝可梦
 - 制作过程
---
# 前言
前天做完了对话系统，今天就可以开始制作开头的剧情了。

# UI 自动本地化
游戏界面的 UI 如果一个个去转化语言就会增加很多的工作量。

例如下面这个开头让玩家输入名字的界面，光是 UI 文本就有三个。

![输入名字](https://files.catbox.moe/c30wfa.jpg)

为了减少无意义的劳动，我写了一个可以自动将关键词转化为本地语言的方法。

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class LocaleText : MonoBehaviour
{
    private Text text;

    private void Awake()
    {
        text = GetComponent<Text>();
    }

    private void Start()
    {
        text.text = GameManager.GetLocaleText(text.text);
    }
}
```

只要在文本对象挂上这个脚本，文本就会自动转化为对应的语言了。

在没改造之前，每一个 UI 界面都要这样：

```
public Text tipText, placeText, btnText;
public InputField inputField;

private void Start()
{
    tipText.text = GameManager.uiTexts["inputNameTip"];
    placeText.text = GameManager.uiTexts["inputPlaceholder"];
    btnText.text = GameManager.uiTexts["confirm"];
}
```

改造之后，不需要代码手动本地化。

只要将文本上面的字符串设置为对应的关键词即可：

![文本设置为关键词](https://files.catbox.moe/lkvpbk.jpg)

接着把脚本挂在这个文本上：

![文本挂上本地化脚本](https://files.catbox.moe/00mhir.jpg)

进入游戏测试：

![测试本地化文本](https://files.catbox.moe/e6cmj9.jpg)

文本上的关键词自动转化成对应的语言文字了！

# 弹窗提示
如果用户输入的名字不合法，需要弹出窗口提示玩家输入不正确。

![弹窗提示](https://files.catbox.moe/kxvbfk.jpg)

做了一个简易的弹窗 UI。

# 按键冲突问题
因为本游戏是用键盘操作的，所以会存在一个按键冲突的问题。

比如玩家打开菜单，此时又弹出一个提示框。

玩家按下方向键菜单还在监听按键，导致出现弹窗前面的窗口还能控制的情况。

![菜单的按键冲突问题](https://files.catbox.moe/2uymmv.jpg)

正常的情况应该是：当玩家打开背包，主菜单的按键监听先屏蔽，等玩家关闭背包了，操作权才返回主菜单。

为了避免这种按键冲突问题，需要实现一种特殊的数据结构「栈」。

栈是一种先进后出的结构，先进来的菜单最后一个关闭，正好符合窗口的设计。

## 栈结构
在全局的 UI 管理类 GameManager 加入栈结构：

```
public static Stack windows = new Stack();


public static void PopWindowStack()
{

    Window temp = windows.Pop() as Window;
    Debug.Log("出栈：" + temp.gameObject.name);

    if (windows.Count != 0)
    {
        Window lastMenu = windows.Pop() as Window;
        lastMenu.SetEnabled();
        windows.Push(lastMenu);
    }
}

public static void PushWindowStack(Window window)
{
    Debug.Log("入栈：" + window.gameObject.name);

    if (windows.Count != 0)
    {
        Window lastMenu = windows.Pop() as Window;
        lastMenu.SetDisabled();
        windows.Push(lastMenu);
    }

    window.SetEnabled();
    windows.Push(window);
}

```

这个结构的原理很简单，如下图所示：

![入栈过程](https://files.catbox.moe/0w7boe.jpg)

当第二个菜单被创建的时候，需要判断栈内是否有其他菜单。

如果有的话，就取出最后一个，将其设置为未激活状态，然后重新压回栈中。

再将新创建的菜单激活压入栈内。

![入栈过程2（没有⑤不用找了）](https://files.catbox.moe/ap0djc.jpg)

出栈的过程同理，就不再“灵魂画图”了。

## 菜单基类

创建一个菜单的基类：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Window : MonoBehaviour
{
    protected bool isEnabled;

    public void SetEnabled()
    {
        isEnabled = true;
    }

    public void SetDisabled()
    {
        isEnabled = false;
    }

    private void OnEnable()
    {
        GameManager.PushWindowStack(this);
    }

    private void OnDestroy()
    {
        GameManager.PopWindowStack();
    }
}
```

这里直接使用 Unity 的生命周期函数 `OnEnable` 和 `OnDestroy` 来控制控制入栈和出栈。

这样可以不用手动去控制进出栈，减少工作量。

当一个菜单被激活时就会自动入栈，当它被销毁时就会自动出栈，全自动化的处理，十分方便。

不仅是弹窗要继承窗口类，包括之前设计的对话系统也要让它继承窗口类。

`isEnabled` 用来控制是否激活菜单的按键控制，在子类中用来控制是否允许按键操作。

## 实际测试
写好之后，进入游戏测试一下。

先执行一段对话，然后隔 1 秒的时候出现一个弹窗。

如果没有栈结构，那么在对话和弹窗同时存在的情况下，按 Z 键时，对话会执行下一句，而弹窗也会消失。

（即按键的监听没有被屏蔽，导致两个窗口可以同时操作）

如果有栈的结构，那么在对话还未结束时又弹出一个弹窗，此时若按下 Z 键，会优先关闭弹窗，再按下 Z 键才会使对话进入下一句。

![菜单的执行顺序](https://files.catbox.moe/pqzt5p.gif)

测试结果没问题，这样菜单按键冲突问题也解决了。

# 宝可梦经典开头
宝可梦系列的开头经常是一个博士出来欢迎，然后随手丢一个精灵。

![宝可梦经典开头](https://files.catbox.moe/8rtdlx.jpg)

仿照原作，我自己用 PS 弄了一张渐变的背景。

![博士的背景](https://files.catbox.moe/s76tjd.jpg)

导入游戏场景测试一下：

![场景测试](https://files.catbox.moe/bppmzx.jpg)

看上去有那么回事了，由于现在还在练习像素画的过程，博士的素材暂时没法完成。

此处先用黑色图块代替。

前天顺便也找了对话框的 UI，风格有点不适合游戏，后面再自己做一个新的。

# 后言
昨天感冒了，这篇文章本来是要 6-6 的时候发布的，结果拖到了今天。

目前个人感觉状态十分良好，只要不出什么意外情况，这个宝可梦的游戏在这个月应该就可以出 Demo 版了。
（迷之自信）