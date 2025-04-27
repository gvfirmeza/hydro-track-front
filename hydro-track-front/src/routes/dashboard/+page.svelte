<script>
    import { onMount } from "svelte";
    import { Chart, registerables } from "chart.js";
  
    Chart.register(...registerables);
  
    let deviceId = "";
    let fluxoChart;
    let diarioChart;
    let fluxoCanvasRef;
    let diarioCanvasRef;
  
    let dadosTempoReal = []; // Armazenar os últimos 20
  
    async function buscarFluxoTempoReal() {
      try {
        const res = await fetch(`https://api-poc-rho.vercel.app/fluxo/recentes/${deviceId}`);
        const data = await res.json();
  
        if (data && data.length > 0) {
          const leituraAtual = {
            timestamp: new Date(data[0].timestamp).toLocaleTimeString(),
            litros: data[0].litros
          };
  
          dadosTempoReal.push(leituraAtual);
  
          if (dadosTempoReal.length > 20) {
            dadosTempoReal.shift(); // Remove o mais antigo se passar de 20
          }
  
          if (fluxoChart) {
            fluxoChart.data.labels = dadosTempoReal.map((d) => d.timestamp);
            fluxoChart.data.datasets[0].data = dadosTempoReal.map((d) => d.litros);
            fluxoChart.update();
          }
        }
      } catch (e) {
        console.error("Erro ao buscar leituras tempo real", e);
      }
    }
  
    async function buscarFluxoDiario() {
      try {
        const res = await fetch(`https://api-poc-rho.vercel.app/leituras-diarias/${deviceId}`);
        const data = await res.json();
  
        if (diarioChart) {
          diarioChart.data.labels = data.map((entry) => entry.data);
          diarioChart.data.datasets[0].data = data.map((entry) => entry.litros_total);
          diarioChart.update();
        }
      } catch (e) {
        console.error("Erro ao buscar leituras diárias", e);
      }
    }
  
    onMount(() => {
      const cookieDeviceId = document.cookie
        .split("; ")
        .find((row) => row.startsWith("deviceId="))
        ?.split("=")[1];
  
      if (!cookieDeviceId) {
        window.location.href = "/";
        return;
      }
  
      deviceId = cookieDeviceId;
  
      // Inicializa o gráfico de tempo real
      fluxoChart = new Chart(fluxoCanvasRef, {
        type: 'line',
        data: {
          labels: [],
          datasets: [
            {
              label: "Litros (Tempo Real)",
              data: [],
              fill: false,
              borderColor: "rgb(59, 130, 246)",
              tension: 0.1,
            },
          ],
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            y: { beginAtZero: true },
          },
        },
      });
  
      // Inicializa o gráfico de histórico diário
      diarioChart = new Chart(diarioCanvasRef, {
        type: 'bar',
        data: {
          labels: [],
          datasets: [
            {
              label: "Litros por Dia",
              data: [],
              backgroundColor: "rgb(34, 197, 94)",
            },
          ],
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            y: { beginAtZero: true },
          },
        },
      });
  
      buscarFluxoTempoReal();
      setInterval(buscarFluxoTempoReal, 1000); // Atualiza a cada 1 segundo
  
      buscarFluxoDiario();
      setInterval(buscarFluxoDiario, 1000); // Atualiza o diário a cada 1 min
    });
  </script>
  
  <div class="min-h-screen max-w-5xl m-auto p-6 space-y-12">
    <h1 class="text-3xl text-center font-bold mb-6">Dashboard - {deviceId}</h1>
  
    <!-- Gráfico de Fluxo Tempo Real -->
    <div class="bg-white shadow rounded p-4" style="height: 400px;">
      <h2 class="text-xl font-semibold mb-2 text-center">Fluxo Tempo Real</h2>
      <canvas bind:this={fluxoCanvasRef}></canvas>
    </div>
  
    <!-- Gráfico de Fluxo Diário -->
    <div class="bg-white shadow rounded p-4" style="height: 400px;">
      <h2 class="text-xl font-semibold mb-2 text-center">Fluxo Diário</h2>
      <canvas bind:this={diarioCanvasRef}></canvas>
    </div>
  </div>
  