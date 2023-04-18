<template>
  <div
    ref="scrollWrapperEle"
    class="scroll-wrapper"
    :style="{ maxHeight: (maxHeight ? maxHeight : 600) + 'px' }"
  >
    <div class="scroll-header" ref="scrollHeaderEle"></div>
    <div ref="scrollContentEle">
      <slot :renderList="virtual.curDisplayList"></slot>
    </div>
    <div class="scroll-loading" ref="scrollLoadingEle" v-show="hasMoreData">
      正在努力加载更多数据中...
    </div>
    <div class="no-more" v-show="virtual.curDisplayList.length && !hasMoreData">全部加载完成~</div>
    <div class="no-data" v-if="!virtual.curDisplayList.length && !error">
      <el-empty description="暂无更多数据" />
    </div>
    <div class="err-warp" v-if="error">
      <div class="err">
        <el-icon :size="180" color="#C0C4CC"><i-ep-FolderDelete /></el-icon>
        <p>出错啦~</p>
      </div>
    </div>
  </div>
</template>
<script lang="ts" setup>
import { computed, onMounted, reactive, ref, watch } from 'vue';
const props = defineProps<{
  perPage?: number; // 分页大小
  maxHeight?: number; // 滚动容器的最大高度
  list: any[]; // 总列表
  loading: boolean; // 数据是否加载中
  error: boolean; // 数据是否有问题（接口报错）
}>();
const scrollWrapperEle = ref();
const scrollContentEle = ref();
const emit = defineEmits(['updateLoading']);
// 交叉观察器
const intersectionObserver = new IntersectionObserver(entries => {
  entries.forEach(it => {
    // 触发目标为底部加载更多容器
    if (it.target.className === 'scroll-loading') {
      // 数据加载完毕前不触发
      if (it.isIntersecting && !props.loading) {
        const { displayList, listPage } = virtual;
        const { page, perPage, total } = listPage;
        // 当前显示的数据不是列表最后一页
        if (page < Math.ceil(total / perPage)) {
          virtual.curDisplayList = [
            ...virtual.curDisplayList,
            ...displayList.slice(page * perPage, (page + 1) * perPage),
          ];
          if (page > 2) {
            // 展示数据为第三页时候开始删除前面的数据
            virtual.curDisplayList.splice(0, perPage);
          }
          listPage.page++;
        }
      }
    } else if (it.target.className === 'scroll-header') {
      // 触发目标为顶部空白容器
      if (it.isIntersecting && !props.loading) {
        const { displayList, listPage } = virtual;
        const { page, perPage, total } = listPage;
        if (page === 3) {
          virtual.curDisplayList.splice(-2 * perPage, 2 * perPage);
          listPage.page -= 2;
        }
        if (page > 3) {
          emit('updateLoading', true);
          scrollWrapperEle.value.style.overflowY = 'hidden'; // 禁止滚动
          virtual.curDisplayList = [
            ...displayList.slice((page - 4) * perPage, (page - 3) * perPage),
            ...virtual.curDisplayList,
          ];
          // 当前显示为最后一页数据
          if (page >= Math.ceil(total / perPage)) {
            // 最后一页数据可能小于分页数
            virtual.curDisplayList.splice(-perPage, total - (page - 1) * perPage);
          } else {
            virtual.curDisplayList.splice(-perPage, perPage);
          }
          // 需要计算更新后的dom的真实高度 这里用setTimeout0
          setTimeout(() => {
            let parentNodeType = scrollContentEle.value.children[0].nodeName;
            let nodes;
            if (parentNodeType === 'UL') {
              nodes = scrollContentEle.value.children[0].children;
            } else if (parentNodeType === 'DIV') {
              nodes = scrollContentEle.value.children;
            }
            let nodesHeight = 0;
            for (let i = 0; i < perPage; i++) {
              nodesHeight += nodes[i].offsetHeight;
            }
            scrollWrapperEle.value.scrollTop = nodesHeight;
            setTimeout(() => {
              emit('updateLoading', false);
              scrollWrapperEle.value.style.overflowY = 'auto';
            }, 600);
          }, 0);
          listPage.page--;
        }
      }
    }
  });
});
interface ListPage {
  page: number;
  perPage: number;
  total: number;
}
interface Virtual {
  displayList: any[];
  curDisplayList: any[];
  listPage: ListPage;
}
const virtual: Virtual = reactive({
  displayList: [],
  curDisplayList: [],
  listPage: {
    page: 1,
    perPage: 3,
    total: 0,
  },
});
const hasMoreData = computed(() => {
  return virtual.listPage.page * virtual.listPage.perPage < virtual.listPage.total;
});
const initPagingInfo = () => {
  let { listPage, displayList } = virtual;
  listPage.page = 1;
  listPage.total = displayList.length;
  if (listPage.total <= listPage.perPage) {
    virtual.curDisplayList = [...displayList];
  } else {
    virtual.curDisplayList = [...displayList.slice(0, listPage.perPage)];
  }
};
const scrollLoadingEle = ref();
const scrollHeaderEle = ref();
onMounted(() => {
  if (props.perPage) {
    virtual.listPage.perPage = props.perPage;
  }
  if (props.list.length) {
    virtual.displayList = JSON.parse(JSON.stringify(props.list));
    initPagingInfo();
  }
  // 开始观察
  intersectionObserver.observe(scrollLoadingEle.value);
  intersectionObserver.observe(scrollHeaderEle.value);
});
watch(
  () => props.list,
  list => {
    virtual.displayList = JSON.parse(JSON.stringify(list));
    initPagingInfo();
    scrollWrapperEle.value.scrollTop = 0; // 重置滚动高度，避免出现数据源切换导致滚动条位置错误
  },
);
</script>
<style lang="less" scoped>
.no-more {
  font-size: 16px;
  text-align: center;
  color: #c0c4cc;
}
.err-warp {
  height: 400px;
  display: flex;
  justify-content: center;
  align-items: center;
  .err {
    text-align: center;
    font-size: 24px;
    color: #c0c4cc;
  }
}
.scroll-wrapper {
  width: 100%;
  overflow-x: hidden;
  overflow-y: auto;
}
.scroll-header {
  height: 30px;
}
.scroll-loading {
  height: 30px;
  text-align: center;
  font-size: 24px;
}
</style>
