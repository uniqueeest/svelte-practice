<script>
  let count = $state(0);
</script>

<h1>Hello Svelte World</h1>
<p>count : {count}</p>
<div>
  <button onclick={() => count++}>+</button>
  <button onclick={() => count--}>-</button>
</div>
