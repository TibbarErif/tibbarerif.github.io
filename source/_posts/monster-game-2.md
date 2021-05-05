---
title: 名为怪物的游戏——《星之魔女》FC小游戏移植（二）
date: 2021-05-03 00:44:28
tags:
 - unity
 - 开发技术

categories:
 - 名为怪物的游戏
 - 制作过程
---
## 前言
前面一篇文章实现了控制角色移动，今天开始实现角色的攻击事件。

## 攻击事件
玩家控制的小魔女可以发射「星之弹」攻击敌人。

这是子弹的特写：

![QQ20210503-004833.jpg](https://i.loli.net/2021/05/03/T34xrIR89U7YlXu.jpg)

演示效果（图片加载比较慢）：

![子弹](https://files.catbox.moe/d1kake.gif)

静帧图：

![子弹2](https://files.catbox.moe/z7fu39.jpg)

大致就是从手中发射星星弹攻击敌人。

要实现角色发射子弹攻击，需要一连串的步骤，接下来就一步一步的进行说明。

### 制作子弹
因为子弹是动态生成的，所以把子弹做成一个 prefab（预制体）。

放在 Resources/Prefabs/MiniGame 目录下备用，子弹还有很多种，这里就把星弹取名为：StarBullet。

![子弹3](https://files.catbox.moe/2wcsfw.jpg)

### 子弹发射口
按下攻击键会发射子弹，按住攻击键会连续发射子弹。

首先创建一个子弹发射的位置，这是一个看不见的透明物体（标红位置）：

![发射口](https://files.catbox.moe/2033ns.jpg)

这个发射口位于角色节点底下，这样就会跟随角色转向而改变位置。

### 按键攻击

我们希望按下 `Z` 或 `J` 键来发射子弹，因此需要修改 unity 自带的按键指令：

![修改按键](https://files.catbox.moe/u3rzlr.jpg)

找到 `Fire1`，别名需要指定两个按键，`negative button` 和 `postive button`，即消极的按钮和积极的按钮。

当按下消极的按钮时，`GetAxis` 的值就会逐渐趋向于 -1，即负向增加；

当按下积极按钮时，`GetAxis` 的值就会逐渐趋向于 1，即正向增加。

对于攻击指令，只要判定 `GetAxis` 不等于 0 即玩家正在按攻击键。

在 Player 脚本添加攻击按键监听：

```
private void ShootEvent()
{
    if (Input.GetAxisRaw("Fire1") != 0)
    {
        Debug.Log("攻击");
    }
}

private void Update()
{
    ShootEvent();
}
```

现在按下攻击键只是在控制台打印“攻击”两个字，接下来要生成上面的子弹。

### 生成子弹

通过 `Resources` 方法动态加载预制体，然后再通过 `Instantiate` 生成游戏对象：

```
private void CreateBullet()
{
    GameObject prefab = Resources.Load("Prefabs/MiniGame/StarBullet") as GameObject;
    GameObject bulletObj = Instantiate(prefab, firePoint.transform);
}
```

然后修改 `ShootEvent`：

```
private void ShootEvent()
{
    if (Input.GetAxisRaw("Fire1") != 0)
    {
        CreateBullet();
    }
}
```

演示效果：

![攻击1](https://files.catbox.moe/ob6aph.gif)

可以看到子弹好像“粘在”角色身上，这是因为指定了 `FirePoint` 作为子弹的父节点，而 `FirePoint` 又挂在 `Player` 下面，子节点肯定是随着父节点改变位置了。

解决方法就是把子弹添加到背景：

```
// 游戏场景
protected GameObject bg;

private void Awake()
{
    // ... 找到背景对象
    bg = GameObject.Find("Background");
}

private void CreateBullet()
{
    GameObject prefab = Resources.Load("Prefabs/MiniGame/StarBullet") as GameObject;
    GameObject bulletObj = Instantiate(prefab, firePoint.transform);

    // 生成子弹后，修改子弹的父节点为游戏场景（背景）
    bulletObj.transform.SetParent(bg.transform);
}
```

修改完成后，再次测试效果：

![攻击2](https://files.catbox.moe/h2q9qk.gif)

这回子弹的位置正常了，但又发现新的问题——可以看到角色移动时没有播放奔跑动画！

### 动画丢失问题

这是因为 `MiniGame_Player` 脚本继承了 `MiniGame_Character`，而动画事件的监听是在父类，现在又在子类重写了 `Update` 方法，导致父类的动画监听事件没了。

解决方法就是把父类的 `Update` 声明为虚方法（virtual），访问修饰符为 `protected` 可以让子类调用：

```
protected virtual void Update()
{
    PlayAnimateListerner();
}
```

接着修改 Player 脚本的 Update 方法：

```
protected override void Update()
{
    // 调用父类的方法
    base.Update();

    // 射击事件
    ShootEvent();
}
```

然后重新测试一次：

![攻击4](https://files.catbox.moe/6gpze2.gif)

我们的女主角终于能快乐的奔跑同时“划出许多星星”了……？！

### 攻击频率
新的问题又出现了，按一下攻击键就生成那么多子弹，甚至造成了卡顿现象。

子弹的发射应该有限制，比如按住攻击键 0.25s 发射一颗，即存在「攻击间隔」，也可以理解为『攻击速度』。

原理与前一篇写的动画事件一样，就不再赘述了。

修改后的效果如下：

![攻击5](https://files.catbox.moe/5a7utg.gif)

看起来像有频率的进行“划水”了。

## 让子弹飞！
现在的攻击只是生出一颗子弹，但是这个子弹就像一张图片一样一动不动。

接下来要让子弹能够向前飞行。

### 子弹基类
除了小魔女发射的「星弹」之外，敌人也会发射各种子弹。

子弹都有共通之处，所以现在新建一个子弹的基类 `MiniGame_Bullet`：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public abstract class MiniGame_Bullet : MonoBehaviour
{
    public float damage;

    [HideInInspector]
    public MiniGame_Character attacker;

    protected abstract void MoveEvent();

    protected void Update()
    {
        MoveEvent();
    }
}
```

所有的子弹都有伤害值，以及攻击者，并且需要在子类实现子弹的移动逻辑。

将 attacker 字段设置为 public 方便赋值，同时再设置 `HideInInspector` 就可以让 attacker 字段不在属性面板显示了。

### 起飞吧，子弹！
新建一个星弹的脚本 `MiniGame_StarBullet`：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_StarBullet : MiniGame_Bullet
{
    public float speed;
    private float direct;

    private void Start()
    {
        direct = attacker.transform.localScale.x;
    }

    protected override void MoveEvent()
    {
        var pos = transform.position;
        pos.x += speed * Time.deltaTime * direct;

        transform.position = pos;
    }
}
```

星弹的逻辑比较简单，根据玩家的朝向向前直线移动。

然后把脚本挂在星弹的预制体上，并且将子弹的速度设置为 1500，伤害值设置为 1。

然后再修改 Player 发射子弹的方法：

```
private void CreateBullet()
{
    GameObject prefab = Resources.Load("Prefabs/MiniGame/StarBullet") as GameObject;
    GameObject bulletObj = Instantiate(prefab, firePoint.transform);
    bulletObj.transform.SetParent(bg.transform);

    MiniGame_Bullet bullet = bulletObj.GetComponent<MiniGame_Bullet>();
    bullet.attacker = this;
}
```

在发射子弹的时候，将 this 赋值给子弹的 attacker，这样这颗子弹的主人就是主角了。

然后打开调试场景测试效果：

![Kapture 2021-05-03 at 9.46.27.gif](https://i.loli.net/2021/05/03/3iXnzk9oZtYsIeG.gif)

看起来已经没问题了，但是……新的问题果然又来了。

打开调试窗口可以发现，发射出去的子弹并没有消失，而是继续一直往前飞行，仍然残留在游戏内：

![Kapture 2021-05-03 at 9.56.18.gif](https://i.loli.net/2021/05/03/4IegDpaqQmfJyEU.gif)

## 子弹消失术
不管玩家的电脑有多大的内存，也扛不住无限生成的子弹。

因此在达到某种条件的时候，应该让子弹“消失”。

### 方案选择

有几个方案：

- 子弹创建 3 秒后自动销毁
- 子弹飞行距离达到某个值后自动销毁
- 子弹离开屏幕后消失

第①个方法和第②个方法实际结果是差不多的。

因为子弹的飞行速度是恒定的，设定 3 秒后销毁子弹，那么子弹最终移动的距离就是固定的。

所以有两种方法让子弹消失：①根据飞行距离 ②判定子弹是否离开屏幕

如果子弹的飞行距离过短，玩家就必须贴脸输出，明明是远程攻击最后却还要拉近距离……

个人感觉体验不好，所以我决定用第二种方法，这也是大多数 FC 游戏的做法。

![场景示意图](https://files.catbox.moe/2odcjn.jpg)

上图是玩家在游戏场景按键攻击的示意图，当子弹飞出屏幕外的时候，即要进行销毁。

常规方法是像这样在游戏场景外生成一片区域，子弹撞到这个区域就会自动销毁。

![QQ20210503-102015.jpg](https://i.loli.net/2021/05/03/ROuA597oyzKlmwf.jpg)

但是还有一种反其道而行之的方法，即把碰撞检测区域放在游戏场景内，改成监听子弹离开这片区域。

![QQ20210503-102429.jpg](https://i.loli.net/2021/05/03/zhknjHylax47vU3.jpg)

第二种方法更好，因为第一种方法碰撞检测区域不规则，需要监听的区域比较多，而且子弹的大小形态各异，如下图：

![QQ20210503-103608.jpg](https://i.loli.net/2021/05/03/vsYMqT4eRylfr2w.jpg)

体积比较大的子弹比起体积小的子弹会提前碰到屏幕边缘而消失。

而且子弹还没离开场景就“不见”了看起来也会很奇怪。

所以这里采用第二种方法来实现。

### 区域检测器

unity 中要实现碰撞检测，其中一方必须是“刚体”，且双方都必须包含碰撞体组件。

这里又面临着一个选择：①刚体组件挂在子弹上面 ②刚体组件挂在区域检测器和敌人身上

常规做法是选择①，因为正常的思路子弹才是“实体”，刚体属于物理组件，理论上应该挂在子弹上面，而区域检测器更像是“触发器”一类的东西。

这样的想法虽然没错，但是这里还是选择反其道而行之，选择②才是最优解。

因为子弹的数量理论上是无限的，频繁创建组件需要耗费性能，而且刚体组件耗费的性能十分感人，如果满屏幕的子弹都是刚体，那很可能会有强烈的卡顿现象。

把刚体加在子弹上面明显是不理智的选择，子弹只需要添加碰撞体并且设置为触发器即可。

而且碰撞检测事件只需要监听「区域检测器」和「敌人」，而不是监听每一颗子弹，这样游戏监听的事件数量就大大减少了。

> 有时候这种反其道而行之的做法，在游戏开发中会有很大的帮助

首先在游戏场景创建一个空白物体 SceneArea，分辨率设置为屏幕大小：1280 * 720。

然后给这个物体添加 `Rigidbody 2D` 组件（2D刚体），然后再添加 `Box Collider 2D` 组件（2D 盒型碰撞体）。

如图所示：

![QQ20210503-155245.jpg](https://i.loli.net/2021/05/03/1rHFbAJKi9EvaTc.jpg)

这里的 `Body Type` 应该设置为 `Dynamic`（动态的）这样才能与触发器产生碰撞检测。

因此还需要修改刚体组件的 `Gravity Scale` 为 0，即让它不受重力影响。

不然开始测试的时候这个区域就会因为重力掉下去，导致测不出来。

`Simulated` 默认是勾选的，此处保持勾选状态，此选项是设置刚体是否模拟物理效果，如果取消勾选则检测不出碰撞。

场景区域这样就设置完毕了，现在虽然可以产生碰撞，但是还未对碰撞事件做出处理。

新建脚本 `MiniGame_SceneArea`：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_SceneArea : MonoBehaviour
{
    private void OnTriggerExit2D(Collider2D collision)
    {
        Debug.Log(collision.gameObject.name + "离开场景");

        string tag = collision.gameObject.tag;

        switch (tag)
        {
            case "Bullet":
                BulletEvent(collision);
                break;
            case "Enemy":
                break;
            case "Item":
                break;
        }
    }

    private void BulletEvent(Collider2D collision)
    {
        Destroy(collision.gameObject);
    }
}
```

这里监听 `OnTriggerExit2D` 事件，需要注意的是 unity 还有一个类似的碰撞检测事件 `OnCollisionExit`。

`Trigger`（触发器） 和 `Collision`（碰撞体）的碰撞回调方法是不一样的。

因为我将子弹视为触发器，所以要用 `Trigger` 作为碰撞的回调方法，当子弹离开这个区域的时候就会触发 `OnTriggerExit2D` 回调。

`OnTriggerExit2D` 回调方法会根据物体的 `tag`（标签）来判断物体属于哪种类型。

因为不仅仅是子弹会在离开屏幕时消失，道具、敌人之类的也会在离开视野范围内消除，所以这里设置通过标签来区分。

其实全部只要触发 `Destroy` 销毁事件即可，但是在这里区分物体的种类，还可以计算「物品搜集率」、「怪物击败率」等等，完美通关还能弄个成就什么的。

### 设置子弹

接下来设置子弹，同样是给子弹加上一个碰撞体，这里选择圆形碰撞体 `Circle Collider 2D`，并且设置好半径，然后把 `Is Trigger` 勾选。

![QQ20210503-153200.jpg](https://i.loli.net/2021/05/03/m75TWwFX9c3Ci2f.jpg)

然后点击属性面板 `Inspector` 顶部的 `Tag`，新增一个 `Bullet` 标签，然后将子弹设置为该标签。

![QQ20210503-155517.jpg](https://i.loli.net/2021/05/03/WjCBiDguzGR72My.jpg)

接下来修改调试场景的配置，场景像素配置修改为 `Free Aspect`，再把 `Maximize On Play`（最大化）点亮。

![QQ20210503-160354.jpg](https://i.loli.net/2021/05/03/xFUiO4JTQPWncEz.jpg)

这样就可以看到屏幕摄像机拍不到的地方了，然后进入调试场景测试：

![Kapture 2021-05-03 at 16.04.59.gif](https://i.loli.net/2021/05/03/1dzFQIGHKsuYgZT.gif)

可以看到子弹离开游戏区域（深色）时就会被自动销毁了。