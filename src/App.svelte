<script>
  import { onMount } from "svelte";
  import { fly } from "svelte/transition";
  import Tailwind from "./Tailwind.svelte";
  import PDFPage from "./PDFPage.svelte";
  import Image from "./Image.svelte";
  import Text from "./Text.svelte";
  import Drawing from "./Drawing.svelte";
  import DrawingCanvas from "./DrawingCanvas.svelte";
  import prepareAssets, { fetchFont } from "./utils/prepareAssets.js";
  import {
    readAsArrayBuffer,
    readAsImage,
    readAsPDF,
    readAsDataURL
  } from "./utils/asyncReader.js";
  import { ggID } from "./utils/helper.js";
  import { save } from "./utils/PDF.js";
  console.log("viewport===>",PDFPage.width,PDFPage.height);
  const genID = ggID();
  let ischeckclick=false;
  let iscloseclick=false;
  let pdfFile;
  let pdfName = "";
  let pages = [];
  let pagesScale = [];
  let allObjects = [];
  let currentFont = "Times-Roman";
  let focusId = null;
  let selectedPageIndex = -1;
  let saving = false;
  let addingDrawing = false;
  let toolbar_click = 0;
  let pos = { x: 0, y: 0 };

  // for test purpose
  onMount(async () => {
    try {
      const res = await fetch("/test.pdf");
      const pdfBlob = await res.blob();
      await addPDF(pdfBlob);
      selectedPageIndex = 0;
      setTimeout(() => {
        fetchFont(currentFont);
        prepareAssets();
      }, 5000);
      // const imgBlob = await (await fetch("/test.jpg")).blob();
      // addImage(imgBlob);
      // addTextField("測試!");
      // addDrawing(200, 100, "M30,30 L100,50 L50,70", 0.5);
    } catch (e) {
      console.log(e);
    }
  });
  async function onUploadPDF(e) {
    const files = e.target.files || (e.dataTransfer && e.dataTransfer.files);
    const file = files[0];
    console.log("PDF file===>",file);
    if (!file || file.type !== "application/pdf") return;
    selectedPageIndex = -1;
    try {
      await addPDF(file);
      selectedPageIndex = 0;
    } catch (e) {
      console.log(e);
    }
  }
  async function addPDF(file) {
    try {
      const pdf = await readAsPDF(file);
      pdfName = file.name;
      pdfFile = file;
      const numPages = pdf.numPages;
      pages = Array(numPages)
        .fill()
        .map((_, i) => pdf.getPage(i + 1));
      allObjects = pages.map(() => []);
      pagesScale = Array(numPages).fill(2);
    } catch (e) {
      console.log("Failed to add pdf.");
      throw e;
    }
  }
  async function onUploadImage(e) {
    const file = e.target.files[0];
    console.log(file);
    if (file && selectedPageIndex >= 0) {
      addImage(file);
    }
    e.target.value = null;
  }
  async function checkimg(){
    if(toolbar_click==1){
      toolbar_click=0;
    }else if(toolbar_click==2){
    toolbar_click=1;
    iscloseclick = !iscloseclick;
  }else{
    toolbar_click=1;
  }
    ischeckclick = !ischeckclick;
  }
  async function addcheckimg(){
    let imageUrl = "../check_img/check-mark.png";
    var res = await fetch(imageUrl);
    var blob = await res.blob();
    console.log("toolBar check===>",toolbar_click);
     addImage(blob);
  }
  async function closeimg(){
    if(toolbar_click==2){
      toolbar_click=0;
    }else if(toolbar_click==1){
    toolbar_click=2;
    ischeckclick = !ischeckclick;
  }else{
    toolbar_click=2;
  }
    iscloseclick = !iscloseclick;
  }
  async function addcloseimg(){
    let imageUrl = "../check_img/close-cross.png";
    var res = await fetch(imageUrl);
    var blob = await res.blob();
    console.log("toolBar check===>",toolbar_click);
    addImage(blob);
   
  }
  async function addImage(file) {
    try {
      // get dataURL to prevent canvas from tainted
      const url = await readAsDataURL(file);
      const img = await readAsImage(url);
      // console.log('img', img);
      const id = genID();
      const { width, height } = img;
      const object = {
        id,
        type: "image",
        width,
        height,
        x: pos.x,
        y: pos.y,
        payload: img,
        file
      };
      allObjects = allObjects.map((objects, pIndex) =>
        pIndex === selectedPageIndex ? [...objects, object] : objects
      );
    } catch (e) {
      console.log(`Fail to add image.`, e);
    }
  }
  function onAddTextField() {
    if (selectedPageIndex >= 0) {
      addTextField();
    }
  }
  function addTextField(text = "New Text Field") {
    const id = genID();
    fetchFont(currentFont);
    const object = {
      id,
      text,
      type: "text",
      size: 16,
      width: 0, // recalculate after editing
      lineHeight: 1.4,
      fontFamily: currentFont,
      x: 0,
      y: 0
    };
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex === selectedPageIndex ? [...objects, object] : objects
    );
  }
  function onAddDrawing() {
    if (selectedPageIndex >= 0) {
      addingDrawing = true;
    }
  }
  function addDrawing(originWidth, originHeight, path, scale = 1) {
    const id = genID();
    const object = {
      id,
      path,
      type: "drawing",
      x: 0,
      y: 0,
      originWidth,
      originHeight,
      width: originWidth * scale,
      scale
    };
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex === selectedPageIndex ? [...objects, object] : objects
    );
  }
  function selectFontFamily(event) {
    const name = event.detail.name;
    fetchFont(name);
    currentFont = name;
  }
  function selectPage(index) {
    console.log("toolbar click===>",toolbar_click);
    
    selectedPageIndex = index;
  }
  function updateObject(objectId, payload) {
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex == selectedPageIndex
        ? objects.map(object =>
            object.id === objectId ? { ...object, ...payload } : object
          )
        : objects
    );
  }
  function deleteObject(objectId) {
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex == selectedPageIndex
        ? objects.filter(object => object.id !== objectId)
        : objects
    );
  }
  function onMeasure(scale, i) {
    pagesScale[i] = scale;
  }
  // FIXME: Should wait all objects finish their async work
  async function savePDF() {
    if (!pdfFile || saving || !pages.length) return;
    // console.log("PDF File===> ", pdfFile);
    // console.log("allObjects===> ", allObjects);
    // console.log("pdfName===> ", pdfName);
    // console.log("pagesScale===> ", pagesScale);
    saving = true;
    try {
      await save(pdfFile, allObjects, pdfName, pagesScale);
    } catch (e) {
      console.log(e);
    } finally {
      saving = false;
    }
  }
