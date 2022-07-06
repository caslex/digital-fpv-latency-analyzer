<script setup lang="ts"></script>

<template>
  <main>
    <div class="appContainer">
      <div class="videoCol">
        <div
          class="dropSection"
          @drop.prevent.stop="dropped"
          @dragenter.prevent.stop
          @dragover.prevent.stop
        >
          Drag and Drop your DJI/Avatar SRT, OpenTX CSV and MP4 files here.
        </div>
        <div class="videoWrapper">
          <video ref="video"></video>
          <div class="videoSubtitles">{{ currentSubtitle }}</div>
        </div>
        <button class="btn btn-primary me-2" @click="playVideo">Play</button>
        <button class="btn btn-secondary" @click="pauseVideo">Pause</button>
        <br />
        OpenTX Offset [s]: <input type="number" v-model.number="openTxOffset" />
        <hr />
        <h3>Statistics</h3>
        <pre>
avg latency: {{ avgLatency }}
median latency: {{ medLatency }}
stdev latency: {{ stdevLatency }}
avg bitrate: {{ avgBitrate }}
median bitrate: {{ medBitrate }}
stdev bitrate: {{ stdevBitrate }}
        </pre>
        <small>by caslex</small>
      </div>
      <div class="dataCol">
        <div class="chart" ref="chartContainer">
          <vue3-chart-js
            ref="chart"
            v-bind="{ ...lineChart }"
            @afterRender="afterRender"
          />
          <div class="scrub" :style="scrubStyle" @mousedown="startScrub"></div>
        </div>
        <h3>Delay Scatter</h3>
        <vue3-chart-js
          ref="delayScatterChart"
          v-bind="{ ...delayScatterChart }"
        />
        <h3>Delay Histogram</h3>
        <vue3-chart-js
          ref="delayHistogramChart"
          v-bind="{ ...delayHistogramChart }"
        />
      </div>
    </div>
  </main>
</template>

<script>
import Vue3ChartJs from "@j-t-mcc/vue3-chartjs";
import "chartjs-adapter-date-fns";
import zoomPlugin from "chartjs-plugin-zoom";
import parse from "date-fns/parse";
import format from "date-fns/format";
Vue3ChartJs.registerGlobalPlugins([zoomPlugin]);

