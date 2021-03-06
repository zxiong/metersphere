<template>
  <ms-container>
    <ms-main-container>
      <el-card v-loading="result.loading">
        <el-row>
          <el-col :span="16">
            <el-row>
              <el-breadcrumb separator-class="el-icon-arrow-right">
                <el-breadcrumb-item :to="{ path: '/performance/test/' + this.projectId }">{{ projectName }}
                </el-breadcrumb-item>
                <el-breadcrumb-item :to="{ path: '/performance/test/edit/' + this.testId }">{{ testName }}
                </el-breadcrumb-item>
                <el-breadcrumb-item>{{ reportName }}</el-breadcrumb-item>
              </el-breadcrumb>
            </el-row>
            <el-row class="ms-report-view-btns">
              <el-button :disabled="isReadOnly || report.status !== 'Running'" type="primary" plain size="mini"
                         @click="dialogFormVisible=true">
                {{ $t('report.test_stop_now') }}
              </el-button>
              <el-button :disabled="isReadOnly || report.status !== 'Completed'" type="success" plain size="mini"
                         @click="rerun(testId)">
                {{ $t('report.test_execute_again') }}
              </el-button>
              <!--<el-button :disabled="isReadOnly" type="info" plain size="mini">
                {{$t('report.export')}}
              </el-button>
              <el-button :disabled="isReadOnly" type="warning" plain size="mini">
                {{$t('report.compare')}}
              </el-button>-->
            </el-row>
          </el-col>
          <el-col :span="8">
            <span class="ms-report-time-desc">
              {{ $t('report.test_duration', [this.minutes, this.seconds]) }}
            </span>
            <span class="ms-report-time-desc">
              {{ $t('report.test_start_time') }}：{{ startTime }}
            </span>
            <span class="ms-report-time-desc">
              {{ $t('report.test_end_time') }}：{{ endTime }}
            </span>
          </el-col>
        </el-row>

        <el-divider/>

        <el-tabs v-model="active" type="border-card" :stretch="true">
          <el-tab-pane :label="$t('load_test.pressure_config')">
            <ms-performance-pressure-config :is-read-only="true" :report="report"/>
          </el-tab-pane>
          <el-tab-pane :label="$t('report.test_overview')">
            <ms-report-test-overview :report="report" ref="testOverview"/>
          </el-tab-pane>
          <el-tab-pane :label="$t('report.test_request_statistics')">
            <ms-report-request-statistics :report="report"/>
          </el-tab-pane>
          <el-tab-pane :label="$t('report.test_error_log')">
            <ms-report-error-log :report="report"/>
          </el-tab-pane>
          <el-tab-pane :label="$t('report.test_log_details')">
            <ms-report-log-details :report="report"/>
          </el-tab-pane>
        </el-tabs>

      </el-card>
      <el-dialog :title="$t('report.test_stop_now_confirm')" :visible.sync="dialogFormVisible" width="30%">
        <p v-html="$t('report.force_stop_tips')"/>
        <p v-html="$t('report.stop_tips')"/>
        <div slot="footer" class="dialog-footer">
          <el-button type="danger" size="small" @click="stopTest(true)">{{ $t('report.force_stop_btn') }}
          </el-button>
          <el-button type="primary" size="small" @click="stopTest(false)">{{ $t('report.stop_btn') }}
          </el-button>
        </div>
      </el-dialog>
    </ms-main-container>
  </ms-container>
</template>

<script>
import MsReportErrorLog from './components/ErrorLog';
import MsReportLogDetails from './components/LogDetails';
import MsReportRequestStatistics from './components/RequestStatistics';
import MsReportTestOverview from './components/TestOverview';
import MsPerformancePressureConfig from "./components/PerformancePressureConfig";
import MsContainer from "../../common/components/MsContainer";
import MsMainContainer from "../../common/components/MsMainContainer";

import {checkoutTestManagerOrTestUser} from "@/common/js/utils";

