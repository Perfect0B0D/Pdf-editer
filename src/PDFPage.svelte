<script>
  import { onMount, onDestroy, createEventDispatcher } from "svelte";
  export let page;
  const dispatch = createEventDispatcher();
  let canvas;
  export let width;
  export let height;
  let clientWidth;
  let mounted;
  let context;
  function measure() {
    dispatch("measure", {
      scale: canvas.clientWidth / width
    });
  }
  async function render() {
    const _page = await page;
    context = canvas.getContext("2d");
    const viewport = _page.getViewport({ scale: 2, rotation: 0 });
    width = viewport.width;
    height = viewport.height;
    console.log("view port==>",width,height);
    await _page.render({
      canvasContext: context,
      viewport
    }).promise;
    measure();
    window.addEventListener("resize", measure);
  }
  onMount(render);
  onDestroy(() => {
    window.removeEventListener("resize", measure);
  });
 

</script>

<div>
  <canvas
    bind:this={canvas}
    class="max-w-full"
    style="width: {width}px;"
    {width}
    {height}
    
    />
</div>
