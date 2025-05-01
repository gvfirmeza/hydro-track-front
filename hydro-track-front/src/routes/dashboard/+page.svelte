<script>
  import { onMount } from "svelte";
  import { Chart, registerables } from "chart.js";
  import Header from "$lib/Header.svelte";

  Chart.register(...registerables);

  let deviceId = "";
  let fluxoChart;
  let diarioChart;
  let fluxoCanvasRef;
  let diarioCanvasRef;

  let dadosTempoReal = [];

  async function buscarFluxoTempoReal() {
    try {
      const res = await fetch(
        `https://api-poc-rho.vercel.app/fluxo/recentes/${deviceId}`,
      );
      const data = await res.json();

      if (data && data.length > 0) {
        const leituraAtual = {
          timestamp: new Date(data[0].timestamp).toLocaleTimeString(),
          litros: data[0].litros,
        };

        dadosTempoReal.push(leituraAtual);

        if (dadosTempoReal.length > 10) {
          dadosTempoReal.shift();
        }

        if (fluxoChart) {
          fluxoChart.data.labels = dadosTempoReal.map((d) => d.timestamp);
          fluxoChart.data.datasets[0].data = dadosTempoReal.map(
            (d) => d.litros,
          );
          fluxoChart.update();
        }
      }
    } catch (e) {
      console.error("Erro ao buscar leituras tempo real", e);
    }
  }

  async function buscarFluxoDiario() {
    try {
      const res = await fetch(
        `https://api-poc-rho.vercel.app/leituras-diarias/${deviceId}`,
      );
      const data = await res.json();

      if (diarioChart) {
        diarioChart.data.labels = data.map((entry) => entry.data);
        diarioChart.data.datasets[0].data = data.map(
          (entry) => entry.litros_total,
        );
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
      type: "line",
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
      type: "bar",
      data: {
        labels: [],
        datasets: [
          {
            label: "Litros por Dia",
            data: [],
            backgroundColor: "rgb(59, 130, 246)",
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

<div class="min-h-screen bg-gray-900 text-gray-200">
  <Header />
  <div class="max-w-[82%] m-auto p-6 space-y-12">
    
    <h1 class="text-4xl text-center font-bold mt-4 mb-10 break-words">
      Gastos do {deviceId}
    </h1>

    <div class="flex flex-wrap gap-14">

      <div class="flex-1 min-w-[45%]">
        <h2 class="text-xl font-semibold mb-2 text-blue-400 text-center">
          Gasto em Tempo Real
        </h2>
        <div class="bg-gray-800 shadow-lg rounded-lg p-6 h-96">
          <canvas bind:this={fluxoCanvasRef}></canvas>
        </div>
      </div>

      <div class="flex-1 min-w-[45%]">
        <h2 class="text-xl font-semibold mb-2 text-blue-400 text-center">
          Gasto Total Últimos 5 Dias
        </h2>
        <div class="bg-gray-800 shadow-lg rounded-lg p-6 h-96">
          <canvas bind:this={diarioCanvasRef}></canvas>
        </div>
      </div>

    </div>
  </div>
</div>
