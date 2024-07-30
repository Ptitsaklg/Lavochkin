<template>
  <v-container>
    <v-row>
      <v-col cols="12">
        <v-btn @click="setModeHistory" :outlined="!isOnlineMode" color="primary" class="mr-6">
          История
        </v-btn>
        <v-btn @click="setModeOnline" :outlined="isOnlineMode" color="primary">
          Онлайн
        </v-btn>
      </v-col>
    </v-row>
      
    <v-row v-if="!isOnlineMode">
      <v-col class="v-col-lg-6">
        <v-card>
          <Line 
            :options="chartOptions" 
            :data="getChartDataCPU" 
          />
        </v-card>
      </v-col>
      <v-col class="v-col-lg-6">
        <v-card>
          <Line 
            :options="chartOptions" 
            :data="getChartDataFreeMemory" 
          />
        </v-card>
      </v-col>
      <v-col class="v-col-lg-6">
        <v-card>
          <Line 
            :options="chartOptions" 
            :data="getChartDataFreeDiskSpace" 
          />
        </v-card>
      </v-col>
      <v-col class="v-col-lg-6">
        <v-row>
          <v-col>
            <v-text-field
              v-model="startTime"
              label="Время С"
              type="time"
              step="1"
              @change="updateStartTime"
              style="min-width: 100px;"
            />
          </v-col>
          <v-col>
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
          <v-col class="text-right">
            <v-btn
              @click="resetTime"
              color="primary"
            >
              Сбросить
            </v-btn>
          </v-col>
        </v-row>
      </v-col>
    </v-row>

    <v-row v-if="isOnlineMode">
      <v-col class="v-col-lg-6">
        <v-card>
          <Line 
            :options="chartOptions" 
            :data="getChartDataCPUOnline" 
          />
        </v-card>
      </v-col>
      <v-col class="v-col-lg-6">
        <v-card>
          <Line 
            :options="chartOptions" 
            :data="getChartDataFreeMemoryOnline" 
          />
        </v-card>
      </v-col>
      <v-col class="v-col-lg-6">
        <v-card>
          <Line 
            :options="chartOptions" 
            :data="getChartDataFreeDiskSpaceOnline" 
          />
        </v-card>
      </v-col>
    </v-row>

    <v-row>
      <v-col class="v-col-lg-12">
        <v-card>
          <v-card-title>Данные сервера</v-card-title>
          <v-data-table
            :headers="tableHeaders"
            :items="serverData"
            class="elevation-1"
          >
            <template v-slot:item.timestamp="{ item }">
              {{ formatDate(item.timestamp) }}
            </template>
          </v-data-table>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import apiClient from "@/utils/axiosConfig";
import { ref, computed, onMounted, onUnmounted } from "vue";
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

const tableHeaders = ref([
  { title: "Время", value: "timestamp" },
  { title: "Загрузка процессора", value: "cpuLoad" },
  { title: "Свободная память", value: "freeMemory" },
  { title: "Свободное место на диске", value: "freeDiskSpace" },
]);

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

const formatDate = (timestamp) => {
  const date = new Date(timestamp);
  return date.toLocaleString('ru-RU', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  });
};

const getChartDataCPU = computed(() => (
    {
      labels: chartLabels.value,
      datasets: [
        {
          label: "Загрузка процессора",
          borderColor: '#1b8525',
          data: chartCPU.value,
          pointStyle: 'circle',
          pointRadius: 6,
          pointHoverRadius: 10
        },
      ],
    }
));

const getChartDataFreeMemory = computed(() => (
    {
      labels: chartLabels.value,
      datasets: [
        {
          label: "Свободная память",
          borderColor: '#175c8d',
          data: chartFreeMemory.value,
          pointStyle: 'circle',
          pointRadius: 6,
          pointHoverRadius: 10
        },
      ],
    }
));

const getChartDataFreeDiskSpace = computed(() => (
    {
      labels: chartLabels.value,
      datasets: [
        {
          label: "Свободное место на диске",
          borderColor: '#f5ab18',
          data: chartFreeDiskSpace.value,
          pointStyle: 'circle',
          pointRadius: 6,
          pointHoverRadius: 10
        },
      ],
    }
));

const chartOptions = computed(() => (
    {
      responsive: true,
      plugins: {
        legend: {
          position: 'top',
        },
        title: {
          display: true,
          text: 'Данные с сервера',
        },
      },
    }
));


// Генератор данных для режима Онлайн
const generateData = (numPoints) => {
  const data = [];
  const startTimestamp = new Date();
  startTimestamp.setMilliseconds(0); // Устанавливаем миллисекунды в 0 для единообразия

  for (let i = 0; i < numPoints; i++) {
    const timestamp = new Date(startTimestamp.getTime() + i * 1000).toISOString();
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

const setModeOnline = () => {
  isOnlineMode.value = true;
  onlineData.value = generateData(10).data; // Генерация начальных данных
  updateOnlineCharts(onlineData.value);

  intervalId = setInterval(() => {
    const newData = generateData(1).data[0];
    onlineData.value.push(newData);
    if (onlineData.value.length > 10) { // Ограничиваем до 10 последних точек данных
      onlineData.value.shift();
    }
    updateOnlineCharts(onlineData.value);
  }, 1000);
};

const setModeHistory = () => {
  isOnlineMode.value = false;
  if (intervalId) {
    clearInterval(intervalId);
    intervalId = null;
  }
};

onMounted(() => {
  getServerData();
});

onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId);
  }
});
</script>


<style scoped>

</style>