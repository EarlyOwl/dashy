<template>
  <div class="pi-hole-queries-v6-wrapper" v-if="results">
    <div v-for="section in results" :key="section.id" class="query-section">
      <p class="section-title">{{ section.title }}</p>
      <div v-for="(query, i) in section.results" :key="i" class="query-row">
        <p class="domain">{{ query.domain }}</p>
        <p class="count">{{ query.count }}</p>
      </div>
    </div>
  </div>
</template>

<script>
import WidgetMixin from '@/mixins/WidgetMixin';
import { showNumAsThousand } from '@/utils/MiscHelpers';

export default {
  mixins: [WidgetMixin],
  data() {
    return {
      results: null,
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
    count() {
      const c = this.options.count;
      return typeof c === 'number' ? c : 10;
    },
    authUrl() {
      return `${this.hostname}/api/auth`;
    },
    endpoint() {
      return `${this.hostname}/api/stats/top_domains`;
    },
  },
  methods: {
    async fetchData() {
      try {
        if (!this.sid || !this.csrfToken) {
          await this.authenticate();
        }
        const topAds = await this.fetchTopDomains(true);
        const topQueries = await this.fetchTopDomains(false);
        this.processData(topAds, topQueries);
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
    /* Make GET request to local pi-hole to fetch top domains */
    async fetchTopDomains(blocked) {
      const url = `${this.endpoint}?blocked=${blocked}&count=${this.count}`;
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
        throw new Error(`Failed to fetch ${blocked ? 'blocked' : 'non-blocked'} domains`);
      }
      const data = await res.json();
      return Array.isArray(data.domains) ? data.domains : [];
    },
    /* Assign data variables to the returned data */
    processData(topAds, topQueries) {
      const format = (arr) => arr.map((item) => ({
        domain: item.domain || 'Unknown',
        count: showNumAsThousand(item.count || 0),
      }));

      this.results = [
        { id: '01', title: 'Top Ads Blocked', results: format(topAds) },
        { id: '02', title: 'Top Queries', results: format(topQueries) },
      ];
    },
  },
};
</script>

<style scoped lang="scss">
.pi-hole-queries-v6-wrapper {
  color: var(--widget-text-color);

  .query-section {
    display: inline-block;
    width: 100%;

    p.section-title {
      margin: 0.75rem 0 0.25rem;
      font-size: 1.2rem;
      font-weight: bold;
    }

    .query-row {
      display: flex;
      justify-content: space-between;
      margin: 0.25rem;

      p.domain {
        margin: 0.25rem 0;
        overflow: hidden;
        text-overflow: ellipsis;
      }

      p.count {
        margin: 0.25rem 0;
        font-family: var(--font-monospace);
      }

      &:not(:last-child) {
        border-bottom: 1px dashed var(--widget-text-color);
      }
    }
  }
}
</style>
