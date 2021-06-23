<script>
  // This interactive Tron-like canvas illustrates the above formula. It's run once every 33.33ms, when each pixel on the canvas is scanned left-to-right, top-to-bottom: x = pixel's x position | y = pixel's y position | m.x = mouse's x position | m.y = mouse's y position. Inspired by Martin Kleppe's tweet here: https://twitter.com/aemkei/status/1378106731386040322?s=20

  import { onMount } from "svelte";
  //import fslightbox from "fslightbox";
  import clonedeep from 'lodash';

  let archive = [];
  let collection = [];
  let current_item;
  let current_filter;
  let viewer;
  let search_term;

  onMount(async () => {
     viewer =
      new URLSearchParams(window.location.search).get("viewer") ||
      "tz2NHCNf862d1j9pfEk37vRRwv8tMCE2hJFv";

    //create a gallery
    console.log("creating gallery");
    await createGallery(viewer);

    await refreshFsLightbox();
    
    //attach lightbox listeners & props
    fsLightboxInstances['gallery-lightbox'].props.slideshowTime = 10000;
    fsLightboxInstances['gallery-lightbox'].props.onClose = (e) => current_item = null;
    fsLightboxInstances['gallery-lightbox'].props.onSlideChange = (fsLightbox) => selectItemByIndex(fsLightbox.stageIndexes.current + 1);
    fsLightboxInstances['gallery-lightbox'].props.customToolbarButtons = [{
      viewBox: '0 0 512.001 512.001',
      d: 'm512.001 256c-15.637-16.845-24.952-31.573-80.444-72.508-58.434-41.209-119.14-62.992-175.557-62.992-30.802 0-62.884 6.508-95.176 19.111l-74.967-74.967-21.213 21.213 66.853 66.853c-17.102 8.605-34.172 18.878-51.053 30.783-55.492 40.934-64.807 55.663-80.444 72.507 15.637 16.845 24.952 31.573 80.444 72.508 58.434 41.21 119.14 62.992 175.556 62.992 30.802 0 62.884-6.508 95.176-19.111l74.968 74.968 21.213-21.213-66.853-66.853c17.102-8.605 34.172-18.878 51.053-30.783 55.737-41.238 64.503-55.281 80.444-72.508zm-413.67 48.413c-26.44-18.578-46.614-37.36-57.655-48.412 11.042-11.052 31.215-29.834 57.655-48.412 12.892-9.058 28.257-18.717 45.462-27.467-14.697 21.663-23.293 47.785-23.293 75.878 0 28.122 8.614 54.268 23.338 75.944-14.447-7.342-29.654-16.393-45.507-27.531zm157.669 57.087c-58.173 0-105.5-47.327-105.5-105.5 0-23.688 7.849-45.578 21.081-63.206l147.625 147.625c-17.628 13.232-39.517 21.081-63.206 21.081zm-63.205-189.918c17.628-13.232 39.517-21.082 63.205-21.082 58.173 0 105.5 47.327 105.5 105.5 0 23.688-7.849 45.578-21.081 63.206zm175.419 160.289c14.693-21.662 23.286-47.781 23.286-75.871 0-28.121-8.613-54.266-23.336-75.942 14.448 7.343 29.654 16.392 45.506 27.53 26.44 18.578 46.614 37.36 57.656 48.412-11.039 11.049-31.213 29.833-57.656 48.412-12.89 9.057-28.253 18.712-45.456 27.459z',
      width: '17px',
      height: '17px',
      title: 'Hide/Show Info Plaque',
      onClick: () => {togglePlaque();}
    }];
    console.log(fsLightboxInstances['gallery-lightbox'].props);
  });

  const query = `
  query collectorGallery($address: String!) {
    hic_et_nunc_token_holder(where: {holder_id: {_eq: $address}, quantity: {_gt: "0"}, token: {supply: {_gt: "0"}}}, order_by: {id: desc}) {
      token {
        id
        artifact_uri
        display_uri
        thumbnail_uri
        timestamp
        mime
        title
        description
        supply
        token_tags {
          tag {
            tag
          }
        }
        creator {
          address
        }
        swaps(where: {status: {_eq: "0"}}, order_by: {price: asc}) {
          amount
          amount_left
          creator_id
          price
        }
      }
    }
  }`;

  async function fetchGraphQL(operationsDoc, operationName, variables) {
    const result = await fetch("https://api.hicdex.com/v1/graphql", {
      method: "POST",
      body: JSON.stringify({
        query: operationsDoc,
        variables: variables,
        operationName: operationName,
      }),
    });

    return await result.json();
  }

  async function doFetch() {
    const { errors, data } = await fetchGraphQL(query, "collectorGallery", {
      address: viewer,
    });
    if (errors) {
      console.error(errors);
    }
    const result = data.hic_et_nunc_token_holder;
    console.log({ result });
    return result;
  }

  const createGallery = async (viewer) => {
    archive = await doFetch();
    collection = archive;
    initCollection();
    console.log("collection1", collection);

  };

  const ipfs_gateway_uri = 'https://ipfs.io/ipfs/';
  const convertIPSUri = (uri) => {
    try{
      return ipfs_gateway_uri+ uri.split('ipfs://')[1];
    }
    catch(e){
      return null;
    }
  }

  

  const displayItem = (item, index) => {
    fsLightboxInstances['gallery-lightbox'].componentsServices.setSlideNumber(index)
    current_item = item;
  }

  const selectItemByIndex = (index) => {
    current_item = collection[index];
  }

  const filterByCreator = async (creator) => {
    console.log('filter by creator', creator);
    current_filter = creator;
    collection = archive.filter((item) => item.token.creator.address === creator )
    await initCollection();
    closeLightbox();
    
  }

  const filterByTag = async (tag) => {
    console.log('filter by tag', tag);
    current_filter = tag;
    collection = archive.filter((item) => {
        if(item.token.token_tags.some((token_tag) => token_tag.tag.tag === tag)){
          return true;
        }
        return false;
    });
    console.log('filtered collection', collection);
    await initCollection();
    closeLightbox();
    
  }

  const clearFilter = async () => {
    current_filter = null;
    collection = archive;
    await initCollection();
  }

  const closeLightbox = () => {
    current_item = null;
    console.log(fsLightboxInstances['gallery-lightbox']);
    //fsLightboxInstances['gallery-lightbox'].close();
    
  }

  const search = async () => {
    console.log(search_term);
    current_filter = search_term;
    collection = archive.filter((item) => {
      console.log('checking', item.token.id.toString() + (item.token.id.toString().indexOf(search_term) > -1 ? 'TRUE': ''));
        if(item.token.id.toString().indexOf(search_term) > -1){
          return true;
        }
        else if(item.token.title.indexOf(search_term) > -1){
          return true;
        }
        else if(item.token.description.indexOf(search_term) > -1){
          return true;
        }
        else if(item.token.creator.address.indexOf(search_term) > -1){
          return true;
        }
        else if(item.token.token_tags.some((token_tag) => token_tag.tag.tag.indexOf(search_term) > -1)){
          return true;
        }
        return false;
    });
    await initCollection();
  }

  const initCollection = async () => {
    collection = collection;
    console.log(collection);
    await refreshFsLightbox();
  }

  let menu_expanded = false;

  const toggleMenu = async () => {
    menu_expanded = !menu_expanded;
  }

  let plaque_expanded = false;

  const togglePlaque = async () => {
    plaque_expanded = !plaque_expanded;
  }

