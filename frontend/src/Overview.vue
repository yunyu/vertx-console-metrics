<template>
    <div class="container-fluid pf-card-grid" v-if="metrics && Object.keys(metrics).length > 0">
        <deploy-verticle ref="deployVerticle"></deploy-verticle>
        <div class="row">
            <div class="col-md-3">
                <pf-aggregate-status-card title="Deployed Verticles" :count="getSimpleMetricValue('vertx_verticles')" iconClass="fa fa-cubes">
                    <a @click="$refs.deployVerticle.displayModal()" class="add" style="cursor: pointer">
                        <span class="pficon pficon-add-circle-o"></span>
                    </a>
                </pf-aggregate-status-card>
            </div>
            <div class="col-md-3">
                <pf-aggregate-status-card title="Open Connections" :count="getSimpleMetricValue('.*_open_connections_.*', true)" iconClass="fa fa-exchange">
                    <span class="pficon pficon-ok"></span>
                </pf-aggregate-status-card>
            </div>
            <div class="col-md-3">
                <pf-aggregate-status-card :title="`Event Loops, ${simpleFormattedData('vertx_event_loop_size').value} Max`" :count="simpleFormattedData('vertx_event_loop_threads').value" iconClass="fa fa-microchip
                ">
                    <span>{{ simpleFormattedData('os_avail_processors').value }} Processors</span>
                </pf-aggregate-status-card>
            </div>
            <div class="col-md-3">
                <pf-aggregate-status-card :count="'Up for ' + uptime" iconClass="fa fa-clock-o">
                    <span>{{ getSimpleMetricValue('os_load_average').toFixed(2) }} Load Avg</span>
                </pf-aggregate-status-card>
            </div>
            <div :class="getColumnClass(1)">
                <pf-card title="Java Heap" :accented="false" :showTitlesSeparator="false">
                    <pf-util-trend labelType="used" donutColor="#EC7A08" :data="javaHeapUsage"></pf-util-trend>
                </pf-card>
            </div>
            <div :class="getColumnClass(1)">
                <pf-card class="match-util-trend" title="System Load" :accented="false" :showTitlesSeparator="false">
                    <pf-trend labelType="used" title="CPU Usage" :data="cpuUsage"></pf-trend>
                    <div class="pf-body-separator"></div>
                    <pf-trend labelType="used" title="HTTP Reqs/sec" :data="avgRequestsPerSecond"></pf-trend>
                    <div class="pf-body-separator"></div>
                    <div class="pf-card-section">
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Nonheap Used" :data="nonHeapUsage"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Collections Run" :data="gcCount"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Total GC Time" :data="gcTotal"></pf-trend-details>
                        </div>
                    </div>
                </pf-card>
            </div>
            <div :class="getColumnClass(2)">
                <pf-card class="match-util-trend" title="HTTP Response Times (ms) " :accented="false" :showTitlesSeparator="false">
                    <pf-stacked-bar :height="288" :data="httpRequests"></pf-stacked-bar>
                </pf-card>
            </div>
            <div :class="getColumnClass(1)">
                <pf-card class="match-util-trend" title="Resources" :accented="false" :showTitlesSeparator="false">
                    <pf-utilization-bar-chart title='Worker Pool' :formatFn="n => n + ' threads'" :value="getSimpleMetricValue('vertx_pools_worker_vert_x_worker_thread_in_use')" :total="getSimpleMetricValue('vertx_pools_worker_vert_x_worker_thread_max_pool_size')" :warning="60" :error="85"></pf-utilization-bar-chart>
                    <!-- <pf-utilization-bar-chart title='Open Files' units='FDs' :value="parseInt(getSimpleMetricValue('process_open_fds'))" :total="parseInt(getSimpleMetricValue('process_max_fds'))" inline :warning="60" :error="85"></pf-utilization-bar-chart> -->
                    <pf-utilization-bar-chart title='Disk Storage' :formatFn="diskUsage.formatFn" :value="diskUsage.used" :total="diskUsage.total" :warning="60" :error="85"></pf-utilization-bar-chart>
                    <div class="pf-body-separator"></div>
                    <div class="pf-card-section">
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Threads Started" :data="simpleFormattedData('jvm_threads_started_total', '0[.]0a')"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Active Threads" :data="simpleFormattedData('jvm_threads_current', '0[.]0a')"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Deadlock Waits" :data="simpleFormattedData('jvm_threads_deadlocked')"></pf-trend-details>
                        </div>
                    </div>
                    <div class="pf-body-separator noline"></div>
                    <div class="pf-card-section">
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Daemon Threads" :data="simpleFormattedData('jvm_threads_daemon')"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Loaded Classes" :data="simpleFormattedData('jvm_classes_loaded', '0[.]0a')"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Total Unloads" :data="simpleFormattedData('jvm_classes_unloaded_total')"></pf-trend-details>
                        </div>
                    </div>
                </pf-card>
            </div>
            <div :class="getColumnClass(1)">
                <pf-card class="match-util-trend" title="Event Bus" :accented="false" :showTitlesSeparator="false">
                    <pf-trend labelType="used" title="Msg Pubs/sec" :data="eventBusMessagesPublishedPerSecond"></pf-trend>
                    <div class="pf-body-separator"></div>
                    <pf-trend labelType="used" title="Active Handlers" :data="simpleFormattedData('vertx_eventbus_handlers', '0[.]0a')"></pf-trend>
                    <div class="pf-body-separator"></div>
                    <div class="pf-card-section">
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Msgs Delivered" :data="simpleFormattedData('vertx_eventbus_messages_delivered_total', '0[.]0a')"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Msgs Published" :data="simpleFormattedData('vertx_eventbus_messages_published_total', '0[.]0a')"></pf-trend-details>
                        </div>
                        <div class="col-sm-4 col-md-4">
                            <pf-trend-details title="Reply Failures" :data="simpleFormattedData('vertx_eventbus_messages_reply_failures_total', '0[.]0a')"></pf-trend-details>
                        </div>
                    </div>
                </pf-card>
            </div>
            <div :class="getColumnClass(2)">
                <pf-card class="match-util-trend" title="HTTP Requests/sec History" :accented="false" :showTitlesSeparator="false">
                    <pf-multi-line :height="288" :data="httpRequestsRates" chartType="area"></pf-multi-line>
                </pf-card>
            </div>
        </div>
    </div>
    <div v-else></div>
