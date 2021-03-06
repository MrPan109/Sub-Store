<template>
  <v-container>
    <v-card class="mb-4">
      <v-subheader>基本信息</v-subheader>
      <v-form class="pl-4 pr-4 pb-0" v-model="formState.basicValid">
        <v-text-field
            v-model="options.name"
            class="mt-2"
            :rules="validations.nameRules"
            required
            label="订阅名称"
            placeholder="填入订阅名称，名称需唯一"
            clearable
            clear-icon="clear"
        />
        <v-textarea
            v-model="options.url"
            class="mt-2"
            rows="2"
            :rules="validations.urlRules"
            required
            label="订阅链接"
            placeholder="填入机场原始订阅链接"
            clearable
            clear-icon="clear"
        />
      </v-form>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn icon @click="save">
          <v-icon>save_alt</v-icon>
        </v-btn>
        <v-dialog max-width="400px" v-model="showShareDialog">
          <template #activator="{on}">
            <v-btn icon v-on="on">
              <v-icon>cloud_circle</v-icon>
            </v-btn>
          </template>
          <v-card class="pl-4 pr-4 pb-4 pt-4">
            <v-card-title>
              <v-icon left>cloud_circle</v-icon>
              配置同步
              <v-spacer/>
              <v-btn icon @click="share">
                <v-icon small>share</v-icon>
              </v-btn>
            </v-card-title>
            <v-textarea
                v-model="importedSub"
                solo
                label="粘贴配置以导入"
                rows="5"
                clearable
                clear-icon="clear"
                :rules="validations.importRules"
            />
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn text color="primary" @click="importSub">确认</v-btn>
              <v-btn text @click="showShareDialog = false">取消</v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>

      </v-card-actions>
    </v-card>
    <v-card class="mb-4">
      <v-subheader>常用选项</v-subheader>
      <v-form class="pl-4 pr-4">
        <v-item-group>
          <v-radio-group
              v-model="options.useless"
              dense
              class="mt-0 mb-0"
          >
            过滤非法节点
            <v-row>
              <v-col>
                <v-radio label="保留" value="KEEP"/>
              </v-col>
              <v-col>
                <v-radio label="删除" value="REMOVE"/>
              </v-col>
              <v-col></v-col>
            </v-row>
          </v-radio-group>

          <v-radio-group
              v-model="options.udp"
              dense
              class="mt-0 mb-0"
          >
            UDP转发
            <v-row>
              <v-col>
                <v-radio label="默认" value="DEFAULT"/>
              </v-col>
              <v-col>
                <v-radio label="强制开启" value="FORCE_OPEN"/>
              </v-col>
              <v-col>
                <v-radio label="强制关闭" value="FORCE_CLOSE"/>
              </v-col>
            </v-row>
          </v-radio-group>

          <v-radio-group
              v-model="options.scert"
              dense
              class="mt-0 mb-0"
          >
            跳过证书验证
            <v-row>
              <v-col>
                <v-radio label="默认" value="DEFAULT"/>
              </v-col>
              <v-col>
                <v-radio label="强制跳过" value="FORCE_OPEN"/>
              </v-col>
              <v-col>
                <v-radio label="强制验证" value="FORCE_CLOSE"/>
              </v-col>
            </v-row>
          </v-radio-group>
          <v-radio-group
              v-model="options.tfo"
              dense
              class="mt-0 mb-0"
          >
            TCP Fast Open
            <v-row>
              <v-col>
                <v-radio label="默认" value="DEFAULT"/>
              </v-col>
              <v-col>
                <v-radio label="强制开启" value="FORCE_OPEN"/>
              </v-col>
              <v-col>
                <v-radio label="强制关闭" value="FORCE_CLOSE"/>
              </v-col>
            </v-row>
          </v-radio-group>
        </v-item-group>
      </v-form>
    </v-card>
    <v-card class="mb-4" id="processors">
      <v-subheader>
        节点操作
        <v-spacer></v-spacer>
        <v-dialog scrollable v-model="dialog">
          <template #activator="{on}">
            <v-btn icon v-on="on">
              <v-icon color="primary">add_circle</v-icon>
            </v-btn>
          </template>
          <v-card>
            <v-card-title>选择节点操作</v-card-title>
            <v-divider></v-divider>
            <v-card-text>
              <v-radio-group dense v-model="selectedProcess">
                <v-radio v-for="(k, idx) in Object.keys(availableProcessors)" :label="availableProcessors[k].name"
                         :key="idx" :value="k"></v-radio>
              </v-radio-group>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn color="primary" text @click="addProcess(selectedProcess)">确认</v-btn>
              <v-btn text @click="dialog = false">取消</v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>

      </v-subheader>
      <v-divider></v-divider>
      <component v-for="p in processors"
                 :is="p.component"
                 :key="p.id"
                 :args="p.args"
                 @dataChanged="dataChanged"
                 @deleteProcess="deleteProcess"
                 @up="moveUp"
                 @down="moveDown"
      >
      </component>
    </v-card>
  </v-container>
