<script>
  import { onMount } from "svelte";
  import { Chart, registerables } from "chart.js";
  import Header from "$lib/Header.svelte";
  import { page } from "$app/stores";

  Chart.register(...registerables);

  let adminId = "";
  let consumoChart;
  let consumoCanvasRef;
  let dadosConsumo = [];
  let dataInicio = new Date();
  let dataFim = new Date();
  let unidadeMedida = "litros";
  let deviceData = {}; // Objeto para armazenar dados agrupados por device_id
  
  // Inicializar período padrão (últimos 7 dias)
  dataInicio.setDate(dataInicio.getDate() - 7);
  // Formatar as datas para o formato YYYY-MM-DD para os inputs
  let dataInicioFormatada = dataInicio.toISOString().split('T')[0];
  let dataFimFormatada = dataFim.toISOString().split('T')[0];

  function converterUnidade(litros) {
    return unidadeMedida === "litros" ? litros : litros / 1000; // 1m³ = 1000 litros
  }

  function getUnidadeTexto() {
    return unidadeMedida === "litros" ? "Litros" : "Metros Cúbicos (m³)";
  }

  function atualizarRotulosGraficos() {
    if (consumoChart) {
      consumoChart.data.datasets.forEach(dataset => {
        dataset.label = `Consumo Total no Período Selecionado (${getUnidadeTexto()})`;
      });
      
      // Atualizar também o título do eixo X
      if (consumoChart.options.scales.x.title) {
        consumoChart.options.scales.x.title.text = getUnidadeTexto();
      }
      
      consumoChart.update();
    }
  }

  async function buscarDadosConsumo() {
    try {
      const res = await fetch(
        `https://api-poc-rho.vercel.app/leituras-diarias/admin/${adminId}`
      );
      const data = await res.json();

      // Agrupar dados por device_id
      deviceData = {};
      
      data.forEach(entry => {
        if (!deviceData[entry.device_id]) {
          deviceData[entry.device_id] = [];
        }
        deviceData[entry.device_id].push({
          data: entry.data,
          litros_total: entry.litros_total
        });
      });

      // Atualizar o gráfico
      atualizarGrafico();
    } catch (e) {
      console.error("Erro ao buscar dados de consumo", e);
    }
  }

  function atualizarGrafico() {
    if (!consumoChart) return;

    // Limpar datasets existentes
    consumoChart.data.datasets = [];
    
    // Coletar todas as datas únicas dos dados
    const todasDatas = new Set();
    Object.values(deviceData).forEach(entries => {
      entries.forEach(entry => todasDatas.add(entry.data));
    });
    
    // Ordenar datas
    const datasOrdenadas = Array.from(todasDatas).sort();
    
    // Filtrar datas dentro do período selecionado
    const dataInicioObj = new Date(dataInicioFormatada);
    const dataFimObj = new Date(dataFimFormatada);
    dataFimObj.setHours(23, 59, 59, 999); // Incluir o dia final completo
    
    const datasLimitadas = datasOrdenadas.filter(dataStr => {
      const data = new Date(dataStr);
      return data >= dataInicioObj && data <= dataFimObj;
    });
    
    // Cores para diferentes dispositivos
    const cores = [
      'rgb(59, 130, 246)', // Azul
      'rgb(16, 185, 129)', // Verde
      'rgb(249, 115, 22)', // Laranja
      'rgb(236, 72, 153)', // Rosa
      'rgb(139, 92, 246)', // Roxo
      'rgb(234, 179, 8)'   // Amarelo
    ];
    
    // Calcular a soma total de consumo para cada dispositivo
    const somasPorDispositivo = [];
    
    Object.entries(deviceData).forEach(([deviceId, entries], index) => {
      // Filtrar entradas dentro do período selecionado
      const entradasNoPeriodo = entries.filter(entry => 
        datasLimitadas.includes(entry.data)
      );
      
      // Calcular a soma total
      const somaTotal = entradasNoPeriodo.reduce(
        (total, entry) => total + converterUnidade(entry.litros_total), 
        0
      );
      
      somasPorDispositivo.push({
        deviceId,
        somaTotal,
        cor: cores[index % cores.length]
      });
    });
    
    // Ordenar dispositivos por consumo (opcional)
    somasPorDispositivo.sort((a, b) => b.somaTotal - a.somaTotal);
    
    // Atualizar labels do gráfico (IDs dos dispositivos)
    // Adicionar um identificador mais claro para cada dispositivo
    consumoChart.data.labels = somasPorDispositivo.map(item => `${item.deviceId}`);
    
    // Adicionar dataset único com todas as somas
    consumoChart.data.datasets.push({
      label: `Consumo Total no Período Selecionado (${getUnidadeTexto()})`,
      data: somasPorDispositivo.map(item => item.somaTotal),
      backgroundColor: somasPorDispositivo.map(item => item.cor),
      borderColor: somasPorDispositivo.map(item => item.cor)
    });
    
    // Atualizar o título do eixo X para refletir a unidade de medida atual
    consumoChart.options.scales.x.title.text = getUnidadeTexto();
    
    consumoChart.update();
  }
  
  function atualizarPeriodo() {
    atualizarGrafico();
  }
  
  function atualizarUnidadeMedida() {
    atualizarRotulosGraficos();
    atualizarGrafico();
  }

  onMount(() => {
    // Obter adminId da URL
    adminId = $page.params.adminId;
    
    if (!adminId) {
      window.location.href = "/";
      return;
    }

    // Inicializa o gráfico de consumo
    consumoChart = new Chart(consumoCanvasRef, {
      type: "bar",
      data: {
        labels: [],
        datasets: []
      },
      options: {
        indexAxis: 'y',  // Barras horizontais
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: { 
            beginAtZero: true,
            title: {
              display: true,
              text: getUnidadeTexto(),
              color: '#e0e0e0'
            },
            ticks: {
              color: '#e0e0e0'
            }
          },
          y: {
            ticks: {
              color: '#e0e0e0',
              font: {
                weight: 'bold'
              }
            }
          }
        },
        plugins: {
          title: {
            display: true,
            text: 'Consumo Total de Água por Dispositivo',
            font: {
              size: 16
            },
            color: '#e0e0e0'
          },
          legend: {
            position: 'top',
            labels: {
              color: '#e0e0e0'
            }
          }
        }
      }
    });

    buscarDadosConsumo();
    // Atualizar a cada 30 segundos
    setInterval(buscarDadosConsumo, 30000);
  });
