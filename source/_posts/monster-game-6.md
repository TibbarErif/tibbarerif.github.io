---
title: 名为怪物的游戏——《星之魔女》FC小游戏移植（六）
date: 2021-05-06 10:00:01
tags:
 - unity
 - 开发技术

categories:
 - 名为怪物的游戏
 - 制作过程
---
## 前言
今天开始设计与角色相关的东西，比如跟随在主角旁边的小鸟、可以吃的加分道具、场景内的敌人，还有 BOSS 战用来显示 BOSS 的血条。

## 宠物跟随
主角在行动时，旁边会有一只小鸟跟随在身边。

（其实就是一个装饰物）

![使魔跟随](https://files.catbox.moe/v6d4wg.jpg)

最早的时候设想过让小鸟可以帮助玩家挡一次子弹，然后小鸟就会死了掉出屏幕。

但是这样太可怜了，所以取消了这个设定。

### 飞行动画
在场景创建一个小鸟对象，然后为它添加飞行动画。

![飞行动画](https://files.catbox.moe/4n7p7y.gif)

### 跟随主角
这里有一个简单的做法就是直接把小鸟放在角色对象底下。

![放在角色对象下](https://files.catbox.moe/b5bsgs.gif)

因为 Pet 变成了 Player 的子节点，所以会自动跟随父节点改变位置和翻转。

然鹅，主角在转向的时候，小鸟会突然“变到”主角身后，很不自然。

### 优化方案
以遛狗为例，狗会在绳子周围走动，人物向前走的时候，狗的移动范围也会发生变化，但狗的行动范围总是不会超过绳子的长度。

因此“绳子”的长度可以理解成宠物的移动范围。

以主角为中心，存在一条“看不见的绳子”，即宠物和角色之间的距离就是小鸟飞行范围的约束。

以角色和小鸟的锚点（即中心点）计算两点的距离即可。

可以用两点间距离公式，也可以直接用 unity 的方法，有现成的方法肯定是直接用啦！

编写 MiniGame_Pet 脚本：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_Pet : MonoBehaviour
{
    public Transform player;
    public float flySpeed = 10f;
    public float followDistance = 10f;

    private void Update()
    {
        if (Vector3.Distance(player.position, transform.position) >= followDistance)
        {
            transform.position = Vector3.Lerp(transform.position, player.position, Time.deltaTime * flySpeed);
        }
    }
}
```

脚本有三个参数，第一个是要跟随的对象，即主角的位置。

第二个是飞行的速度，决定了当主角甩开小鸟的时候，小鸟会以多快的速度追上主角。

第三个参数是跟随范围的距离，当超过这个距离的时候，小鸟就会开始追主角。

![跟随效果](https://files.catbox.moe/h2vvif.gif)

看起来丝滑了很多，小鸟现在还不能跟随角色转身，根据玩家的朝向让小鸟也转向就好了。

![转身效果](https://files.catbox.moe/p1c2xw.gif)

这扑腾翅膀的样子也太萌了~

## 角色系统
基于“万物皆对象”的思想，可以把主角和敌人的相似点抽取出来，做成基类。

主角有动画，敌人也有动画；

主角有血量，敌人也有血量；

诸如此类，它们身上存在许多相似点，所以可以进行抽取。

### 角色基类

新建一个游戏角色的脚本 `MiniGame_Character`。

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public abstract class MiniGame_Character : MonoBehaviour
{
    public float maxHP;
    public GameObject[] bullets;

    protected float currentHP;

    protected Rigidbody2D rb;
    protected Animator animator;

    private void Awake()
    {
        currentHP = maxHP;
        rb = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
    }

    public IEnumerator TakeDamage(float damage)
    {
        animator.SetBool("hurt", true);
        currentHP -= damage;

        yield return new WaitForSeconds(0.05f);

        animator.SetBool("hurt", false);

        if (currentHP <= 0)
        {
            DeadCallback();
        }
    }

    protected abstract void DeadCallback();
}
```

在唤醒的时候自动获取对象的刚体和动画组件，并且赋予当前 HP 属性。

并且声明了一个抽象方法 `DeadCallback`（死亡回调），即当角色死亡的时候会发生什么事情，必须在子类中实现。

虽然游戏中的角色都会死亡，但是玩家死亡了会触发 GameOver，但是敌人死亡了会爆金币，所以要单独实现。

`TakeDamage` 方法是角色的受伤事件，传入一个 damage（伤害值）。

类似攻击事件，这里用到协程，在 0.05s 后解除受伤动画状态。

要调用协程方法必须用 `StartCoroutine`，这样很不方便。

### 延迟类封装

可以将延迟函数封装成一个工具类，然后让子类继承这个类就可以直接使用延迟函数了。

在 `Assets/Scripts` 目录新建一个 Core 文件夹，用来保存游戏的核心类文件。

新建脚本 `TimeManger`：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class TimeManager : MonoBehaviour
{
    /**
     * 重复执行
     */
    protected void RepeatForever(string methodName, float time, float repeatRate)
    {
        InvokeRepeating(methodName, time, repeatRate);
    }

    /**
     * 延迟执行
     */
    protected void Wait(System.Action action, float seconds)
    {
        StartCoroutine(WaitIEnumerable(action, seconds));
    }

    /**
     * 延迟执行方法封装
     */
    private IEnumerator WaitIEnumerable(System.Action action, float seconds)
    {
        yield return new WaitForSeconds(seconds);
        action();
    }

    /**
     * 关闭该脚本上的Timer方法
     */
    protected void ClearTimeer()
    {
        CancelInvoke();
    }
}
```

`TimeManger` 封装了协程的方法，只要直接调用 `Wait` 就可以实现延迟执行。

还有一些其他定时重复执行的方法，以后会用到。

而且这个类必须继承 `MonoBehaviour`，不然没办法调用 unity 提供的延迟函数。

而且继承了 `MonoBehaviour` 可以当做组件挂在场景中的游戏物体上。

### 修改调用方法

未修改前：

```
public IEnumerator TakeDamage(float damage)
{
    animator.SetBool("hurt", true);
    currentHP -= damage;

    if(currentHP <= 0)
    {
        DeadCallback();
    }

    yield return new WaitForSeconds(0.05f);

    animator.SetBool("hurt", false);
}
```

修改后：

```
public void TakeDamage(float damage)
{
    animator.SetBool("hurt", true);
    currentHP -= damage;

    Wait(delegate
    {
        animator.SetBool("hurt", false);

        if (currentHP <= 0)
        {
            DeadCallback();
        }

    }, 0.05f);
}
```

`TakeDamage` 方法已经不再是返回 `IEnumerator` 类型了，因此可以作为普通方法调用。

`delegate` 是 `C#` 的委托类型，可以传入一个匿名函数作为回调，匿名函数的类型是 `System.Action`，可以作为函数的参数。

`TimeManager` 封装的方法：

```
protected void Wait(System.Action action, float seconds)
{
    StartCoroutine(WaitIEnumerable(action, seconds));
}
```

只要传入一个委托和延迟执行的时间，就会自动去调用协程方法。

同理，再去修改之前的 Player 脚本的 Shoot 方法，这样代码又整洁了一些。

## 制作敌人
小游戏只有一关，所以敌人数量不会很多。

包括数个小怪以及最后的 BOSS。

上面已经封装了角色类，现在再封装一层敌人的通用类。

### 敌人基类
敌人也具有共通点，比如死亡会掉落金币（道具）。

把基础方法抽取出来可以提高代码的复用性，节省时间。

新建 `MiniGame_Enemy` 类：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public abstract class MiniGame_Enemy : MiniGame_Character
{
    protected override void DeadCallback()
    {
        throw new System.NotImplementedException();
    }

    protected abstract void MoveAction();

    private void Update()
    {
        MoveAction();
    }
}
```

死亡回调留个空，后面再完善就行了。

然后声明了一个抽象的 `MoveAction`（移动控制），每个敌人的行动模式都不一样，所以要在子类单独实现。

### 漂浮幽灵
幽灵敌人比较简单，它只会在一个区域范围内进行巡逻。

制作飞行动画：

![漂浮幽灵](https://files.catbox.moe/v9ilid.gif)

然后再写一个控制幽灵的逻辑。

幽灵的移动轨迹示意图：

![幽灵的移动轨迹](https://files.catbox.moe/cij098.jpg)

演示效果：

![演示幽灵移动](https://files.catbox.moe/sm5xoq.gif)

### 垃圾桶怪
垃圾桶怪是固定不动的敌人，只有玩家走进攻击范围或者被玩家攻击的时候才会变成攻击形态。

待机状态：

![垃圾桶怪待机动画](https://files.catbox.moe/snldse.gif)

愤怒形态：

![垃圾桶怪愤怒形态](https://files.catbox.moe/h9pa0a.gif)

攻击形态：

![攻击形态](https://files.catbox.moe/8zprhn.gif)

动画的状态转化关系也比较简单。

![垃圾桶怪的动画状态机](https://files.catbox.moe/2rebvq.jpg)

然后为垃圾桶怪添加逻辑处理。

垃圾桶怪前方有一块警戒区域，当玩家进入这个区域的时候，垃圾桶怪就会变成攻击形态追击玩家。

![垃圾桶怪的警戒区域](https://files.catbox.moe/idqygz.jpg)

还有第二种情况，当玩家用子弹攻击垃圾桶怪的时候，如果垃圾桶怪藏在桶里，此时是无敌的。

如果攻击达到一定次数，即使玩家没有踏入警戒区，垃圾桶怪也会变得愤怒，然后开始追击玩家。

（打扰到它睡觉了）

警戒区域也是一个碰撞体，设置为触发器即可。

![警戒区域设置](https://files.catbox.moe/bpsoxg.jpg)

演示效果：

![进入垃圾桶怪的警戒区域](https://files.catbox.moe/3l8ids.gif)

这里会出现一个问题，因为使用了物理系统，所以会产生物理碰撞效果。

![与敌人的物理碰撞效果](https://files.catbox.moe/81hsjo.gif)

但这并不是游戏中应该有的效果，主角撞到敌人时，不应该把敌人撞飞。

只要将刚体组件的类型设置为 `Static`（静态不受外力）即可。

![静态刚体](https://files.catbox.moe/562zu9.jpg)

修正后的效果：

![无法被击飞的垃圾桶怪](https://files.catbox.moe/mq0eet.gif)

垃圾桶怪的行为逻辑还没完成，不过还有不少怪物的动画没完成，今天已经挺晚了，就留到明天再补充。

### 自爆怪
自爆怪是一个组合类型的敌人，包括头顶的“竹蜻蜓”以及本体的“炸弹”。

当它靠近主角的时候，竹蜻蜓就会飞走，然后本体掉落在地上产生爆炸。

![自爆怪](https://files.catbox.moe/6yuzhl.jpg)

添加了“竹蜻蜓”的动画效果：

![竹蜻蜓动画](https://files.catbox.moe/omynqq.gif)

具体的分离逻辑也留到明天完成。

## 后言
今天的进度先到这里了，明天写完最后一篇，小游戏应该就能结束了。
