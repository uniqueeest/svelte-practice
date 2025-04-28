<script>
  import Wordle from '$lib/components/Wordle.svelte';
</script>

<style>
  main {
    max-width: 600px;
    margin: 0 auto;
    padding: 1rem;
    text-align: center;
  }

  h1 {
    margin-bottom: 2rem;
    color: #333;
  }
</style>

<main class="mx-auto max-w-xl p-4 text-center">
  <h1 class="mb-8 text-3xl font-bold text-gray-800">Wordle 게임</h1>
  <Wordle />
</main>
