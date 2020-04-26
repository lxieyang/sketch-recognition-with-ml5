<script>
  import { onMount } from 'svelte';
  import p5 from 'p5';
  import ml5 from 'ml5';

  let myp5;
  let featureExtractor;
  let classifier;
  let modelStatus = '';

  let labels = new Map();
  labels.set('rect', []);
  labels.set('circle', []);
  labels.set('line', []);
  $: canTrain = checkCanTrain(labels);
  let newLabel = '';
  $: toAdd = newLabel.trim().toLowerCase();

  let activeLabelButton = '';

  let mode = 'train'; // 'train' or 'test'
  let classificationResults = [
    // { label: 'rect', confidence: 0.7213 },
    // { label: 'line', confidence: 0.05 },
    // { label: 'circle', confidence: 0.23 },
  ];

  function checkCanTrain(lbs = new Map()) {
    let cantrain = true;
    Array.from(lbs.keys()).forEach(lb => {
      if (lbs.get(lb) && lbs.get(lb).length === 0) {
        cantrain = false;
      }
    });
    return cantrain;
  }

  let sketch = function(p5) {
    let canvas;
    let canvasWidth = 200;
    let canvasHeight = 150;

    p5.setup = () => {
      canvas = p5.createCanvas(canvasWidth, canvasHeight);
      canvas.parent('canvas-holder');
      p5.background('white');
      p5.strokeWeight(6);
      p5.stroke(0);
    };

    p5.touchMoved = () => {
      p5.line(p5.mouseX, p5.mouseY, p5.pmouseX, p5.pmouseY);
      return false;
    };

    p5.touchEnded = () => {
      if (p5.mouseX >= 0 && p5.mouseY >= 0 && p5.mouseX <= canvasWidth && p5.mouseY <= canvasHeight) {
        if (mode === 'test' && classifier && classify) {
          classify();
        }
      }
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

  function trainModel() {
    modelStatus = 'loading model...';
    featureExtractor = ml5.featureExtractor('MobileNet', { numLabels: Array.from(labels.keys()).length }, () => {
      modelStatus = 'adding training examples...';
      let promises = [];
      Array.from(labels.keys()).forEach(lb => {
        const examples = labels.get(lb);
        examples.forEach(dataURL => {
          let img = document.createElement('img');
          img.src = dataURL;
          promises.push(classifier.addImage(img, lb));
        });
      });
      Promise.all(promises).then(() => {
        console.log('training examples added');
        classifier.train(function(lossValue) {
          if (lossValue) {
            modelStatus = 'training, loss: ' + lossValue;
          } else {
            modelStatus = 'done!';
          }
        });
      });
    });

    classifier = featureExtractor.classification();
  }

  function classify() {
    let img = document.createElement('img');
    img.src = document.querySelector('#canvas-holder canvas').toDataURL();
    classifier.classify(img, (err, results) => {
      if (err) {
        console.log(err);
      }
      // console.log(results);
      if (results && results[0]) {
        classificationResults = results;
        // myp5.clearCanvas();
      }
    });
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

  .result-item {
    display: flex;
    align-items: center;
    margin: 8px;
  }

  .result-item .item {
    display: flex;
    align-items: center;
    min-width: 110px;
    margin-right: 8px;
  }

  .result-item .item .tag {
    margin: 0px 4px;
  }

  .result-item .progress-bar-container {
    margin: 0px 4px;
    flex: 1;
  }
</style>

<div class="container">
  <h1 class="title is-3">
    Simple Sketch Recognition with
    <a href="https://ml5js.org/" target="_blank">ML5</a>
  </h1>

  <h1 class="title is-5">Labels</h1>
  {#if Array.from(labels.keys()).length < 2}
    <em class="content has-text-danger is-small">Please add at least two labels</em>
  {/if}
  <div class="field is-grouped is-grouped-multiline">
    {#each Array.from(labels.keys()) as label}
      <div class="control">
        <div class="tags has-addons">
          <span class="tag is-link is-light">{label}</span>
          {#if mode === 'train'}
            <span class="tag is-delete" on:click={e => removeLabel(label)} />
          {/if}
        </div>
      </div>
    {/each}
  </div>

  {#if mode === 'train'}
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
  {/if}

  <br />
  <div class="columns">
    <div class="column is-one-third">
      <div id="canvas-holder" />
      <button
        class="button is-light is-small"
        on:click={_ => {
          myp5.clearCanvas();
          if (mode === 'test') {
            classificationResults = [];
          }
        }}>
        Clear
      </button>
    </div>
    {#if mode === 'train'}
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
    {/if}
  </div>

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
    <div class="title is-6">
      Status
      <span class="content has-background-warning has-text-dark">{modelStatus}</span>
    </div>
    <div>
      <button class="button is-link" disabled={!canTrain} on:click={trainModel}>Train</button>
    </div>
  {:else if mode === 'test'}
    {#if classificationResults.length === 0}
      <div class="title is-6">Start by drawing on the canvas</div>
    {:else}
      <div class="title is-6">Classification results:</div>
    {/if}
    {#each classificationResults as item, idx}
      <div class="result-item">
        <div class="item">
          Label:
          <span class="tag {idx === 0 ? 'is-success' : ''}">{item.label}</span>
        </div>
        <div class="item">
          Confidence:
          <span class="tag {idx === 0 ? 'is-success is-light' : ''}">{item.confidence.toFixed(4)}</span>
        </div>

        <div class="progress-bar-container">
          <progress class="progress {idx === 0 ? 'is-success' : ''}" value={item.confidence} max="1">
            {item.confidence}
          </progress>
        </div>

      </div>
    {/each}
  {/if}
</div>