export default {
  name: "PerformanceReportView",
  components: {
    MsReportErrorLog,
    MsReportLogDetails,
    MsReportRequestStatistics,
    MsReportTestOverview,
    MsContainer,
    MsMainContainer,
    MsPerformancePressureConfig
  },
  data() {
    return {
      result: {},
      active: '1',
      reportId: '',
      status: '',
      reportName: '',
      testId: '',
      testName: '',
      projectId: '',
      projectName: '',
      startTime: '0',
      endTime: '0',
      minutes: '0',
      seconds: '0',
      title: 'Logging',
      report: {},
      isReadOnly: false,
      websocket: null,
      dialogFormVisible: false,
      testPlan: {testResourcePoolId: null}
    }
  },
  methods: {
    initBreadcrumb() {
      if (this.reportId) {
        this.result = this.$get("/performance/report/test/pro/info/" + this.reportId, res => {
          let data = res.data;
          if (data) {
            this.reportName = data.name;
            this.testId = data.testId;
            this.testName = data.testName;
            this.projectId = data.projectId;
            this.projectName = data.projectName;
          }
        })
      }
    },
    initReportTimeInfo() {
      if (this.reportId) {
        this.result = this.$get("/performance/report/content/report_time/" + this.reportId)
          .then(res => {
            let data = res.data.data;
            if (data) {
              this.startTime = data.startTime;
              this.endTime = data.endTime;
              let duration = data.duration;
              this.minutes = Math.floor(duration / 60);
              this.seconds = duration % 60;
            }
          }).catch(() => {
            this.clearData();
          })
      }
    },
    initWebSocket() {
      let protocol = "ws://";
      if (window.location.protocol === 'https:') {
        protocol = "wss://";
      }
      const uri = protocol + window.location.host + "/performance/report/" + this.reportId;
      this.websocket = new WebSocket(uri);
      this.websocket.onmessage = this.onMessage;
      this.websocket.onopen = this.onOpen;
      this.websocket.onerror = this.onError;
      this.websocket.onclose = this.onClose;
    },
    checkReportStatus(status) {
      switch (status) {
        case 'Error':
          this.$warning(this.$t('report.generation_error'));
          break;
        case 'Starting':
          this.$alert(this.$t('report.start_status'));
          break;
        case 'Reporting':
        case 'Running':
        case 'Completed':
        default:
          break;
      }
    },
    clearData() {
      this.startTime = '0';
      this.endTime = '0';
      this.minutes = '0';
      this.seconds = '0';
    },
    stopTest(forceStop) {
      this.result = this.$get('/performance/stop/' + this.reportId + '/' + forceStop, () => {
        this.$success(this.$t('report.test_stop_success'));
        if (forceStop) {
          this.$router.push('/performance/report/all');
        } else {
          this.report.status = 'Completed';
        }
      });
      this.dialogFormVisible = false;
    },
    rerun(testId) {
      this.$confirm(this.$t('report.test_rerun_confirm'), '', {
        confirmButtonText: this.$t('commons.confirm'),
        cancelButtonText: this.$t('commons.cancel'),
        type: 'warning'
      }).then(() => {
        this.result = this.$post('/performance/run', {id: testId, triggerMode: 'MANUAL'}, (response) => {
          this.reportId = response.data;
          this.$router.push({path: '/performance/report/view/' + this.reportId});
          // 注册 socket
          this.initWebSocket();
        })
      }).catch(() => {
      });
    },
    onOpen() {
      window.console.log("socket opening.");
    },
    onError(e) {
      window.console.error(e)
    },
    onMessage(e) {
      this.$set(this.report, "refresh", e.data); // 触发刷新
      this.$set(this.report, "status", 'Running');
      this.initReportTimeInfo();
      window.console.log('receive a message:', e.data);
    },
    onClose(e) {
      this.$set(this.report, "refresh", Math.random()); // 触发刷新
      this.$set(this.report, "status", 'Completed');
      this.initReportTimeInfo();
      window.console.log("socket closed.");
    }
  },
  created() {
    this.isReadOnly = false;
    if (!checkoutTestManagerOrTestUser()) {
      this.isReadOnly = true;
    }
    this.reportId = this.$route.path.split('/')[4];
    this.result = this.$get("/performance/report/" + this.reportId, res => {
      let data = res.data;
      if (data) {
        this.status = data.status;
        this.$set(this.report, "id", this.reportId);
        this.$set(this.report, "status", data.status);
        this.$set(this.report, "testId", data.testId);
        this.$set(this.report, "loadConfiguration", data.loadConfiguration);
        this.checkReportStatus(data.status);
        if (this.status === "Completed" || this.status === "Running") {
          this.initReportTimeInfo();
        }
        this.initBreadcrumb();
        this.initWebSocket();
      } else {
        this.$error(this.$t('report.not_exist'))
      }
    });

  },
  beforeDestroy() {
    this.websocket.close() //离开路由之后断开websocket连接
  },
  watch: {
    '$route'(to) {
      if (to.name === "perReportView") {
        this.isReadOnly = false;
        if (!checkoutTestManagerOrTestUser()) {
          this.isReadOnly = true;
        }
        let reportId = to.path.split('/')[4];
        this.reportId = reportId;
        if (reportId) {
          this.$get("/performance/report/test/pro/info/" + reportId, response => {
            let data = response.data;
            if (data) {
              this.status = data.status;
              this.reportName = data.name;
              this.testName = data.testName;
              this.testId = data.testId;
              this.projectName = data.projectName;

              this.$set(this.report, "id", reportId);
              this.$set(this.report, "status", data.status);

              this.checkReportStatus(data.status);
              if (this.status === "Completed") {
                this.result = this.$get("/performance/report/content/report_time/" + this.reportId).then(res => {
                  let data = res.data.data;
                  if (data) {
                    this.startTime = data.startTime;
                    this.endTime = data.endTime;
                    let duration = data.duration;
                    this.minutes = Math.floor(duration / 60);
                    this.seconds = duration % 60;
                  }
                }).catch(() => {
                  this.clearData();
                })
              } else {
                this.clearData();
              }
            } else {
              this.$error(this.$t('report.not_exist'));
            }
          });

        }
      }
    }
  }
}
</script>

<style scoped>

.ms-report-view-btns {
  margin-top: 15px;
}

.ms-report-time-desc {
  text-align: left;
  display: block;
  color: #5C7878;
}

</style>
