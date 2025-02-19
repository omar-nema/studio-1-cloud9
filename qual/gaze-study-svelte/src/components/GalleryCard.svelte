<script>
  import {
    pageState,
    selectedImage,
    screenWidth,
    screenHeight,
    cardInView,
    tooltipText,
    tooltipShow,
    // infoTipIndex,
  } from '../stores/pageState';
  import { dbGet } from '../utils/firebaseUtils.js';
  import { onMount } from 'svelte';
  import { contourMapBlur } from '../utils/generateVisuals';
  import Tooltip from './Tooltip.svelte';
  import { fade } from 'svelte/transition';
  import { updateTooltip } from '../utils/tooltipUtils';
  import GalleryCardTip from './GalleryCardTip.svelte';
  import jump from '../utils/jumpSection';

  //CUSTOM WIDTH AND HEIGHT CALC
  export let data;
  let maxW = Math.min($screenWidth - 600, 800),
    maxH = $screenHeight - 190;
  let width = 'auto',
    ht = 'auto',
    styleSubstring = '';

  let dimWidthToHt = data.width / data.height;
  if (dimWidthToHt < 1) {
    ht = maxH + 'px';
    styleSubstring = 'height: 100%';
  } else {
    width = maxW + 'px';
    styleSubstring = 'width: 100%';
  }

  //GET CURRENT SESSION DATA
  let currSessionKey, currSession, sessionData;
  let currFrame = 0;
  let sessions = [];
  if (data.sessionData) {
    sessions = data.sessionData;
  }
  let sessionsArray = [];
  if (sessions) {
    sessionsArray = Object.keys(sessions);
  }
  let currSessionIndex = sessionsArray.length - 1;
  currSessionKey = sessionsArray[currSessionIndex];
  //SWITCH SESSIONS
  $: {
    currSession = sessions[currSessionKey];
    getSessionData(currSessionKey);
  }
  async function getSessionData(key) {
    sessionData = await dbGet('sessionData/' + key);
    createClips();
    //reset the slider
    if (sessionData) {
      currFrame = 0;
    }
  }
  //RE-CALCULATE SLIDER STEPS ON SESSION SWITCH
  let sliderMax = 100;
  $: {
    if (sessionData) {
      sliderMax = sessionData.length;
    }
    if (viewMode == 'aggregate') {
      contourMapBlur(
        sessionData,
        `#${data.key} .img-holder`,
        `#${data.key}-contour`,
        data.url
      );
    }
  }

  //helper text insantiation
  let helperTextPositions = [[], [0, 0], [0, 0]];
  let imgNav, visFilter, gazeBtn, card;
  function updateHelperTextPos(index, clientRect, cardRect) {
    let x, y;

    x = Math.min(
      clientRect.x + clientRect.width / 4 - cardRect.x,
      cardRect.width - 450
    );

    if (index == 0) {
      y = clientRect.y - cardRect.y - 150;
    } else {
      y = clientRect.y - cardRect.y + 50;
    }
    helperTextPositions[index] = [x, y];
  }
  $: {
    if (imgNav && visFilter && gazeBtn) {
      updateHelperTextPos(
        0,
        imgNav.getBoundingClientRect(),
        card.getBoundingClientRect()
      );
      updateHelperTextPos(
        1,
        visFilter.getBoundingClientRect(),
        card.getBoundingClientRect()
      );
      updateHelperTextPos(
        2,
        gazeBtn.getBoundingClientRect(),
        card.getBoundingClientRect()
      );
    }
  }

  //managing slices
  let viewMode = 'slice'; //other mode = aggregate
  let playStatus = 'pause';

  //SLICE VISUAL - PROB BEST TO MOVE OUT
  $: {
    if (playStatus == 'play' && currFrame < sessionData.length - 1) {
      setTimeout(() => {
        currFrame++;
      }, 50);
    } else if (playStatus == 'play' && currFrame == sessionData.length - 1) {
      playStatus = 'pause';
    }
  }
  $: {
    currFrame;
    if (sessionData && currFrame < sessionData.length - 1) {
      moveClips(sessionData[currFrame].xPct, sessionData[currFrame].yPct);
    }
  }
  let clips = [];
  function createClips() {
    clips = [];
    let numClips = 40;
    let clipMaxSize = 33;
    let clipMinR = 19;
    let clipInc = (clipMaxSize - clipMinR) / numClips;
    let blurMax = 10;
    let blurInc = blurMax / numClips;
    let opacityInc = 1 / numClips;
    for (let i = numClips; i > 0; i--) {
      let r = clipInc * i + clipMinR;
      let blur = blurInc * i;
      let opacity = opacityInc * (numClips - i);
      clips.push({
        class: 'clip',
        r: r,
        ctrx: sessionData[0].xPct,
        ctry: sessionData[0].yPct,
        blur: blur,
        opacity: opacity,
        src: data.url,
      });
    }
  }
  let clipHolder, domClips;
  $: {
    if (clipHolder) {
      domClips = clipHolder.childNodes;
    }
  }
  function moveClips(centerx, centery) {
    domClips.forEach((d) => {
      let currClipPath = d.style['clip-path'];
      let split = currClipPath.split('at ');
      let prefix = split[0];
      let newCtr = `at ${centerx}% ${centery}%)`;
      d.style['clip-path'] = prefix + newCtr;
    });
  }
  $: clips;
  let infoTipIndex = -1;
