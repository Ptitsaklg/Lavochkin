<template>
  <v-container>
    <v-row>
      <v-col cols="12" class="d-flex justify-space-between align-center">
        <div>
          <v-btn @click="setModeHistory" :outlined="!isOnlineMode" color="primary" class="mr-6">
            История
          </v-btn>
          <v-btn @click="setModeOnline" :outlined="isOnlineMode" color="primary">
            Онлайн
          </v-btn>
        </div>
        <v-switch
          v-if="!isOnlineMode"
          v-model="isDark"
          color="black"
          @click="toggleTheme"
          label="Темная тема"
        ></v-switch>
      </v-col>
    </v-row>
      
    <v-row v-if="!isOnlineMode">
      <v-col class="v-col-lg-6">
        <v-card>
          <div class="chart-container">
            <Line 
              ref="chartInstance"
              :options="chartOptions" 
              :data="getCombinedChartData" 
              @click="handleChartClick"
            />
          </div>
          <v-row>
            <v-col class="ma-3">
              <v-text-field
                v-model="startTime"
                label="Время С"
                type="time"
                step="1"
                @change="updateStartTime"
                style="min-width: 100px;"
              />
            </v-col>
            <v-col class="ma-3">
              <v-text-field
                v-model="endTime"
                label="Время По"
                type="time"
                step="1"
                @change="updateEndTime"
                style="min-width: 100px;"
              />
            </v-col>
          </v-row>
          <v-row>
            <v-col class="text-right mb-3 mr-3">
              <v-btn
                @click="resetTime"
                color="primary"
              >
                Сбросить
              </v-btn>
            </v-col>
          </v-row>
        </v-card>
      </v-col>

      <v-col class="v-col-lg-6">
        <v-card>
          <v-card-title>Данные сервера</v-card-title>
          <v-data-table
            :headers="tableHeaders"
            :items="serverData"
            class="elevation-1"
            :items-per-page-text="'Элементов на странице:'"
            :items-per-page-all-text="'Все'"
            :page-text="'{0}-{1} из {2}'"
          >
            <template v-slot:item.timestamp="{ item }">
              {{ formatDate(item.timestamp) }}
            </template>
          </v-data-table>
        </v-card>
      </v-col>
    </v-row>


    <v-row v-if="isOnlineMode">
      <v-col class="v-col-lg-6">
        <v-card>
          <div class="chart-container">
            <Line 
              :options="chartOptionsOnline" 
              :data="getChartDataCPUOnline" 
            />
        </div>
        </v-card>
      </v-col>
      <v-col class="v-col-lg-6">
        <v-card>
          <div class="chart-container">
            <Line 
              :options="chartOptionsOnline" 
              :data="getChartDataFreeMemoryOnline" 
            />
        </div>
        </v-card>
      </v-col>
      <v-col class="v-col-lg-6">
        <v-card>
          <div class="chart-container">
            <Line 
              :options="chartOptionsOnline" 
              :data="getChartDataFreeDiskSpaceOnline" 
            />
        </div>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import apiClient from "@/utils/axiosConfig";
import { useTheme } from 'vuetify';
import { ref, computed, onMounted, onUnmounted} from "vue";
import {Line} from "vue-chartjs";
import {
  BarElement,
  CategoryScale,
  Chart as ChartJS,
  Legend,
  LinearScale,
  LineElement,
  PointElement,
  Title,
  Tooltip
} from "chart.js";

ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, PointElement, LineElement)

const startTime = ref('00:00:00');
const endTime = ref('23:59:59');
const theme = useTheme();
const isDark = ref(false);

const serverData = ref([]);
const chartLabels = ref([]);
const chartCPU = ref([]);
const chartFreeMemory = ref([]);
const chartFreeDiskSpace = ref([]);

const onlineData = ref([]);
const chartLabelsOnline = ref([]);
const chartCPUOnline = ref([]);
const chartFreeMemoryOnline = ref([]);
const chartFreeDiskSpaceOnline = ref([]);
const isOnlineMode = ref(false);
let intervalId = null;

