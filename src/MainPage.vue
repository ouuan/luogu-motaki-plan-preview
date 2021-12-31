<template>
  <n-space
    id="main"
    vertical
  >
    <n-page-header title="Luogu Motaki 计划预览" />

    <n-card title="Repositories">
      <n-space>
        <template
          v-for="repo of repos"
          :key="repo"
        >
          <n-a :href="`https://github.com/ouuan/${repo}`">
            {{ repo }}
          </n-a>
        </template>
      </n-space>
    </n-card>

    <n-card title="方案">
      <n-space vertical>
        <n-radio-group
          v-model:value="type"
          name="type"
        >
          <n-space>
            <n-radio value="server">
              服务器
            </n-radio>
            <n-radio value="file">
              文件
            </n-radio>
          </n-space>
        </n-radio-group>

        <div v-if="type === 'server'">
          <n-text>服务器地址（不带 <code>/file</code> 等路径</n-text>
          <n-input
            v-model:value="serverUrl"
            type="text"
            placeholder="https://example.com"
            :on-change="onUrlChanged"
          />
        </div>

        <div v-else>
          <input
            ref="fileInput"
            type="file"
            name="motaki-plan"
            accept="application/json"
            @change="onFileChanged"
          >
        </div>
      </n-space>
    </n-card>

    <n-card
      v-show="ok"
      title="预览"
    >
      <n-space justify="space-around">
        <canvas
          ref="canvas"
          :width="WIDTH"
          :height="HEIGHT"
        />
      </n-space>
    </n-card>

    <n-card
      v-if="ok"
      :title="type === 'server' ? '选择以添加至客户端（不选相当于全选）' : '任务列表'"
    >
      <n-space vertical>
        <n-data-table
          :data="tableData"
          :columns="tableColumns"
          :row-key="(row) => row.name"
          @update:checked-row-keys="updateCheckedKeys"
        />
        <n-p v-if="type === 'server'">
          复制这一行添加到客户端配置文件即可：
          <br>
          <n-space>
            <pre>{{ clientConfig }}</pre>
            <n-button
              type="primary"
              @click="copyClientConfig"
            >
              复制
            </n-button>
          </n-space>
        </n-p>
      </n-space>
    </n-card>
  </n-space>
</template>

<script setup lang="ts">
import {
  DataTableColumns,
  NA,
  NButton,
  NCard,
  NDataTable,
  NInput,
  NP,
  NPageHeader,
  NRadio,
  NRadioGroup,
  NSpace,
  NText,
  useMessage,
} from 'naive-ui';
import {
  computed,
  onMounted,
  ref,
  watch,
} from 'vue';
import axios from 'axios';
import { isPlan, validatePlan } from './validatePlan';
import { HEIGHT, palette, WIDTH } from './constants';

const type = ref<'server' | 'file'>('server');
const serverUrl = ref('');
const repos = ['luogu-motaki', 'luogu-motaki-plan-editor', 'luogu-motaki-plan-preview', 'luogu-motaki-server', 'luogu-motaki-client'];
const ok = ref(false);
const fileInput = ref<HTMLInputElement | null>(null);
const canvas = ref<HTMLCanvasElement | null>(null);
const msg = useMessage();
const tableData = ref<Array<{
  name: string;
  leftTop: string;
  rightBottom: string;
  size: string;
}
>>([]);
const columns: DataTableColumns = [
  {
    title: '名称',
    key: 'name',
  }, {
    title: '左上坐标',
    key: 'leftTop',
  }, {
    title: '右下坐标',
    key: 'rightBottom',
  }, {
    title: '大小',
    key: 'size',
  },
];
const tableColumns = computed(() => {
  if (type.value === 'file') return columns;
  return ([{ type: 'selection' }] as DataTableColumns).concat(columns);
});

function processPlan(plan: unknown) {
  if (typeof plan !== 'object' || plan === null || !isPlan(plan)) {
    msg.error('Invalid plan format');
    return;
  }
  const validate = validatePlan(plan);
  if (validate !== true) {
    msg.error(`Invalid plan: ${validate}`);
  }
  tableData.value = [];
  const ctx = canvas.value?.getContext('2d');
  if (!ctx) return;
  const imageData = ctx.createImageData(WIDTH, HEIGHT);
  Object.entries(plan).forEach(([name, { x, y, data }]) => {
    const lines = data.split('\n');
    const width = lines.length;
    const height = lines[0].length;
    tableData.value.push({
      name,
      leftTop: `(${x}, ${y})`,
      rightBottom: `(${x + width - 1}, ${y + height - 1})`,
      size: `${width} x ${height} = ${width * height}`,
    });
    for (let dx = 0; dx < width; dx += 1) {
      for (let dy = 0; dy < height; dy += 1) {
        const colorIndex = parseInt(lines[dx][dy], 32);
        const index = (x + dx) + (y + dy) * WIDTH;
        for (let i = 0; i < 3; i += 1) {
          imageData.data[index * 4 + i] = palette[colorIndex][i];
        }
        imageData.data[index * 4 + 3] = 255;
      }
    }
  });
  ctx.putImageData(imageData, 0, 0);
  ok.value = true;
}

function onFileChanged() {
  ok.value = false;
  const file = fileInput.value?.files?.[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = () => {
      const { result } = reader;
      if (typeof result === 'string') {
        const plan = JSON.parse(result);
        processPlan(plan);
      }
    };
    reader.readAsText(file);
  }
}

function onUrlChanged() {
  ok.value = false;
  if (serverUrl.value.length < 3) {
    msg.error('Server URL too short');
    return;
  }
  axios.get(`${serverUrl.value.trim()}/plan`)
    .then((res) => {
      if (res.status === 200) {
        const { data } = res;
        processPlan(data);
      }
    }).catch((err) => {
      msg.error(err.toString());
    });
}

watch([type], () => {
  ok.value = false;
  if (type.value === 'file') {
    if (fileInput.value?.files?.length) {
      onFileChanged();
    }
  } else if (serverUrl.value) {
    onUrlChanged();
  }
});

onMounted(() => {
  const query = new URLSearchParams(window.location.search);
  const server = query.get('server');
  if (server) {
    serverUrl.value = server;
    onUrlChanged();
  }
});

const checkedKeys = ref<(string|number)[]>([]);

function updateCheckedKeys(newKeys: (string|number)[]) {
  checkedKeys.value = newKeys;
}

const clientConfig = computed(() => {
  const selectedTasks = checkedKeys.value.map((key) => key.toString());
  return `${serverUrl.value} ${selectedTasks.join(' ')}`;
});

function copyClientConfig() {
  navigator.clipboard.writeText(clientConfig.value)
    .then(() => { msg.success('复制成功'); })
    .catch((err) => msg.error(`复制失败: ${err}`));
}

</script>

<style scoped>
#main {
  max-width: 80%;
  margin: auto;
  margin-top: 20px;
}

canvas {
  border: 2px solid grey;
}
</style>