</script>

{#if $tooltipShow}
  <div transition:fade={{ duration: 100 }}>
    <Tooltip />
  </div>
{/if}

<div
  class="card-outer"
  id={data.key}
  class:active={data.key == $cardInView}
  bind:this={card}
>
  {#if infoTipIndex >= 0}
    <GalleryCardTip {helperTextPositions} bind:infoTipIndex />
  {/if}
  <div class="card-header">
    <h2 style="display: flex; align-items: center;">
      <div
        style="font-weight: 400; color: rgb(126 123 123);margin-right: 15px;"
      >
        Gaze Collection
      </div>
      <a
        class="clickable"
        style="display: flex; align-items: center;"
        href={data.origLink}
        target="_blank"
      >
        <span>
          {data.artist}, <i>{data.title}</i>
        </span>
        <span
          class="material-icons-round"
          style="font-size: 12px; margin-left: 6px"
        >
          open_in_new
        </span>
      </a>
    </h2>
    {#if $screenWidth > 800}
      <div
        class="clickable"
        on:click={() => {
          jump(data.key);
          infoTipIndex = 0;
          // infoTipIndex.set(0);
        }}
      >
        <span class="material-icons-round md-14" style="color: #bfb9b9"
          >info</span
        >
      </div>
    {/if}
  </div>

  <div class="card-filters">
    <div
      class="visual-filter filter-group"
      bind:this={visFilter}
      class:info-highlight={infoTipIndex == 1}
    >
      <div class="filter-options">
        <div
          class="filter time clickable"
          class:selected={viewMode == 'slice'}
          on:click={() => {
            if (playStatus == 'pause') {
              viewMode = 'slice';
              console.log('session length', sessionData.length);
              if (
                currFrame == sessionData.length ||
                currFrame == sessionData.length - 1
              ) {
                currFrame = 0;
              }
              playStatus = 'play';
            } else if (viewMode == 'slice' && playStatus !== 'pause') {
              playStatus = 'pause';
            }
          }}
        >
          {#if playStatus == 'pause'}
            <span class="play-toggle">Animate</span>
          {:else}
            <span
              on:click={() => {
                // playStatus = 'pause';
              }}
              class="play-toggle">Pause</span
            >
          {/if}

          <span id="slider-holder">
            <input
              type="range"
              id="slider"
              name="slider"
              min="0"
              max={sliderMax}
              step="1"
              on:input={() => {
                viewMode = 'slice';
              }}
              bind:value={currFrame}
            />
          </span>
        </div>
        <div
          class="filter clickable"
          class:selected={viewMode == 'aggregate'}
          on:click={() => {
            playStatus = 'pause';
            viewMode = 'aggregate';
            contourMapBlur(
              sessionData,
              `#${data.key} .img-holder`,
              `#${data.key}-contour`,
              data.url
            );
          }}
        >
          <!-- <span class="material-icons-round md-14">image</span> -->
          <span>Aggregate</span>
        </div>
        <div
          class="filter clickable"
          class:selected={viewMode == 'original'}
          on:click={() => {
            viewMode = 'original';
          }}
        >
          <!-- <span class="material-icons-round md-14">compare</span> -->
          <span>Original</span>
        </div>
      </div>
    </div>
    <div
      class="viewer-filter  filter-group"
      class:info-highlight={infoTipIndex == 2}
    >
      <!-- <div class="label compact">
        <span class="material-icons-round md-14">people</span>
        <span>Viewer</span>
      </div> -->
      {#if $screenWidth > 950}
        <div class="filter-options">
          <!-- <div class="label compact">
          <span class="material-icons-round md-14">add</span>
  
        </div> -->
          <div
            bind:this={gazeBtn}
            class="filter clickable add"
            on:click={() => {
              selectedImage.set(data);
              pageState.set('record');
            }}
          >
            <span> Add Your Gaze</span>
          </div>
        </div>
      {/if}
    </div>
  </div>
  <div class="center">
    <div
      class="img-holder"
      style="width: {width}; height: {ht}; max-width: {data.width}px; max-height: {data.height}px"
      on:mouseover={(e) => {
        // let str;
        // let name = sessions[sessionsArray[currSessionIndex]].name;
        // if (viewMode == 'aggregate') {
        //   str = `Visualization of ${name}'s Gaze. Focused portions of the image represent where ${name} was looking the most, while blurred portions were paid less attention to. `;
        // } else if (viewMode == 'slice' && playStatus == 'pause') {
        //   str = `Currently displaying a single slice - a frame representing just 0.05 seconds of viewing. Click on the Animate button on the top left to play back ${name}'s viewing session.`;
        // } else if (viewMode == 'slice' && playStatus == 'play') {
        //   str = `Playing back ${name}'s viewing session. The focused section represents what they were looking at a given moment.`;
        // } else if (viewMode == 'original') {
        //   str = `Showing ${data.artist}'s ${data.title}, without a gaze representation.`;
        // }
        // updateTooltip(e.x, e.y, str);
      }}
      on:mousemove={(e) => {
        // updateTooltip(e.x, e.y);
      }}
      on:mouseleave={(e) => {
        // updateTooltip();
      }}
    >
      <img
        class:slice={viewMode == 'slice'}
        class:agg={viewMode == 'aggregate'}
        src={data.url}
        style={styleSubstring}
        class="main"
      />
      <svg
        class="contour"
        class:active={viewMode == 'aggregate'}
        id="{data.key}-contour"
        style="width: 100%; height: 100%; position: absolute; top:0; left:0; z-index:10"
      />
      {#if viewMode == 'slice'}
        <div bind:this={clipHolder}>
          {#each clips as clip}
            <img
              class="clip"
              style="clip-path: circle({clip.r}% at {clip.ctrx}% {clip.ctry}%); filter: blur({clip.blur}px);opacity: ${clip.opacity}"
              src={data.url}
            />
          {/each}
        </div>
      {/if}

      <!-- <img src={data.url} style={styleSubstring} /> -->
    </div>

    <div
      class="filter person"
      class:info-highlight={infoTipIndex == 0}
      class:info-hide={viewMode == 'original'}
    >
      <div
        class="arrow-nav clickable"
        class:disabled={currSessionIndex == 0}
        on:click={() => {
          currSessionIndex--;
          currSessionKey = sessionsArray[currSessionIndex];
        }}
      >
        <span class="material-icons-round md-18 nav" style="font-size: 26px"
          >arrow_left</span
        >
        <span>Prev</span>
      </div>

      <select class="clickable" bind:value={currSessionKey} bind:this={imgNav}>
        {#each sessionsArray as session, index}
          <option value={session}>
            <span style="font-weight: 600; color: black;"
              >{sessions[session].name}'s Gaze</span
            >
            <span style="font-weight: 400; color: gray"
              >({index + 1} of {sessionsArray.length})</span
            >
          </option>
        {/each}
      </select>
      <div
        class="clickable arrow-nav"
        class:disabled={currSessionIndex == sessionsArray.length - 1}
        on:click={() => {
          currSessionIndex++;
          currSessionKey = sessionsArray[currSessionIndex];
        }}
      >
        <span>Next</span>
        <span class="material-icons-round md-18 nav" style="font-size: 26px"
          >arrow_right</span
        >
      </div>
    </div>
  </div>
</div>

<style>
  .card-outer {
    opacity: 0.2;
    position: relative;
    transition: opacity 0.3s ease-in-out;
  }
  .card-outer.active {
    opacity: 1;
  }
  .card-header {
    display: flex;
  }

  #contour-overlay {
    background: black;
  }
  .card-filters {
    display: flex;
    align-items: center;
    font-weight: 500;
    flex-wrap: wrap;
    flex-direction: row;
    margin-bottom: 25px;
    justify-content: space-between;
  }
  a {
    color: black;
    transition: all 0.15s linear;
  }
  a:hover {
    text-decoration: none;
  }

  p {
    margin-top: 10px;
    padding: 0;
  }
  h2 {
    text-align: left;
    width: 100%;
    margin-top: 0;
  }
  i {
    text-decoration: underline;
    text-decoration-color: gray;
  }

  .label .material-icons-round {
    margin-right: 10px;
    color: gray;
  }
  .material-icons-round {
    font-size: 18px;
    margin-right: 5px;
  }

  .img-holder {
    max-width: 100%;
    max-height: 100%;
    display: flex;
    justify-content: center;
    overflow: hidden;
    position: relative;
    display: inline-block;
    margin: auto;
    border: 1px solid rgba(0, 0, 0, 0.1);
  }

  img.agg {
    filter: blur(5px);
  }
  img.main {
    transition: all 0.3s ease-in-out;
  }
  img.slice {
    filter: blur(10px);
  }

  .filter-group {
    /* min-width: 350px; */
    font-size: var(--font-size-filter);
    height: 34px;
  }
  .info-hide {
    opacity: 0;
  }
  .info-highlight {
    border: 1px dashed var(--color-accent-faded) !important;
    box-shadow: var(--box-shadow-med);
    border-radius: 5px;
    opacity: 1;
  }

  .filter-group,
  .filter-options {
    display: flex;
    flex-direction: row;
  }
  .filter-options {
    overflow: hidden;
    flex-grow: 0.7;
    border-radius: 5px;
  }
  .arrow-nav {
    display: flex;
    align-items: center;
  }

  select {
    background: none;
    border: none;
    padding: 0;
    margin: 0;
    width: 100%;
    text-align: center;
    text-transform: capitalize;
    text-overflow: ellipsis;
    overflow-wrap: break-word;
    -webkit-appearance: none;
    -moz-appearance: none;
    text-indent: 1px;
    text-overflow: '';
    font-weight: 600;
  }
  /* select option {
    color: white;
  } */

  .filter {
    background: var(--bg-contrast-darker);
    padding: 0 10px;
    height: 100%;
    border: 0.5px dashed transparent;
    transition: all 0.15s linear;
    border-right: 0.5px solid rgba(0, 0, 0, 0.1);
  }
  .filter.add {
    background: var(--bg-gradient-dark);
  }
  .filter:last-child {
    border-right: none;
  }
  .filter.person {
    background: none;
    width: 100%;
    margin: auto;
    padding: 5px 20px;
  }
  .filter.selected {
    background: var(--bg-gradient-dark);
    /* border: 0.5px solid #0000004f; */
  }
  .filter,
  .label {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 10px 20px;
  }
  .label.compact {
    padding-left: 0;
    padding-right: 0;
  }

  .name {
    padding: 0 10px;
    min-width: 100px;
    text-align: center;
  }

  .clickable:hover {
    color: var(--color-accent);
  }
  .play-toggle {
    width: 50px;
  }

  .disabled {
    opacity: 0.1;
    pointer-events: none;
  }
  .viewer-filter {
    margin-right: 15px;
  }

  input {
    margin: 0;
    padding: 0;
  }

  #slider-holder {
    display: flex;
    margin-left: 15px;
    align-items: center;
    width: 50px;
  }
  input {
    width: 100%;
  }

  .filter.time span {
    margin: 0 4px;
  }
  input[type='range'] {
    -webkit-appearance: none !important;
    height: 7px;
    border: none;
    border-radius: 0;
    background: var(--bg-contrast);
  }
  input[type='range']::-webkit-slider-thumb {
    -webkit-appearance: none !important;
    background: white;
    border-radius: 100%;
    border: none;
    box-shadow: none;
    height: 15px;
    width: 15px;
    cursor: pointer !important;
    transition: all 0.1s linear;
    box-shadow: var(--box-shadow-light);
  }
  input[type='range']::-ms-fill-lower {
    background: blue !important;
  }
  input[type='range']::-moz-range-thumb {
    -webkit-appearance: none !important;
    background: white;
    border-radius: 100%;
    border: none;
    box-shadow: none;
    height: 15px;
    width: 15px;
    cursor: pointer !important;
    transition: all 0.15s ease-in-out;
    box-shadow: var(--box-shadow-light);
  }
  input[type='range']::-moz-range-thumb:hover {
    box-shadow: 0 0 2px 2px rgba(0, 0, 0, 0.2);
  }

  input[type='range']::-webkit-slider-thumb:hover {
    box-shadow: 0 0 2px 2px rgba(0, 0, 0, 0.2);
  }
  input[type='range']:focus {
    outline: none; /* Removes the blue border. You should probably do some kind of focus styling for accessibility reasons though. */
  }

  .clip {
    position: absolute;
    top: 0;
    left: 0;
    max-width: 100%;
    margin: auto;
    opacity: 1;
    transition: all 0.1s ease-in-out;
  }

  .center {
    display: flex;
    justify-content: center;
    flex-direction: column;
  }

  .contour {
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.15s ease-in-out;
  }
  .contour.active {
    opacity: 1;
  }
</style>
