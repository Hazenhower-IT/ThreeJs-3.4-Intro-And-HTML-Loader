<template>
  <canvas ref="canvasRef"></canvas>
  <div class="loading-bar"></div>
</template>

<script setup>
import { ref, onMounted, onUnmounted} from "vue"
import { OrbitControls} from "three/examples/jsm/controls/OrbitControls"
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader"
import * as THREE from "three"
import gsap from "gsap"
import * as dat from "dat.gui"


//GUI
const gui = new dat.GUI()

//CANVAS
let canvasRef = ref();

var controls

//Utilizziamo i valori del viewport per impostare la grandezza del canvas
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

//Aggiungiamo l'event listner per il resize del canvas
window.addEventListener("resize", ()=>{
  //update sizes for the canvas
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  //update the camera aspect ratio
  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  //update renderer
  renderer.setSize(sizes.width, sizes.height)
  //HANDLE PIXEL RATIO
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})


//HANDLE FULL SCREEN (Il codice aggiuntivo è per permettere il funzionamento corretto anche con browser Safari)
window.addEventListener("dblclick",()=>{
  const fullscreenElement = document.fullscreenElement ||  document.webkitFullscreenElement
  if(!fullscreenElement){
    if(canvasRef.value.requestFullscreen){
      canvasRef.value.requestFullscreen()
    }
    else if(canvasRef.value.webkitRequestFullscreen){
      canvasRef.value.webkitRequestFullscreen()
    }
    
  }
  else{
    if(document.exitFullscreen)
    {
      document.exitFullscreen()
    }
    else if(document.webkitExitFullscreen){
      document.webkitExitFullscreen()
    }
  }
})

let scene = new THREE.Scene()

// 1*  Let's create the overlay to use for the loading intro
//OVERLAY
const overlayGeo = new THREE.PlaneGeometry(2,2,1,1)
const overlayMat = new THREE.ShaderMaterial({
    transparent: true,
    uniforms:{
      uAlpha: { value: 1}
    },
    vertexShader:`
    void main()
    {
      gl_Position = vec4(position, 1.0);
    }
    `,
    fragmentShader:`
    uniform float uAlpha;
    
    void main()
    {
      gl_FragColor = vec4(0.0, 0.0, 0.0, uAlpha);
    }
    `
})
const overlay = new THREE.Mesh(overlayGeo, overlayMat)
scene.add(overlay)

//now we want to know when the scene and all the assets in it are loaded
//to both display the loading bars, and fadeout the overlay when the load is complete

//we can use a LoadingManager as parameters for both the GLTF abd CubeTexture loaders
//the loadingManager provide control of this loading session , to make us know the status of the loading, and when it's complete 
//(so go up where we instantiate the loaders and apply that 2* ) 


let renderer

//Lights
const directionalLight = new THREE.DirectionalLight("#ffffff",3)
directionalLight.position.set(0.25, 3, -2.25)
directionalLight.castShadow = true
directionalLight.shadow.camera.far = 15
directionalLight.shadow.mapSize.set(1024, 1024)
gui.add(directionalLight, "intensity").min(0).max(10).step(0.001).name("lightIntensity")
gui.add(directionalLight.position, "x").min(-5).max(5).step(0.001).name("light X")
gui.add(directionalLight.position, "y").min(-5).max(5).step(0.001).name("light Y")
gui.add(directionalLight.position, "z").min(-5).max(5).step(0.001).name("light Z")
scene.add(directionalLight)

// const directionalLightCameraHelper = new THREE.CameraHelper(directionalLight.shadow.camera)
// scene.add(directionalLightCameraHelper)




// CAMERA
let camera = new THREE.PerspectiveCamera(
  75, 
  sizes.width / sizes.height, 
  0.1,
  100  
);
camera.position.set(0,0,3)
scene.add(camera);

const clock = new THREE.Clock()

//Animate Loop
const animate = () =>{
  const elapsedTime = clock.getElapsedTime()

  controls.update()
  
  renderer.render(scene, camera)
}

onMounted(() => {
  

  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true
  });
  
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2)) // in questo modo il max valore di pixel ratio utilizzaro è 2 (altrimenti devi renderizzare troppi pixel, dispiego enorme di potenza gpu)
  renderer.outputColorSpace = THREE.SRGBColorSpace

  renderer.toneMapping = THREE.ReinhardToneMapping
  renderer.toneMappingExposure = 3

  gui.add(renderer, "toneMapping", {
    Not: THREE.NoToneMapping,
    Linear: THREE.LinearToneMapping,
    Reinhard: THREE.ReinhardToneMapping,
    Cineon: THREE.CineonToneMapping,
    ACESFilmic: THREE.ACESFilmicToneMapping
  }).onChange(()=>{
    renderer.toneMapping = Number(renderer.toneMapping)
    updateAllMaterials()
  })


  //add the toneMappingExposure too to dat.GUI
  gui.add(renderer, "toneMappingExposure").min(0).max(10).step(0.001)


  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap


  renderer.render(scene, camera)
  renderer.setAnimationLoop(animate)

  //CONTROLS
  controls = new OrbitControls(camera, canvasRef.value)
  controls.enableDamping = true; // permette uno effetto smooth una volta che rilasciamo l'elemento, rallenta fino a fermarsi

  ////////////////////////////////// INTRO AND HTML LOADER /////////////////////////////////////////////////////
//We are going to create a loader that show the loading status of the scene, and then display the scene only when all is loaded and ready

