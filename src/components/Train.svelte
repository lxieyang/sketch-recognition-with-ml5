<script>
  import { onMount } from 'svelte';
  import p5 from 'p5';
  import ml5 from 'ml5';

  let myp5;
  let featureExtractor;
  let classifier;

  let labels = new Map();
  labels.set('rect', []);
  labels.set('circle', []);
  labels.set('line', []);
  let newLabel = '';
  $: toAdd = newLabel.trim().toLowerCase();
  let activeLabelButton = '';

  let sketch = function(p5) {
    let canvas;

    p5.setup = () => {
      canvas = p5.createCanvas(200, 150);
      canvas.parent('canvas-holder');
      p5.background('white');
      p5.strokeWeight(6);
      p5.stroke(0);
    };

    p5.touchMoved = () => {
      p5.line(p5.mouseX, p5.mouseY, p5.pmouseX, p5.pmouseY);
      return false;
    };

    p5.clearCanvas = () => {
      p5.clear();
      p5.background('white');
    };
  };

  onMount(_ => {
    myp5 = new p5(sketch);
  });

  function addNewLabel() {
    if (toAdd) {
      labels.set(toAdd, []);
      labels = labels;
      newLabel = '';
    }
  }

  function removeLabel(target) {
    labels.delete(target);
    labels = labels;
  }

  function addTrainingExampleFor(target) {
    activeLabelButton = target;
    const dataURL = document.querySelector('#canvas-holder canvas').toDataURL();
    labels.get(target).push(dataURL);
    labels = labels;
    myp5.clearCanvas();
  }
</script>

<style>
  .add-examples-container {
    padding: 5px;
    height: 160px;
    display: flex;
    flex-direction: column;
    flex-wrap: wrap;
    overflow-x: auto;
  }

  .add-example-group {
    margin: 5px 0px;
    display: flex;
    align-items: center;
  }

  .example-count {
    margin: 0px 10px;
  }
</style>

<h1 class="title is-5">Labels</h1>
{#if Array.from(labels.keys()).length < 2}
  <em class="content has-text-danger is-small">Please add at least two labels</em>
{/if}
<div class="field is-grouped is-grouped-multiline">
  {#each Array.from(labels.keys()) as label}
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

<br />
<h1 class="title is-5">Training examples</h1>
<div class="columns">
  <div class="column is-one-quarter" id="canvas-holder" />
  <div class="column add-examples-container">
    {#each Array.from(labels.keys()) as label}
      <div class="add-example-group">
        <div class="example-count">{labels.get(label).length}</div>
        <button
          class="button {activeLabelButton === label ? 'is-info' : ''}"
          on:click={_ => addTrainingExampleFor(label)}>
          Add as {label}
        </button>
      </div>
    {/each}
  </div>
</div>
