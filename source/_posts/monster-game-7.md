---
title: 名为怪物的游戏——《星之魔女》FC小游戏移植（七）
date: 2021-05-07 10:10:25
tags:
 - unity
 - 开发技术

categories:
 - 名为怪物的游戏
 - 制作过程
---
## 前言
FC 小游戏的最终篇。

完成怪物逻辑设计、界面 UI、BOSS 战、场景设计等。

## 受伤判定
主角受到的伤害来自两种，第一是被敌人发射的子弹击中，第二是碰到敌人。

而敌人受伤只来源于玩家的攻击，主角碰到敌人只有主角会受到伤害，敌人不会受伤。

给子弹加上 Bullet 的 Tag（标签），给敌人加上 Enemy 标签。

之后就可以在碰撞回调中通过标签区分碰撞对象。

### 玩家受伤
玩家受到伤害事件的处理。

#### 对象类型
首先需要知道碰到玩家的物体是什么，可以用上面说的标签来区分。

如果是碰到道具则应该获得加分或加血，如果是敌人和子弹才会受到伤害。

编辑 `MiniGame_Player` 脚本，添加碰撞监听事件：

```
private void OnCollisionEnter2D(Collision2D collision)
{
    string tag = collision.gameObject.tag;

    switch (tag)
    {
        case "Enemy":
            TouchEnemy(collision);
            break;
        case "Item":
            TouchItem(collision);
            break;
        case "Bullet":
            TouchBullet(collision);
            break;
    }
}

private void TouchEnemy(Collision2D collision)
{
}

private void TouchItem(Collision2D collision)
{

}

private void TouchBullet(Collision2D collision)
{

}
```

现在的碰撞事件是两个刚体之间的碰撞，因此需要使用 `OnCollisionEnter2D` 监听。

这里设置角色与 3 种类型的物体发生碰撞的处理，即敌人、道具、子弹。

具体的方法留空，接下来逐一进行实现。

#### 受伤/死亡动画
增加受伤和死亡动画的状态机。

任意状态都可以直接进入受伤状态，而当受伤判定为死亡时，进入死亡动画。

