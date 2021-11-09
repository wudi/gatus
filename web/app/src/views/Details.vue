<template>
  <router-link to="../"
               class="absolute top-2 left-2 inline-block px-2 pb-0.5 text-lg text-black bg-gray-100 rounded hover:bg-gray-200 focus:outline-none border border-gray-200 dark:bg-gray-700 dark:text-gray-200 dark:border-gray-500 dark:hover:bg-gray-600">
    &larr;
  </router-link>
  <div>
    <slot v-if="endpointStatus">
      <h1 class="text-xl xl:text-3xl font-mono text-gray-400">RECENT CHECKS</h1>
      <hr class="mb-4"/>
      <Endpoint
          :data="endpointStatus"
          :maximumNumberOfResults="20"
          @showTooltip="showTooltip"
          @toggleShowAverageResponseTime="toggleShowAverageResponseTime"
          :showAverageResponseTime="showAverageResponseTime"
      />
      <Pagination @page="changePage"/>
    </slot>
    <div v-if="endpointStatus && endpointStatus.key" class="mt-12">
      <h1 class="text-xl xl:text-3xl font-mono text-gray-400">UPTIME</h1>
      <hr/>
      <div class="flex space-x-4 text-center text-2xl mt-6 relative bottom-2 mb-10">
        <div class="flex-1">
          <h2 class="text-sm text-gray-400 mb-1">Last 7 days</h2>
          <img :src="generateUptimeBadgeImageURL('7d')" alt="7d uptime badge" class="mx-auto" />
        </div>
        <div class="flex-1">
          <h2 class="text-sm text-gray-400 mb-1">Last 24 hours</h2>
          <img :src="generateUptimeBadgeImageURL('24h')" alt="24h uptime badge" class="mx-auto" />
        </div>
        <div class="flex-1">
          <h2 class="text-sm text-gray-400 mb-1">Last hour</h2>
          <img :src="generateUptimeBadgeImageURL('1h')" alt="1h uptime badge" class="mx-auto" />
        </div>
      </div>
    </div>
    <div v-if="endpointStatus && endpointStatus.key" class="mt-12">
      <h1 class="text-xl xl:text-3xl font-mono text-gray-400">RESPONSE TIME</h1>
      <hr/>

      <LineChart :chartData="chartData" :options="chartOptions" />

      <img :src="generateResponseTimeChartImageURL()" alt="response time chart" class="mt-6" />
      <div class="flex space-x-4 text-center text-2xl mt-6 relative bottom-2 mb-10">
        <div class="flex-1">
          <h2 class="text-sm text-gray-400 mb-1">Last 7 days</h2>
          <img :src="generateResponseTimeBadgeImageURL('7d')" alt="7d response time badge" class="mx-auto mt-2" />
        </div>
        <div class="flex-1">
          <h2 class="text-sm text-gray-400 mb-1">Last 24 hours</h2>
          <img :src="generateResponseTimeBadgeImageURL('24h')" alt="24h response time badge" class="mx-auto mt-2" />
        </div>
        <div class="flex-1">
          <h2 class="text-sm text-gray-400 mb-1">Last hour</h2>
          <img :src="generateResponseTimeBadgeImageURL('1h')" alt="1h response time badge" class="mx-auto mt-2" />
        </div>
      </div>
    </div>
    <div v-if="endpointStatus && endpointStatus.key">
      <h1 class="text-xl xl:text-3xl font-mono text-gray-400 mt-4">EVENTS</h1>
      <hr class="mb-4"/>
      <div>
        <slot v-for="event in events" :key="event">
          <div class="p-3 my-4">
            <h2 class="text-lg">
              <img v-if="event.type === 'HEALTHY'" src="../assets/arrow-up-green.png" alt="Healthy"
                   class="border border-green-600 rounded-full opacity-75 bg-green-100 mr-2 inline" width="26"/>
              <img v-else-if="event.type === 'UNHEALTHY'" src="../assets/arrow-down-red.png" alt="Unhealthy"
                   class="border border-red-500 rounded-full opacity-75 bg-red-100 mr-2 inline" width="26"/>
              <img v-else-if="event.type === 'START'" src="../assets/arrow-right-black.png" alt="Start"
                   class="border border-gray-500 rounded-full opacity-75 bg-gray-100 mr-2 inline" width="26"/>
              {{ event.fancyText }}
            </h2>
            <div class="flex mt-1 text-sm text-gray-400">
              <div class="flex-1 text-left pl-10">
                {{ new Date(event.timestamp).toISOString() }}
              </div>
              <div class="flex-1 text-right">
                {{ event.fancyTimeAgo }}
              </div>
            </div>
          </div>
        </slot>
      </div>
    </div>
  </div>
  <Settings @refreshData="fetchData"/>
</template>


<script>
import Settings from '@/components/Settings.vue'
import Endpoint from '@/components/Endpoint.vue';
import {SERVER_URL} from "@/main.js";
import {helper} from "@/mixins/helper.js";
import Pagination from "@/components/Pagination";

