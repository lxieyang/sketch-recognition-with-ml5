<script>
  import Train from './components/Train.svelte';
  import Test from './components/Test.svelte';

  let mode = 'train'; // 'train' or 'test'
</script>

<style>

</style>

<div class="container">
  <h1 class="title is-3">
    Simple Sketch Recognition with
    <a href="https://ml5js.org/" target="_blank">ML5</a>
  </h1>

  <div class="tabs is-medium">
    <ul>
      <li class={mode === 'train' ? 'is-active' : ''}>
        <a on:click={_ => (mode = 'train')}>Train</a>
      </li>
      <li class={mode === 'test' ? 'is-active' : ''}>
        <a on:click={_ => (mode = 'test')}>Test</a>
      </li>
    </ul>
  </div>
  {#if mode === 'train'}
    <Train />
  {:else if mode === 'test'}
    <Test />
  {/if}

</div>
