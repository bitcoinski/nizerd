<svelte:head>
  <meta property="og:image" content="thumbnail.jpg" />
</svelte:head>

<script>

  // This interactive Tron-like canvas illustrates the above formula. It's run once every 33.33ms, when each pixel on the canvas is scanned left-to-right, top-to-bottom: x = pixel's x position | y = pixel's y position | m.x = mouse's x position | m.y = mouse's y position. Inspired by Martin Kleppe's tweet here: https://twitter.com/aemkei/status/1378106731386040322?s=20 
  
  import { onMount } from "svelte";

  let context;

  let factor = 1;
  let width = window.screen.availWidth;
  let height = window.screen.availHeight;
  let m = { x: width/ 2, y: height/2 };

  onMount(async () => {
    context = c.getContext("2d");
    console.log('mounted')
  });

  

  let mouseMoving = false;
  function handleMousemove(event) {
    mouseMoving = true;
    m.x = event.clientX;
    m.y = event.clientY;
    draw();
  }

  const randomNumber = (min, max) =>
    Math.floor(Math.random() * (max - min + 1) + min);
  const randomByte = () => randomNumber(0, 255);
  const randomPercent = () => (randomNumber(50, 100) * 0.01).toFixed(2);
  const randomCssRgba = () =>
    `rgba(${[randomByte(), randomByte(), randomByte(), randomPercent()].join(
      ","
    )})`;

    
      
  const draw = () => {
    context.clearRect(0, 0, width, height);
    context.fillStyle = "rgba(0,0,0,1)";
    context.fillRect(0, 0, width, height);
    
    for (let x = 0; x < width; x++) {
      for (let y = 0; y < height; y++) {
        if (((x - m.x / factor) * m.x) % (y - m.y / factor)) {
          context.fillStyle = 'black';
        } else {
          context.fillStyle = 'white';
        }
        context.fillRect(
            x * factor,
            y * factor,
            factor * factor,
            factor * factor
          );
      }
    }

    mouseMoving = false;
  };

  
  
</script>


<canvas id="c" width="{width}" height="{height}" on:mousemove={handleMousemove}/>

<style>
  html {
    background-color: black !important;
  }

</style>