</template>

<script>
import {showError, showInfo} from "@/utils";
import TypeFilter from "@/components/TypeFilter";
import RegionFilter from "@/components/RegionFilter";
import KeywordFilter from "@/components/KeywordFilter";
import RegexFilter from "@/components/RegexFilter";
import SortOperator from "@/components/SortOperator";
import KeywordRenameOperator from "@/components/KeywordRenameOperator";
import RegexRenameOperator from "@/components/RegexRenameOperator";
import KeywordDeleteOperator from "@/components/KeywordDeleteOperator";
import RegexDeleteOperator from "@/components/RegexDeleteOperator";
import FlagOperator from "@/components/FlagOperator";
import ScriptFilter from "@/components/ScriptFilter";
import ScriptOperator from "@/components/ScriptOperator";
import KeywordSortOperator from "@/components/KeywordSortOperator";

const AVAILABLE_PROCESSORS = {
  "Flag Operator": {
    component: "FlagOperator",
    name: "国旗"
  },
  "Type Filter": {
    component: "TypeFilter",
    name: "类型过滤器"
  },
  "Region Filter": {
    component: "RegionFilter",
    name: "区域过滤器"
  },
  "Keyword Filter": {
    component: "KeywordFilter",
    name: "关键词过滤器"
  },
  "Regex Filter": {
    component: "RegexFilter",
    name: "正则过滤器"
  },
  "Sort Operator": {
    component: "SortOperator",
    name: "节点排序"
  },
  "Keyword Sort Operator": {
    component: "KeywordSortOperator",
    name: "关键词排序"
  },
  "Keyword Rename Operator": {
    component: "KeywordRenameOperator",
    name: "关键词重命名"
  },
  "Regex Rename Operator": {
    component: "RegexRenameOperator",
    name: "正则重命名"
  },
  "Keyword Delete Operator": {
    component: "KeywordDeleteOperator",
    name: "删除关键词"
  },
  "Regex Delete Operator": {
    component: "RegexDeleteOperator",
    name: "删除正则"
  },
  "Script Filter": {
    component: "ScriptFilter",
    name: "脚本过滤器"
  },
  "Script Operator": {
    component: "ScriptOperator",
    name: "脚本操作"
  }
}

