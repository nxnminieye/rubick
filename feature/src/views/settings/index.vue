<template>
  <div class="settings">
    <div class="left-menu">
      <a-menu v-model:selectedKeys="currentSelect" mode="inline">
        <a-menu-item key="normal">
          <template #icon>
            <ToolOutlined />
          </template>
          基本设置
        </a-menu-item>
        <a-menu-item key="global">
          <template #icon>
            <LaptopOutlined />
          </template>
          全局快捷键
        </a-menu-item>
      </a-menu>
    </div>
    <div class="settings-detail">
      <div v-if="currentSelect[0] === 'normal'">
        <div class="setting-item">
          <div class="title">
            快捷键(需要使用 option/ctrl/shift/command 键修饰)
          </div>
          <div class="settings-item-li">
            <div class="label">显示/隐藏快捷键</div>
            <div
              class="value"
              tabIndex="-1"
              @keyup="(e) => changeShortCut(e, 'showAndHidden')"
            >
              {{ shortCut.showAndHidden }}
            </div>
          </div>
<!--          <div class="settings-item-li">-->
<!--            <div class="label">插件分离快捷键</div>-->
<!--            <div-->
<!--              class="value"-->
<!--              tabIndex="-1"-->
<!--              @keyup="(e) => changeShortCut(e, 'separate')"-->
<!--            >-->
<!--              {{ shortCut.separate }}-->
<!--            </div>-->
<!--          </div>-->
<!--          <div class="settings-item-li">-->
<!--            <div class="label">返回主界面</div>-->
<!--            <div-->
<!--              class="value"-->
<!--              tabIndex="-1"-->
<!--              @keyup="(e) => changeShortCut(e, 'quit')"-->
<!--            >-->
<!--              {{ shortCut.quit }}-->
<!--            </div>-->
<!--          </div>-->
        </div>
        <div class="setting-item">
          <div class="title">通用</div>
          <div class="settings-item-li">
            <div class="label">输入框自动粘贴</div>
            <a-switch
              v-model:checked="common.autoPast"
              checked-children="开"
              un-checked-children="关"
            ></a-switch>
          </div>
          <div class="settings-item-li">
            <div class="label">开机启动</div>
            <a-switch
              v-model:checked="common.start"
              checked-children="开"
              un-checked-children="关"
            ></a-switch>
          </div>
          <div class="settings-item-li">
            <div class="label">空格执行</div>
            <a-switch
              v-model:checked="common.space"
              checked-children="开"
              un-checked-children="关"
            ></a-switch>
          </div>
        </div>
<!--        <div class="setting-item">-->
<!--          <div class="title">本地搜索启动</div>-->
<!--          <div class="settings-item-li">-->
<!--            <div class="label">搜索启动应用&文件</div>-->
<!--            <a-switch-->
<!--              v-model:checked="local.search"-->
<!--              checked-children="开"-->
<!--              un-checked-children="关"-->
<!--            ></a-switch>-->
<!--          </div>-->
<!--        </div>-->
      </div>
      <div v-if="currentSelect[0] === 'global'">
        <a-collapse>
          <a-collapse-panel key="1" header="说明及示例">
            <div>
              按下快捷键，自动搜索对应关键字，当关键字结果完全匹配，且结果唯一时，会直接指向该功能。
            </div>
            <h3 style="margin-top: 10px;">示例</h3>
            <a-divider style="margin: 5px 0;" />
            <a-list item-layout="horizontal" :data-source="examples">
              <template #renderItem="{ item }">
                <a-list-item>
                  <a-list-item-meta :description="item.desc">
                    <template #title>
                      <div>{{ item.title }}</div>
                    </template>
                  </a-list-item-meta>
                </a-list-item>
              </template>
            </a-list>
          </a-collapse-panel>
        </a-collapse>
        <div class="feature-container">
          <div class="keywords item">
            <div>快捷键</div>
            <a-tooltip placement="top" trigger="click">
              <template #title>
                  <span>先按功能键（Ctrl、Shift、Alt、Option、Command），再按其他普通键。或按
                    F1-F12 单键
                  </span>
              </template>
              <div
                :key="index"
                v-for="(item, index) in global"
                class="value"
                tabIndex="2"
                @keyup="(e) => changeGlobalKey(e, index)"
              >
                {{ item.key }}
              </div>
            </a-tooltip>
          </div>
          <div class="short-cut item">
            <div>功能关键字</div>
            <a-input
              :key="index"
              :value="item.value"
              v-for="(item, index) in global"
              class="value"
              :disabled="!item.key"
              @change="(e) => changeGlobalValue(index, e.target.value)"
            />
          </div>
        </div>
        <div @click="addConfig" class="add-global">+ 新增全局快捷功能</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ToolOutlined, LaptopOutlined } from "@ant-design/icons-vue";
import debounce from "lodash.debounce";
import { ref, reactive, watch, toRefs, toRaw } from "vue";
import keycodes from "./keycode";

const { remote, ipcRenderer } = window.require("electron");

