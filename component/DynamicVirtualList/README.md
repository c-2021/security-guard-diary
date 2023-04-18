### 不定高虚拟滚动

- 开发技术 vue3+ts
- 实现原理利用 IntersectionObserverAPI 和分页，监听列表顶部和底部出现时机进行数据的切换
- 组件接收 prop
  1. perPage 分页大小
  2. maxHeight 滚动容器的最大高度
  3. list 总列表
  4. loading 数据是否加载中
  5. error 数据是否有问题（eg: 接口报错）
- 组件上抛事件 updateLoading
  1. 携带参数为 true/false
  2. 触发时机 往回(上)滚数据且当前页大于第三页的时候
  3. 上抛该事件原因 当需要重新加载之前已删除页数据时候，为了避免出现不必要的 bug 需要禁止滚动，故为了用户体验加了 loading 效果
