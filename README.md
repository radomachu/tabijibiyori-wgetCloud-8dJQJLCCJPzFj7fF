---

> **引擎版本**：Cocos Creator 3.x
> **阅读时间**：5分钟（能帮你省8小时调试）

---

## 🚑 翻车现场

那天是周五下午 4 点半，我正打算提交代码下班。
测试小姐姐突然跑过来一句：
“你这个商城列表，在我手机上滑动的时候卡得我想摔手机！”

我一脸懵逼：
“啊？我在 iPhone 14 Pro 上测试很丝滑啊！”

拿过她的红米 Note 9 一滑——

**卧槽，真的卡。**

再试试快速滑动——

**卧槽，更卡了。**

我打开 Chrome DevTools 看性能面板，帧率从 60 狂掉到 25，一大片红色警告条刺眼地躺在那里：

```
OnScrolling() - 35ms ⚠️
UpdateVisibleItems() - 28ms ⚠️
CalculateVisibleRange() - 15ms ⚠️
```

心里默默地叹了口气：
**“完蛋鸟，又是一个不眠的周末。”**

---

## 🤔 问题的根源

我翻开了以前写的虚拟列表代码：

```
// 我之前的写法（估计很多人也这么写）
private onScrolling(): void {
    this.updateVisibleItems(); // 每次滚动都刷新
}
```

看起来挺合理的对吧？
滚动更新可见项，天经地义。

但我忽略了一个残酷事实：

Cocos 的 `scrolling` 事件，在快速滑动时，**1 秒能触发 60 次！**

也就是说，我的 `updateVisibleItems()` 每秒执行 60 次，干的事还挺重：

* 计算可见区间（遍历位置数组）
* 回收旧节点（遍历 Map）
* 创建新节点（对象池、设置位置、更新数据）

**60 次 × 每次 30ms = 1800ms 的计算量。**

一秒钟才 1000ms，非要干 1800ms 的活儿——
不卡才怪。

---

## 💡 灵魂拷问：真的需要刷新那么快吗？

我冷静想了想。
当用户快速甩动列表时，他真的能看清里面的内容吗？

答案是：**根本看不清。**

你可以自己试试，打开淘宝、微信朋友圈，快速滑动时会发现：

* 内容略模糊（其实没完整渲染）
* 停下来才变清晰
* 但完全不会觉得“卡”

**原来大厂早就在玩这个骚操作：快速滚动时，故意降低刷新率。**

就像拍电影——打斗戏 24 帧就够，定格镜头才用 60 帧。

---

## 🎯 我的方案：让列表自己决定要不要刷新

核心思路特别简单，就四句话：

1️⃣ 慢速滑动 → 用户能看清 → 必须刷新 → 60fps
2️⃣ 快速滑动 → 用户看不清 → 少刷几次 → 20fps
3️⃣ 卡顿明显 → 设备吃不消 → 降级模式
4️⃣ 滑动停止 → 强制刷新 → 确保显示完整

听起来挺复杂，其实不到 100 行代码就能搞定。

---

## 🧭 第一步：给列表装个“速度计”

```
export class PerformanceManager {
    private lastPos = 0;
    private lastTime = 0;
    private speed = 0;

    shouldUpdate(currentPos: number): boolean {
        const now = Date.now();
        const timePassed = now - this.lastTime;
        if (timePassed < 16) return false; // 每帧间隔16ms

        const moved = Math.abs(currentPos - this.lastPos);
        this.speed = moved / (timePassed / 1000);

        if (this.speed > 2000) {
            if (timePassed < 50) return false; // 飞速滑动，20fps
        } else if (this.speed > 1000) {
            if (timePassed < 33) return false; // 快速滑动，30fps
        }

        this.lastPos = currentPos;
        this.lastTime = now;
        return true;
    }
}
```

简单说：
速度越快 → 刷新越少 → 性能越稳。

---

## 🧩 第二步：改造滚动逻辑

```
private onScrolling(): void {
    const offset = this.scrollView.getScrollOffset();
    const pos = this.scrollView.vertical ? offset.y : offset.x;

    if (this.perfManager.shouldUpdate(pos)) {
        this.updateVisibleItems(); // 该刷才刷
    }

    this.onScrollProgress();
}

private onScrollEnded(): void {
    this.updateVisibleItems(); // 停下来强制刷新
}
```

只加了一个判断，改动极小，零侵入。

---

## 🛡️ 第三步：加个“低性能模式”

```
recordFrameTime(time: number): void {
    this.frameTimes.push(time);
    if (this.frameTimes.length > 5) this.frameTimes.shift();

    const slowFrames = this.frameTimes.filter(t => t > 16).length;
    this.isLowPerformanceMode = slowFrames >= 3;
    if (this.isLowPerformanceMode) {
        console.warn('⚠️ 进入低性能模式，降低刷新率保证流畅');
    }
}
```

当连续几帧掉帧，就自动降级刷新频率。
你说是不是很“智能”？😎

---

## 📊 实测结果

| 手机型号 | 优化前 | 优化后 | 提升幅度 | 刷新次数/秒 |
| --- | --- | --- | --- | --- |
| iPhone 14 Pro | 54 fps | **60 fps** | ↑ 11% | 60 → 35 |
| 小米 12 | 38 fps | **55 fps** | ↑ 45% | 60 → 28 |
| 红米 Note 9 | 23 fps | **42 fps** | ↑ 83% | 60 → 18 |

没看错，**低端机性能提升了 83%！**
而代码改动量，不到 100 行。

---

## 🧠 进阶优化：动态缓冲区

再来一个锦上添花的小优化——

```
private getBufferSize(): number {
    const speed = this.perfManager.getSpeed();

    if (speed > 2000) return 3; // 飞速滑动：3屏缓冲
    if (speed > 1000) return 2; // 快速滑动：2屏缓冲
    return 1;                   // 慢速滑动：1屏缓冲
}
```

滑得越快，就多加载几屏内容，防止白屏。

---

## 🧩 核心总结

1️⃣ 不要盲目追求高刷新率。
2️⃣ 用数据说话，不凭感觉。
3️⃣ 性能优化要有取舍，不是一味地“更快”。

---

## 🌙 写在最后

虚拟列表的性能优化，说到底是一场“感知”与“取舍”的平衡。

**一句话总结：**

> 好的性能优化，不是让程序跑得更快，而是让程序知道什么时候该偷懒。

就像武侠小说里的高手，不是每一招都用尽全力，而是知道——
何时发力，何时卸力。

---

## 查看源码

👉 [《高性能实用虚拟列表》](https://github.com)

**核心特性：**

* 支持垂直/水平/网格布局
* 动态高度 / 宽度
* 自适应性能优化
* 对象池自动管理
* 插入 / 删除动画

---

## 🙋 常见问题

**Q：会看到空白吗？**
A：不会。快速滑动时自动扩展缓冲区。

**Q：兼容性如何？**
A：Cocos Creator 2.x / 3.x 通吃，原生、小游戏、H5 全支持。

**Q：老项目能用吗？**
A：能，只需在 `onScrolling` 里加一句判断。

---

## ❤️ 最后

如果你也在为性能掉帧头疼，希望这篇文章能帮你少熬几个夜。
也欢迎留言分享你的思路，让我们一起把 Cocos 的生态做得更好。

**祝大家的列表都丝滑如德芙！🍫**

本博客参考[楚门加速器](https://koumen.org)。转载请注明出处！
