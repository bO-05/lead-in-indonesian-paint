<script>
  import Dashboard from './lib/Dashboard.svelte';
</script>

<main class="min-h-screen w-full flex flex-col justify-between bg-[#f5f7fa]">
  <div class="container mx-auto px-4 py-8">
    <Dashboard />
  </div>
  <footer class="w-full bg-gray-100 py-4 text-center text-gray-600 text-sm">
    <a href="https://github.com/bO-05/lead-in-indonesian-paint" target="_blank" rel="noopener noreferrer" class="hover:text-gray-900 transition-colors">
      View on GitHub
    </a>
  </footer>
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
