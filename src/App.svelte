<script>
  import Dashboard from './lib/Dashboard.svelte';
</script>

<main class="min-h-screen w-full flex justify-center bg-[#f5f7fa]">
  <div class="container mx-auto px-4 py-8">
    <Dashboard />
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
  }

  .container {
    max-width: 1400px;
    width: 100%;
  }
</style>
