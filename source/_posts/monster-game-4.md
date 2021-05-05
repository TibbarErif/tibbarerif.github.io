---
title: 名为怪物的游戏——《星之魔女》FC小游戏移植（四）
date: 2021-05-04 18:33:46
tags:
 - unity
 - 开发技术

categories:
 - 名为怪物的游戏
 - 制作过程
---
## 前言
由于自己写一个控制系统的工作量太大，因此决定改用 unity 的物理系统来完成这个平台跳跃 FC 小游戏。

虽然又得重头开始了，但这样也是为了让整个游戏的制作速度更快和更好。

## 角色控制系统

话不多说，直接开始。

### 重力系统

新建场景，然后创建一个角色图像，给角色加上 `Rigid Body 2D` 组件和 `Capsule Collider 2D` 组件。

![QQ20210504-184126.jpg](https://i.loli.net/2021/05/04/WXdDeG4w8UOoSEh.jpg)

`Rigid Body 2D` 是刚体组件，带有物理属性，`Capsule Collider 2D` 则是胶囊状碰撞体。

然后进入调试场景，发现角色已经会受到重力自由下落了。

![重力效果](https://files.catbox.moe/tgafa6.gif)

### 创建地板
由于没有地板支撑，角色会掉到屏幕外面。

接下来创建一个地板，加上 `Rigid Body 2D` 和 `Box Collider 2D` 组件。

![地板](https://files.catbox.moe/tnvyq1.jpg)

同时地板的刚体组件类型设置为 `Kinematic`：

![刚体类型](https://files.catbox.moe/lccafr.jpg)

然后打开调试场景测试：

![地板效果](https://files.catbox.moe/h2l724.gif)

两个刚体之间产生了碰撞，因此角色可以站在地板上面。

### 水平移动
使用物理系统控制角色移动十分简单，只要让角色受到水平方向的力就可以了。

新建 `MiniGame_Player` 脚本：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_Player : MonoBehaviour
{
    public float moveSpeed = 100f;

    private Rigidbody2D rb;
    private float horizontal;

    private void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    private void Update()
    {
        horizontal = Input.GetAxisRaw("Horizontal");
    }

    private void FixedUpdate()
    {
        rb.velocity = new Vector2(horizontal * moveSpeed, rb.velocity.y);
    }
}
```

在唤醒物体时，取得身上绑定的刚体组件，然后监听水平方向的按键，并且赋值给 `horizontal` 变量。

然后在 `FixedUpdate` 方法里对物体施加外力，测试结果：

![水平移动](https://files.catbox.moe/xd0xkl.gif)

发现角色受到外力直接倒下了……

这是因为角色的重心太高了，受到外力很容易倒下。（unity 复现了 100% 物理效果）

解决方法很简单，只要把 z 轴的旋转冻结就可以。

![冻结z轴旋转](https://files.catbox.moe/j3bkmm.jpg)

然后再进入游戏场景进行测试：

![重新测试](https://files.catbox.moe/chgu4h.gif)

### 角色朝向
虽然角色可以左右滑动了，但是角色的朝向并没有改变。

修改 `FixedUpdate` 代码：

```
private void FixedUpdate()
{
    rb.velocity = new Vector2(horizontal * moveSpeed, rb.velocity.y);

    if (horizontal != 0)
    {
        transform.localScale = new Vector3(horizontal, 1, 1);
    }
}
```

加上一个判断水平按键的条件，根据玩家控制的移动方向改变角色的翻转。

测试效果：

![角色朝向](https://files.catbox.moe/pv53o4.gif)

### 角色跳跃
同理，只需要让角色受到一个向上的力，角色就会“跳起来”了。

因为跳跃只能触发一次，而不像水平方向移动一样没有限制，所以要增加一个变量用来判断玩家是否按下跳跃键。

在跳跃状态下就不能再按跳跃键了，因为这个小游戏没有二段跳。

```
private bool jumpPressed;
```

接下来这里有个小技巧，可以解决之前提到过的按键监听 `GetButtonDown` 手感不好的问题。

即在 Update 方法里监听按键，在 FixedUpdate 方法里写处理逻辑。

```
private void Update()
{
    horizontal = Input.GetAxisRaw("Horizontal");

    if (Input.GetButtonDown("Jump"))
    {
        jumpPressed = true;
    }
}


private void FixedUpdate()
{
    rb.velocity = new Vector2(horizontal * moveSpeed, rb.velocity.y);

    if (horizontal != 0)
    {
        transform.localScale = new Vector3(horizontal, 1, 1);
    }

    if (jumpPressed && isJump == false)
    {
        jumpPressed = false;
        rb.velocity = new Vector2(rb.velocity.x, jumpSpeed);
    }
}
```

现在按下空格键角色已经可以跳跃了，但是可以在空中无限跳。

正确的逻辑应该是角色跳跃之后，就不能再按空格键进行二段跳或者三段跳。

而是应该落到地板上面才能重新按跳跃键。

### 落地问题
现在的跳跃机制没有判断落到地板的情况，要实现这个判断实际上很复杂。

如果直接用碰撞系统无法避免“陷入物体”的情况（详情见前文）。

完美的解决方法就是上一篇文章中提到过的“射线检测机制”，但是要自己手动写一个检测系统非常困难。

幸运的是 unity 已经实现了类似的方法。

`Physics2D.Overlap` 相关方法可以绘制某种图形，然后判断与某个物体是否相交。

这个跟我自己设想的「探知领域」差不多。

我今天折腾了一个下午，到底是为了什么……

在角色的脚底创建一个空的物体，当做角色的“脚”用来判断角色是否与地板接触。

![地板检测器](https://files.catbox.moe/qw5jf7.jpg)

然后选中游戏场景中的地板，添加一个新的 Layout（图层），命名为 Ground：

![Layout](https://files.catbox.moe/w0n4yr.jpg)

图层是用来控制游戏中场景的层级关系，在这里也可以作为检测用的“标签”。

图层 Layout 也可以作为参数传给脚本，类型是 LayerMask，在 Player 脚本添加：

```
public LayerMask layerMask;
private bool isGround;
```

`layerMask` 用来赋值地板的图层参数，`isGround` 用来判断是否站在地板上面。

在 FixedUpdate 方法添加一行代码;

```
isGround = Physics2D.OverlapCircle(groundCheck.position, 0.5f, layerMask);
```

`Physics2D.OverlapCircle` 方法会在脚底的位置画一个半径是 0.5 的圆圈，如果圆圈与 layerMask 的图层相交时就会返回 true。

其实就是下图这样：

![圆圈判定](https://files.catbox.moe/jeeovz.jpg)

这个圆的半径不能太大，不然角色还没碰到地板就会被判定成站在地板上面了（毕竟踩着那么大一颗球）。

进入游戏场景进行测试：

![跳跃测试](https://files.catbox.moe/x1dj9g.gif)

只有落地了才能继续跳跃，无法在空中进行二段跳。

跳跃功能也完成了！

## 整理代码

顺便优化一下代码，完整的 `MiniGame_Player` 脚本如下：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_Player : MonoBehaviour
{
    public float moveSpeed = 400f;
    public float jumpSpeed = 400f;

    public Transform groundCheck;

    private Rigidbody2D rb;
    private float horizontal;

    private bool jumpPressed;

    public LayerMask layerMask;
    private bool isGround;

    private void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    private void Update()
    {
        PressedKey();
    }

    private void FixedUpdate()
    {
        Move();
        Jump();
        GroundCheck();
    }

    private void PressedKey()
    {
        horizontal = Input.GetAxisRaw("Horizontal");

        if (Input.GetButtonDown("Jump") && isGround)
        {
            jumpPressed = true;
        }
    }

    private void Move()
    {
        rb.velocity = new Vector2(horizontal * moveSpeed, rb.velocity.y);

        if (horizontal != 0)
        {
            transform.localScale = new Vector3(horizontal, 1, 1);
        }
    }

    private void Jump()
    {
        if (jumpPressed)
        {
            jumpPressed = false;
            rb.velocity = new Vector2(rb.velocity.x, jumpSpeed);
        }
    }

    private void GroundCheck()
    {
        isGround = Physics2D.OverlapCircle(groundCheck.position, 0.5f, layerMask);
    }
}
```

这里有个小知识，可以发现上面我把方法都抽取出来，然后再在 `FixedUpdate` 里面进行调用。

是否会多此一举？

```
private void FixedUpdate()
{
    Move();
    Jump();
    GroundCheck();
}
```

看到网上有人在讨论这个问题，正好思考了一下。

程序是堆栈调用，为了代码的美观而封装成单独的方法调用，岂不是增加了入栈和出栈的成本？

确实可能会有一些影响，但这里并不只是为了代码美观整洁才封装的。

之所以封装成单独的方法调用是因为可以节省内存。

如果把全部的代码写在一坨，那定义的一些临时变量就会占着内存不放。

只有函数执行结束的时候回收机制才会销毁函数内部的临时变量。

代码的美观整洁也是十分重要的，抽取方法还可以实现代码的复用。

为了以后方便维护，在觉得一些地方写的不够好的时候，我会回头优化一下，如有必要也会像现在这样直接推翻整个系统重来。

## 手感调整
现在虽然实现了角色控制系统，但是操作手感却很不好。

接下来就开始优化。

### 重力系数
因为重力太小的原因，角色跳跃看起来很“假”。

打开顶部的菜单 `Edit` 然后选择 `Project Settings` 进入游戏参数配置。

选中左侧的 `Physics 2D` 把 `Gravity` 的 y 值改成 -45.5，如下：

![QQ20210505-094234.jpg](https://i.loli.net/2021/05/05/PWCBsafOUnMbg86.jpg)

然后再进入游戏场景测试：

![Kapture 2021-05-05 at 09.47.53.gif](https://i.loli.net/2021/05/05/vcIY8dFeonqx1Sf.gif)

掉落的速度看起来好多了。

### 不自然停止问题
角色跳跃的过程，如果立即放开水平移动键，就会像上图那样直接停止水平移动，看起来有些不自然。

如果要优化操作手感，应该给水平方向一些惯性，即使玩家松开按键，角色也会向前方保持减速运动直到停止，而不是立即停下来。

想要实现平滑过渡，将原来的 `GetAxisRaw` 改成 `GetAxis` 即可，前者返回 0、1、-1 三个数，而后者却返回 -1~1 的范围值。

```
// 原来的水平移动按键监听
horizontal = Input.GetAxisRaw("Horizontal");

// 修改之后
horizontal = Input.GetAxis("Horizontal");
```

然后再修改 Move 方法：

```
private void Move()
{
    rb.velocity = new Vector2(horizontal * moveSpeed, rb.velocity.y);

    if (horizontal > 0)
    {
        transform.localScale = new Vector3(1, 1, 1);
    }
    else if (horizontal < 0)
    {
        transform.localScale = new Vector3(-1, 1, 1);
    }
}
```

修改后的效果如下：

![Kapture 2021-05-05 at 09.58.03.gif](https://i.loli.net/2021/05/05/RmaBeMxqQ3plJKk.gif)

虽然只是很细微的差别，但实际操作会感觉“丝滑”一些。

水平方向即使松开按键也会保持一小段减速，而不是直接停下来。

### 弹起问题
角色落到地上，有一个软绵绵的弹起效果。

但这并不是我们需要的，在 unity 老版本中可以修改刚体的弹力，但是我用的是新版的 2021.1.5f1c1，在刚体上面已经找不到弹力设置项了。

这里需要修改碰撞检测类型，默认值是 `Discrete` （离散的），需要修改为 `Continuous`（连续的）：

![QQ20210505-101708.jpg](https://i.loli.net/2021/05/05/mCw1TkOal9UfhbI.jpg)

然后再进入游戏测试：

![Kapture 2021-05-05 at 10.21.56.gif](https://i.loli.net/2021/05/05/MOnekuH9oDQp2m6.gif)

现在角色已经站在“钢”做成的地板上了。

### 卡住问题
当角色与刚体的侧边接触时，会出现卡住的情况：

![Kapture 2021-05-05 at 10.24.03.gif](https://i.loli.net/2021/05/05/yTiMqgX5Y1VhzIc.gif)

这是因为 unity 的物理系统也模拟了摩擦力，所以角色与边缘接触时，会因为强大的摩擦力而被“吸住”。

只需要修改摩擦力就可以解决此问题。

在 `Assets` 新建一个 `Physic Material` 来保存物理材质。

然后右键打开菜单，选择 `2D`，然后再选择 `Physic Material 2D`，因为这个游戏是 2D 的，所以要选择 2D 的材质。

![QQ20210505-102756.jpg](https://i.loli.net/2021/05/05/oP59YgpvwZqCNB6.jpg)

将材质文件命名为 Player，然后在右侧打开的属性中，将 `Friction`（摩擦力）设置为 0。

![QQ20210505-102843.jpg](https://i.loli.net/2021/05/05/CJNH1xpS9juiVW5.jpg)

最后，点选场景中的角色，选中刚体组件，点击 `Material` 旁边的小圈，选择刚才创建好的材质：

![QQ20210505-103025.jpg](https://i.loli.net/2021/05/05/VLWBEymwSHpGeNl.jpg)

测试效果：

![Kapture 2021-05-05 at 10.33.05.gif](https://i.loli.net/2021/05/05/GI3S9fUlXMqApJg.gif)

加上材质的女主角已经变得十分“光滑”了！

## 总结
有时总想自己从零开始造轮子，但其实轮子别人已经造好了，直接用就可以了。

不应该执着于制作过程，而且学习 unity 提供的功能也是一种提升能力的办法。