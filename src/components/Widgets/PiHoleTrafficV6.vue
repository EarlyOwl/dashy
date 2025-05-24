<template>
  <div :id="chartId" class="pi-hole-traffic-v6"></div>
</template>

<script>
import WidgetMixin from '@/mixins/WidgetMixin';
import ChartingMixin from '@/mixins/ChartingMixin';

export default {
  mixins: [WidgetMixin, ChartingMixin],
  data() {
    return {
      sid: null,
      csrfToken: null,
    };
  },
  computed: {
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
    authUrl() {
      return `${this.hostname}/api/auth`;
    },
    dataUrl() {
      return `${this.hostname}/api/history`;
    },
  },
  methods: {
    /* Make GET request to local pi-hole instance to fetch query history */
    async fetchData() {
      try {
        if (!this.sid || !this.csrfToken) {
          await this.authenticate();
        }

        const res = await fetch(this.dataUrl, {
          method: 'GET',
          headers: {
            'X-FTL-SID': this.sid,
            'X-FTL-CSRF': this.csrfToken,
            Accept: 'application/json',
          },
          credentials: 'include',
        });

        if (!res.ok) {
          throw new Error(`Failed to fetch history: HTTP ${res.status}`);
        }

        const { history } = await res.json();
        this.processData(history);
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
    /* Assign data variables to the returned data */
    processData(history) {
      const labels = [];
      const total = [];
      const blocked = [];

      history.forEach((point) => {
        labels.push(this.formatTime(point.timestamp * 1000));
        total.push(point.total || 0);
        blocked.push(point.blocked || 0);
      });

      const chartData = {
        labels,
        datasets: [
          { name: 'Total Queries', type: 'line', values: total },
          { name: 'Blocked', type: 'line', values: blocked },
        ],
      };

      this.generateChart(chartData);
    },
    /* Generate chart showing the umber of queries */
    generateChart(chartData) {
      return new this.Chart(`#${this.chartId}`, {
        title: 'Query History',
        data: chartData,
        type: 'axis-mixed',
        height: this.chartHeight,
        colors: ['#20e253', '#f80363'],
        truncateLegends: true,
        lineOptions: {
          regionFill: 1,
          hideDots: 1,
        },
        axisOptions: {
          xIsSeries: true,
          xAxisMode: 'tick',
        },
      });
    },
  },
};
</script>

<style scoped>
.pi-hole-traffic-v6 {
  width: 100%;
}
</style>