import { LineChart } from 'vue-chart-3';
import { Chart, CategoryScale, LineController, LinearScale, LineElement, PointElement, TimeScale, Title, Tooltip} from 'chart.js';
Chart.register(LineController, CategoryScale, LinearScale, LineElement, PointElement, TimeScale, Title, Tooltip);


export default {
  name: 'Details',
  components: {
    Pagination,
    Endpoint,
    Settings,
    LineChart,
  },
  emits: ['showTooltip'],
  mixins: [helper],
  methods: {
    fetchData() {
      // XXX: This should probably be called every 15 minutes or so
      //console.log("[Details][fetchData] Fetching data");
      fetch(`${this.serverUrl}/api/v1/endpoints/${this.$route.params.key}/statuses?page=${this.currentPage}`)
          .then(response => response.json())
          .then(data => {
            if (JSON.stringify(this.endpointStatus) !== JSON.stringify(data)) {
              this.endpointStatus = data;
              this.uptime = data.uptime;
              let events = [];
              for (let i = data.events.length - 1; i >= 0; i--) {
                let event = data.events[i];
                if (i === data.events.length - 1) {
                  if (event.type === 'UNHEALTHY') {
                    event.fancyText = 'Endpoint is unhealthy';
                  } else if (event.type === 'HEALTHY') {
                    event.fancyText = 'Endpoint is healthy';
                  } else if (event.type === 'START') {
                    event.fancyText = 'Monitoring started';
                  }
                } else {
                  let nextEvent = data.events[i + 1];
                  if (event.type === 'HEALTHY') {
                    event.fancyText = 'Endpoint became healthy';
                  } else if (event.type === 'UNHEALTHY') {
                    if (nextEvent) {
                      event.fancyText = 'Endpoint was unhealthy for ' + this.prettifyTimeDifference(nextEvent.timestamp, event.timestamp);
                    } else {
                      event.fancyText = 'Endpoint became unhealthy';
                    }
                  } else if (event.type === 'START') {
                    event.fancyText = 'Monitoring started';
                  }
                }
                event.fancyTimeAgo = this.generatePrettyTimeAgo(event.timestamp);
                events.push(event);
              }
              this.events = events;
            }
            this.fetchUptimeChartData();
          });
    },
    fetchUptimeChartData() {
      fetch(`${this.serverUrl}/api/v1/endpoints/${this.$route.params.key}/response-times/24h`).then(response => {
        response.json().then(data => {
          let chart = {
            labels: [],
            datasets: [
              {
                label: 'Average response time (ms)',
                data: [],
                borderColor: 'rgb(75, 192, 192)',
              },
            ]
          };
          let latest = null;
          for (const [key] of Object.entries(data)) {
              latest = key;
          }
          for (let i = 24; i >= 0; i--) {
            let date = new Date((latest*1000)-(i*3600000));
            chart.labels.push(date.toLocaleTimeString().replaceAll(":00", ""));
            if (data[date.getTime()/1000]) {
              chart.datasets[0].data.push(data[date.getTime()/1000]);
            } else {
              chart.datasets[0].data.push(0);
            }
          }
          this.chartData = chart;
        })
      });
    },
    generateUptimeBadgeImageURL(duration) {
      return `${this.serverUrl}/api/v1/endpoints/${this.endpointStatus.key}/uptimes/${duration}/badge.svg`;
    },
    generateResponseTimeBadgeImageURL(duration) {
      return `${this.serverUrl}/api/v1/endpoints/${this.endpointStatus.key}/response-times/${duration}/badge.svg`;
    },
    generateResponseTimeChartImageURL() {
      return `${this.serverUrl}/api/v1/endpoints/${this.endpointStatus.key}/response-times/24h/chart.svg`;
    },
    prettifyUptime(uptime) {
      if (!uptime) {
        return '0%';
      }
      return (uptime * 100).toFixed(2) + '%'
    },
    prettifyTimeDifference(start, end) {
      let minutes = Math.ceil((new Date(start) - new Date(end)) / 1000 / 60);
      return minutes + (minutes === 1 ? ' minute' : ' minutes');
    },
    changePage(page) {
      this.currentPage = page;
      this.fetchData();
    },
    showTooltip(result, event) {
      this.$emit('showTooltip', result, event);
    },
    toggleShowAverageResponseTime() {
      this.showAverageResponseTime = !this.showAverageResponseTime;
    },
  },
  data() {
    return {
      endpointStatus: {},
      uptime: {},
      events: [],
      hourlyAverageResponseTime: {},
      // Since this page isn't at the root, we need to modify the server URL a bit
      serverUrl: SERVER_URL === '.' ? '..' : SERVER_URL,
      currentPage: 1,
      showAverageResponseTime: true,
      chartData: {labels: [], datasets: [{data: []}]},
      chartOptions: {
        scales: {
          y: {
            min: 0,
            title: {
              display: true,
              text: 'Average response time (ms)'
            }
          }
        }
      }
    }
  },
  created() {
    this.fetchData();
  }
}
</script>

<style scoped>
.endpoint {
  border-radius: 3px;
  border-bottom-width: 3px;
}
</style>