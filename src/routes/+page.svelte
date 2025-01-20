<script>
  import { onMount, onDestroy } from 'svelte';
  
  let ws;
  let name = '';
  let isNameSet = false;
  let selectedScore = null;
  let allScores = {};
  let showResults = false;
  let activeTab = 'player';
  let totalPlayers = 0;
  let isConnected = false;
  
  const scores = [1, 2, 3, 5, 8, 13, 21];
  
  function connectWebSocket() {
    const wsUrl = import.meta.env.VITE_WS_URL;
    ws = new WebSocket(wsUrl);

    ws.onopen = () => {
      isConnected = true;
    };

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      if (message.type === 'state') {
        allScores = message.data.scores;
        showResults = message.data.showResults;
        totalPlayers = message.data.totalPlayers;
      } else if (message.type === 'reset') {
        allScores = message.data.scores;
        showResults = message.data.showResults;
        totalPlayers = message.data.totalPlayers;
        selectedScore = null;
      }
    };

    ws.onclose = () => {
      isConnected = false;
      setTimeout(connectWebSocket, 1000);
    };
  }
  
  onMount(() => {
    const savedName = localStorage.getItem('playerName');
    if (savedName) {
      name = savedName;
      isNameSet = true;
    }
    
    connectWebSocket();
  });
  
  onDestroy(() => {
    if (ws) ws.close();
  });
  
  function sendMessage(message) {
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(message));
    }
  }
  
  function handleNameSubmit(e) {
    e.preventDefault();
    if (name.trim()) {
      localStorage.setItem('playerName', name);
      isNameSet = true;
    }
  }
  
  function handleScoreSelect(score) {
    selectedScore = score;
    sendMessage({
      type: 'score',
      name,
      score
    });
  }
  
  function handleShowResults() {
    sendMessage({
      type: 'show_results',
      show: true
    });
  }
  
  function handleReset() {
    selectedScore = null;
    sendMessage({
      type: 'reset'
    });
  }
  
  $: votedCount = Object.keys(allScores).length;
  $: remainingVotes = totalPlayers - votedCount;
  $: canShowResults = votedCount === totalPlayers;
</script>

{#if !isConnected}
<div class="fixed top-0 left-0 right-0 bg-red-500 text-white p-2 text-center">
    กำลังเชื่อมต่อกับเซิร์ฟเวอร์...
</div>
{/if}

{#if !isNameSet}
  <div class="max-w-md mx-auto mt-8 p-4">
    <div class="bg-white rounded-lg shadow-md">
      <div class="p-6">
        <form on:submit={handleNameSubmit} class="space-y-4">
          <input
            type="text"
            placeholder="ใส่ชื่อของคุณ"
            bind:value={name}
            class="w-full px-4 py-2 border rounded-md"
          />
          <button type="submit" class="w-full px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700">
            เริ่มต้น
          </button>
        </form>
      </div>
    </div>
  </div>
{:else}
  <div class="max-w-2xl mx-auto mt-8 p-4 space-y-6">
    <div class="flex justify-between items-center">
      <h2 class="text-2xl font-bold">สวัสดี {name}</h2>
      <div class="space-x-4">
        <button 
          on:click={handleReset}
          class="px-4 py-2 border rounded-md hover:bg-gray-100"
        >
          🔄 เริ่มใหม่
        </button>
        <button 
          on:click={() => isNameSet = false}
          class="px-4 py-2 border rounded-md hover:bg-gray-100"
        >
          ✏️ เปลี่ยนชื่อ
        </button>
      </div>
    </div>

    <div class="border-b">
      <nav class="flex space-x-4">
        <button
          class="px-4 py-2 {activeTab === 'player' ? 'border-b-2 border-blue-600' : ''}"
          on:click={() => activeTab = 'player'}
        >
          👥 เลือกคะแนน
        </button>
        <button
          class="px-4 py-2 {activeTab === 'display' ? 'border-b-2 border-blue-600' : ''}"
          on:click={() => activeTab = 'display'}
        >
          👁️ แสดงผล
        </button>
      </nav>
    </div>

    {#if activeTab === 'player'}
      {#if !showResults}
        <div class="space-y-6">
          <div class="grid grid-cols-4 gap-4">
            {#each scores as score}
              <button
                class="h-16 text-xl border rounded-md {selectedScore === score ? 'bg-blue-600 text-white' : 'hover:bg-gray-100'}"
                on:click={() => handleScoreSelect(score)}
              >
                {score}
              </button>
            {/each}
          </div>
          
          {#if selectedScore}
            <div class="p-4 bg-blue-50 border border-blue-200 rounded-md">
              <p>คุณเลือกคะแนน: {selectedScore}</p>
            </div>
          {/if}
          
          <button 
            class="w-full px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:bg-gray-400"
            on:click={handleShowResults}
            disabled={!canShowResults}>
            แสดงผลคะแนนทั้งหมด
          </button>
        </div>
      {:else}
        <div class="bg-white rounded-lg shadow-md">
          <div class="p-6">
            <h3 class="text-xl font-semibold mb-4">ผลคะแนนทั้งหมด</h3>
            <div class="space-y-2">
              {#each Object.entries(allScores) as [playerName, score]}
                <div class="flex justify-between items-center">
                  <span>{playerName}</span>
                  <span class="font-semibold">{score}</span>
                </div>
              {/each}
            </div>
          </div>
        </div>
      {/if}
    {:else}
      <div class="space-y-6">
        {#if !showResults}
          <div class="bg-white rounded-lg shadow-md">
            <div class="p-6">
              <h3 class="text-xl font-semibold mb-4">กำลังรอโหวต...</h3>
              <div class="space-y-4">
                <div class="flex justify-between items-center">
                  <span>👥 จำนวนทั้งหมด:</span>
                  <span class="font-semibold">{totalPlayers}</span>
                </div>
                <div class="flex justify-between items-center">
                  <span>โหวตแล้ว:</span>
                  <span class="font-semibold">{votedCount} คน</span>
                </div>
                <div class="flex justify-between items-center">
                  <span>ยังไม่ได้โหวต:</span>
                  <span class="font-semibold">{remainingVotes} คน</span>
                </div>
                {#each Object.entries(allScores) as [playerName]}
                  <div class="flex justify-between items-center">
                    <span>{playerName}</span>
                    <span class="font-semibold">✓</span>
                  </div>
                {/each}
              </div>
            </div>
          </div>
        {:else}
          <div class="bg-white rounded-lg shadow-md">
            <div class="p-6">
              <h3 class="text-xl font-semibold mb-4">ผลคะแนนทั้งหมด</h3>
              <div class="space-y-4">
                {#each Object.entries(allScores) as [playerName, score]}
                  <div class="flex justify-between items-center">
                    <span>{playerName}</span>
                    <span class="font-semibold">{score}</span>
                  </div>
                {/each}
              </div>
            </div>
          </div>
        {/if}
      </div>
    {/if}
  </div>
{/if}
