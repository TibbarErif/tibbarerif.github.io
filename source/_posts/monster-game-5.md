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
unity 自带了一套动画系统，动画系统还内置了“状态机”。

状态机是一种可以实现不同状态之间互相转换的机制。

本篇开始制作角色的攻击动作以及其他一些动画。

unity 的动画系统比起 cocos 复杂很多，在网上没找到比较好的教程，因此决定自己看官方文档了。

官方文档：[https://docs.unity3d.com/cn/2020.3/Manual/AnimationSection.html](https://docs.unity3d.com/cn/2020.3/Manual/AnimationSection.html)

## 攻击系统
在制作动画之前先把攻击动作完成了。

### 攻击事件
前文写的攻击系统有个地方需要优化，就是角色发射子弹的按键控制。

这里取消按键连发的设定，而是根据玩家按一下攻击键就发射一颗子弹，这样操作手感比较丝滑。

监听按键时，增加监听攻击键：

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

### 刚体碰撞问题
因为现在使用了物理系统，所以之前使用的区域检测法让子弹消失会出现问题。

如下图：

![内部碰撞](https://files.catbox.moe/osp1cd.jpg)

因为碰撞区域设置为刚体，而现在角色也是刚体，刚体之间的碰撞会产生物理效果，导致碰撞区域发生了倾斜。

所以现在不能让碰撞区域作为刚体了，而是要让子弹、敌人和玩家作为刚体，区域检测器设置为触发器。

尽管让子弹加上刚体会影响性能，但至少要先把功能实现了再说，如果出现卡顿现象，到时候再想办法优化。

### 子弹实例
将碰撞区域的刚体移除，将碰撞盒子设置为 Trigger。

再给子弹加上 2D 刚体组件。

修改原来子弹代码移动逻辑，改成用外力进行推动而不是直接修改坐标，原理与控制角色一样。

首先需要在父类中获得刚体组件：

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

子弹离开边界的时候已经可以销毁了。