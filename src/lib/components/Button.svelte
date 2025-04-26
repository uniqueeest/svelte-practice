<script lang="ts">
  import { cn } from '$lib/utils';
  import type { HTMLButtonAttributes } from 'svelte/elements';
  import type { Snippet } from 'svelte';

  type Props = {
    size?: 'sm' | 'md' | 'lg';
    color?: 'primary' | 'secondary' | 'danger';
    class?: string;
    children: Snippet;
    disabled?: boolean;
  } & HTMLButtonAttributes;

  const props: Props = $props();

  const sizeClasses = {
    sm: 'px-2 py-1 text-sm',
    md: 'px-4 py-2 text-base',
    lg: 'px-6 py-3 text-lg',
  };
  const baseColorClasses = {
    primary: 'bg-blue-600 text-white',
    secondary: 'bg-gray-200 text-black',
    danger: 'bg-red-600 text-white',
  };
  const hoverColorClasses = {
    primary: 'hover:bg-blue-700',
    secondary: 'hover:bg-gray-300',
    danger: 'hover:bg-red-700',
  };

  const btnClass = cn(
    'rounded-md font-medium focus:outline-none transition',
    sizeClasses[props.size ?? 'md'],
    baseColorClasses[props.color ?? 'primary'],
    !props.disabled && hoverColorClasses[props.color ?? 'primary'],
    props.disabled ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer',
    props.class,
  );
</script>

<button class={btnClass} {...props} disabled={props.disabled}>
  {@render props.children()}
</button>
