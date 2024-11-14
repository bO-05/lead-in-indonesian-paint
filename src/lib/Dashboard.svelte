<script>
  import { onMount, tick } from 'svelte';
  import { Chart, registerables } from 'chart.js';
  import paintData from '../data/paint.json';
  import brandDistribution from '../data/distributionBrands.json';
  import colorDistribution from '../data/distributionColor.json';
  import manufacturers from '../data/manufacturers.json';
  import companyResponses from '../data/companyResponses.json';

  Chart.register(...registerables);
  let tableData = paintData;
  let searchTerm = '';
  let activeTab = 'samples'; // Changed default tab
  
  // Add sort state tracking
  let sortConfig = {
    key: null,
    direction: 'asc'
  };

  // Helper function to parse lead content
  function parsePPM(value) {
    if (typeof value === 'number') return value;
    if (typeof value !== 'string') return 0;
    
    // Handle "< X" values
    if (value.includes('<')) {
      return parseFloat(value.replace(/[^0-9.]/g, ''));
    }
    return parseFloat(value.replace(/,/g, ''));
  }

  // Updated sort function
  const sortData = (key) => {
    // Toggle direction if clicking same column
    if (sortConfig.key === key) {
      sortConfig.direction = sortConfig.direction === 'asc' ? 'desc' : 'asc';
    } else {
      sortConfig.key = key;
      sortConfig.direction = 'asc';
    }

    tableData = [...tableData].sort((a, b) => {
      let comparison = 0;
      
      if (key === 'Lead Content (ppm)') {
        comparison = parsePPM(a[key]) - parsePPM(b[key]);
      } else {
        comparison = a[key].toString().localeCompare(b[key].toString());
      }

      return sortConfig.direction === 'asc' ? comparison : -comparison;
    });
  };

  // Helper to get sort indicator
  const getSortIndicator = (key) => {
    if (sortConfig.key !== key) return '↕️';
    return sortConfig.direction === 'asc' ? '↑' : '↓';
  };

  // Filter function
  $: {
    if (searchTerm) {
      tableData = paintData.filter(item => 
        Object.values(item).some(val => 
          val.toString().toLowerCase().includes(searchTerm.toLowerCase())
        )
      );
    } else {
      tableData = paintData;
    }
  }

  // Row class function
  function getRowClass(leadContent) {
    const value = parsePPM(leadContent);
    if (value > 10000) return 'high-risk';
    if (value > 90) return 'medium-risk';
    return 'safe';
  }

  let brandTableData = brandDistribution;

  // Add sorting function for brand table
  const sortBrandData = (key) => {
    if (sortConfig.key === key) {
      sortConfig.direction = sortConfig.direction === 'asc' ? 'desc' : 'asc';
    } else {
      sortConfig.key = key;
      sortConfig.direction = 'asc';
    }

    brandTableData = [...brandTableData].sort((a, b) => {
      let comparison = 0;
      
      // Handle numeric columns
      if (typeof a[key] === 'number') {
        comparison = a[key] - b[key];
      } else {
        comparison = a[key].toString().localeCompare(b[key].toString());
      }

      return sortConfig.direction === 'asc' ? comparison : -comparison;
    });
  };

  // Add to script section
  let showModal = false;
  let modalData = {
    title: '',
    items: []
  };

  // Add new sort state for modal
  let modalSortConfig = {
    key: null,
    direction: 'asc'
  };

  // Add sort function for modal data
  function sortModalData(key) {
    if (modalSortConfig.key === key) {
      modalSortConfig.direction = modalSortConfig.direction === 'asc' ? 'desc' : 'asc';
    } else {
      modalSortConfig.key = key;
      modalSortConfig.direction = 'asc';
    }

    modalData.items = [...modalData.items].sort((a, b) => {
      let comparison = 0;
      
      if (key === 'Lead Content (ppm)') {
        comparison = parsePPM(a[key]) - parsePPM(b[key]);
      } else {
        comparison = a[key].toString().localeCompare(b[key].toString());
      }

      return modalSortConfig.direction === 'asc' ? comparison : -comparison;
    });
  }

  // Update the initializeCharts function
  async function initializeCharts() {
    await tick();

    const leadCtx = document.getElementById('leadChart');
    if (leadCtx) {
      new Chart(leadCtx, {
        type: 'bar',
        data: {
          labels: ['Safe (<90 ppm)', 'Medium (90-10k ppm)', 'High (>10k ppm)'],
          datasets: [{
            label: 'Number of Samples',
            data: [
              paintData.filter(p => parsePPM(p['Lead Content (ppm)']) < 90).length,
              paintData.filter(p => {
                const val = parsePPM(p['Lead Content (ppm)']);
                return val >= 90 && val <= 10000;
              }).length,
              paintData.filter(p => parsePPM(p['Lead Content (ppm)']) > 10000).length
            ],
            backgroundColor: ['#4caf50', '#ffa500', '#ff4444']
          }]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          plugins: {
            legend: { position: 'top' },
            title: { display: true, text: 'Lead Content Distribution' }
          },
          onClick: (event, elements) => {
            if (elements.length > 0) {
              const index = elements[0].index;
              let items = [];
              let title = '';

              if (index === 0) {
                title = 'Safe Samples (<90 ppm)';
                items = paintData.filter(p => parsePPM(p['Lead Content (ppm)']) < 90);
              } else if (index === 1) {
                title = 'Medium Risk Samples (90-10,000 ppm)';
                items = paintData.filter(p => {
                  const val = parsePPM(p['Lead Content (ppm)']);
                  return val >= 90 && val <= 10000;
                });
              } else {
                title = 'High Risk Samples (>10,000 ppm)';
                items = paintData.filter(p => parsePPM(p['Lead Content (ppm)']) > 10000);
              }

              modalData = { title, items };
              showModal = true;
            }
          }
        }
      });
    }

    const colorCtx = document.getElementById('colorChart');
    if (colorCtx) {
      const colorLabels = colorDistribution.map(item => item.Colour);
      const colorData = colorDistribution.map(item => item['No. of Samples']);
      const colorBackgrounds = colorLabels.map(getColorCode);

      new Chart(colorCtx, {
        type: 'pie',
        data: {
          labels: colorLabels,
          datasets: [{
            data: colorData,
            backgroundColor: colorBackgrounds,
            borderColor: '#e5e7eb',
            borderWidth: 1
          }]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          plugins: {
            legend: { position: 'right' },
            title: { display: true, text: 'Paint Color Distribution' }
          },
          onClick: (event, elements) => {
            if (elements.length > 0) {
              const index = elements[0].index;
              const color = colorLabels[index];
              const colorInfo = colorDistribution[index];
              
              modalData = {
                title: `${color} Paint Samples`,
                items: paintData.filter(p => p['Color of Paint'] === color)
              };
              showModal = true;
            }
          }
        }
      });
    }
  }

  // Call initializeCharts when the visual tab is active
  function switchTab(tab) {
    activeTab = tab;
    searchTerm = '';
    
    switch(tab) {
      case 'visual':
        initializeCharts(); // Initialize charts when switching to the visual tab
        break;
      case 'samples':
        tableData = paintData;
        break;
      case 'brands':
        brandTableData = brandDistribution;
        break;
      case 'manufacturers':
        tableData = manufacturers;
        break;
      case 'responses':
        tableData = companyResponses;
        break;
    }
  }

  onMount(() => {
    console.log('Initializing charts...');

    // Update the switchTab function to initialize charts when the visual tab is activated
    function switchTab(tab) {
      activeTab = tab;
      searchTerm = '';
      
      switch(tab) {
        case 'visual':
          initializeCharts(); // Initialize charts when switching to the visual tab
          break;
        case 'samples':
          tableData = paintData;
          break;
        case 'brands':
          brandTableData = brandDistribution;
          break;
        case 'manufacturers':
          tableData = manufacturers;
          break;
        case 'responses':
          tableData = companyResponses;
          break;
      }
    }
  });

  function getColorCode(color) {
    const colorMap = {
      'Red': '#dc2626',
      'Yellow': '#eab308',
      'Green': '#16a34a',
      'Blue': '#2563eb',
      'Orange': '#ea580c',
      'White': '#ffffff',
      'Grey': '#4b5563'
    };
    return colorMap[color] || '#000000';
  }

  function getTextClass(leadContent) {
    const value = parsePPM(leadContent);
    if (value > 10000) return 'text-red-700';
    if (value > 90) return 'text-orange-700';
    return 'text-green-700';
  }