export default {
  data() {
    const zoomOpts = {
      pan: {
        enabled: true,
      },
      zoom: {
        wheel: {
          enabled: true,
        },
        pinch: {
          enabled: true,
        },
        mode: "y",
      },
    };
    return {
      subtitles: {},
      subtitleTimes: [],
      videoInterval: null,
      currentSubtitle: "",
      openTxData: [],
      openTxOffset: 0,
      firstInit: true,
      scrubStyle: {
        inset: "0 auto 100% 0",
      },
      lineChart: {
        id: "dataChart",
        type: "line",
        data: {
          labels: [],
          datasets: [],
        },
        options: {
          animation: false,
          ticks: {
            source: "data",
          },
          scales: {
            x: {
              type: "time",
              parsing: false,
              time: {
                tooltipFormat: "t",
                displayFormats: {
                  millisecond: "t",
                  second: "t",
                  minute: "t",
                  hour: "t",
                  day: "t",
                  week: "t",
                  month: "t",
                  quarter: "t",
                  year: "t",
                },
              },
            },
            y: {
              type: "logarithmic",
            },
          },
          plugins: {
            zoom: {
              ...zoomOpts,
            },
          },
        },
      },
      delayScatterChart: {
        id: "delayScatterChart",
        type: "scatter",
        data: {
          labels: [],
          datasets: [],
        },
        options: {
          animation: false,
          plugins: {},
          scales: {
            y: {
              type: "logarithmic",
            },
          },
        },
      },
      delayHistogramChart: {
        id: "delayHistogramChart",
        type: "bar",
        data: {
          labels: [],
          datasets: [],
        },
        options: {
          animation: false,
          plugins: {},
          scales: {
            y: {
              type: "logarithmic",
            },
          },
        },
      },
    };
  },
  computed: {
    avgLatency() {
      const latencies = Object.values(this.subtitles).map((s) => s.data.delay);
      return this.mean(latencies);
    },
    medLatency() {
      const latencies = Object.values(this.subtitles).map((s) => s.data.delay);
      return this.median(latencies);
    },
    stdevLatency() {
      const latencies = Object.values(this.subtitles).map((s) => s.data.delay);
      return this.standardDeviation(latencies);
    },
    avgBitrate() {
      const bitrates = Object.values(this.subtitles).map((s) => s.data.bitrate);
      return this.mean(bitrates);
    },
    medBitrate() {
      const bitrates = Object.values(this.subtitles).map((s) => s.data.bitrate);
      return this.median(bitrates);
    },
    stdevBitrate() {
      const bitrates = Object.values(this.subtitles).map((s) => s.data.bitrate);
      return this.standardDeviation(bitrates);
    },
  },
  components: {
    Vue3ChartJs,
  },
  watch: {
    /*subtitles: {
      deep: true,
      handler() {
        this.handleDataChange();
      },
    },
    openTxData: {
      deep: true,
      handler() {
        this.handleDataChange();
      },
    },*/
    openTxOffset: {
      deep: true,
      handler() {
        this.handleDataChange();
      },
    },
  },
  mounted() {
    this.videoInterval = window.setInterval(() => {
      const currentTime = this.$refs.video.currentTime;
      const subtitle = this.findSubtitle(currentTime);
      if (subtitle !== null) {
        this.currentSubtitle = subtitle.text;
      }

      const xPixel = Math.max(
        this.$refs.chart?.chartJSState?.chart?.scales?.x.getPixelForValue(
          currentTime * 1000
        ),
        0
      );
      const top = this.$refs.chart?.chartJSState?.chart?.scales?.y?.top;
      const bottom =
        this.$refs.chart?.chartJSState?.chart?.height -
        this.$refs.chart?.chartJSState?.chart?.scales?.y?.bottom;
      this.scrubStyle.inset = `${top}px auto ${bottom}px ${xPixel}px`;
    }, 100);
    document.onkeydown = this.checkKey;
  },
  beforeUnmount() {
    if (this.videoInterval !== null) {
      window.clearInterval(this.videoInterval);
      this.videoInterval = null;
    }
    document.onkeydown = null;
  },
  methods: {
    handleDataChange() {
      let shownByDefault = ["delay", "bitrate", "distance"];
      if (!this.firstInit) {
        shownByDefault = this.$refs.chart.chartJSState.chart.legend.legendItems
          .filter((l) => l.hidden === false)
          .map((l) => l.text);
      }
      this.firstInit = false;

      this.lineChart.data.datasets.splice(0);
      this.lineChart.data.labels.splice(0);
      const colors = [
        "#000000",
        "#004949",
        "#009292",
        "#ff6db6",
        "#ffb6db",
        "#490092",
        "#006ddb",
        "#b66dff",
        "#6db6ff",
        "#b6dbff",
        "#920000",
        "#924900",
        "#db6d00",
        "#24ff24",
        "#ffff6d",
      ];
      let subtitleDuration = null;

      if (this.subtitleTimes.length > 0) {
        subtitleDuration = Math.max(...this.subtitleTimes);
        const dataProps = Object.keys(
          this.subtitles[this.subtitleTimes[0]].data
        );

        const datasets = [];
        for (const [i, dataProp] of dataProps.entries()) {
          datasets.push({
            label: dataProp,
            borderColor: colors[i % (colors.length - 1)],
            hidden: !shownByDefault.includes(dataProp),
            data: this.subtitleTimes.map((s) => ({
              x: this.subtitles[s].startSeconds * 1000,
              y: this.subtitles[s].data[dataProp],
            })),
          });
        }

        this.lineChart.data.datasets.push(...datasets);
        this.lineChart.data.labels.push(...this.subtitleTimes);

        // Delay data
        const delays = Object.values(this.subtitles)
          .map((s) => s.data.delay)
          .sort((a, b) => a - b);

        this.delayScatterChart.data.datasets.splice(0);
        this.delayScatterChart.data.datasets.push({
          label: "Delay",
          data: delays.map((delay, idx) => ({
            x: idx,
            y: delay,
          })),
        });
        this.$refs.delayScatterChart.update();

        // Histogram
        const numBuckets = 50;
        const maxDelay = delays[delays.length - 1];
        const bucketSize = Math.ceil(maxDelay / numBuckets);
        const buckets = [];
        const bucketLabels = [];
        for (let i = 0; i < numBuckets; i++) {
          buckets.push(0);
          if (bucketSize <= 1) {
            bucketLabels.push(`${i * bucketSize}`);
          } else {
            bucketLabels.push(`${i * bucketSize}-${(i + 1) * bucketSize - 1}`);
          }
        }
        for (const delay of delays) {
          let i;
          for (i = 0; i < numBuckets; i++) {
            if (delay < (i + 1) * bucketSize) {
              break;
            }
          }
          buckets[i]++;
        }
        this.delayHistogramChart.data.labels = bucketLabels;
        this.delayHistogramChart.data.datasets.splice(0);
        this.delayHistogramChart.data.datasets.push({
          label: "# of delays",
          data: buckets,
        });
        this.$refs.delayHistogramChart.update();
      }

      if (this.openTxData.length > 0) {
        const datasets = [];
        const headers = Object.keys(this.openTxData[0].raw);

        for (const [idxHeader, header] of headers.entries()) {
          const entries = [];
          for (const openTxData of this.openTxData) {
            const timeSinceStart =
              openTxData.timeSinceStart + this.openTxOffset;
            if (timeSinceStart < 0) continue;
            if (subtitleDuration !== null) {
              if (timeSinceStart > subtitleDuration) break;
            }
            entries.push({
              x: timeSinceStart * 1000,
              y: openTxData.raw[header],
            });
          }
          const dataName = `TX ${header}`;
          datasets.push({
            label: dataName,
            borderColor: colors[idxHeader % (colors.length - 1)],
            hidden: !shownByDefault.includes(dataName),
            data: entries,
          });
        }

        this.lineChart.data.datasets.push(...datasets);
        this.lineChart.data.labels.push(...headers);
      }

      this.$refs.chart.update(1);
    },
    mean(arr) {
      const tot = arr.reduce((a, b) => a + b, 0);
      return tot / arr.length || 0;
    },
    median(arr) {
      if (arr.length === 0) return 0;
      arr.sort((a, b) => a - b);

      return arr[Math.floor(arr.length / 2)];
    },
    standardDeviation(arr) {
      const mean = this.mean(arr);
      return Math.sqrt(
        arr.map((x) => Math.pow(x - mean, 2)).reduce((a, b) => a + b, 0) /
          arr.length
      );
    },
    checkKey(event) {
      if (event.keyCode === 37) {
        this.$refs.video.currentTime -= 0.005;
      } else if (event.keyCode === 39) {
        this.$refs.video.currentTime += 0.005;
      }
    },
    afterRender(event) {
      if (event?.chartRef?.value?.chartJSState?.chart?.scales?.x) {
        this.scrubStyle.bottom =
          event?.chartRef?.value?.chartJSState?.chart?.scales?.x.bottom;
      }
    },
    async dropped(dropEvent) {
      const numFiles = dropEvent?.dataTransfer?.files?.length;
      if (numFiles > 0) {
        const promises = [];
        for (let i = 0; i < numFiles; i++) {
          const file = dropEvent.dataTransfer.files[i];
          if (this.getExtension(file.name) === "mp4") {
            this.readVideo(file);
          } else if (this.getExtension(file.name) === "srt") {
            promises.push(this.readSubtitle(file));
          } else if (this.getExtension(file.name) === "csv") {
            promises.push(this.readOpenTx(file));
          }
        }
        await Promise.all(promises).then(() => {
          this.handleDataChange();
        });
      }
    },
    readVideo(file) {
      const reader = new FileReader();
      reader.onloadend = (loadendEvent) => {
        if (loadendEvent.target.readyState === FileReader.DONE) {
          this.$refs.video.setAttribute(
            "src",
            (window.URL || window.webkitURL).createObjectURL(file)
          );
        }
      };
      reader.readAsDataURL(file);
    },
    readSubtitle(file) {
      return new Promise((resolve) => {
        const reader = new FileReader();
        reader.onloadend = (loadendEvent) => {
          if (loadendEvent.target.readyState === FileReader.DONE) {
            this.parseSubtitle(reader);
            resolve();
          }
        };
        reader.readAsText(file);
      });
    },
    readOpenTx(file) {
      return new Promise((resolve) => {
        const reader = new FileReader();
        reader.onloadend = (loadendEvent) => {
          if (loadendEvent.target.readyState === FileReader.DONE) {
            this.parseOpenTx(reader);
            resolve();
          }
        };
        reader.readAsText(file);
      });
    },
    parseSubtitle(reader) {
      this.subtitles = {};
      this.subtitleTimes = [];
      const subtitleEntries = reader.result.split("\n\n");
      for (const subtileEntry in subtitleEntries) {
        const subtitleParts = subtitleEntries[subtileEntry].split("\n");
        if (subtitleParts.length >= 2) {
          const timeParts = subtitleParts[1].split(" --> ");
          const start = timeParts[0];
          const end = timeParts[1];
          let text = subtitleParts[2];
          if (subtitleParts.length > 2) {
            for (let j = 3; j < subtitleParts.length; j++) {
              text += "\n" + subtitleParts[j];
            }
          }
          const startSeconds = this.timeToSeconds(start);

          const isAvatar = text.indexOf("Distance") >= 0;
          const isDji = text.indexOf("rcSignal") >= 0;
          let data = null;
          if (isAvatar || isDji) {
            if (isAvatar) {
              // Avatar
              const regex =
                /Signal:(\d*) CH:(\d*) FightTime:(\d*) SBat:(\d*\.?\d*)V GBat:(\d*\.?\d*)V Delay:(\d*)ms Bitrate:(\d*\.?\d*)Mbps Distance:(\d*)m/gm;
              const match = regex.exec(text);
              data = {
                type: "avatar",
                signal: match[1],
                ch: match[2],
                flightTime: match[3],
                uavBat: match[4],
                glsBat: match[5],
                uavBatCells: null,
                glsBatCells: null,
                delay: parseFloat(match[6]),
                bitrate: parseFloat(match[7]),
                rcSignal: null,
                distance: match[8],
              };
            } else if (isDji) {
              const regex =
                /signal:(\d*) ch:(\d*) flightTime:(\d*) uavBat:(\d*\.?\d*)V glsBat:(\d*\.?\d*)V?%? uavBatCells:(\d*) glsBatCells:(\d*) delay:(\d*)ms bitrate:(\d*\.?\d*)Mbps rcSignal:(\d*)/gm;
              const match = regex.exec(text);
              if (match !== null) {
                data = {
                  type: "dji",
                  signal: match[1],
                  ch: match[2],
                  flightTime: match[3],
                  uavBat: match[4],
                  glsBat: match[5],
                  uavBatCells: match[6],
                  glsBatCells: match[7],
                  delay: parseFloat(match[8]),
                  bitrate: parseFloat(match[9]),
                  rcSignal: match[10],
                  distance: null,
                };
              }
            }
          }

          this.subtitles[startSeconds] = {
            start,
            end,
            startSeconds,
            text,
            data,
          };
          this.subtitleTimes.push(startSeconds);
        }
      }
      this.subtitleTimes.sort((a, b) => a - b);
    },
    parseOpenTx(reader) {
      const csvLines = reader.result.split("\n");
      const headerLine = csvLines.shift();
      const headers = headerLine.split(",");
      const openTxData = [];
      let firstStart = null;
      for (const [idxCsvLine, csvLine] of csvLines.entries()) {
        const csvData = csvLine.split(",");
        if (csvData.length !== headers.length) continue;
        const data = {
          raw: {},
        };

        for (const [idx, header] of headers.entries()) {
          data.raw[header] = csvData[idx];
        }

        // Set relative time
        const entryTime = format(
          parse(
            `${data.raw["Date"]} ${data.raw["Time"]}`,
            "yyyy-MM-dd HH:mm:ss.SSS",
            new Date()
          ),
          "t"
        );
        if (idxCsvLine === 0) {
          data.timeSinceStart = 0;
          firstStart = entryTime;
        } else {
          data.timeSinceStart = entryTime - firstStart;
        }

        openTxData.push(data);
      }
      this.openTxData = openTxData;
    },
    findSubtitle(seconds) {
      let subtitle = null;
      for (const subtitleSeconds of this.subtitleTimes) {
        if (subtitleSeconds > seconds) {
          break;
        }
        subtitle = this.subtitles[subtitleSeconds] ?? null;
      }
      return subtitle;
    },
    getExtension(fileName) {
      const parts = fileName.split(".");
      return parts[parts.length - 1];
    },
    timeToSeconds(timeString) {
      let seconds = 0.0;
      const parts = timeString.split(":");
      for (let i = 0; i < parts.length; i++) {
        seconds = seconds * 60 + parseFloat(parts[i].replace(",", "."));
      }
      return seconds;
    },
    strip(s) {
      return s.replace(/^\s+|\s+$/g, "");
    },
    playVideo() {
      this.$refs.video.play();
    },
    pauseVideo() {
      this.$refs.video.pause();
    },
    startScrub() {
      document.onmouseup = this.stopScrub;
      document.onmousemove = this.scrub;
    },
    scrub(e) {
      const mouseX = e.clientX - this.$refs.chartContainer.offsetLeft;
      this.$refs.video.currentTime =
        this.$refs.chart.chartJSState.chart.scales.x.getValueForPixel(mouseX) /
        1000;
    },
    stopScrub() {
      document.onmouseup = null;
      document.onmousemove = null;
    },
  },
};
</script>

<style lang="scss">
.appContainer {
  width: 100%;
  display: flex;
}
.dropSection {
  padding: 2rem;
  background-color: #eee;
  border: 5px dashed #ccc;
  text-align: center;
  margin-bottom: 1rem;
}
video {
  max-width: 100%;
  height: auto;
}
.videoWrapper {
  position: relative;
  max-width: 600px;
  .videoSubtitles {
    position: absolute;
    text-align: center;
    left: 0;
    right: 0;
    bottom: 20px;
    background-color: #000;
    color: #fff;
  }
}
.videoCol {
  flex: 1;
}
.dataCol {
  flex: 2;
}
.chart {
  position: relative;
}
.scrub {
  position: absolute;
  background-color: red;
  width: 2px;
  cursor: col-resize;
}
</style>