</template>

<style lang="scss">
.pf-card-grid {
    padding: 20px 30px 5px;
}

.match-util-trend .card-pf-body {
    height: 308px;
}

.match-util-trend-title .card-pf-body {
    height: 287px;
}

.card-pf-body .pf-body-separator {
    height: 1px;
    background: #d1d1d1;
    margin-top: 24px;
    margin-bottom: 24px;
}

.card-pf-body .pf-body-separator.noline {
    background: transparent;
    margin-top: 8px;
    margin-bottom: 8px;
}

@media (min-width: 1200px) and (max-width: 1600px) {
    .card-pf-body {
        .pf-body-separator {
            margin-top: 16px;
            margin-bottom: 16px;
        }

        .progress {
            margin-bottom: 16px;
        }

        .pf-body-separator.noline {
            margin-top: 6px;
            margin-bottom: 6px;
        }
    }
}

@media (min-width: 992px) and (max-width: 1060px) {
    .pf-card-grid .card-pf.card-pf-aggregate-status {
        height: 104px;
    }
}

.card-pf-aggregate-status-notifications {
    font-size: 20px;
}

.pf-card-section {
    overflow: hidden;
}

.pf-card-section>div {
    padding: 0;
    padding-right: 5px;
}
</style>

<script>
import numeral from 'numeral';
import formatSeconds from './formatseconds.js';

import Metrics from './metrics.js';
Metrics.initialize('/metrics');

import DeployVerticle from './DeployVerticle.vue';

