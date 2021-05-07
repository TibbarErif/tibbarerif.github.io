---
title: 名为怪物的游戏——《星之魔女》FC小游戏移植（五）
date: 2021-05-05 10:37:17
tags:
 - unity
 - 开发技术

categories:
 - 名为怪物的游戏
 - 制作过程
---
## 前言
unity 自带了一套动画系统，还内置了“状态机”。

状态机是一种可以实现不同状态之间互相转换的机制。

本篇开始制作角色的攻击动作以及其他一些动画。

unity 的动画系统比起 cocos 复杂很多，在网上没找到比较好的教程，因此决定自己看官方文档。

官方文档：[https://docs.unity3d.com/cn/2020.3/Manual/AnimationSection.html](https://docs.unity3d.com/cn/2020.3/Manual/AnimationSection.html)

## 攻击系统
制作动画之前，要先把角色的攻击功能做出来。

### 攻击事件
前文写的方法是通过按键实现连发攻击。

现在为了让手感更加丝滑，改成按一下攻击键就发射一颗子弹。

使用 `GetButtonDown` 来监听玩家按下攻击键：

```
private void PressedKey()
{
    horizontal = Input.GetAxis("Horizontal");

    if (Input.GetButtonDown("Jump") && isGround)
    {
        jumpPressed = true;
    }

    // 新增
    if (Input.GetButtonDown("Fire1"))
    {
        Shoot();
    }
}
```

然后编写 `Shoot` 方法，在这里创建子弹：

```
private void Shoot()
{
    GameObject prefab = Resources.Load("Prefabs/MiniGame/StarBullet") as GameObject;
    GameObject bulletObj = Instantiate(prefab, firePoint);

    MiniGame_Bullet bullet = bulletObj.GetComponent<MiniGame_Bullet>();
    bullet.attacker = this;

    bulletObj.transform.SetParent(bg);
}
```

这样按键攻击的功能就完成了。

演示效果：

![按键射击](https://files.catbox.moe/tszeso.gif)

这里的子弹以及如何让子弹飞离屏幕就消失，前面发的博文已经有介绍了，故不再重复说明。

### 刚体碰撞问题
因为现在使用了物理系统，所以之前使用的区域检测法让子弹消失会出现问题。

如下图：

![内部碰撞](https://files.catbox.moe/osp1cd.jpg)

这是刚体和刚体之间会发生碰撞，产生物理效果。

虽然碰撞区域移除了重力影响，但现在这个场景里面，角色身上有刚体组件，地板也有刚体组件，这样必然会触发物理系统，结果就是碰撞区域发生了偏移。

因为我已经把角色的操控系统改成用物理效果来实现了，原理已经不同了。

现在不能让碰撞区域作为刚体，而是要让子弹、敌人和玩家作为刚体，区域检测器设置为触发器。

尽管让子弹加上刚体会影响性能，但至少要先把功能实现了再说，如果出现卡顿现象，到时候再想办法优化。

（这个小游戏不是弹幕游戏，应该不至于会出现性能问题）

### 子弹实例
移除碰撞区域的刚体，并设置为 Trigger，这样碰撞区域就是一个触发器了。

再给子弹加上 2D 刚体组件。

修改原来子弹代码移动逻辑，改成用外力进行推动而不是直接修改坐标，与控制角色的代码一样。

在父类中获得刚体组件：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public abstract class MiniGame_Bullet : MonoBehaviour
{
    public float damage;

    [HideInInspector]
    public MiniGame_Character attacker;

    protected Rigidbody2D rb;

    private void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    protected abstract void MoveEvent();

    protected void Update()
    {
        MoveEvent();
    }
}
```

然后修改子类的移动逻辑：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_StarBullet : MiniGame_Bullet
{
    public float speed = 1000f;
    private float direct;

    private void Start()
    {
        direct = attacker.transform.localScale.x;
    }

    protected override void MoveEvent()
    {
        rb.velocity = new Vector2(speed * direct, rb.velocity.y);
    }
}
```

测试效果：

![子弹消失](https://files.catbox.moe/0l2djk.gif)

可以看到，子弹离开边界的时候已经被销毁了。

## 角色动画
为了方便观察动画效果，先调整一下 unity 引擎的界面布局。

![unity 布局调整](https://files.catbox.moe/wpdw8j.jpg)

接下来就可以开始制作动画了。

在 Assets 目录下创建一个 Animations 文件夹用来存放动画，依次创建 MiniGame、Player 子文件夹。

这里有个比较坑爹的地方，我用的 2021 版 unity 默认会把动画的播放间隔参数隐藏起来。

如下图：

![动画界面](https://files.catbox.moe/t2bkg3.jpg)

其实只要点开界面的右上方，选择更多，在弹出的菜单中再选择 `ShowSampleRate`：

![showSampleRate](https://files.catbox.moe/vgl65s.jpg)

勾选以后，就可以在动画界面看到设定帧数的输入框了，默认值是 60，也就是说 1 秒钟有 60 帧：

![帧数设置](https://files.catbox.moe/g48voy.jpg)

实际上我们没有那么多的素材能用，一般也就几帧而已，每帧对应一张图片素材。

### 待机动画
待机动画是角色静止不动时的动作。

动画是女主角眨眼的动作，这个动画只播放一次，不循环。

完整动作只有 4 张素材，也就是把眼睛闭上的动作。

睁眼的动作其实就反过来倒序播放而已。

顺序播放：1234321，一共有 7 帧。

60 / 7 = 8.5，因此这里可以四舍五入取 9 作为帧数，即将 1 秒钟划分成 9 份。

演示效果：

![待机动画](https://files.catbox.moe/6c0sgn.gif)

动画默认是循环播放的，我们这个待机效果只要播放一次就够。

找到刚才存放动画的文件：

![待机动画文件](https://files.catbox.moe/4sewgq.jpg)

双击选中，然后在右侧的属性面板中把 `Loop Time` 的勾去掉即可：

![动画属性](https://files.catbox.moe/c9b7b4.jpg)

### 奔跑动画

继续制作奔跑动画，创建新的动画然后把素材拖进去：

![奔跑动画](https://files.catbox.moe/xlkzmh.gif)

奔跑的动画是循环的，所以不需要调整。

### 奔跑攻击动画
角色一边奔跑一边按攻击键，也有独立的动画。

直接将素材拖进去即可：

![奔跑攻击](https://files.catbox.moe/zqdfm6.gif)

### 攻击动画
攻击动画只有一张素材，也拖进去即可。

![攻击动画](https://files.catbox.moe/5qsyf4.jpg)

### 受伤动画
受伤也只有一张。

![受伤动画](https://files.catbox.moe/4bwpzy.jpg)

### 倒下动画
角色 gg 的时候倒地动画。

![倒地动画](https://files.catbox.moe/2frez6.jpg)

### 未完成动画
其实还有跳跃、跳跃攻击的动画，但是没有做出来。

角色的动画这样就算弄好了。

## 状态转换
事物具有“状态”属性，遇到不同条件时，状态就会发生变化。

比如天冷的时候，水结冰，此时水是固态，而当天气变热，冰化了，变成液态的水，然后天气继续升温，水被蒸发了，变成气态的水蒸气，水蒸气遇到冷空气又会变成雨。

这个过程就是水的状态转化机制，动画系统也是同理。

当玩家操控角色行走时，由待机动作转化为奔跑动作，而玩家又按下了攻击键，则角色的动画就会从奔跑转变为奔跑攻击，攻击动作完成后又会变回奔跑动作，然后玩家松开方向键停止奔跑，角色的动画就会从奔跑转为待机。

一般来说，除了自发的转换之外，动画转换基本是根据玩家的操作来决定的。

来看看令人头大的动画状态控制器：

![动画状态控制器](https://files.catbox.moe/82a86r.jpg)

### 初始状态
物体总是有一个初始状态，其中 `Entry` 箭头指向的就是默认进入的动画效果。

![初始状态](https://files.catbox.moe/4g2m3y.jpg)

这里的意思是，游戏开始时，角色就会进入 `Idle`（待机动画）。

打开调试场景测试：

![初始动画](https://files.catbox.moe/2lsdgf.gif)

可以发现角色播放了眨眼动画，说明设置成功了。

而且这里只眨眼了一次，说明上面设置的取消循环也成功了。

### 待机-奔跑
角色只有在玩家按键操作的时候，才会从静止状态变为奔跑状态。

因此可以用两种方法实现状态的转化，第一种是根据玩家的按键，第二种是根据角色当前的移动速度。

第一种方法是主动变化，第二种方法是被动触发。

这里选择第一种。

右键 `Player_Idle` 动画，在弹出的菜单中选择 `Make Transtition` 创建一个新的转换关系。

![新建关系](https://files.catbox.moe/hh7o6d.jpg)

待机动画和奔跑动画是可以相互转换的，所以要建立双向关系。

动画转换条件只要监听玩家按键就可以了，控制玩家移动的 `GetAxis` 方法会返回 -1~1 的值。

当玩家没有按键的时候，返回的是 0，向左移动返回负数，向右返回整数。

因此这个值就可以当做动画转换的条件。

```
// 保存动画组件
private Animator animator;

// 在唤醒的时候获取动画组件
animator = GetComponent<Animator>();

// 在移动方法里获得水平按键参数
animator.SetFloat("horizontal", Mathf.Abs(horizontal));
```

这里将水平参数转换成绝对值，因为只要正数就够了，方向并不影响动画的播放。

好了，现在这样就可以进入游戏场景调试了：

![待机和奔跑之间的转换](https://files.catbox.moe/d9qysu.gif)

简直不要太神奇……如果是自己来写状态机，指不定要花多长的时间呢！

### 待机-攻击
从待机到攻击比较简单，只要在待机状态下玩家按下攻击键就认为是攻击状态。

这里我对代码进行了一些修改，把按键判定放在 Update，然后在 FixedUpdate 实际发射出子弹。

因为攻击动作太快（只有 1 帧）会导致以肉眼看不见的速度结束，所以要用到延迟函数，这里我通过协程来实现。

```
private void FixedUpdate()
{
    StartCoroutine(Shoot());
    Move();
    Jump();
    GroundCheck();
}

private void PressedKey()
{
    horizontal = Input.GetAxis("Horizontal");

    if (Input.GetButtonDown("Jump") && isGround)
    {
        jumpPressed = true;
    }

    if (Input.GetButtonDown("Fire1"))
    {
        isShoot = true;
    }
}

private IEnumerator Shoot()
{
    if (isShoot)
    {
        isShoot = false;
        animator.SetBool("shoot", true);

        GameObject prefab = Resources.Load("Prefabs/MiniGame/StarBullet") as GameObject;
        GameObject bulletObj = Instantiate(prefab, firePoint);

        MiniGame_Bullet bullet = bulletObj.GetComponent<MiniGame_Bullet>();
        bullet.attacker = this;

        bulletObj.transform.SetParent(bg);

        yield return new WaitForSeconds(0.05f);

        animator.SetBool("shoot", false);
    }
}
```

在生成子弹后 0.05s 将攻击状态变为 false，即切换动画状态。

演示效果：

![待机-攻击](https://files.catbox.moe/ps485v.gif)

奔跑状态下按攻击键没有触发攻击动画，是因为还没设置。

### 奔跑-奔跑攻击
普通攻击是停下来的，而奔跑攻击有单独的动画。

演示效果如下：

![奔跑-奔跑攻击](https://files.catbox.moe/8odefg.gif)

这里出现了一个比较“呆萌”的效果，主角奔跑攻击然后松开方向键，会出现“原地小跑”的情况，然后才停下来。

![原地小跑](https://files.catbox.moe/ojwxmg.gif)

这是因为动画先从 Player_RunAttack 转换到 Player_Run，本来应该立即从 Player_Run 转化为 Player_Idle 的，但是因为我用的是 `GetAxis` 会有一段缓冲减速效果，因此这段极短的缓冲时间就是播放奔跑动画的时间，所以就会出现原地小跑的情况。

还有这段伸手动作：

![伸手](https://files.catbox.moe/8qzngf.gif)

这是因为奔跑攻击动画用了循环，而且帧数比较多，所以才会有点“迟钝”的样子。

只要改成不循环然后减少帧数解决可以解决“伸手”的问题，但还是因为看起来很呆萌，所以就保留下来了。

> 火兔语录：所有的 bug 都是游戏彩蛋！

### 未完成动画
还剩下两个动画没完成，一个是受伤另一个是倒地。

这两个动画要等制作出敌人才能实现，所以就留到下一篇了。

## 后言
参考资料：[https://www.bilibili.com/video/BV1sE411g7jK?t=1301](https://www.bilibili.com/video/BV1sE411g7jK?t=1301)

最终还是找到了一个视频资料。

第一次看到 unity 的动画状态机还以为会是很难理解的东西，但实际体验了一下，感觉并不是想象中那么困难。

有时候对于未知的事物，还是要勇敢的尝试一下才知道是不是真的很难。

最开始学 unity 是 4 年前，跟着教学制作了一个 3D 小球的 Demo。

虽然做出来了，但是全英文的界面让我感觉到学习很困难。

而且 3D 的摄像机弄了半天也没搞清楚原理，最后就不了了之了。

也许是因为那时刚毕业，对自己的技术不那么自信，所以才会打退堂鼓。

但是经过了这么多年以后，技术提高了，自信心也增强了。

所谓“功夫不负有心人”，我开始相信只要是想学的技术，肯定能学会。

关键在于决心强不强烈。

成就感可以驱动行动，如果一个人擅长做一件事，而且这件事具有一定的挑战性，完成这件事就能得到成就感。

成就感是一种正反馈，得到的正反馈越多，学习的动力越强。

但如果一个人不擅长做某事，而且尝试过一次之后就失败了，这样就会得到与成就感相反的——『挫败感』。

如果这个人的内心又比较脆弱，意志也不够坚定，积累了一定的挫败感超过承受能力最终就会打退堂鼓。

像现在这样记录博客实际上也是在积累「成就感」，边做边写也能够督促自己每天坚持更新。

如果哪一天没更新了，说明自己偷懒了。