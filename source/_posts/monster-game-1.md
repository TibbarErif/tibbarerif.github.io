---
title: 名为怪物的游戏——《星之魔女》FC小游戏移植（一）
date: 2021-05-02 20:43:32
tags:
 - unity
 - 开发技术

categories:
 - 名为怪物的游戏
 - 制作过程
---

## 前言
短篇 AVG 游戏的流程已经差不多了，现在就差 cee 把游戏要用的素材提供给我，最后再导入测试就基本完成了。

所以趁这个时间，我打算把原来在 cocos creator 引擎上《名为怪物的游戏》制作进度移植到 unity 引擎。

同时花了一晚上的时间把火兔游戏的官网重建成这样一个博客，后续将会以博文的方式直播制作过程或者发布游戏预告。

（直播制作过程主要是为了防鸽……）

## 移植原因
首先我很喜欢 cocos creator 引擎，不仅是因为国人制作的，而且上手简单。

在制作了游戏的序章之后，发现 cocos creator 不能满足我们的要求，因为我们打算发布的是 PC 端，
而 cocos creator 主打移动游戏，比方说在游戏内调节分辨率 cocos 就不支持，还有因为 JavaScript
对文件读写什么的也有限制，要解决这些问题估计得花很多时间，但对于我们来说可以游刃有余的时间并不多了，
所以选择对单机游戏支持比较友好的 unity。

## 星之魔女
星之魔女是《名为怪物的游戏》中的一个怀旧向像素风 FC 游戏。