const examples = [
  {
    title: "快捷键 「 Alt + W」 关键字 「 微信」",
    desc: "按下Alt + W 直接打开本地微信应用",
  },
  {
    title: "快捷键 「 Alt + Q」 关键字 「 取色」",
    desc: "按下Alt + Q 直接打开屏幕取色功能",
  },
];

const state = reactive({
  shortCut: {},
  common: {},
  local: {},
  global: [],
});

const currentSelect = ref(["normal"]);

const {perf, global: defaultGlobal} = remote.getGlobal("OP_CONFIG").get();

state.shortCut = perf.shortCut;
state.common = perf.common;
state.local = perf.local;
state.global = defaultGlobal;

const setConfig = debounce(() => {
  remote.getGlobal("OP_CONFIG").set(JSON.parse(JSON.stringify({
    perf: {
      shortCut: state.shortCut,
      common: state.common,
      local: state.local,
    },
    global: state.global,
  })));
  ipcRenderer.send("re-register");
}, 2000);

watch(state, setConfig);

const changeShortCut = (e, key) => {
  if (e.altKey && e.keyCode !== 18) {
    const compose = `Option+${keycodes[e.keyCode].toUpperCase()}`;
    state.shortCut[key] = compose;
  }
  if (e.ctrlKey && e.keyCode !== 17) {
    const compose = `Ctrl+${keycodes[e.keyCode].toUpperCase()}`;
    state.perf.shortCut[key] = compose;
  }
  if (e.shiftKey && e.keyCode !== 16) {
    const compose = `Shift+${keycodes[e.keyCode].toUpperCase()}`;
    state.perf.shortCut[key] = compose;
  }
  if (e.metaKey && e.keyCode !== 93) {
    const compose = `Command+${keycodes[e.keyCode].toUpperCase()}`;
    state.perf.shortCut[key] = compose;
  }
};

const changeGlobalKey = (e, index) => {
  let compose;
  if (e.altKey && e.keyCode !== 18) {
    compose = `Alt+${keycodes[e.keyCode].toUpperCase()}`;
  }
  if (e.ctrlKey && e.keyCode !== 17) {
    compose = `Ctrl+${keycodes[e.keyCode].toUpperCase()}`;
  }
  if (e.shiftKey && e.keyCode !== 16) {
    compose = `Shift+${keycodes[e.keyCode].toUpperCase()}`;
  }
  if (e.metaKey && e.keyCode !== 93) {
    compose = `Command+${keycodes[e.keyCode].toUpperCase()}`;
  }
  if (compose) {
    state.global[index].key = compose;
  }
  // f1 - f12
  if (e.keyCode >= 112 && e.keyCode <= 123) {
    compose = keycodes[e.keyCode].toUpperCase();
  }
  if (compose) {
    state.global[index].key = compose;
  }
};

const changeGlobalValue = (index, value) => {
  state.global[index].value = value;
}

const addConfig = () => {
  state.global.push({
    key: "",
    value: "",
  });
};

const {shortCut, common, local, global} = toRefs(state);
</script>

<style lang="less">
.settings {
  box-sizing: border-box;
  width: 100%;
  overflow-x: hidden;
  background: #f3efef;
  height: calc(~"100vh - 46px");
  display: flex;
  .left-menu {
    width: 200px;
    height: 100%;
    .search-container {
      padding: 10px;
    }
    .ant-input-affix-wrapper {
      border: none;
    }
    .ant-menu {
      background: #F3EFEF;
      .ant-menu-item-selected {
        background-color: #E2E2E2;
        color: #141414;
        &:after {
          display: none;
        }
      }
    }
  }
  .settings-detail {
    padding: 20px;
    box-sizing: border-box;
    flex: 1;
    overflow: auto;
    height: 100%;
    background: #fff;
    .setting-item {
      margin-bottom: 20px;
      .ant-form-item {
        margin-bottom: 0;
      }
      .title {
        color: #6c9fe2;
        font-size: 15px;
        margin-bottom: 10px;
      }
      .settings-item-li {
        padding-left: 20px;
        display: flex;
        width: 100%;
        align-items: center;
        justify-content: space-between;
        margin-bottom: 10px;
        .label {
          color: #646464;
        }
        .value {
          width: 300px;
          text-align: center;
          border: 1px solid #ddd;
          color: #6c9fe2;
          font-size: 14px;
          height: 24px;
          font-weight: lighter;
        }
      }
    }
  }
  .feature-container {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-top: 10px;
    font-size: 14px;
    .item {
      flex: 1;
    }
    .short-cut {
      margin-left: 20px;
    }
    .value {
      text-align: center;
      border: 1px solid #ddd;
      color: #6c9fe2;
      font-size: 14px;
      height: 24px;
      font-weight: lighter;
      margin-top: 10px;
    }
  }
  .add-global {
    color: #6c9fe2;
    margin-top: 20px;
    width: 100%;
    text-align: center;
    cursor: pointer;
  }
}
</style>
