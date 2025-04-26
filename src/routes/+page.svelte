<script>
  import Button from '$lib/components/Button.svelte';
  import { cn } from '$lib/utils';
</script>

<section class={cn('flex flex-col items-center justify-center gap-4', 'h-screen')}>
  <h1 class="text-4xl font-bold">Hello World</h1>
  <Button onclick={() => console.log('clicked')}>Click me</Button>
</section>