export default {
    name: 'Overview',
    components: {
        'deploy-verticle': DeployVerticle
    },
    data() {
        return {
            requestedMetrics: [],
            metrics: null,
            deployModalDisplayed: false
        }
    },
    beforeMount() {
        this.numeral = numeral;
        this.lastUpdated = -1;
        this.updateRateInSeconds = 1;

        // Necessary for reactivity
        this.metricsCallback = metrics => {
            this.metrics = metrics;
            // console.log(JSON.stringify(this.metrics, null, 4));
            const prevUpdated = this.lastUpdated;
            this.lastUpdated = performance.now();

            if (this.lastUpdated !== -1) {
                this.updateRateInSeconds = (this.lastUpdated - prevUpdated) / 1000;
            }
        };
        Metrics.addCallback(this.metricsCallback);

        this.columnClasses = [];
        this.regexNameCache = {};
    },
    beforeDestroy() {
        Metrics.removeCallback(this.metricsCallback);
    },
    methods: {
        getColumnClass(width) {
            let existingValue = this.columnClasses[width];
            if (existingValue === undefined) {
                existingValue = 'col-sm-12 col-md-' + 6 * width + ' col-lg-' + 3 * width;
                this.columnClasses[width] = existingValue;
            }
            return existingValue;
        },
        getMetricByName(name, isRegex) {
            if (isRegex) {
                const cachedKey = this.regexNameCache[name];
                if (cachedKey !== undefined) {
                    const v = this.metrics[cachedKey];
                    if (v !== undefined) {
                        return v;
                    } else {
                        delete this.regexNameCache[name];
                    }
                }
                let regex = new RegExp(name);
                for (let [k, v] of Object.entries(this.metrics)) {
                    if (regex.test(k)) {
                        this.regexNameCache[name] = k;
                        return v;
                    }
                }
                return undefined;
            } else {
                return this.metrics[name]
            }
        },
        getSimpleMetricValue(name, isRegex) {
            return parseFloat(this.getMetricByName(name, isRegex).metrics.value);
        },
        simpleFormattedData(key, format) {
            let formatFn;
            if (format) {
                formatFn = n => numeral(n).format(format);
            }
            return {
                value: this.getSimpleMetricValue(key),
                formatFn: formatFn
            }
        }
    },
    // TODO switch to configurable computed props
    computed: {
        diskUsage() {
            return {
                used: this.getSimpleMetricValue('disk_space_bytes_used'),
                total: this.getSimpleMetricValue('disk_space_bytes_max'),
                formatFn: n => numeral(n).format('0.0 b')
            }
        },
        javaHeapUsage() {
            return {
                used: parseFloat(this.getMetricByName('jvm_memory_bytes_used').metrics.area.heap.value),
                total: parseFloat(this.getMetricByName('jvm_memory_bytes_max').metrics.area.heap.value),
                formatFn: n => numeral(n).format('0.0 b')
            }
        },
        gcStats() {
            const gcMetrics = this.getMetricByName('jvm_gc_collection_seconds').metrics.gc;
            let count = 0;
            let sum = 0;
            for (let [k, v] of Object.entries(gcMetrics)) {
                if (v.count) {
                    count += parseFloat(v.count);
                }
                if (v.sum) {
                    sum += parseFloat(v.sum);
                }
            }
            return { count: count, sum: sum };
        },
        gcTotal() {
            return {
                value: this.gcStats.sum,
                formatFn: n => numeral(n).format('0') + ' sec'
            };
        },
        gcCount() {
            return {
                value: this.gcStats.count,
                formatFn: n => numeral(n).format('0[.]0a')
            };
        },
        cpuUsage() {
            return {
                value: this.getSimpleMetricValue('os_system_cpu_load'),
                formatFn: n => numeral(n).format('0.0 %')
            }
        },
        nonHeapUsage() {
            return {
                value: parseFloat(this.getMetricByName('jvm_memory_bytes_used').metrics.area.nonheap.value),
                formatFn: n => numeral(n).format('0.0 b')
            }
        },
        httpRequests() {
            const requestsMetric = this.getMetricByName('vertx_http_servers_.*:\\d+_requests', true);
            const requestsDistMetric = this.getMetricByName('vertx_http_servers_.*_\\d+_requests_dist', true);

            const perc50 = parseFloat(requestsMetric.metrics.quantiles['0.5']);
            const perc95 = parseFloat(requestsMetric.metrics.quantiles['0.95']);
            const perc99 = parseFloat(requestsMetric.metrics.quantiles['0.99']);
            return {
                indices: [new Date()],
                values: {
                    '50th': numeral(perc50 * 1000).format('0.0'),
                    '95th': numeral(perc95 * 1000).format('0.0'),
                    '99th': numeral(perc99 * 1000).format('0.0')
                },
                colors: {
                    '50th': '#7cc2e8',
                    '95th': '#f9d67a',
                    '99th': '#f39c3d'
                },
                baseline: {
                    value: numeral(requestsDistMetric.metrics.stat.mean.value * 1000).format('0.0'),
                    text: 'Overall average'
                }
            }
        },
        httpRequestsRates() {
            const requestsRateMetric = this.getMetricByName('vertx_http_servers_.*_\\d+_requests_rate', true);

            const oneMin = parseFloat(requestsRateMetric.metrics.interval['oneMinute'].value);
            const fiveMin = parseFloat(requestsRateMetric.metrics.interval['fiveMinute'].value);
            const fifteenMin = parseFloat(requestsRateMetric.metrics.interval['fifteenMinute'].value);

            return {
                indices: [new Date()],
                values: {
                    '1 minute': numeral(oneMin).format('0.00'),
                    '5 minutes': numeral(fiveMin).format('0.00'),
                    '15 minutes': numeral(fifteenMin).format('0.00')
                }
            }
        },
        uptime() {
            return formatSeconds(Math.floor(Date.now() / 1e3 - this.getSimpleMetricValue('process_start_time_seconds')));
        },
        avgRequestsPerSecond() {
            const totalReqs = this.getMetricByName('vertx_http_servers_.*:\\d+_requests', true).metrics.count;
            let rps;
            if (this.lastTotalReqs === undefined) {
                rps = 0;
            } else {
                rps = Math.round((totalReqs - this.lastTotalReqs) / this.updateRateInSeconds);
            }
            this.lastTotalReqs = totalReqs;
            return {
                value: rps
            };
        },
        eventBusMessagesPublishedPerSecond() {
            const totalPublished = this.getSimpleMetricValue('vertx_eventbus_messages_published_total');
            let mpps;
            if (this.lastTotalPublished === undefined) {
                mpps = 0;
            } else {
                mpps = Math.round((totalPublished - this.lastTotalPublished) / this.updateRateInSeconds);
            }
            this.lastTotalPublished = totalPublished;
            return {
                value: mpps,
                formatFn: n => numeral(n).format('0[.]0a')
            }
        }
    }
}
</script>
