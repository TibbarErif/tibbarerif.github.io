---
title: 代号：宝可梦（一）
date: 2021-06-05 11:42:54
tags:
 - unity
 - 开发技术

categories:
 - 代号：宝可梦
 - 制作过程
---

# 前言
宝可梦同人游戏正式开始制作。

与怪物游戏同时进行开发，由于许多系统是类似的，所以完成其中一个再改造一下另一个游戏也能使用。

想过用 RPG Maker 来制作这个游戏。

自己从零开始写系统要花费好几倍的精力……

想到原创系统的梦想，还是咬咬牙坚持下去吧。

# 对话系统
昨天晚上就做好了对话系统，但是 UI 还没做。

![对话系统](https://files.catbox.moe/35k8kn.gif)

虽然功能完成了，但还不能直接使用。

![对话系统代码](https://files.catbox.moe/f39ii6.jpg)

现在还只是纯代码的形式，每一句话都要写一段代码。

为了简化工作，对话要做成配置表或者文本形式。

因为宝可梦的对话系统比较简单，不用配置头像，所以我选择最简单的文本形式作为对话系统的数据存储。

![对话文本](https://files.catbox.moe/3jjq7n.jpg)

这个对话系统的数据存储也是想了好久才决定用这种形式。

第一是要直观的看出对话内容，第二是要方便设置和修改。

用 excel 虽然可以实现更加复杂的对话系统，但是配置对话文本花的时间也会翻倍。

按照目前的情况时间并不充裕，我要在最短的时间把游戏赶制出来，选择简单文本是最佳方案。

# 选项系统
要让剧情连贯起来不是一件容易的事情。

所以我的第一步计划是不用连贯的方法，而是分开实现各个系统。

假如有一个场景是一个 NPC 跑过来跟玩家对话，然后问了玩家一个问题，要让玩家做出二选一，选完之后 NPC 就会离开。

这个情景看起来很简单，但是要连贯起来是非常困难的。

在一些大游戏会专门设计一个可视化剧情系统来完成整个剧情动画。

但是我现在的状况不允许慢悠悠的去做一套剧情系统，相反如果不用连贯的方式来实现就会简单得多。

比如对话完了要调出一个选项，那就先结束对话，接着创建一个选项，玩家选择完之后，再重新调用下一段的对话。

理论上可以实现剧情系统的连贯，但是得花好多时间去完成一整套的剧情系统。

缺点是代码会有很多层嵌套，玩家层面感知不到，所以没关系。

能用人力解决的，就先用人力代替吧，节省时间。

![选项系统](https://files.catbox.moe/9dsndy.gif)

选项系统和对话系统是分开的，因此在一个剧情对话中，需要把一段原本完整的剧情分割成多个。

这也是比较麻烦的地方，但还好问题不大。

# 联动效果
为了方便调用，创建对话和选项的方法要封装起来。

创建一个 GameManager 游戏管理类，用于调用一些通用的方法。

![封装方法](https://files.catbox.moe/fid6fp.jpg)

然后试着写一下对话和选项的联动实现剧情的代码实现。

![对话和选项联动的代码](https://files.catbox.moe/si4hkc.jpg)

如果剧情对话的分支比较多的话，就比较蛋疼了……

测试结果：

![剧情对话和选项的联动](https://files.catbox.moe/ysobfs.gif)

看起来是没啥问题了，但代码写起来不是很优雅。

而且在创建选项的时候，这里的文字是没有本地化语言处理的：

```
 private void Select()
{
    List<OptionData> optionDatas = new List<OptionData>();

    optionDatas.Add(new OptionData
    {
        text = "男孩",
        callback = Dialog_2
    });

    optionDatas.Add(new OptionData
    {
        text = "女孩",
        callback = Dialog_3
    });

    GameManager.CallSelectPanel(optionDatas);
}
```

目前游戏有两种语言：简体中文和繁体中文。

如果这样写死代码的话，就没办法实现多语言了。

> 有时候真的想去游戏公司实习一下，看看别人是怎么做的，网上几乎找不到相关的教程，经常卡到头大

要保证能够多语言化，这里的选项文字就得提取出来。

所以我决定单独把 UI 的文本提取成一个 TXT 文件。

# UI 文本
创建一个本地化文本 ui.txt：

![ui.txt](https://files.catbox.moe/m6f6gc.jpg)

这个文件保存了界面 UI 上的文字和对话选项的文字。

UI 文本需要在游戏启动时进行加载。

创建一个 Loading 场景加载资源，加载完成后即跳转到测试场景。

在 GameManager 里用一个静态变量保存所有 UI 文本。

```
public class GameManager : MonoBehaviour
{
    public static Dictionary<string, string> uiTexts = new Dictionary<string, string>();

    public static void LoadLocaleSetting()
    {
        string lang = "zh-cn";
        TextAsset uiText = Resources.Load("Locale/" + lang + "/common/ui") as TextAsset;
        string[] data = uiText.text.Split(Environment.NewLine.ToCharArray());

        foreach (string item in data)
        {
            if (item != "")
            {
                string[] res = item.Split('=');
                uiTexts.Add(res[0], res[1]);
            }
        }
    }
}
```

并且提供了一个加载资源的方法。

这样文本就会全部载入到静态变量中，全局都可以调用。

# 选项本地化
修改之前生成选项的方法：

```
private void Select()
{
    List<OptionData> optionDatas = new List<OptionData>();

    optionDatas.Add(new OptionData
    {
        text = "boy",
        callback = Dialog_2
    });

    optionDatas.Add(new OptionData
    {
        text = "girl",
        callback = Dialog_3
    });

    GameManager.CallSelectPanel(optionDatas);
}
```

这里只要传入 UI 文本中等号左边的 key（键名）就可以。

boy 和 girl 是在 ui.txt 文本中定义的。

只要传入这个键名，就会转化成对应的键值：男孩和女孩。

最后在选项数据里加一个将 key 转化为对应文本的方法即可：

```
public class OptionData
{
    public string text;
    public System.Action callback;

    public string GetText()
    {
        GameManager.uiTexts.TryGetValue(text, out string value);
        return value;
    }
}
```

测试一下大功告成~

这样看起来舒服多了~