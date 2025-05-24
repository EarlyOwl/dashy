<template>
  <div class="pi-hole-stats-v6-wrapper">
    <!-- Current Status -->
    <div v-if="status && !hideStatus" class="status">
      <span class="status-lbl">{{ $t('widgets.pi-hole.status-heading') }}:</span>
      <span :class="`status-val ${getStatusColor(status)}`">{{ status | capitalize }}</span>
    </div>
    <!-- Block Pie Chart -->
    <p :id="chartId" class="block-pie"></p>
    <!-- More Data -->
    <div v-if="dataTable" class="data-table">
      <div class="data-table-row" v-for="(row, inx) in dataTable" :key="inx">
        <p class="row-label">{{ row.lbl }}</p>
        <p class="row-value">{{ row.val }}</p>
      </div>
    </div>
  </div>
</template>

<script>
// import axios from 'axios';
import WidgetMixin from '@/mixins/WidgetMixin';
import ChartingMixin from '@/mixins/ChartingMixin';
import { capitalize } from '@/utils/MiscHelpers';

export default {
  mixins: [WidgetMixin, ChartingMixin],
  data() {
    return {
      status: null,
      dataTable: null,
      csrfToken: null,
      sid: null,
    };
  },
  computed: {
    /* Let user select which comic to display: random, latest or a specific number */
    hostname() {
      const usersChoice = this.parseAsEnvVar(this.options.hostname);
      if (!usersChoice) this.error('You must specify the hostname for your Pi-Hole server');
      return usersChoice || 'http://pi.hole';
    },
    apiKey() {
      const usersChoice = this.parseAsEnvVar(this.options.apiKey);
      if (!usersChoice) this.error('App Password is required, please see the docs');
      return usersChoice;
    },
    endpoint() {
      return `${this.hostname}/api/stats/summary`;
    },
    authUrl() {
      return `${this.hostname}/api/auth`;
    },
    hideStatus() { return this.options.hideStatus; },
    hideChart() { return this.options.hideChart; },
    hideInfo() { return this.options.hideInfo; },
  },
  filters: {
    capitalize,
  },
  methods: {
    async fetchData() {
      try {
        if (!this.sid || !this.csrfToken) {
          await this.authenticate();
        }
        await this.fetchBlockingStatus();
        await this.queryStats();
      } catch (err) {
        this.error(`[Pi-hole] fetchData error: ${err.message}`);
      }
    },
    /* Make POST request to local pi-hole instance for authentication */
    async authenticate() {
      try {
        const res = await fetch(this.authUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          credentials: 'include',
          body: JSON.stringify({ password: this.apiKey }),
        });
        /* Error message for missing TOTP if 2FA enabled */
        if (!res.ok) {
          const errorData = await res.json();
          if (res.status === 400 && errorData.error?.message?.includes('No 2FA token found')) {
            throw new Error('2FA token missing: You must provide a TOTP token or use an app password');
          }
          throw new Error(`Auth failed: HTTP ${res.status}`);
        }
        /* Error message for failed auth */
        const { session } = await res.json();
        if (!session) throw new Error('Missing session info in auth response');
        if (session.valid !== true) {
          throw new Error('Authentication failed: Invalid credentials or 2FA token required');
        }
        /* Error message for missing CSRF token and/or SID in response */
        const { sid, csrf } = session;
        if (!sid || !csrf) throw new Error('No CSRF token or SID received');
        this.sid = sid;
        this.csrfToken = csrf;
      } catch (err) {
        this.error(`[Pi-hole] Authentication error: ${err.message}`);
        throw err;
      }
    },
    /* Make GET request to local pi-hole to fetch blocking status */
    async fetchBlockingStatus() {
      if (!this.sid) throw new Error('No SID available');

      const url = `${this.hostname}/api/dns/blocking`;
      const res = await fetch(url, {
        method: 'GET',
        headers: {
          'X-FTL-SID': this.sid,
          'X-FTL-CSRF': this.csrfToken,
          Accept: 'application/json',
        },
        credentials: 'include',
      });

      if (!res.ok) {
        throw new Error(`Blocking status fetch failed: HTTP ${res.status}`);
      }

      const data = await res.json();
      this.status = data.blocking || 'unknown';
    },
    /* Make GET request to local pi-hole to fetch stats */
    async queryStats() {
      try {
        const res = await fetch(this.endpoint, {
          method: 'GET',
          headers: {
            'X-FTL-SID': this.sid,
            'X-FTL-CSRF': this.csrfToken,
            Accept: 'application/json',
          },
          credentials: 'include',
        });

        if (!res.ok) {
          throw new Error(`Stats fetch failed: HTTP ${res.status}`);
        }

        const data = await res.json();
        this.processData(data);
      } catch (err) {
        this.error(`[Pi-hole] Stats query error: ${err.message}`);
        throw err;
      }
    },
    /* Assign data variables to the returned data */
    processData(data) {
      const {
        queries: { total, blocked, percent_blocked: blockedPercent },
        clients: { active, total: totalClients },
        gravity: { domains_being_blocked: blocklistSize },
      } = data;

      if (!this.hideInfo) {
        this.dataTable = [
          { lbl: 'Active Clients', val: `${active}/${totalClients}` },
          { lbl: 'Ads Blocked', val: blocked },
          { lbl: 'Total DNS Queries', val: total },
          { lbl: 'Blocklist Size', val: blocklistSize },
        ];
      }

      if (!this.hideChart) {
        this.generateBlockPie(blockedPercent);
      }
    },
    getStatusColor(status) {
      if (status === 'enabled') return 'green';
      if (status === 'disabled') return 'red';
      return 'blue';
    },
    /* Generate pie chart showing the proportion of queries blocked */
    generateBlockPie(blockedToday) {
      const chartData = {
        labels: ['Blocked', 'Allowed'],
        datasets: [{ values: [blockedToday, 100 - blockedToday] }],
      };
      return new this.Chart(`#${this.chartId}`, {
        title: 'Block Percent',
        data: chartData,
        type: 'donut',
        height: 250,
        strokeWidth: 18,
        colors: ['#f80363', '#20e253'],
        tooltipOptions: {
          formatTooltipY: d => `${Math.round(d)}%`,
        },
      });
    },
  },
};
</script>

<style scoped lang="scss">
.pi-hole-stats-v6-wrapper {
  display: flex;
  flex-direction: column;

  .status {
    margin: 0.5rem 0;

    .status-lbl {
      color: var(--widget-text-color);
      font-weight: bold;
    }

    .status-val {
      margin-left: 0.5rem;
      font-family: var(--font-monospace);

      &.green { color: var(--success); }
      &.red { color: var(--danger); }
      &.blue { color: var(--info); }
    }
  }

  img.block-percent-chart {
    margin: 0.5rem auto;
    max-width: 8rem;
    width: 100%;
  }

  .block-pie {
    margin: 0;
  }

  .data-table {
    display: flex;
    flex-direction: column;

    .data-table-row {
      display: flex;
      justify-content: space-between;

      p {
        margin: 0.2rem 0;
        color: var(--widget-text-color);
        font-size: 0.9rem;

        &.row-value {
          font-family: var(--font-monospace);
        }
      }

      &:not(:last-child) {
        border-bottom: 1px dashed var(--widget-text-color);
      }
    }
  }
}
</style>