</script>

<div class="dashboard">
  <header class="mb-8">
    <h1 class="text-2xl md:text-3xl font-bold mb-2">Indonesian Paint Lead Content Analysis</h1>
    <p class="text-gray-600 text-sm md:text-base mb-4">
      Analysis of lead content in paint brands across Indonesia
    </p>
    <div class="text-xs text-gray-500 space-y-1 border-t pt-4">
      <div class="bg-blue-50 border-l-4 border-blue-400 p-4 mb-4">
        <p class="text-sm text-blue-700">
          <strong>Disclaimer:</strong> This is a simple data visualization of the research paper below. 
          I take no credit for the research, data collection, or findings. 
          All credit goes to the original authors and researchers. I made this for future reference when buying paint.
        </p>
      </div>
      <p>Original Research:</p>
      <p><strong>"Lead in Solvent-Based Paints in Indonesia 2021"</strong></p>
      <p>DOI: <a href="https://doi.org/10.13140/RG.2.2.17539.02081" 
                 target="_blank"
                 class="text-blue-600 hover:underline">
        10.13140/RG.2.2.17539.02081
      </a></p>
      <p>
        <a href="https://ipen.org/sites/default/files/documents/lead_in_solvent-based_paints_in_indonesia_2021_nexus3_english.pdf" 
           target="_blank" 
           class="text-blue-600 hover:underline">
          PDF Report
        </a>
      </p>
    </div>
  </header>

  <!-- Tab Navigation -->
  <div class="mb-6">
    <nav class="flex space-x-4 border-b">
      <button 
        class="px-6 py-3 font-medium transition-colors duration-200 ease-in-out
          {activeTab === 'visual' 
            ? 'border-b-2 border-blue-500 text-blue-600 bg-blue-50' 
            : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'}"
        on:click={() => switchTab('visual')}>
        Visualizations
      </button>
      <button 
        class="px-6 py-3 font-medium transition-colors duration-200 ease-in-out
          {activeTab === 'samples' 
            ? 'border-b-2 border-blue-500 text-blue-600 bg-blue-50' 
            : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'}"
        on:click={() => switchTab('samples')}>
        All Paint Samples
      </button>
      <button 
        class="px-6 py-3 font-medium transition-colors duration-200 ease-in-out
          {activeTab === 'brands' 
            ? 'border-b-2 border-blue-500 text-blue-600 bg-blue-50' 
            : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'}"
        on:click={() => switchTab('brands')}>
        Brand Distribution
      </button>
      <button 
        class="px-6 py-3 font-medium transition-colors duration-200 ease-in-out
          {activeTab === 'manufacturers' 
            ? 'border-b-2 border-blue-500 text-blue-600 bg-blue-50' 
            : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'}"
        on:click={() => switchTab('manufacturers')}>
        Manufacturers
      </button>
      <button 
        class="px-6 py-3 font-medium transition-colors duration-200 ease-in-out
          {activeTab === 'responses' 
            ? 'border-b-2 border-blue-500 text-blue-600 bg-blue-50' 
            : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'}"
        on:click={() => switchTab('responses')}>
        Company Responses
      </button>
    </nav>
  </div>

  <!-- Tab Content -->
  {#if activeTab === 'visual'}
    <h2 class="text-xl font-semibold mb-4">Data Visualization</h2>
    
    <!-- Add total count -->
    <div class="bg-white p-4 rounded-lg shadow-sm mb-6">
      <div class="text-center">
        <span class="text-3xl font-bold text-gray-700">Total Samples: {paintData.length}</span>
      </div>
    </div>

    <div class="grid grid-cols-1 gap-6 mb-8">
      <div class="bg-white p-4 rounded-lg shadow-sm">
        <h3 class="text-lg font-medium mb-2">Lead Content Distribution</h3>
        <div class="chart-container" style="height: 300px;">
          <canvas id="leadChart"></canvas>
        </div>
      </div>
      <div class="bg-white p-4 rounded-lg shadow-sm">
        <h3 class="text-lg font-medium mb-2">Paint Color Distribution</h3>
        <div class="chart-container" style="height: 300px;">
          <canvas id="colorChart"></canvas>
        </div>
      </div>
    </div>

  {:else if activeTab === 'samples'}
    <div class="controls bg-white p-4 rounded-lg shadow-sm mb-6">
      <div class="flex flex-col md:flex-row gap-4 items-center">
        <div class="w-full md:w-1/2">
          <input 
            type="text" 
            bind:value={searchTerm} 
            placeholder="Search brands, manufacturers, colors..."
            class="w-full border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 text-base"
            style="padding: 12px 24px; font-size: 16px; border-width: 2px; margin-bottom: 16px; margin-top: 16px;"
          />
        </div>
      </div>
    </div>

    <div class="w-full overflow-hidden bg-white rounded-lg shadow-sm mb-8">
      <div class="w-full overflow-x-auto -mx-4 sm:mx-0">
        <table class="w-full whitespace-nowrap">
          <thead>
            <tr class="bg-gray-50">
              <th class="px-4 py-3 text-left text-sm font-semibold cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortData('Brand Name')}>
                Brand Name 
                <span class="sort-indicator">{getSortIndicator('Brand Name')}</span>
              </th>
              <th class="px-4 py-3 text-left text-sm font-semibold cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortData('Manufacturer')}>
                Manufacturer
                <span class="sort-indicator">{getSortIndicator('Manufacturer')}</span>
              </th>
              <th class="px-4 py-3 text-left text-sm font-semibold cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortData('Color of Paint')}>
                Color
                <span class="sort-indicator">{getSortIndicator('Color of Paint')}</span>
              </th>
              <th class="px-4 py-3 text-left text-sm font-semibold cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortData('Lead Content (ppm)')}>
                Lead Content (ppm)
                <span class="sort-indicator">{getSortIndicator('Lead Content (ppm)')}</span>
              </th>
            </tr>
          </thead>
          <tbody>
            {#each tableData as row}
              <tr class="hover:bg-gray-50">
                <td class="px-4 py-3 text-sm">{row['Brand Name']}</td>
                <td class="px-4 py-3 text-sm">{row['Manufacturer']}</td>
                <td class="px-4 py-3 text-sm">
                  <div class="flex items-center gap-2">
                    <span class="w-3 h-3 rounded-full border border-gray-300" 
                          style="background-color: {getColorCode(row['Color of Paint'])}">
                    </span>
                    <span style="color: {row['Color of Paint'] === 'White' ? '#000000' : getColorCode(row['Color of Paint'])}">
                      {row['Color of Paint']}
                    </span>
                  </div>
                </td>
                <td class="px-4 py-3 text-sm">
                  <div class="relative">
                    <div class={`absolute inset-0 opacity-20 ${getRowClass(row['Lead Content (ppm)'])}`}></div>
                    <span class={`relative z-10 font-medium ${getTextClass(row['Lead Content (ppm)'])}`}>
                      {row['Lead Content (ppm)']}
                    </span>
                  </div>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>

    <div class="w-full overflow-hidden bg-white rounded-lg shadow-sm mb-8">
      <div class="w-full overflow-x-auto -mx-4 sm:mx-0">
        <table class="w-full whitespace-nowrap text-sm">
          <thead>
            <tr>
              <th class="py-2 px-3 bg-gray-50 text-left font-medium text-gray-600">Risk Level</th>
              <th class="py-2 px-3 bg-gray-50 text-left font-medium text-gray-600">PPM Range</th>
              <th class="py-2 px-3 bg-gray-50 text-right font-medium text-gray-600">Count</th>
            </tr>
          </thead>
          <tbody>
            <tr class="border-l-4 border-green-500">
              <td class="py-2 px-3 font-medium text-green-800">Safe</td>
              <td class="py-2 px-3 text-green-600">&lt;90 ppm</td>
              <td class="py-2 px-3 text-right font-bold text-green-700">
                {paintData.filter(p => parsePPM(p['Lead Content (ppm)']) < 90).length}
              </td>
            </tr>
            <tr class="border-l-4 border-orange-500">
              <td class="py-2 px-3 font-medium text-orange-800">Medium Risk</td>
              <td class="py-2 px-3 text-orange-600">90-10,000 ppm</td>
              <td class="py-2 px-3 text-right font-bold text-orange-700">
                {paintData.filter(p => {
                  const val = parsePPM(p['Lead Content (ppm)']);
                  return val >= 90 && val <= 10000;
                }).length}
              </td>
            </tr>
            <tr class="border-l-4 border-red-500">
              <td class="py-2 px-3 font-medium text-red-800">High Risk</td>
              <td class="py-2 px-3 text-red-600">&gt;10,000 ppm</td>
              <td class="py-2 px-3 text-right font-bold text-red-700">
                {paintData.filter(p => parsePPM(p['Lead Content (ppm)']) > 10000).length}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

  {:else if activeTab === 'brands'}
    <div class="bg-white rounded-lg shadow-sm p-4 w-full">
      <div class="overflow-x-auto w-full">
        <table class="w-full">
          <thead>
            <tr class="bg-gray-50">
              <th class="px-4 py-3 text-left cursor-pointer hover:bg-gray-100"
                  on:click={() => sortBrandData('Brand Name')}>
                Brand Name
                <span class="sort-indicator">{getSortIndicator('Brand Name')}</span>
              </th>
              <th class="px-4 py-3 text-left cursor-pointer hover:bg-gray-100"
                  on:click={() => sortBrandData('No. of Samples')}>
                No. of Samples
                <span class="sort-indicator">{getSortIndicator('No. of Samples')}</span>
              </th>
              <th class="px-4 py-3 text-left cursor-pointer hover:bg-gray-100"
                  on:click={() => sortBrandData('No. of Samples <90 ppm')}>
                Samples &lt;90 ppm
                <span class="sort-indicator">{getSortIndicator('No. of Samples <90 ppm')}</span>
              </th>
              <th class="px-4 py-3 text-left cursor-pointer hover:bg-gray-100"
                  on:click={() => sortBrandData('No. of Samples >90 ppm')}>
                Samples &gt;90 ppm
                <span class="sort-indicator">{getSortIndicator('No. of Samples >90 ppm')}</span>
              </th>
              <th class="px-4 py-3 text-left cursor-pointer hover:bg-gray-100"
                  on:click={() => sortBrandData('No. of Samples >10,000 ppm')}>
                Samples &gt;10,000 ppm
                <span class="sort-indicator">{getSortIndicator('No. of Samples >10,000 ppm')}</span>
              </th>
            </tr>
          </thead>
          <tbody>
            {#each brandTableData as brand}
              <tr class="hover:bg-gray-50">
                <td class="px-4 py-3">{brand['Brand Name']}</td>
                <td class="px-4 py-3">{brand['No. of Samples']}</td>
                <td class="px-4 py-3 text-green-600">{brand['No. of Samples <90 ppm']}</td>
                <td class="px-4 py-3 text-orange-600">{brand['No. of Samples >90 ppm']}</td>
                <td class="px-4 py-3 text-red-600">{brand['No. of Samples >10,000 ppm']}</td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>

  {:else if activeTab === 'manufacturers'}
    <div class="bg-white rounded-lg shadow-sm p-4 w-full">
      <div class="overflow-x-auto w-full">
        <table class="w-full">
          <thead>
            <tr class="bg-gray-50">
              <th class="px-4 py-3 text-left">Manufacturer</th>
              <th class="px-4 py-3 text-left">Type</th>
              <th class="px-4 py-3 text-left">Website</th>
            </tr>
          </thead>
          <tbody>
            {#each manufacturers as mfr}
              <tr class="hover:bg-gray-50">
                <td class="px-4 py-3">{mfr.Manufacturer}</td>
                <td class="px-4 py-3">{mfr['Type of company']}</td>
                <td class="px-4 py-3">
                  {#if mfr.Website !== 'N/A'}
                    <a href={mfr.Website} target="_blank" class="text-blue-600 hover:underline">
                      {mfr.Website}
                    </a>
                  {:else}
                    N/A
                  {/if}
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>

  {:else if activeTab === 'responses'}
    <div class="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
      {#each companyResponses as response}
        <div class="bg-white rounded-lg shadow-sm p-4">
          <h3 class="font-bold text-lg mb-2">{response['Company Name']}</h3>
          <div class="text-sm text-gray-600 mb-2">
            Response Date: {response['Response Date']}
          </div>
          <div class="text-sm mb-2">
            <strong>PIC:</strong> {response['PIC']}
          </div>
          <div class="text-sm">
            <strong>Response:</strong> {response['Response']}
          </div>
          {#if response['Statement']}
            <div class="mt-2 text-sm text-gray-700 italic">
              "{response['Statement']}"
            </div>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
</div>

<style>
  :global(body) {
    background-color: #f5f7fa;
    margin: 0;
    padding: 0;
  }

  .dashboard {
    width: 100%;
    margin: 0 auto;
  }

  .stat-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    min-width: 100px;
  }

  .high-risk { background-color: theme('colors.red.500'); }
  .medium-risk { background-color: theme('colors.orange.500'); }
  .safe { background-color: theme('colors.green.500'); }

  @media (max-width: 640px) {
    .dashboard {
      padding: 0;
    }
    
    td, th {
      padding: 0.5rem !important;
      font-size: 0.75rem !important;
    }

    :global(.container) {
      padding-left: 0;
      padding-right: 0;
    }

    .chart-container {
      height: 250px !important;
    }

    nav.flex {
      overflow-x: auto;
      -webkit-overflow-scrolling: touch;
      padding-bottom: 1px;
    }

    nav.flex::-webkit-scrollbar {
      display: none;
    }

    nav button {
      white-space: nowrap;
      padding: 0.5rem 1rem;
    }
  }

  .sort-indicator {
    display: inline-block;
    margin-left: 4px;
    font-size: 0.8em;
    opacity: 0.6;
  }

  th:hover .sort-indicator {
    opacity: 1;
  }

  .dashboard {
    width: 100%;
  }

  table {
    border-collapse: collapse;
    width: 100%;
  }

  th, td {
    text-align: left;
    padding: 1rem;
  }

  :global(#chartjs-tooltip) {
    background: rgba(0, 0, 0, 0.85) !important;
    border-radius: 4px;
    color: white;
    opacity: 1;
    pointer-events: auto !important;
    transition: all .1s ease;
    max-height: 300px !important;
    overflow-y: auto !important;
  }

  :global(#chartjs-tooltip .tooltip-header) {
    padding-bottom: 6px;
    border-bottom: 1px solid rgba(255,255,255,0.1);
    margin-bottom: 6px;
    font-weight: bold;
  }

  :global(#chartjs-tooltip .tooltip-body) {
    max-height: 250px;
    overflow-y: auto;
    padding-right: 5px;
  }

  :global(#chartjs-tooltip .tooltip-body::-webkit-scrollbar) {
    width: 4px;
  }

  :global(#chartjs-tooltip .tooltip-body::-webkit-scrollbar-track) {
    background: rgba(255,255,255,0.1);
    border-radius: 2px;
  }

  :global(#chartjs-tooltip .tooltip-body::-webkit-scrollbar-thumb) {
    background: rgba(255,255,255,0.3);
    border-radius: 2px;
  }
</style> 

{#if showModal}
  <div class="fixed inset-0 bg-black bg-opacity-50 z-50 flex items-center justify-center p-4">
    <div class="bg-white rounded-lg max-w-4xl w-full max-h-[90vh] overflow-hidden">
      <div class="p-4 border-b flex justify-between items-center">
        <h3 class="text-lg font-semibold">{modalData.title}</h3>
        <button 
          class="text-gray-500 hover:text-gray-700"
          on:click={() => showModal = false}>
          ✕
        </button>
      </div>
      <div class="p-4 overflow-auto max-h-[calc(90vh-8rem)]">
        <table class="w-full">
          <thead>
            <tr class="bg-gray-50">
              <th class="px-4 py-2 text-left cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortModalData('Brand Name')}>
                Brand Name
                <span class="sort-indicator">
                  {modalSortConfig.key === 'Brand Name' 
                    ? (modalSortConfig.direction === 'asc' ? '↑' : '↓') 
                    : '↕️'}
                </span>
              </th>
              <th class="px-4 py-2 text-left cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortModalData('Manufacturer')}>
                Manufacturer
                <span class="sort-indicator">
                  {modalSortConfig.key === 'Manufacturer' 
                    ? (modalSortConfig.direction === 'asc' ? '↑' : '↓') 
                    : '↕️'}
                </span>
              </th>
              <th class="px-4 py-2 text-left cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortModalData('Color of Paint')}>
                Color
                <span class="sort-indicator">
                  {modalSortConfig.key === 'Color of Paint' 
                    ? (modalSortConfig.direction === 'asc' ? '↑' : '↓') 
                    : '↕️'}
                </span>
              </th>
              <th class="px-4 py-2 text-left cursor-pointer hover:bg-gray-100" 
                  on:click={() => sortModalData('Lead Content (ppm)')}>
                Lead Content (ppm)
                <span class="sort-indicator">
                  {modalSortConfig.key === 'Lead Content (ppm)' 
                    ? (modalSortConfig.direction === 'asc' ? '↑' : '↓') 
                    : '↕️'}
                </span>
              </th>
            </tr>
          </thead>
          <tbody>
            {#each modalData.items as item}
              <tr class="border-t">
                <td class="px-4 py-2">{item['Brand Name']}</td>
                <td class="px-4 py-2">{item['Manufacturer']}</td>
                <td class="px-4 py-2">
                  <div class="flex items-center gap-2">
                    <span class="w-3 h-3 rounded-full border border-gray-300" 
                          style="background-color: {getColorCode(item['Color of Paint'])}">
                    </span>
                    <span style="color: {item['Color of Paint'] === 'White' ? '#000000' : getColorCode(item['Color of Paint'])}">
                      {item['Color of Paint']}
                    </span>
                  </div>
                </td>
                <td class="px-4 py-2">
                  <span class={getTextClass(item['Lead Content (ppm)'])}>
                    {item['Lead Content (ppm)']}
                  </span>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </div>
  </div>
{/if} 