</script>

<div class="min-h-screen">
  <Header />
  <div class="max-w-[90%] m-auto p-6 space-y-8">
    
    <h1 class="text-4xl text-center font-bold mt-4 mb-10 break-words">
      Dashboard Admin: {adminId}
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

    <div class="w-full">
      <div class="flex justify-between items-center mb-2">
        <h2 class="text-xl font-semibold text-blue-400">
          Consumo Total por Dispositivo
        </h2>
        <div class="flex items-center space-x-2">
          <div class="flex items-center">
            <label for="dataInicio" class="mr-2 text-sm">De:</label>
            <input 
              id="dataInicio"
              type="date" 
              bind:value={dataInicioFormatada} 
              class="bg-gray-700 border border-gray-600 rounded px-2 py-1 text-sm"
              on:change={atualizarPeriodo}
            />
          </div>
          <div class="flex items-center">
            <label for="dataFim" class="mr-2 text-sm">Até:</label>
            <input 
              id="dataFim"
              type="date" 
              bind:value={dataFimFormatada} 
              class="bg-gray-700 border border-gray-600 rounded px-2 py-1 text-sm"
              on:change={atualizarPeriodo}
            />
          </div>
        </div>
      </div>
      <div class="bg-gray-800 shadow-lg rounded-lg p-6 h-[500px]">
        <canvas bind:this={consumoCanvasRef}></canvas>
      </div>
    </div>
  </div>
</div>