使用 cocos creator 引擎开发的画面：[https://www.bilibili.com/video/BV167411L7vJ/](https://www.bilibili.com/video/BV167411L7vJ/)

<iframe src="//player.bilibili.com/player.html?aid=89983856&bvid=BV167411L7vJ&cid=153683294&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

## 移植过程

### 素材导入
由于之前已经做完了，所以素材可以直接导入。

像素风的序列帧：

![QQ20210502-214701.jpg](https://i.loli.net/2021/05/02/ZjeiXxRbSLBlHG7.jpg)

### 场景配置
新建一个 1280 * 720 的场景。

![QQ20210502-214837.jpg](https://i.loli.net/2021/05/02/94OvjVgNtaY1HFG.jpg)

### 让角色动起来
现在场景有了，但角色只是一张静态图片，要让角色可以通过按键移动，就要开始编写角色控制脚本了。

新建名为 `MiniGame_MoveEvent` 的脚本，用来控制角色移动：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_MoveEvent : MonoBehaviour
{
    public float speed = 500f;

    private Vector3 left = new Vector3(-1, 1, 1);
    private Vector3 right = new Vector3(1, 1, 1);

    void FixedUpdate()
    {
        if (Input.GetKey(KeyCode.A) || Input.GetKey(KeyCode.LeftArrow))
        {
            gameObject.transform.localScale = left;
            transform.Translate(Vector3.left * Time.deltaTime * speed);
        }

        if (Input.GetKey(KeyCode.D) || Input.GetKey(KeyCode.RightArrow))
        {
            gameObject.transform.localScale = right;
            transform.Translate(Vector3.right * Time.deltaTime * speed);
        }
    }
}
```

在 unity 中加载游戏会加载所有脚本，所以脚本的名字加上 `MiniGame_` 来区分。

为了让角色的朝向也能改变，在按下移动键的时候，顺便改变图片的翻转。

这里可以使用 WASD 来控制移动，也可以用方向键控制移动，目前只有左右移动，把这个脚本挂在 Player 对象上即可用键盘控制角色了：

![Kapture 2021-05-02 at 21.54.23.gif](https://i.loli.net/2021/05/02/ZC73l8bPK6TiDho.gif)

实现角色移动还可以直接用 unity 自带的物理引擎，碰撞检测阻止移动就比较方便，但是我这里选择自己写脚本逻辑，因为 FC 游戏里的一些操作是不符合物理规律的。

### 角色小动作
现在角色能动起来了，但是看起来就是在移动一张图片，没有游戏的感觉。

为了让角色变得“生动”，就要给角色加上动画演出效果。

unity 内置了动画系统，但是我这里也选择自己写脚本来控制。

新建一个名为 `MiniGame_Character` 的抽象类，因为不仅主角可以播放动画，敌人也有动画效果。

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public abstract class MiniGame_Character : MonoBehaviour
{
    public Sprite[] idle;
    public Sprite[] hurt;
    public Sprite[] attack;
    public Sprite[] run;

    // 是否循环播放动画
    private bool animateLoop;
    // 当前播放的动画
    private string currentAnimate;
    // 当前动画index
    private int animateIndex;
    // 当前动画精灵
    public Sprite[] currentAnimateSprites;
    // 人物行走图
    private Image character;
    // 动画播放间隔
    private float animateInterval;
    // 当前时间
    private float time;

    private void Awake()
    {
        character = GetComponent<Image>();
    }

    // 死亡回调
    protected abstract void DeadCallback();

    public void SetAnimate(string animate)
    {
        // 不播放重复动画
        if (currentAnimate == animate)
        {
            return;
        }

        Debug.Log("切换动画：" + animate);

        currentAnimate = animate;
        animateIndex = 0;

        switch (animate)
        {
            case "idle":
                animateLoop = false;
                animateInterval = 0.1f;
                currentAnimateSprites = idle;
                break;
            case "run":
                animateLoop = true;
                animateInterval = 0.1f;
                currentAnimateSprites = run;
                break;
            case "attack":
                animateLoop = false;
                currentAnimateSprites = attack;
                break;
            case "hurt":
                animateLoop = false;
                currentAnimateSprites = hurt;
                break;
        }
    }

    private void PlayAnimateListerner()
    {
        if (animateLoop == false && animateIndex > currentAnimateSprites.Length - 1)
        {
            return;
        }

        if (Time.time < time)
        {
            return;
        }

        // 时间自增
        time = Time.time + animateInterval;

        if (animateLoop == true)
        {
            if (animateIndex > currentAnimate.Length - 1)
            {
                animateIndex = 0;
            }
        }

        character.sprite = currentAnimateSprites[animateIndex];
        animateIndex++;
    }

    private void Update()
    {
        PlayAnimateListerner();
    }
}

```

再新建一个用于玩家控制角色的脚本：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_Player : MiniGame_Character
{
    protected override void DeadCallback()
    {
        // TODO 进入GameOver场景
        throw new System.NotImplementedException();
    }
}
```

动画的播放逻辑很简单，就是循环播放图片，以下是主角的序列帧行走图：

![QQ20210502-220534.jpg](https://i.loli.net/2021/05/02/23e9XzWpJdjHBIw.jpg)

跟制作动画的原理一样，就是以肉眼难以辨别的速度播放细微不同的图片，所以看起来像“动起来”一样。

为了间隔一定时间播放一张图片，这里用了一个计时器，定义下一个切换图片的时间点，比如 0.1 秒以后，如果当前时间等于 0.1 秒后，就播放下一张图片，然后切换图片的时间点等于当前时间加上 0.1s。

如果是循环播放类的图片，在图片全部播完之后，就会从第一张开始继续播放，如果是不循环的动画，就停止继续播放。

声明一个公开的方法 `SetAnimate`，只要传入要播放的动画，就会自动配置对应的参数，比如当前播放的图片数组和是否循环播放以及播放间隔。

这里要加一个判断，防止重复播放相同的动画：

```
if (currentAnimate == animate)
{
    return;
}
```

如果当前已经在播这个动画了，再调用这个方法就直接返回。

抽象父类还定义了一个 `DeadCallback` 死亡回调方法，即当目标死亡时会做什么事情。

玩家死亡了就是 Gamover，敌人死亡了就爆金币。

现在先来做基本的动画：待机小动作和跑步动作。

要播放动画就调用 `SetAnimate` 方法，在 `MiniGame_Character` 加入代码：

```
private void Start()
{
    SetAnimate("idle");
}
```

这样角色在进入场景的时候，就会自动播放待机动画。

把 `MiniGame_Character` 脚本挂在场景的 Player 节点，并且在脚本组件上把待机动画图拖进去：

![QQ20210502-221735.jpg](https://i.loli.net/2021/05/02/IrwmhDUNsGRy8vl.jpg)

进入场景就可以看到效果了：

![Kapture 2021-05-02 at 22.19.09.gif](https://i.loli.net/2021/05/02/ytqZiRHB5nG6k98.gif)

一个简单的眨眼小动作！

同理要让角色有奔跑动画只需要在 `MiniGame_MoveEvent` 控制角色移动的时候播放动画即可：

```
void FixedUpdate()
{
    if (Input.GetKey(KeyCode.A) || Input.GetKey(KeyCode.LeftArrow))
    {
        player.SetAnimate("run");
        gameObject.transform.localScale = left;
        transform.Translate(Vector3.left * Time.deltaTime * speed);
    }

    if (Input.GetKey(KeyCode.D) || Input.GetKey(KeyCode.RightArrow))
    {
        player.SetAnimate("run");
        gameObject.transform.localScale = right;
        transform.Translate(Vector3.right * Time.deltaTime * speed);
    }

    // 判断松开按键
    if (Input.GetKeyUp(KeyCode.A) || Input.GetKeyUp(KeyCode.LeftArrow) ||
        Input.GetKeyUp(KeyCode.D) || Input.GetKeyUp(KeyCode.RightArrow)
        )
    {
        player.SetAnimate("idle");
    }
}
```

这里还要判断当键盘松开的时候，要改成待机动画。

然后把奔跑的动画素材也拖到脚本组件上，演示效果：

![Kapture 2021-05-02 at 22.26.16.gif](https://i.loli.net/2021/05/02/hvgxd8Y3pkworau.gif)

### 小问题修正
这里其实还存在一个小问题，就是当按键和弹起的一瞬间，有几率出现判定失误，`GetKey` 方法是监听按键，不管是弹起还是按下都会触发，而 `GetKeyUp` 则是监听弹起事件，因此会存在监听到弹起事件的一瞬间同时也被判定为按下的情况，导致人物虽然停止移动了，但是奔跑动画却还在播放的情况。

除此之外，还有一个比较特殊的情况，就是如果玩家同时按下左方向和右方向，这样也会变得很奇怪。

为了修正上述两个问题，重新修改 `MiniGame_MoveEvent` 脚本：

```
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MiniGame_MoveEvent : MonoBehaviour
{
    public float speed = 400f;

    private MiniGame_Player player;

    private Vector3 left = new Vector3(-1, 1, 1);
    private Vector3 right = new Vector3(1, 1, 1);

    private void Start()
    {
        player = GetComponent<MiniGame_Player>();
    }

    void FixedUpdate()
    {
        float raw = Input.GetAxis("Horizontal");
        float moveSpeed = raw * Time.deltaTime * speed;

        var pos = transform.position;
        pos.x += moveSpeed;
        transform.position = pos;

        if (raw < 0)
        {
            player.SetAnimate("run");
            gameObject.transform.localScale = left;
        }
        else if (raw > 0)
        {
            player.SetAnimate("run");
            gameObject.transform.localScale = right;
        } else
        {
            player.SetAnimate("idle");
        }
    }
}
```

监听按键的方法改成 `GetAxis`，这个方法会返回 -1~1的浮点数，

这样还可以让角色有一个起跑短暂加速的感觉，而且在松开按键的时候，也会有缓冲效果。

修改后的演示效果：

![Kapture 2021-05-02 at 23.17.15.gif](https://i.loli.net/2021/05/02/jUuVS9l6sPQzTO2.gif)

另外，unity 还有一个 `GetAxisRaw` 方法，类似 `GetAxis`，但是它只会返回三个值：-1、0、1。

如果使用 `GetAxisRaw` 方法，就没有平滑起跑的效果了，而是直接以最大的速度奔跑，改成 `GetAxisRaw` 后的演示效果如下：

![Kapture 2021-05-02 at 23.20.22.gif](https://i.loli.net/2021/05/02/agpb3rl1ihWxLo7.gif)

两种效果都各有好坏，有缓冲效果感觉更加笨重，但是比较真实，以最大速度起跑操作体验更好。

这里我就采用直接最大速度开始奔跑的方案，另外，如果同时按下左右方向键，则行动会立即停止，不会因为同时按而产生奇奇怪怪的结果了。

而动画是根据按压的 raw 返回值来判断的，也就不会出现行动停止奔跑动画还在继续的情况。