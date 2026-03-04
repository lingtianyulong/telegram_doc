# Telegram 区域绘制机制

[toc]

在 `Telegram` 中, 区域裁剪分“三层”做的，不是只靠一个 setClipRect：

- 第一层：Qt 的重绘区域

  paintEvent(QPaintEvent *e) 先拿 e->rect()，这就是本次需要重绘的脏区（Qt 底层也会按这个区域做实际裁剪）。

- 第 2 层：数据级裁剪（只遍历可见 item）

  通过 lower_bound 算出 [from, to)，只绘制和 clip 相交的消息项，避免遍历全量列表。

- 第 3 层：子模块内再次相交判断

  clip 被塞进 ChatPaintContext，后续 `message/reaction/date/userpic `绘制会继续做 intersects() 判断，局部再裁一次。

  