</script>

<svelte:head>
  <meta property="og:image" content="thumbnail.jpg" />
  <script type="text/javascript" src="js/fslightbox/fslightbox.js" />
</svelte:head>

<div class="menu" style="{menu_expanded ? 'left:0;' : 'left:-100vw;'}">
  <div class="search">
    <input type="text" placeholder="search id, title, description, creator address & tags" bind:value="{search_term}"on:keyup="{search}" />
  </div>
  
  <div class="about">
    <a target="_blank" href="https://twitter.com/bitcoinski">@bitcoinski</a>
  </div>

  {#if current_filter}
    <div class="current_filter">
      <div class="text">Filtering by: {current_filter}</div> | 
      <div class="reset_button" on:click="{clearFilter}">clear filter</div>
    </div>
  {/if}

</div>
<div class="menu_expand_button_div" style="{menu_expanded ? 'bottom:80px;' : 'bottom:0;'}">
  <div class="menu_expand_button" on:click="{toggleMenu}">
    {menu_expanded ? 'CLOSE MENU' : 'MENU'}
  </div>
</div>


{#if current_item}
  
  <div class="plaque" style="{plaque_expanded ? 'left:-90vw;' : 'left:0;'}">
    <div class="title"><a target="_blank" href="{'https://www.hicetnunc.xyz/objkt/' + current_item.token.id}" >{current_item.token.title}</a></div>
    <div class="description">{current_item.token.description}</div>
    <div class="author">Creator: <a on:click="{() => filterByCreator(current_item.token.creator.address)}">{current_item.token.creator.address}</a></div>
    <div class="tags">
      {#each current_item.token.token_tags as token_tag}
        <a on:click="{() => filterByTag(token_tag.tag.tag)}">{token_tag.tag.tag}</a>&nbsp;
      {/each}
    </div>
  </div>
{/if}
<div class="container">
  {#each collection as item, index}
    {#if item.token.mime}
      {#if item.token.mime.indexOf('image') > -1 }
      <a class="gallery_item" data-fslightbox="gallery-lightbox" href={convertIPSUri(item.token.artifact_uri)} on:click={() => {displayItem(item, index)}}>
        <img src={convertIPSUri(item.token.display_uri)} />
      </a>
      {:else if item.token.mime.indexOf('video') > - 1 || item.token.mime.indexOf('audio') > - 1}
      <a class="gallery_item" data-fslightbox="gallery-lightbox" href={convertIPSUri(item.token.artifact_uri)}  on:click={() => {displayItem(item, index)}}>
        <video playsinline="" loop="true" controls="true" src="{convertIPSUri(item.token.artifact_uri)}" poster="{convertIPSUri(item.token.display_uri)}" ></video>
      </a>
      {:else if item.token.mime.indexOf('x-directory') > -1}
      <a class="gallery_item" data-fslightbox="gallery-lightbox" href={'#iframe_' + item.token.id}  on:click={() => {displayItem(item, index)}}>
        <img src={convertIPSUri(item.token.display_uri)} />
        <iframe id="{'iframe_' + item.token.id}" title="html-embed" src="{convertIPSUri(item.token.artifact_uri )+ '/?creator=' +  item.token.creator.address + '&viewer=' + viewer + '&objkt=' + item.token.id}" sandbox="allow-scripts allow-same-origin" allow="accelerometer; camera; gyroscope; microphone; xr-spatial-tracking;" style="{'height: 80vh; width: 80vw; display:'+ (current_item && current_item.token.id === item.token.id ? 'inherit': 'none') +'; border: none;'}"></iframe>
      </a>
      
      {/if}
    {/if}
  {/each}
</div>



<style>
  body {
    background-color: black !important;
  }
  .gallery_item{
    max-width: 32vw;
  }
  .gallery_item img, .gallery_item video{
    max-width: 32vw;
  }
  .menu{
    z-index:999999999;
    background-color: rgba(0,0,0,0.5);
    color:white;
    position:fixed;
    bottom:0;
    left:0;
    font-size:11px;
    padding:1vw;
    width:100vw;
    height:80px;
    min-height:80px;
  }
  
  .menu_expand_button_div{
    z-index:999999999;
    display:block;
    width:100vw;
    position:fixed;
    bottom:0;
    text-align:center;
  }
  .menu_expand_button{
    width:100px;
    display:inline-block;
    color:white;
    background-color:black;
    padding:1px;
    border-radius: 5px 5px 0px 0px;
    font-size:12px;
  }
  .search{
    display:inline-block;
    width:100%;
    max-width:75vw;
  }
  .about{
    display:inline-block;
    width:100%;
    max-width:20vw;
    color:#444444;
    text-align:right;
  }
  .about a{
    color:#444444;
    text-decoration: none;
  }
  .search input{
    background-color:rgba(0,0,0,0.8);
    color:rgba(255,255,255, 1);
    border:0;
    padding:1vw;
    border-radius: 5px;
    width:50%;
  }
  
  .current_filter .text, .current_filter .reset_button{
    display:inline-block;
    cursor:pointer;
  }
  .current_filter .reset_button:hover{
    text-decoration: underline;
  }
  
  .plaque{
    z-index:1000000001;
    background-color: rgba(0,0,0,0.5);
    color:white;
    position:fixed;
    bottom:0;
    left:0;
    font-size:12px;
    padding:2vw;
    border-radius: 0px 15px 0px 0px;
  }

  .plaque .title{
    font-size:14px;
    font-weight: bold;
  }
  .plaque a{
    cursor:pointer;
    color:white;
  }
  .plaque a:hover{
    text-decoration: underline;
  }
  
</style>