export default {
  components: {
    FlagOperator,
    KeywordFilter,
    RegexFilter,
    RegionFilter,
    TypeFilter,
    SortOperator,
    KeywordRenameOperator,
    KeywordSortOperator,
    RegexRenameOperator,
    KeywordDeleteOperator,
    RegexDeleteOperator,
    ScriptFilter,
    ScriptOperator,
  },
  data: function () {
    return {
      selectedProcess: null,
      showShareDialog: false,
      importedSub: "",
      dialog: false,
      validations: {
        nameRules: [
          v => !!v || "订阅名称不能为空！",
          v => /^[\w-_]*$/.test(v) || "订阅名称只能包含英文字符、横杠和下划线！"
        ],
        urlRules: [
          v => !!v || "订阅链接不能为空！",
          v => /^https?:\/\//.test(v) || "订阅链接不合法！"
        ],
        importRules: [
          v => !!v || "不能导入空配置！"
        ]
      },
      formState: {
        basicValid: false
      },
      options: {
        name: "",
        url: "",
        useless: "KEEP",
        udp: "DEFAULT",
        scert: "DEFAULT",
        tfo: "DEFAULT",
      },
      process: [],
    }
  },
  created() {
    const name = this.$route.params.name;
    const sub = (typeof name === 'undefined' || name === 'UNTITLED') ? {} : this.$store.state.subscriptions[name];
    this.$store.commit("SET_NAV_TITLE", sub.name ? `订阅编辑 -- ${sub.name}` : "新建订阅");
    const {options, process} = loadSubscription(this.options, sub);
    this.options = options;
    this.process = process;
  },
  computed: {
    availableProcessors() {
      return AVAILABLE_PROCESSORS;
    },

    processors() {
      return this.process.map(p => {
        return {
          component: AVAILABLE_PROCESSORS[p.type].component,
          args: p.args,
          id: p.id
        }
      });
    }
  },
  methods: {
    save() {
      if (this.options.name && this.options.url) {
        const sub = buildSubscription(this.options, this.process);
        if (this.$route.params.name !== "UNTITLED") {
          this.$store.dispatch("UPDATE_SUBSCRIPTION", {
            name: this.$route.params.name,
            sub
          }).then(() => {
            showInfo(`成功保存订阅：${this.options.name}！`);
          }).catch(() => {
            showError(`发生错误，无法保存订阅！`);
          });
        } else {
          this.$store.dispatch("NEW_SUBSCRIPTION", sub).then(() => {
            showInfo(`成功创建订阅：${this.options.name}！`);
          }).catch(() => {
            showError(`发生错误，无法创建订阅！`);
          });
        }
      }
    },

    share() {
      let sub = buildSubscription(this.options, this.process);
      sub.name = "「订阅名称」";
      sub.url = "「订阅链接」";
      sub = JSON.stringify(sub);
      this.$clipboard(sub);
      this.$store.commit("SET_SUCCESS_MESSAGE", "导出成功，订阅已复制到剪贴板！");
      this.showShareDialog = false;
    },

    importSub() {
      if (this.importedSub) {
        const sub = JSON.parse(this.importedSub);
        const {options, process} = loadSubscription(this.options, sub);
        delete options.name;
        delete options.url;

        Object.assign(this.options, options);
        this.process = process;

        this.$store.commit("SET_SUCCESS_MESSAGE", "成功导入订阅！");
        this.showShareDialog = false;
      } else {
        this.$store.commit("SET_ERROR_MESSAGE", "不能导入空配置！");
      }
    },

    dataChanged(content) {
      let index = 0;
      for (; index < this.process.length; index++) {
        if (this.process[index].id === content.idx) {
          break;
        }
      }
      this.process[index].args = content.args;
    },

    addProcess(type) {
      this.process.push({type, id: uuidv4()});
      this.dialog = false;
    },

    deleteProcess(id) {
      let index = 0;
      for (; index < this.process.length; index++) {
        if (this.process[index].id === id) {
          break;
        }
      }
      this.process.splice(index, 1);
    },

    moveUp(id) {
      let index = 0;
      for (; index < this.process.length; index++) {
        if (this.process[index].id === id) {
          break;
        }
      }
      if (index === 0) return;
      // otherwise swap with previous one
      const prev = this.process[index - 1];
      const cur = this.process[index];
      this.process.splice(index - 1, 2, cur, prev);
    },

    moveDown(id) {
      let index = 0;
      for (; index < this.process.length; index++) {
        if (this.process[index].id === id) {
          break;
        }
      }
      // otherwise swap with latter one
      const cur = this.process[index];
      const next = this.process[index + 1]
      this.process.splice(index, 2, next, cur);
    },
  }
}

function loadSubscription(options, sub) {
  options = {
    ...options,
    name: sub.name,
    url: sub.url,
  }
  let process = []

  // flag
  for (const p of (sub.process || [])) {
    switch (p.type) {
      case 'Useless Filter':
        options.useless = "REMOVE";
        break
      case 'Set Property Operator':
        options[p.args.key] = p.args.value ? "FORCE_OPEN" : "FORCE_CLOSE";
        break
      default:
        p.id = uuidv4();
        process.push(p);
    }
  }
  return {options, process};
}

function buildSubscription(options, process) {
  const sub = {
    name: options.name,
    url: options.url,
    process: []
  };
  // useless filter
  if (options.useless === 'REMOVE') {
    sub.process.push({
      type: "Useless Filter"
    });
  }
  // udp, tfo, scert
  for (const opt of ['udp', 'tfo', 'scert']) {
    if (options[opt] !== 'DEFAULT') {
      sub.process.push({
        type: "Set Property Operator",
        args: {key: opt, value: options[opt] === 'FORCE_OPEN'}
      });
    }
  }
  for (const p of process) {
    sub.process.push(p);
  }
  return sub;
}

function uuidv4() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
    var r = Math.random() * 16 | 0, v = c == 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
  });
}
</script>

<style>
.v-label {
  font-size: small;
}
</style>