![主角的受伤和死亡状态](https://files.catbox.moe/6xpk7k.jpg)

#### 抽离受伤事件
因为角色受到伤害会进入一个保护状态，要与敌人区分开来。

因此需要修改之前写的游戏角色基类，将 `TakeDamage` 方法改成抽象方法：

```
public abstract void TakeDamage(float damage);
```

接着在 `MiniGame_Player` 方法里实现受伤处理：

```
public override void TakeDamage(float damage)
{
    Debug.Log("受到伤害：" + damage);

    animator.SetBool("hurt", true);
    currentHP -= damage;

    Wait(delegate
    {
        animator.SetBool("hurt", false);

        if (currentHP <= 0)
        {
            DeadCallback();
        }

    }, 0.25f);
}
```

#### 碰到敌人
玩家碰到敌人时，显示受伤动画，减少血量，进入场景测试。

![撞到敌人测试](https://files.catbox.moe/3i5m3g.gif)

发现撞到敌人之后，虽然播放了受伤动画，但是却没有在 0.25s 后解除。

这是因为受伤动画没有转换成其他动画的设置。

#### 受伤动画不解除问题

修改动画状态机，当解除受伤动画时，让主角变成待机状态。

![受伤动画转换为待机动画](https://files.catbox.moe/0q2ogj.jpg)

然后再进入游戏测试：

![受伤修改后测试](https://files.catbox.moe/8856n6.gif)

受伤动画不解除的问题解决了，但是可以发现，如果继续停留在原地，角色与敌人依然保持接触状态，却不会再触发受伤事件了。这是因为两个刚体组件发生碰撞时，会出现弹开的情况。

#### 无敌时间
当主角受伤的时候会进入短暂的无敌，避免玩家连续碰到敌人还没反应过来就直接 gg 了。

增加一个变量用来设定角色无敌状态的持续时间，另一个变量保存当前无敌剩余时间：

```
public float pretectedTime = 1f;
private float currentPretectedTime;
```

当玩家受伤时，就赋予无敌时间，持续时间在 Update 方法里减少。

修改受伤事件：

```
private void TouchEnemy(Collision2D collision)
{
    if (currentPretectedTime > 0)
    {
        return;
    }

    currentPretectedTime = pretectedTime;

    TakeDamage(1f);
}
```

如果在无敌时间里，再次调用受伤方法就直接返回，否则计算伤害同时赋予玩家无敌时间。

接着消除无敌时间：

```
private void FixedUpdate()
{
    ProtectedTime();
    Shoot();
    Move();
    Jump();
    GroundCheck();
}

private void ProtectedTime()
{
    if (currentPretectedTime > 0)
    {
        currentPretectedTime -= Time.deltaTime;
    }
}
```

在 `FixedUpdate` 方法里计算无敌时间。

最后进入游戏场景测试：

![无敌时间测试](https://files.catbox.moe/oa3v53.gif)

主角跟敌人碰撞之后，还是“停在原地”，看起来依然与敌人保持着接触。

实际上，无敌时间确实生效了，但是玩家跟敌人保持接触却没有受到伤害不是因为无敌时间的关系。

而是玩家在撞到怪物身上的时候，发生“弹开”的情况。

只要稍微修改一下碰撞盒子就可以看出效果了：

![碰撞盒子修改测试](https://files.catbox.moe/q4qmhr.gif)

把敌人的碰撞盒子变大的时候，可以看到主角被“击退”了一步。

但是这个弹力实际上很小，所以肉眼看不出来。

#### 击退效果
修改弹力需要创建一个物理材质：

![弹性材质](https://files.catbox.moe/p9ud5b.jpg)

然后把材质拖到敌人的碰撞盒子上。

为了测试弹力效果，先将弹力设置成一个比较大的值：10.

观察效果：

![增强弹力效果](https://files.catbox.moe/roegwf.gif)

可以观察到角色撞到敌人之后被弹开了一段较大的距离。

如果仔细观察的话，还能发现角色又会向前挪动，继续与敌人发生碰撞，然后反复碰撞出现“抖动”的情况。

原因是角色受伤状态下仍然可以按方向键向前移动，所以又与前方的敌人发生了碰撞。除此之外，由于移动是用 `GetAxis` 来监听的，即使松开按键也存在一个缓冲的过程，速度并不会直接降低为 0，所以还会保持向前移动一小段距离，又与敌人产生碰撞。

只要修改移动方法，在无敌时间里禁止角色受到水平方向的力推动玩家就行了。

```
 private void Move()
{
    if (currentPretectedTime > 0)
    {
        return;
    }

    animator.SetFloat("horizontal", Mathf.Abs(horizontal));

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

测试效果：

![修改之后的弹力效果](https://files.catbox.moe/e1zyfr.gif)

因为弹力太大所以主角撞到敌人之后直接飞出屏幕外面了~

把弹力跳到 0.25 重新测试：

![降低弹力的效果](https://files.catbox.moe/n1x4sd.gif)

可以看到这样好多了，但是主角在受伤之后会进入“小跑动画”。

#### 小跑动画问题
这是因为玩家在受伤的时候仍然可以按住水平方向键，因此还会播放奔跑动画。

在控制角色移动的方法中，修改动画参数就可以了：

```
private void Move()
{
    if (currentPretectedTime > 0)
    {
        // 设置水平参数为0，即不会再播放奔跑动画了
        animator.SetFloat("horizontal", 0);
        return;
    }

    animator.SetFloat("horizontal", Mathf.Abs(horizontal));

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

还需要注意一个问题，这里因为用了 `currentPretectedTime` 来作为判定时间，就必须让受伤动画的“硬直”时间与无敌时间相同，否则无敌状态还没解除，受伤动画就先解除了，角色就会变成待机状态。

![受伤动画时间不一致问题](https://files.catbox.moe/kydjby.gif)

#### 动画不一致问题
修改受伤动画解除事件与无敌时间保持一致：

```
public override void TakeDamage(float damage)
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

    }, pretectedTime);
}
```

`Wait` 方法的第二个参数改成无敌时间即可，然后继续测试：

![受伤动画优化效果](https://files.catbox.moe/nogqt4.gif)

#### 受伤下的操作限制

但是又有新的问题，受伤的时候还可以跳跃和发射子弹。

![受伤跳跃和攻击](https://files.catbox.moe/cu01km.gif)

受伤跳跃还可以接受，但是受伤了还能发射子弹就有点离谱。

修改 `Shoot` 方法，当角色在无敌状态时，不能发射子弹攻击。

```
private void Shoot()
{
    // 加上无敌时间判断
    if (currentPretectedTime < 0 && isShoot)
    {
        isShoot = false;
        animator.SetBool("shoot", true);

        GameObject prefab = Resources.Load("Prefabs/MiniGame/StarBullet") as GameObject;
        GameObject bulletObj = Instantiate(prefab, firePoint);

        MiniGame_Bullet bullet = bulletObj.GetComponent<MiniGame_Bullet>();
        bullet.attacker = this;

        bulletObj.transform.SetParent(bg);

        Wait(delegate
        {
            animator.SetBool("shoot", false);
        }, 0.05f);
    }
}
```

演示效果：

![受伤时限制攻击](https://files.catbox.moe/bb6thw.gif)

### 敌人受伤
### 死亡处理

## 怪物设计
接上篇，未完成的敌人设计。

### 垃圾桶怪

### 球球怪

### 布偶怪（BOSS）

## 对象销毁

## 道具系统
比如加分道具、加血道具。

因为只有一关，所以不打算制作更多的道具。

### 动态效果
### 加分道具
### 加血道具

## BOSS 战

### 切换BGM

### 显示血条

### 逻辑处理

## 游戏场景

## GameOver
### 角色死亡判断
### BOSS击败判断

## 标题界面
### 动画效果
### 按键开始