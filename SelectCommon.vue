<script lang="ts" setup>
import { nextTick, onMounted, ref, watch } from 'vue';
import { message } from "@/utils/message";

interface FetchParams {
    [key: string]: any;
}

const props = defineProps({
    fetchData: {
        type: Function,
        required: true
    },
    modelValue: {
        type: [String, Number],
        default: ''
    },
    name: {
        type: String,
        default: ''
    },
    placeholder: String,
    disabled: Boolean,
    param: {
        type: Object,
        default: () => ({})
    }
});

//双向绑定更新数据
const emit = defineEmits(['update:modelValue']);

const value = ref(props.modelValue);
const itemList = ref([]);
const totalItems = ref(0);
const loading = ref(false);
const currentPage = ref(1);
const hasMore = ref(true);
const noDataText = ref('无数据');

// 监听 modelValue 的变化
watch(() => props.modelValue, (newVal) => {
    value.value = newVal;
});

// 监听 name 的变化
watch(() => props.name, (newVal) => {
    if (newVal && value.value) {
        // 如果 name 有值，将其添加到选项列表中
        const existingItem = itemList.value.find(item => item.value === value.value);
        if (!existingItem) {
            itemList.value = [{
                label: newVal,
                value: value.value
            }, ...itemList.value];
        }
    }
});

// 异步获取数据的方法
const getData = async (params: FetchParams, isLoadMore = false) => {
    if (!hasMore.value && isLoadMore) return;
    
    loading.value = true;
    try {
        const res = await props.fetchData(params);
        if (!res.error) {
            const newItems = res.items?.map(item => ({
                label: item.name,
                value: item.id
            })) || [];
            
            if (isLoadMore) {
                // 加载更多时，追加数据
                itemList.value = [...itemList.value, ...newItems];
            } else {
                // 首次加载或搜索时，替换数据
                itemList.value = newItems;
            }
            
            // 如果有 name 值，确保它在列表中
            if (props.name && value.value) {
                const existingItem = itemList.value.find(item => item.value === value.value);
                if (!existingItem) {
                    itemList.value = [{
                        label: props.name,
                        value: value.value
                    }, ...itemList.value];
                }
            }
            
            // 判断是否还有更多数据
            hasMore.value = newItems.length === params.MaxResultCount;
            totalItems.value = itemList.value.length;
        } else {
            // 如果接口返回错误，清空数据
            itemList.value = [];
            hasMore.value = false;
            totalItems.value = 0;
        }
    } catch (error) {
        // 如果发生错误，清空数据
        itemList.value = [];
        hasMore.value = false;
        totalItems.value = 0;
        message(error, { customClass: 'el', type: 'error' });
    } finally {
        loading.value = false;
    }
};

// 远程搜索方法
const remoteSearch = debounce(async (query: string) => {
    currentPage.value = 1; // 重置页码
    hasMore.value = true; // 重置加载更多状态
    if (query) {
        const params: FetchParams = {
            ...props.param,
            skipCount: 1,
            MaxResultCount: 10,
            keyword: query
        };
        await getData(params);
    } else {
        // 如果搜索词为空，则加载初始数据
        const params: FetchParams = {
            ...props.param,
            skipCount: 1,
            MaxResultCount: 10
        };
        await getData(params);
    }
}, 300);

onMounted(async () => {
    // 初始加载数据
    const params: FetchParams = {
        ...props.param,
        skipCount: 1,
        MaxResultCount: 10
    };
    await getData(params);
});

const handleVisibleChange = async (visible) => {
    if (visible) {
        // 先清空数据
        itemList.value = [];
        totalItems.value = 0;
        hasMore.value = true;
        currentPage.value = 1;
        
        // 加载初始数据
        const params: FetchParams = {
            ...props.param,
            skipCount: 1,
            MaxResultCount: 10
        };
        await getData(params);
        
        // 添加滚动事件监听
        const dropdown = document.querySelector('.myselect-loadmore .el-select-dropdown__wrap');
        if (dropdown) {
            dropdown.addEventListener('scroll', handleScroll);
            // 重置滚动位置
            dropdown.scrollTop = 0;
        }
    } else {
        // 移除滚动事件监听
        const dropdown = document.querySelector('.el-select-dropdown');
        if (dropdown) {
            dropdown.removeEventListener('scroll', handleScroll);
        }
    }
};

//防抖函数
function debounce<T extends (...args: any[]) => any>(func: T, delay: number) {
    let timeout: NodeJS.Timeout;
    return function (this: any, ...args: Parameters<T>) {
        const context = this;
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(context, args), delay);
    };
}

const handleScroll = debounce(async (event) => {
    // 判断是否滚动到底部
    const bottom = event.target.scrollHeight === event.target.scrollTop + event.target.clientHeight;
    if (bottom && !loading.value && hasMore.value) {
        currentPage.value++;
        // 调用父组件传递的函数，并传入子组件的参数
        const params: FetchParams = {
            ...props.param,
            skipCount: currentPage.value,
            MaxResultCount: 10
        };
        await getData(params, true);
    }
}, 100);
</script>

<template>
    <el-select 
        v-model="value" 
        class="myselect" 
        :placeholder="placeholder || '请选择！'" 
        popper-class="myselect-loadmore"
        filterable 
        remote
        remote-show-suffix
        :remote-method="remoteSearch"
        clearable 
        :disabled="disabled" 
        @visible-change="handleVisibleChange"
        @update:modelValue="(val) => emit('update:modelValue', val)">
        <el-option v-for="(item, index) in itemList" :key="item.value" :label="item.label" :value="item.value" />
    </el-select>
</template>

<style scoped>
.myselect-loadmore :deep(.el-select-dropdown__wrap) {
    max-height: 274px;
}

</style>
