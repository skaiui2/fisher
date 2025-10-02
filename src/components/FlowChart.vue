<template>
  <div>
    <h2>Flow Data</h2>
    <canvas id="rateChart"></canvas>
    <table border="1">
      <thead>
      <tr>
        <th>timestamp</th>
        <th>src_ip</th>
        <th>src_port</th>
        <th>dst_ip</th>
        <th>dst_port</th>
        <th>protocol</th>
        <th>rate_bps</th>
        <th>instant_rate_bps</th>
        <th>smoothed_rate_bps</th>
        <th>peak_rate_bps</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="(msg, index) in messages" :key="index">
        <td>{{ msg.timestamp }}</td>
        <td>{{ msg.src_ip }}</td>
        <td>{{ msg.src_port }}</td>
        <td>{{ msg.dst_ip }}</td>
        <td>{{ msg.dst_port }}</td>
        <td>{{ msg.protocol }}</td>
        <td>{{ msg.rate_bps }}</td>
        <td>{{ msg.instant_rate_bps }}</td>
        <td>{{ msg.smoothed_rate_bps }}</td>
        <td>{{ msg.peak_rate_bps }}</td>
      </tr>
      </tbody>
    </table>

    <h2>Rate over Time</h2>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { Chart, LineController, LineElement, PointElement, LinearScale, Title, CategoryScale } from 'chart.js'

Chart.register(LineController, LineElement, PointElement, LinearScale, Title, CategoryScale)

const messages = ref([])
let chart

onMounted(() => {
  const ctx = document.getElementById('rateChart').getContext('2d')
  chart = new Chart(ctx, {
    type: 'line',
    data: {
      labels: [], // timestamps
      datasets: [{
        label: 'rate_bps',
        data: [],
        borderColor: 'blue',
        fill: false,
        tension: 0.1
      }]
    },
    options: {
      responsive: true,
      animation: false,
      scales: {
        x: {
          title: { display: true, text: 'timestamp' }
        },
        y: {
          title: { display: true, text: 'rate_bps' }
        }
      }
    }
  })

  const ws = new WebSocket("ws://localhost:8081/ws/flow")

  ws.onopen = () => {
    console.log("WebSocket connected")
  }

  ws.onmessage = (event) => {
    try {
      const data = JSON.parse(event.data)
      messages.value.push(data)

      // update chart
      chart.data.labels.push(data.timestamp)
      chart.data.datasets[0].data.push(data.rate_bps)

      // keep only last 20 points
      if (chart.data.labels.length > 200) {
        chart.data.labels.shift()
        chart.data.datasets[0].data.shift()
      }

      chart.update()
    } catch (e) {
      console.warn("Non-JSON message:", event.data)
    }
  }

  ws.onerror = (err) => {
    console.error("WebSocket error:", err)
  }
})
</script>