const selectedRowIndex = ref(null);
const chartInstance = ref(null);


function toggleTheme () {
  theme.global.name.value = theme.global.current.value.dark ? 'light' : 'dark'
}

const applyTheme = () => {
  if (!isOnlineMode.value) {
    theme.global.name.value = isDark.value ? 'dark' : 'light';
  } else {
    theme.global.name.value = 'light';
  }
};

const setModeOnline = () => {
  isOnlineMode.value = true;
  applyTheme();

  if (intervalId) {
    clearInterval(intervalId);
  }

  if (onlineData.value.length === 0) {
    const baseTime = new Date();
    baseTime.setMilliseconds(0);
    onlineData.value = generateData(10, new Date(baseTime.getTime() - 9000)).data;
  }

  updateOnlineCharts(onlineData.value);

  intervalId = setInterval(() => {
    const currentTime = new Date();
    currentTime.setMilliseconds(0);
    const newData = generateData(1, currentTime).data[0];
    onlineData.value.push(newData);

    if (onlineData.value.length > 10) {
      onlineData.value.shift();
    }

    updateOnlineCharts(onlineData.value);
  }, 1000);
};


const setModeHistory = () => {
  isOnlineMode.value = false;
  applyTheme();

  if (intervalId) {
    clearInterval(intervalId);
    intervalId = null;
  }
};

const updateStartTime = () => {
  filterData();
};

const updateEndTime = () => {
  filterData();
};

const filterData = () => {
  if (!startTime.value || !endTime.value) return;

  const [startHours, startMinutes, startSeconds] = startTime.value.split(':').map(Number);
  const [endHours, endMinutes, endSeconds] = endTime.value.split(':').map(Number);

  const startTimestamp = startHours * 3600 + startMinutes * 60 + startSeconds;
  const endTimestamp = endHours * 3600 + endMinutes * 60 + endSeconds;

  const filteredData = serverData.value.filter(item => {
    const date = new Date(item.timestamp);
    const itemTime = date.getHours() * 3600 + date.getMinutes() * 60 + date.getSeconds();
    return itemTime >= startTimestamp && itemTime <= endTimestamp;
  });

  updateChartData(filteredData);
};

const updateChartData = (data) => {
  chartLabels.value = data.map(item => formatDate(item.timestamp));
  chartCPU.value = data.map(item => item.cpuLoad);
  chartFreeMemory.value = data.map(item => item.freeMemory);
  chartFreeDiskSpace.value = data.map(item => item.freeDiskSpace);
};

const resetTime = () => {
  startTime.value = '00:00:00';
  endTime.value = '23:59:59';
  filterData();
};

const formatDate = (timestamp) => {
  const date = new Date(timestamp);
  return date.toLocaleTimeString();
};

const tableHeaders = ref([
  { title: "Время", value: "timestamp" },
  { title: "Загрузка процессора, %", value: "cpuLoad" },
  { title: "Свободная память, *100 МБ", value: "freeMemory" },
  { title: "Свободное место на диске, *100 ГБ", value: "freeDiskSpace" },
]);


// Имитация получения данных с сервера
const getServerData = async () => {
  try {
    const response = await apiClient.get('/server-data.json');
    serverData.value = response.data.data;

    const labels = [];
    const cpuData = [];
    const freeMemoryData = [];
    const freeDiskSpaceData = [];

    serverData.value.forEach((item) => {
      labels.push(formatDate(item.timestamp));
      cpuData.push(item.cpuLoad); 
      freeMemoryData.push(item.freeMemory); 
      freeDiskSpaceData.push(item.freeDiskSpace); 
    });

    chartLabels.value = labels;
    chartCPU.value = cpuData;
    chartFreeMemory.value = freeMemoryData;
    chartFreeDiskSpace.value = freeDiskSpaceData;
  } catch (error) {
    console.error("Ошибка при загрузке данных:", error);
  }
};

