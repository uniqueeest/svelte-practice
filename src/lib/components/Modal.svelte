<script lang="ts">
  import { cn } from '$lib/utils';

  let { title, message, onClose, buttonText } = $props<{
    title: string;
    message?: string | Node;
    onClose?: () => void;
    buttonText?: string;
  }>();
</script>

<div class={cn('bg-opacity-50 fixed inset-0 z-50 flex items-center justify-center bg-black/30')}>
  <div
    class={cn(
      'relative',
      'flex flex-col items-center gap-4',
      'min-h-[200px] w-[calc(100%-2rem)] max-w-md p-4',
      'rounded-lg bg-white',
      'shadow-md',
    )}
  >
    <h2 class={cn('text-xl font-bold')}>{title}</h2>
    {#if message}
      <p class="text-center whitespace-pre-line">{message}</p>
    {/if}
    {#if onClose && buttonText}
      <button class={cn('absolute bottom-4', 'px-4 py-2', 'rounded-md bg-blue-500 text-white')} onclick={onClose}>
        {buttonText}
      </button>
    {/if}
  </div>
</div>
