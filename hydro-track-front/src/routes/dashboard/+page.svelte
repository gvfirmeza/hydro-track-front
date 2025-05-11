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
  let diasHistorico = 5;
  let alturaMaximaGrafico = 10;
  let unidadeMedida = "litros";

  function converterUnidade(litros) {
    return unidadeMedida === "litros" ? litros : litros / 1000; // 1m³ = 1000 litros
  }

  function getUnidadeTexto() {
    return unidadeMedida === "litros" ? "Litros" : "Metros Cúbicos";
  }

  function atualizarRotulosGraficos() {
    if (fluxoChart) {
      fluxoChart.data.datasets[0].label = `${getUnidadeTexto()} (Tempo Real)`;
      fluxoChart.update();
    }
    if (diarioChart) {
      diarioChart.data.datasets[0].label = `${getUnidadeTexto()} por Dia`;
      diarioChart.update();
    }
  }

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
            (d) => converterUnidade(d.litros),
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
        `https://api-poc-rho.vercel.app/leituras-diarias/${deviceId}?dias=${diasHistorico}`,
      );
      const data = await res.json();

      if (diarioChart) {
        diarioChart.data.labels = data.map((entry) => entry.data);
        diarioChart.data.datasets[0].data = data.map(
          (entry) => converterUnidade(entry.litros_total),
        );
        diarioChart.update();
      }
    } catch (e) {
      console.error("Erro ao buscar leituras diárias", e);
    }
  }
  
  function atualizarDiasHistorico() {
    buscarFluxoDiario();
  }
  
  function atualizarUnidadeMedida() {
    atualizarRotulosGraficos();
    buscarFluxoTempoReal();
    buscarFluxoDiario();
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
            label: `${getUnidadeTexto()} (Tempo Real)`,
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
          y: { 
            beginAtZero: true,
            suggestedMax: alturaMaximaGrafico,
          },
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
            label: `${getUnidadeTexto()} por Dia`,
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
    
    <div class="flex justify-center mb-6">
      <div class="bg-gray-800 rounded-lg p-3 inline-flex items-center">
        <span class="mr-3 text-sm font-medium">Unidade de Medida:</span>
        <div class="flex space-x-4">
          <label class="inline-flex items-center cursor-pointer">
            <input 
              type="radio" 
              name="unidade" 
              value="litros" 
              bind:group={unidadeMedida} 
              on:change={atualizarUnidadeMedida}
              class="form-radio h-4 w-4 text-blue-500 bg-gray-700 border-gray-600"
              checked
            />
            <span class="ml-2 text-sm">Litros</span>
          </label>
          <label class="inline-flex items-center cursor-pointer">
            <input 
              type="radio" 
              name="unidade" 
              value="m3" 
              bind:group={unidadeMedida} 
              on:change={atualizarUnidadeMedida}
              class="form-radio h-4 w-4 text-blue-500 bg-gray-700 border-gray-600"
            />
            <span class="ml-2 text-sm">Metros Cúbicos (m³)</span>
          </label>
        </div>
      </div>
    </div>

    <div class="flex flex-wrap gap-14">

      <div class="flex-1 min-w-[45%]">
        <div class="flex justify-between items-center mb-2">
          <h2 class="text-xl font-semibold text-blue-400">
            Gasto em Tempo Real
          </h2>
          <div class="flex items-center">
            <label for="alturaMaxima" class="mr-2 text-sm">Altura máx. ({unidadeMedida === "litros" ? "litros" : "m³"}):</label>
            <input 
              id="alturaMaxima"
              type="number" 
              bind:value={alturaMaximaGrafico} 
              min="1" 
              max="100" 
              class="w-16 bg-gray-700 border border-gray-600 rounded px-2 py-1 text-sm"
              on:change={() => {
                if (fluxoChart) {
                  fluxoChart.options.scales.y.suggestedMax = alturaMaximaGrafico;
                  fluxoChart.update();
                }
              }}
            />
          </div>
        </div>
        <div class="bg-gray-800 shadow-lg rounded-lg p-6 h-96">
          <canvas bind:this={fluxoCanvasRef}></canvas>
        </div>
      </div>

      <div class="flex-1 min-w-[45%]">
        <div class="flex justify-between items-center mb-2">
          <h2 class="text-xl font-semibold text-blue-400">
            Gasto Total Últimos Dias
          </h2>
          <div class="flex items-center">
            <label for="diasHistorico" class="mr-2 text-sm">Dias:</label>
            <input 
              id="diasHistorico"
              type="number" 
              bind:value={diasHistorico} 
              min="1" 
              max="30" 
              class="w-16 bg-gray-700 border border-gray-600 rounded px-2 py-1 text-sm"
              on:change={atualizarDiasHistorico}
            />
          </div>
        </div>
        <div class="bg-gray-800 shadow-lg rounded-lg p-6 h-96">
          <canvas bind:this={diarioCanvasRef}></canvas>
        </div>
      </div>

    </div>
  </div>
</div>
