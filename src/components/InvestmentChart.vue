<template>
  <div class="container mb-4">
    <h5>Monthly investment change ({{ accountName }})</h5>
    <p v-if="loading">Loading chart…</p>
    <p v-else-if="error" class="text-danger">{{ error }}</p>
    <canvas ref="canvas" v-show="!loading && !error"></canvas>
  </div>
</template>

<script>
import { API } from 'ynab';
import {
  Chart, BarController, BarElement, CategoryScale, LinearScale, Tooltip,
} from 'chart.js';

Chart.register(BarController, BarElement, CategoryScale, LinearScale, Tooltip);

// Adjust these if your account or payee names differ
const ACCOUNT_NAME = 'Roth IRA';
const PAYEES = ['Manual Balance Adjustment', 'Dividend'];
// The API only returns the last year of transactions unless since_date is set
const SINCE_DATE = '2000-01-01';

// Module-level so Vue 3 doesn't wrap the chart instance in a reactive proxy
let chart = null;

function monthRange(first, last) {
  const out = [];
  let [y, m] = first.split('-').map(Number);
  const [ly, lm] = last.split('-').map(Number);
  while (y < ly || (y === ly && m <= lm)) {
    out.push(`${y}-${String(m).padStart(2, '0')}`);
    m += 1;
    if (m > 12) { m = 1; y += 1; }
  }
  return out;
}

export default {
  props: ['token', 'planId'],
  data() {
    return { loading: false, error: null, accountName: ACCOUNT_NAME };
  },
  watch: {
    planId() { this.load(); },
  },
  mounted() {
    this.load();
  },
  beforeUnmount() {
    if (chart) { chart.destroy(); chart = null; }
  },
  methods: {
    async load() {
      this.loading = true;
      this.error = null;
      try {
        const api = new API(this.token);
        const accounts = (await api.accounts.getAccounts(this.planId)).data.accounts;
        const matches = accounts.filter(
          (a) => !a.deleted && a.name.trim().toLowerCase() === ACCOUNT_NAME.toLowerCase()
        );
        if (!matches.length) throw new Error(`No account named "${ACCOUNT_NAME}" found`);

        const totals = {};
        for (const acct of matches) {
          const res = await api.transactions.getTransactionsByAccount(
            this.planId, acct.id, SINCE_DATE
          );
          for (const t of res.data.transactions) {
            if (t.deleted || !PAYEES.includes(t.payee_name)) continue;
            const month = t.date.slice(0, 7);
            totals[month] = (totals[month] || 0) + t.amount / 1000;
          }
        }

        const keys = Object.keys(totals).sort();
        if (!keys.length) throw new Error('No matching transactions found');
        const labels = monthRange(keys[0], keys[keys.length - 1]);
        const values = labels.map((m) => Math.round((totals[m] || 0) * 100) / 100);

        this.loading = false;
        this.$nextTick(() => this.draw(labels, values));
      } catch (e) {
        this.loading = false;
        this.error = (e && e.error && e.error.detail) || (e && e.message) || 'Something went wrong';
      }
    },
    draw(labels, values) {
      if (chart) chart.destroy();
      chart = new Chart(this.$refs.canvas, {
        type: 'bar',
        data: {
          labels,
          datasets: [{
            data: values,
            backgroundColor: values.map((v) => (v >= 0 ? '#2e9e5b' : '#d64545')),
          }],
        },
        options: {
          plugins: { legend: { display: false } },
          scales: { y: { ticks: { callback: (v) => '$' + v.toLocaleString() } } },
        },
      });
    },
  },
};
</script>
