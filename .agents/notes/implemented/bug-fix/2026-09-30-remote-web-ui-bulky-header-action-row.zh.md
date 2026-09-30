# Agent Note: 竖屏适配层把第三方宽按钮行留在标题行

Status: implemented

## Problem

竖屏触摸适配层（`packages/dsh-remote-web-ui/src/client/mobile-adapt.ts`）会把官方的页头动作——agent 预设模式标签与后台任务徽章——从标题簇重排到标签页行：`body.dsh-remote-header-seated` 用注入的 CSS 把该槽位移出文档流，`alignActionsText()` 再收敛出一个 `translate(dx, dy)` 把它画到标签页行上。

这个座位是按两个小徽章量的。第三方插件往同一个槽位注入的东西多得多：`dsh-session-manager` 把 `.sm-header` 渲染进 `conversation.session.header.actions`，六个按钮（归档、星标、待回看、标签/备注、移动至工作区、删除会话）。在真实 GUI 上以 393x851 竖屏 + 触摸模拟实测，这行被 seat 之后：

- `actions` 报 `position: absolute`、`translate(-28px, 65px)`、`offsetParent === wSkVaW_root`——不是标题簇，因为 `titleRow`、`titleCluster` 与页头全是 `position: static`，绝对定位的盒子落到了最近的已定位祖先，也就是应用根节点；
- 前五个按钮落在 y=65，而页头只有 76 px 高，`删除会话` 折到 y=103，即掉到页头下方、浮在消息流上；
- 它发布的 reserve 是 `--dsh-remote-header-actions-reserve: 401px`，比 393 px 的视口还宽，于是标签页行按 `max(0px, 100% - 401px)` 自我收缩，只剩 8 px 宽，对话 / 轨迹 / 上下文三个标签一起消失。

桌面端不受影响：1600x900 下同一会话的槽位是 `static`，六个按钮在 y=10 一行排开。

## Decision

带着第三方宽按钮行的槽位保留桌面端的位置——行留在标题簇里、留在文档流内，由 React 持有。

- `holdsBulkyActionRow(actions)` 检测槽位内的 `.sm-header`。`seatHeaderActions()` 把它并入 `seated` 判定，该槽位因此不被 seat；`alignActionsText()` 对它提前返回。两处都调用 `clearSeatOffsets(actions)`：清掉节点上的 `transform`，并从 `<html>` 上移除 `--dsh-remote-header-actions-reserve`，这样热更新的页面不会留着过期的偏移，也不会留着一个仍在挤压标签页行的 reserve。
- 四条 CSS 规则全部限定在 `body.dsh-remote-portrait` 且带 `:has(.sm-header)`，让这行占据标题簇而不是被画到别处：标题簇 `flex-wrap: wrap` 配 4 px 行距，槽位吃 `flex: 1 0 100%` 并 `flex-wrap: wrap`、`justify-content: flex-start`，`.sm-headerBtn` 收紧到 `min-height: 26px; padding: 2px 7px; font-size: 12px`。

改动后在 393x851 实测：槽位是 `static` 且没有 transform，按钮折成两行紧凑排在 166 px 高页头内的 y=74 与 y=104，标签页行恢复 345 px，reserve 为空，六个按钮的命中测试全部命中自身。1600x900 下每一项实测值——槽位 `[354, 10, 607, 32]`、`.sm-header` `[531, 10, 430, 32]`、六个按钮全在 y=10——与改动前基线逐值一致。

## Alternatives considered

按宽度而不是按类名判定（`actions.getBoundingClientRect().width > N`）：否决，太脆。宽度取决于语言、按钮文案和 React 的绘制时机，对某个插件刚好的阈值对下一个就错了。`.sm-header` 是注入方已经公开的稳定契约。

把行压缩到能塞进座位（只留图标，或收成一个溢出菜单）：否决，那会藏掉让六个动作可被发现的文案，并且等于改另一个插件渲染什么。报的是位置问题，不是密度问题。

限制 reserve 的上限（`min(reserve, 40vw)`）同时继续 seat：否决。它只修好 8 px 的标签页行，行依旧被画在页头下方、压在消息流上——而那正是用户看到的缺陷。

把节点搬进标签页行（v67 之前的行为）：已被[拥有该座位的 note](2026-09-09-mobile-remote-tap-and-adaptation-fixes.md) 否决——搬动 React 持有的宿主节点会破坏 React 后续 `insertBefore` / `removeChild` 的锚点，而里面的徽章是有活处理函数的 React 按钮，克隆留不住。本次改动搬得更少，不是更多。

## Consequences

适配层现在区分同一个官方槽位里的两类占用者：官方小徽章保留为它们测量的座位，第三方宽按钮行保留桌面端位置。判据是另一个包拥有的类名，因此是对 `dsh-session-manager` 标记的一次耦合：那个类名一旦变化，这一行会静默回落到 seat 行为，本缺陷复现。在对齐阶段判定「宽到座位装不下」可以泛化到任意插件，代价是在发现页头的那一拍多一次布局读取；类名检查便宜，且对今天唯一发布这种行的插件是精确的。

两条规则保持由 `:has(.sm-header)` 门控，而不是套用到每个标题簇，因此当官方徽章是唯一占用者时它的布局不变。`:has()` 需要 Chrome 105 及以上；报缺陷的设备是 ArkWeb / Chromium 144。

## Testing

`packages/dsh-remote-web-ui/tests/mobile-adapt.spec.ts` 新增两条用例：一条断言带 `.sm-header` 的槽位永不被 seat，也不发布 transform 或 reserve；一条断言紧凑规则由 `:has(.sm-header)` 限定范围，而不是施加到每个标题簇。两者在现有 jsdom 环境中与座位相关用例一起跑，后者原样通过——官方几何仍然被 seat。

真实 GUI 验证用 headless Chrome 经 CDP，393x851、`pointer: coarse`、竖屏、五点触摸与报缺陷的华为浏览器 UA：改动前后在两个视口各读一次几何，并对六个按钮做 `elementFromPoint` 命中测试、对消息区做重叠探测。截图：[手机](../../../packages/dsh-remote-web-ui/docs/evidence-bulky-action-row-phone.png)、[桌面](../../../packages/dsh-remote-web-ui/docs/evidence-bulky-action-row-desktop.png)。
