<template>
    <div class="main">
        <a-space wrap>
            <a-button type="primary" @click="$router.push('/add')">添加</a-button>
            <a-button type="primary" @click="syncData">同步数据</a-button>
            <a-popconfirm title="确定清除当前页面所有Cookie？" ok-text="Yes" cancel-text="No" @confirm="removeCookie">
                <a-button danger block>清除当前页面cookie</a-button>
            </a-popconfirm>
            <a-popconfirm title="确定删除所有Cookie？" ok-text="Yes" cancel-text="No" @confirm="deleteAllCookies">
                <a-button type="primary" danger block>删除所有cookie</a-button>
            </a-popconfirm>
        </a-space>
        <a-table :columns="columns" :data-source="data" :scroll="{ x: '100%' }" size='small'
            style="height: 100%;margin-top: 8px;" bordered>
            <template #bodyCell="{ column, record, index }">
                <template v-if="column.dataIndex === 'name'">
                    {{ record.name }}
                </template>
                <template v-else-if="column.dataIndex === 'cookies' && record.cookiesArr">
                    <div class="cookieClass">
                        <div color="geekblue" v-for="(item, index) in record.cookiesArr" :key="index">
                            <a-tag color="#2db7f5"> {{ JSON.stringify(item) }}</a-tag>
                        </div>
                    </div>
                </template>
                <template v-else-if="column.dataIndex === 'Action'">
                    <div>
                        <a-tooltip>
                            <template #title>编辑</template>
                            <FormOutlined
                                @click="$router.push({ path: '/edit', query: { row: JSON.stringify(record) } })" />
                        </a-tooltip>
                        <a-divider type="vertical" />
                        <a-popconfirm title="你确定删除吗？" ok-text="Yes" cancel-text="No"
                            @confirm="delCookie(index)">
                            <a-tooltip>
                                <template #title>删除</template>
                                <DeleteOutlined />
                            </a-tooltip>
                        </a-popconfirm>
                        <a-divider type="vertical" />
                        <a-popconfirm title="你确定注入吗？" ok-text="Yes" cancel-text="No"
                            @confirm="injectCookie(record)">
                            <a-tooltip>
                                <template #title>注入</template>
                                <AimOutlined :class="{ 'highlighted': record.name === usedCookie.name }" />
                            </a-tooltip>
                        </a-popconfirm>
                        <a-divider type="vertical" />
                        <a-tooltip>
                            <template #title>拷贝当前窗口所有cookie</template>
                            <RedoOutlined @click="copyAllCookie(record.name)" />
                        </a-tooltip>
                    </div>
                </template>
            </template>
        </a-table>
    </div>
</template>
<script setup>
import { apiReqs } from '@/api'
import { DeleteOutlined, AimOutlined, FormOutlined, RedoOutlined } from '@ant-design/icons-vue';
import { message } from 'ant-design-vue';
import { ref, inject, onMounted  } from 'vue';
const methods = inject('globalMethods');
const data = ref([]);
const usedCookie = ref({});
const columns = [
    {
        title: '名称',
        dataIndex: 'name',
    },
    {
        title: 'cookies',
        dataIndex: 'cookies',
        width: '40%'
    },
    {
        title: '操作',
        dataIndex: 'Action',
        fixed: 'right'
    },
];

const syncData = () => {
    // 同步数据  
    apiReqs.getData({
        // 如果上传文件，则设置formData为true，这里暂时不用。
        success: (res) => {
            if (Array.isArray(res) && res.length > 0) {
                const dataList = res
                    .filter((item) => Array.isArray(item.Cookie_json) && item.Cookie_json.length > 0)
                    .map((item) => ({
                        name: item.username,
                        cookiesArr: item.Cookie_json,
                    }));

                if (dataList.length === 0) {
                    message.success('接口暂无有效数据');
                    return;
                }
                data.value = dataList; //更新组件的数据状态，触发页面重新渲染
                methods.getStorage('cookie', async (data) => {
                    let newdata = [];
                    if (!Array.isArray(data) || data.length === 0) {
                        newdata = dataList;
                    } else {
                        dataList.forEach((element) => {
                            const existing = data.findIndex((item) => item.name === element.name);
                            if (existing !== -1) {
                                // 更新已存在的元素
                                data[existing] = element;
                            } else {
                                // 将不存在的元素插入到数组末尾
                                newdata.push(element);
                            }
                        });
                        newdata = [...newdata, ...data];
                    }


                    message.success('接口获取数据成功');
                    await methods.setStorage({ cookie: newdata });
                    getList();
                });
            } else {
                message.success('接口暂无数据');
            }
        },
        fail: (res) => {
            console.log('接口获取数据失败', res)
            message.error('接口获取数据失败');
        },
    })

}

const copyAllCookie = (name) => {
    chrome.tabs.query({ active: true, currentWindow: true }, function (tabs) {
        if (!tabs) {
            message.error('获取当前窗口域名失败');
        }
        chrome.runtime.sendMessage({ action: 'copyAllCookie', tabs, name }, (response) => {
            console.log('收到来自 background.js 的响应:', response);
            getList();
        });
    });
}


const delCookie = (index) => {
    data.value.splice(index, 1);
    methods.setStorage({ 'cookie': data.value });
}

const removeCookie = () => {
    chrome.tabs.query({ active: true, currentWindow: true }, function (tabs) {
        if (!tabs) {
            message.error('获取当前窗口域名失败');
        }
        chrome.runtime.sendMessage({ action: 'delCookie', tabs: tabs }, (response) => {
            console.log('收到来自 background.js 的响应:', response);
        });
    });
}

const injectCookie = (data) => {
    // 注入cookie 发送消息到 background.js
    if (!data.cookiesArr) {
        console.log('cookie为空')
    }
    chrome.tabs.query({ active: true, currentWindow: true }, function (tabs) {
        if (!tabs) {
            console.log('获取当前窗口域名失败')
        }
        chrome.runtime.sendMessage({ action: 'injectCookie', data: data.cookiesArr, tabs: tabs }, (response) => {
            console.log('收到来自 background.js 的响应:', response);
            methods.setStorage({ 'usedCookie': data });
            usedCookie.value = data
        });
    });
}

const getUsedCookie = () => {
    // 获取已注入cookie
    methods.getStorage(('usedCookie'), async (data) => {
        console.log("usedCookie list is", data)
        if (data) {
            usedCookie.value = data
        }
    });
}



const getList = () => {
    // 获取cookie列表
    methods.getStorage(('cookie'), async (list) => {
        console.log("Cookie list is", list)
        if (Array.isArray(list) && list.length > 0) {
            data.value = [...list]; // 修改数据的引用，触发组件的重新渲染
        } else {
            methods.delAllStorage();
        }
    });
}


const deleteAllCookies = () => {
    methods.delAllStorage(); // 调用删除所有 cookie 的方法
    data.value = []; // 清空 data 列表，可能需要根据具体情况进行处理
}

onMounted(() => {
    // 列表数据
    getList()
    // 已注入cookie
    getUsedCookie()
})


</script>
<style scoped>
.highlighted {
    color: #00BFFF;
}

.main {
    padding: 8px;
    flex: 1
}

.cookieClass {
    display: flex;
    flex-direction: column;
    overflow-y: hidden
}
</style>
