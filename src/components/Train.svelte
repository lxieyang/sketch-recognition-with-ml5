<script>
  let labels = new Set(['rect', 'circle', 'line']);
  let newLabel = '';
  $: toAdd = newLabel.trim().toLowerCase();

  function addNewLabel() {
    if (toAdd) {
      labels.add(toAdd);
      labels = labels;
      newLabel = '';
    }
  }

  function removeLabel(target) {
    labels.delete(target);
    labels = labels;
  }
</script>

<h1 class="title is-5">Labels</h1>
{#if Array.from(labels).length < 2}
  <em class="content has-text-danger is-small">Please add at least two labels</em>
{/if}
<div class="field is-grouped is-grouped-multiline">

  {#each Array.from(labels) as label}
    <div class="control">
      <div class="tags has-addons">
        <span class="tag is-link is-light">{label}</span>
        <span class="tag is-delete" on:click={e => removeLabel(label)} />
      </div>
    </div>
  {/each}
</div>

<div class="field has-addons">
  <div class="control">
    <input class="input is-small" type="text" placeholder="Add a label" bind:value={newLabel} />
  </div>
  <div class="control">
    <button
      class="button is-small is-primary"
      disabled={toAdd.length === 0 || labels.has(toAdd)}
      on:click={addNewLabel}>
      Add
    </button>
    {#if labels.has(toAdd)}
      <em class="content has-text-danger is-small">exists!</em>
    {/if}
  </div>
</div>