</script>
<!-- //style -->
<style>
	.toolbar_click {
		background-color: rgba(231, 37, 111, 0.356);
	}
</style>
<svelte:window
  on:dragenter|preventDefault
  on:dragover|preventDefault
  on:drop|preventDefault={onUploadPDF} />
<Tailwind />
<main class="flex flex-col items-center py-16 bg-gray-100 min-h-screen">
  <div
    class="fixed z-10 top-0 left-0 right-0 h-12 flex justify-center items-center
    bg-gray-200 border-b border-gray-300">
    <input
      type="file"
      name="pdf"
      id="pdf"
      on:change={onUploadPDF}
      class="hidden" />
    <input
      type="file"
      id="image"
      name="image"
      class="hidden"
      on:change={onUploadImage} />
    <label
      class="whitespace-no-wrap bg-blue-500 hover:bg-blue-700 text-white
      font-bold py-1 px-3 md:px-4 rounded mr-3 cursor-pointer md:mr-4"
      for="pdf">
      Choose PDF
    </label>
    <div
      class="relative mr-3 flex h-8 bg-gray-400 rounded-sm overflow-hidden
      md:mr-4">
         <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        for="image"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}>
        <img src="image.svg" alt="An icon for adding images" />
      </label> 
       <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        for="check"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}
        on:click={checkimg}>
        <img class:toolbar_click={ischeckclick} src="check.svg" alt="check image" />
      </label>
      <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        for="fault"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}
         on:click={closeimg}>
        <img class:toolbar_click={iscloseclick} src="close.svg" alt="fault image" />
      </label>
       <label
        class="flex items-center justify-center h-full w-6 hover:bg-gray-500
        cursor-pointer"
        for="v-line"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}>
        <img src="vertical-line.svg" alt="vertical line image" />
      </label>
      <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        for="text"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}
        on:click={onAddTextField}>
        <img src="your.svg" alt="An icon for adding text" />
      </label>
      <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        for="drawing"
        on:click={onAddDrawing}
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}>
        <img src="gesture.svg" alt="An icon for adding drawing" />
      </label>
    </div>
    <div class="justify-center mr-3 md:mr-4 w-full max-w-xs hidden md:flex">
      <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
      <input
        placeholder="Rename your PDF here"
        type="text"
        class="flex-grow bg-transparent"
        bind:value={pdfName} />
    </div>
    <button
      on:click={savePDF}
      class="w-20 bg-blue-500 hover:bg-blue-700 text-white font-bold py-1 px-3
      md:px-4 mr-3 md:mr-4 rounded"
      class:cursor-not-allowed={pages.length === 0 || saving || !pdfFile}
      class:bg-blue-700={pages.length === 0 || saving || !pdfFile}>
      {saving ? 'Saving' : 'Save'}
    </button>
  </div>
  {#if addingDrawing}
    <div
      transition:fly={{ y: -200, duration: 500 }}
      class="fixed z-10 top-0 left-0 right-0 border-b border-gray-300 bg-white
      shadow-lg"
      style="height: 50%;">
      <DrawingCanvas
        on:finish={e => {
          const { originWidth, originHeight, path } = e.detail;
          let scale = 1;
          if (originWidth > 500) {
            scale = 500 / originWidth;
          }
          addDrawing(originWidth, originHeight, path, scale);
          addingDrawing = false;
        }}
        on:cancel={() => (addingDrawing = false)} />
    </div>
  {/if}
  {#if pages.length}
    <div class="flex justify-center px-5 w-full md:hidden">
      <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
      <input
        placeholder="Rename your PDF here"
        type="text"
        class="flex-grow bg-transparent"
        bind:value={pdfName} />
    </div>
    <div class="w-full">
      {#each pages as page, pIndex (page)}
        <div
          class="w-full flex flex-col items-center overflow-hidden pdf-page" style="margin-top: 20px" id="page_{pIndex}"
         
          on:mousedown={() => selectPage(pIndex)}
          on:mousedown="{event => {
            console.log(event);
            let pageE = document.querySelector('.pdf-page');
            let pageContent = document.querySelector('.pdf-content');
            pos.x = event.pageX - (Math.floor(pageE.clientWidth) - Math.floor(pageContent.clientWidth))/2;
            pos.x = pos.x/2;
            pos.y = event.pageY;
            let pH = Math.floor(pageE.clientHeight) + 20;
            console.log('pos, ph>>>>>>>>>>>>', pos, pH);
             if(pos.y>pH){pos.y=(pos.y-83)%pH}else{pos.y=pos.y-83;}
            pos.y/=2;
            console.log("mouse pos===>",pos.x,pos.y);
            if(toolbar_click===1){
                 addcheckimg();                
            }else if(toolbar_click===2){
                 addcloseimg();          
            }else{
              return;
            }
          }}"
          on:touchstart={() => selectPage(pIndex)}>
          <div
            class="relative shadow-lg pdf-content"
            class:shadow-outline={pIndex === selectedPageIndex}>
            <PDFPage          
              on:measure={e => onMeasure(2, pIndex)}
              {page} />
            <div
              class="absolute top-0 left-0 transform origin-top-left"
              style="transform: scale({pagesScale[pIndex]}); touch-action: none;">
              {#each allObjects[pIndex] as object (object.id)}
                {#if object.type === 'image'}
                  <Image
                    on:update={e => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    file={object.file}
                    payload={object.payload}
                    x={object.x}
                    y={object.y}
                    width={object.width}
                    height={object.height}
                    pageScale={pagesScale[pIndex]} />
                {:else if object.type === 'text'}
                  <Text
                    on:update={e => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    on:selectFont={selectFontFamily}
                    text={object.text}
                    x={object.x}
                    y={object.y}
                    size={object.size}
                    lineHeight={object.lineHeight}
                    fontFamily={object.fontFamily}
                    pageScale={pagesScale[pIndex]} />
                {:else if object.type === 'drawing'}
                  <Drawing
                    on:update={e => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    path={object.path}
                    x={object.x}
                    y={object.y}
                    width={object.width}
                    originWidth={object.originWidth}
                    originHeight={object.originHeight}
                    pageScale={pagesScale[pIndex]} />
                {/if}
              {/each}

            </div>
          </div>
        </div>
      {/each}
    </div>
  {:else}
    <div class="w-full flex-grow flex justify-center items-center">
      <span class=" font-bold text-3xl text-gray-500">Drag something here</span>
    </div>
  {/if}
</main>
