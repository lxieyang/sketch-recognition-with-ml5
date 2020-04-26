<script>
  import { onMount } from 'svelte';
  import p5 from 'p5';
  import JSZip from 'jszip/dist/jszip.min';
  import FileSaver from 'file-saver';

  let myp5;
  let canvasWidth = 200;
  let canvasHeight = 150;
  let featureExtractor;
  let classifier;
  let modelStatus = '';

  let labels = new Map();
  labels.set('Rectangle', []);
  labels.set('Circle', []);
  labels.set('Line', []);
  $: canTrain = checkCanTrain(labels);
  let modelCanBeSaved = false;
  let newLabel = '';
  $: toAdd = newLabel.trim().toLowerCase();

  let activeLabelButton = '';

  let modalOpen = false;
  let modalLabel = '';

  let mode = 'train'; // 'train' or 'test'
  let classificationResults = [
    // { label: 'rect', confidence: 0.7213 },
    // { label: 'line', confidence: 0.05 },
    // { label: 'circle', confidence: 0.23 },
  ];

  function checkCanTrain(lbs = new Map()) {
    let cantrain = true;
    if (lbs.size === 0) {
      cantrain = false;
    }
    Array.from(lbs.keys()).forEach(lb => {
      if (lbs.get(lb) && lbs.get(lb).length === 0) {
        cantrain = false;
      }
    });
    return cantrain;
  }

  let sketch = function(p5) {
    let canvas;
    // let canvasWidth = 200;
    // let canvasHeight = 150;

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
    const canvasHolderDiv = document.querySelector('#canvas-holder');
    canvasWidth = canvasHolderDiv.clientWidth;
    canvasHeight = canvasHolderDiv.clientHeight;

    myp5 = new p5(sketch);

    window.addEventListener('beforeunload', event => {
      let msg = 'Are you sure you want to leave?';
      // Chrome requires returnValue to be set.
      event.returnValue = msg;
      return msg;
    });
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
            modelCanBeSaved = true;
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

  function saveModel() {
    classifier.save();
    let zip = new JSZip();
    let imgs = zip.folder('training-images');
    Array.from(labels.keys()).forEach(lb => {
      let examples = labels.get(lb);
      examples.forEach((example, idx) => {
        imgs.file(`${lb}-${idx + 1}.png`, example.split(',')[1], { base64: true });
      });
    });
    zip.generateAsync({ type: 'blob' }).then(function(content) {
      FileSaver.saveAs(content, 'training-examples.zip');
    });
  }

  function loadModel(e) {
    modelStatus = 'Loading custom models...';
    featureExtractor = ml5.featureExtractor('MobileNet', { numLabels: Array.from(labels.keys()).length }, () => {
      modelStatus = 'MobileNet loaded...';
      classifier.load(e.target.files).then(_ => {
        modelStatus = 'Custom model ready!';
      });
    });

    classifier = featureExtractor.classification();
  }

  async function loadTrainingImages(e) {
    const f = e.target.files[0];

    if (f.type !== 'application/zip') {
      alert('Need a zip file.');
      return;
    }

    Object.filter = (obj, predicate) =>
      Object.keys(obj)
        .filter(key => predicate(obj[key]))
        .reduce((res, key) => ((res[key] = obj[key]), res), {});

    let images = Object.filter((await JSZip.loadAsync(f)).files, f => f.dir === false);

    const filenames = Object.keys(images);
    let newLabels = new Map();
    const strippedFilenames = new Set(
      filenames.map(fn => fn.replace('training-images/', '')).map(fn => fn.split('-')[0]),
    );
    strippedFilenames.forEach(lb => {
      newLabels.set(lb, []);
    });

    for (let i = 0; i < filenames.length; i++) {
      let content = await images[filenames[i]].async('base64');
      let label = filenames[i].replace('training-images/', '').split('-')[0];

      newLabels.get(label).push('data:image/png;base64,' + content);
    }

    labels = newLabels;
  }

  function closeModal() {
    modalOpen = false;
    modalLabel = '';
  }
</script>

<style>
  .labels-container {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
  }

  .labels-container .labels-title {
    margin-right: 6px;
  }

  .canvas-container {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    /* height: 190px; */
  }

  #canvas-holder {
    width: 200px;
    height: 160px;
    margin-right: 10px;
  }

  .add-examples-container {
    flex: 1;
    flex-basis: 300px;
    padding: 5px;
    height: 100%;
    display: flex;
    flex-direction: column;
    flex-wrap: wrap;
    overflow: auto;
  }

  .add-example-group {
    margin: 5px 0px;
    display: flex;
    align-items: center;
    flex-wrap: wrap;
  }

  .add-example-group .buttons {
    margin-bottom: 0px;
  }

  .add-example-group .buttons .button {
    margin-bottom: 0px;
  }

  .example-images-container {
    flex: 1;
    height: 40px;
    display: flex;
    align-items: center;
  }

  .example-images-container img {
    border: 1px dotted lightgray;
    margin: 0px 4px;
    height: 80%;
  }

  .example-in-modal {
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 6px;
  }

  .example-in-modal img {
    width: 300px;
    border: 1px dotted lightgray;
    margin-right: 10px;
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

  <div class="labels-container">
    <h1 class="labels-title is-capitalized is-size-4 has-text-weight-bold">Labels:</h1>
    <div class="field is-grouped is-grouped-multiline">
      {#each Array.from(labels.keys()) as label}
        <div class="control">
          <div class="tags has-addons">
            <span class="tag is-link is-light is-medium">{label}</span>
            {#if mode === 'train'}
              <span class="tag is-delete is-medium" on:click={e => removeLabel(label)} />
            {/if}
          </div>
        </div>
      {/each}
    </div>
  </div>

  {#if Array.from(labels.keys()).length < 2}
    <em class="content has-text-danger is-small">Please add at least two labels</em>
  {/if}

  {#if mode === 'train'}
    <div class="field has-addons" style="margin-top: 5px;">
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
  <div class="canvas-container">
    <div class="">
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
      <div class="add-examples-container">
        {#each Array.from(labels.keys()) as label}
          <div class="add-example-group">
            <div class="buttons has-addons">
              <button
                class="button is-small {activeLabelButton === label ? 'is-info' : ''}"
                on:click={_ => addTrainingExampleFor(label)}>
                Add as a
                <span class="has-text-weight-bold" style="margin-left: 4px">{label}</span>
              </button>
              <button
                title="Edit examples"
                class="button is-small"
                on:click={_ => {
                  modalLabel = label;
                  modalOpen = true;
                }}>
                Count: {labels.get(label).length}
              </button>
            </div>

            <div class="example-images-container">
              {#each [...labels.get(label)].reverse().slice(0, 5) as img}
                <img src={img} alt="" />
              {/each}
              {#if labels.get(label).length > 5}...{/if}
            </div>

          </div>
        {/each}
      </div>
    {/if}
  </div>

  <div class="modal {modalOpen ? 'is-active' : ''}">
    <div class="modal-background" />
    <div class="modal-card">
      <header class="modal-card-head">
        <p class="modal-card-title">
          Training examples for {modalLabel}
          {#if modalLabel.length > 0}(Total: {labels.get(modalLabel).length}){/if}
        </p>
        <button class="delete" aria-label="close" on:click={closeModal} />
      </header>
      {#if modalLabel.length > 0}
        <section class="modal-card-body">
          {#if labels.get(modalLabel).length === 0}
            <em class="content has-text-danger ">No training examples available. Please add some!</em>
          {/if}
          {#each [...labels.get(modalLabel)].reverse() as img}
            <div class="example-in-modal">
              <img src={img} alt="" />
              <button
                class="button is-danger"
                on:click={_ => {
                  let examples = labels.get(modalLabel).filter(item => item !== img);
                  labels.set(modalLabel, examples);
                  labels = labels;
                }}>
                Delete
              </button>
            </div>
          {/each}

        </section>
      {/if}
      <footer class="modal-card-foot">
        <button class="button" on:click={closeModal}>Close</button>
      </footer>
    </div>
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
      <button class="button is-success" disabled={!modelCanBeSaved} on:click={saveModel}>
        Save Model & Training examples
      </button>
      <br />
      <br />
      <span class="tag is-info">Load model:</span>
      <input type="file" name="myfile" multiple on:change={loadModel} />
      <br />
      <br />
      <span class="tag is-info">Load training images:</span>
      <input type="file" name="myfile" on:change={loadTrainingImages} />
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