// We want also a black overlay that fades out when the scene in ready
//we have many solution to achieve that:
//- Animate the canvas in css
//- Animate a div above the canvas in css
//- Animate a black rectangle in front of the camera

//NOTE that is best to keep everything inside the webGL scene, and don't mix with HTML, so if you can put all or as much as possible in the webgl scene

//so , instead using an html element to do the fade, we are going to use a threejs plane 
//to put a plane in front of the camera we can go in 2 ways:
//1) we could add it to the camera object and move it forward (in this way the plane follow the camera, but it's hard to find the right size to use that depends on the resolution)
//2) We could put the plane on the scene and position its vertices using a vertex shader

//we are going to use the second way..let's go after the instantiation of the scene object and create a plane 1*



//LOADER AND LOADING SECTION

//Loading Bar ELEMENT
const loadingBarElement = document.querySelector(".loading-bar")

//2*
//First we need to instantiate the loading manager 
const loadingManager = new THREE.LoadingManager(

//and now we can provide as parameters 3 functions as parameters: 

//the first if for the loaded callback (when the loading is complete)  

//the second one is the progress callback, and can be used to know the status of the loading  (triggered every time one assets is loaded) 

//(optional) the error callback, that is triggered in case of an error in the loading occur

//So we are going to use the loaded callback to hide the overlay when all the stuff is loaded, and the progress callback to display the progress on the progressbar


//Loaded
  ()=>{

    //set a timeout before stard loading, to prevent laggy bugs (we wait half a second)
    window.setTimeout(()=>{
    //Overlay Fadeout Animation
    gsap.to(overlayMat.uniforms.uAlpha, {duration:3, value:0})
    loadingBarElement.classList.add("ended")
    loadingBarElement.style.transform = ""
    }, 500)


   
  },



//Progress
//this function has 3 argument: itemUrl (the url of the items loaded), itemsLoaded (how much assets were loaded), and itemsTotal(the total number of assets to load)
  (itemUrl, itemsLoaded, itemsTotal)=>{

    //we are going to add a progressBar that match the assets loading
    //note that this process can be made (and be better) using another vertexGeometry to create the loading
    //but we are going to use an html bar as loader, just to practice mixing html and webgl

    //calculate the progress ratio
    const progressRatio = itemsLoaded / itemsTotal 
    
    if(loadingBarElement){
       //update the style of the loading bar element
    loadingBarElement.style.transform = "scaleX("+ progressRatio +")"
    }
   

    console.log("progress")
  }
)


//then provide the loadingManager as parameter on both the loader

//Instantiate the GLTFLoader
const gltfLoader = new GLTFLoader(loadingManager)

//Instantiate the CubeLoader
const cubeTextureLoader = new THREE.CubeTextureLoader(loadingManager)


const environmentMap = cubeTextureLoader.load([
  "/textures/environmentMaps/0/px.jpg",
  "/textures/environmentMaps/0/nx.jpg",
  "/textures/environmentMaps/0/py.jpg",
  "/textures/environmentMaps/0/ny.jpg",
  "/textures/environmentMaps/0/pz.jpg",
  "/textures/environmentMaps/0/nz.jpg",
])

// 2* change the environmentMap encoding property from linear to sRGBEncoding
environmentMap.colorSpace = THREE.SRGBColorSpace

//and apply the environment map on the background of the render
scene.background = environmentMap


//Load our MODELS
gltfLoader.load("/models/FlightHelmet/glTF/FlightHelmet.gltf", (gltf)=>{
  
  gltf.scene.scale.set(10,10,10)
  gltf.scene.position.set(0,-4,0)
  gltf.scene.rotation.y = -Math.PI * 0.25 
  scene.add(gltf.scene)
  
  gui.add(gltf.scene.rotation, "y").min(-Math.PI).max(Math.PI).step(0.001).name("rotation")

  //*1(come here later when you find another *1) and then call the function after the models il loaded into the scene
  updateAllMaterials()
})


// Function for Updating materials based on the environment Maps
const updateAllMaterials = () =>{
  scene.traverse((child)=>{  //the tranverse function "traverse" every object, every child, every child of child and so on
  
  //Set a control to change the material only for Mesh objects (or what we want. but if you don't do this way, it change also the lights and all the other type of objects materials), and only for object that has a MeshStandardMaterial
  if(child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial){

    //and finally now we can apply the environmentMap to the envMap property of the selected objects
    child.material.envMap = environmentMap
    child.material.envMapIntensity = debugObject.envMapIntensity
    child.material.needsUpdate = true

    child.castShadow = true
    child.receiveShadow = true
  }
  
  })
}



//now create a debugObject to store property that can't be store anywhere else (because dat.gui requires that it's an object property to tweak) (Note that can be better to create this in the first section of the code, i'll go here just for stay clear)
const debugObject = {}

//let's start adding the envMapIntensity to the debug object
debugObject.envMapIntensity = 5

//then add a tweak on gui
gui.add(debugObject, "envMapIntensity").min(0).max(10).step(0.001).name("EnvMap Intensity").onChange(updateAllMaterials)

});

onUnmounted(() => {
  renderer.setAnimationLoop(null)
});

</script>

<style>
canvas {
  width: 100%;
  height: 100%;
  display: block;
}

.loading-bar{
  position:absolute;
  top: 50%;
  width: 100%;
  height: 2px;
  background: #fff;
  transform: scaleX(0);
  transform-origin: top left;
  transition: transform 0.7s;
  will-change: transform;
}

.loading-bar.ended{
  transform-origin: top right;
  transition: transform 1.8s ease-in-out;
}
</style>