const getCombinedChartData = computed(() => ({
  labels: chartLabels.value,
  datasets: [
    {
      label: 'Загрузка процессора, %',
      borderColor: '#42A5F5',
      backgroundColor: '#42A5F5',
      fill: false,
      data: chartCPU.value,
    },
    {
      label: 'Свободная память, *100 МБ',
      borderColor: '#66BB6A',
      backgroundColor: '#66BB6A',
      fill: false,
      data: chartFreeMemory.value,
    },
    {
      label: 'Свободное место на диске, *100 ГБ',
      borderColor: '#FFCA28',
      backgroundColor: '#FFCA28',
      fill: false,
      data: chartFreeDiskSpace.value,
    },
  ],
}));

const chartOptions = computed(() => (
    {
      responsive: true,
      plugins: {
        legend: {
          position: 'top',
        },
      },
    }
));

// Для выделения строки в таблице при клике на точку на графике 
const handleChartClick = (event) => {
  const chart = chartInstance.value.chart;
  const points = chart.getElementsAtEventForMode(event, 'nearest', { intersect: true }, true);
  if (points.length) {
    const index = points[0].index;
    selectedRowIndex.value = index;
    highlightRow(index);
  }
};

const highlightRow = (index) => {
  const rows = document.querySelectorAll('tbody .v-data-table__tr');
  if (index >= 0 && index < rows.length) {
    rows.forEach((row) => {
      row.classList.remove('highlighted-row');
    });
    rows[index].classList.add('highlighted-row');
  }
};


// Генератор данных для режима Онлайн
const generateData = (numPoints, startTime = new Date()) => {
  const data = [];
  startTime.setMilliseconds(0);

  for (let i = 0; i < numPoints; i++) {
    const timestamp = new Date(startTime.getTime() + i * 1000).toISOString();
    const cpuLoad = Math.floor(Math.random() * 100);
    const freeMemory = Math.floor(Math.random() * 3000);
    const freeDiskSpace = Math.floor(Math.random() * 50000);
    data.push({
      timestamp,
      cpuLoad,
      freeMemory,
      freeDiskSpace,
    });
  }

  return { data };
};

const updateOnlineCharts = (data) => {
  const labels = data.map(item => formatDate(item.timestamp));
  const cpuLoadData = data.map(item => item.cpuLoad);
  const freeMemoryData = data.map(item => item.freeMemory);
  const freeDiskSpaceData = data.map(item => item.freeDiskSpace);

  chartLabelsOnline.value = labels;
  chartCPUOnline.value = cpuLoadData;
  chartFreeMemoryOnline.value = freeMemoryData;
  chartFreeDiskSpaceOnline.value = freeDiskSpaceData;
};

const getChartDataCPUOnline = computed(() => (
    {
    labels: chartLabelsOnline.value,
    datasets: [
      {
        label: 'Загрузка процессора',
        backgroundColor: '#42A5F5',
        data: chartCPUOnline.value,
      },
    ],
  }
));

const getChartDataFreeMemoryOnline = computed(() => (
    {
    labels: chartLabelsOnline.value,
    datasets: [
      {
        label: 'Свободная память',
        backgroundColor: '#66BB6A',
        data: chartFreeMemoryOnline.value,
      },
    ],
  }
));

const getChartDataFreeDiskSpaceOnline = computed(() => (
    {
    labels: chartLabelsOnline.value,
    datasets: [
      {
        label: 'Свободное место на диске',
        backgroundColor: '#FFA726',
        data: chartFreeDiskSpaceOnline.value,
      },
    ],
  }
));

const chartOptionsOnline = computed(() => (
    {
      responsive: true,
      animation: {
        duration: 10000,
      },
      plugins: {
        legend: {
          position: 'top',
        },

      },
    }
));


onMounted(() => {
  getServerData();
});

onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId);
  }
});
</script>


<style lang="sass">
.chart-container
  position: relative
  width: 100%
  height: 0
  padding-bottom: 50%

.highlighted-row
  background-color: #d3d3d3
</style>