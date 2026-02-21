// ================= SETTINGS =================
const CHUNK_SIZE = 16;
const RENDER_DISTANCE = 2;
let worldSeed = 1337;
const PLAYER_HEIGHT = 1.8;
const PLAYER_WIDTH = 0.6;

// ================= THREE =================
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87CEEB);

const camera = new THREE.PerspectiveCamera(75,innerWidth/innerHeight,0.1,1000);
camera.position.set(0,20,0);

const renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth,innerHeight);
document.body.innerHTML="";
document.body.appendChild(renderer.domElement);

const sun = new THREE.DirectionalLight(0xffffff,1);
sun.position.set(100,200,100);
scene.add(sun);
scene.add(new THREE.AmbientLight(0xffffff,0.4));

// ================= MATERIALS =================
const geo = new THREE.BoxGeometry();
const grassMat = new THREE.MeshLambertMaterial({color:0x00aa00});
const dirtMat  = new THREE.MeshLambertMaterial({color:0x8B4513});
const stoneMat = new THREE.MeshLambertMaterial({color:0x777777});

const hardness = {grass:1,dirt:1,stone:3};

// ================= WORLD =================
let blocks=[];
let chunks={};

function noise(x,z){
  return Math.abs(Math.sin(x*0.08+worldSeed)*Math.cos(z*0.08));
}

function generateChunk(cx,cz){
  const key=cx+","+cz;
  if(chunks[key]) return;

  const group=new THREE.Group();

  for(let x=0;x<CHUNK_SIZE;x++){
    for(let z=0;z<CHUNK_SIZE;z++){

      const wx=cx*CHUNK_SIZE+x;
      const wz=cz*CHUNK_SIZE+z;
      const h=Math.floor(noise(wx,wz)*8)+2;

      for(let y=0;y<=h;y++){
        let mat,type;
        if(y===h){mat=grassMat;type="grass";}
        else if(y>h-3){mat=dirtMat;type="dirt";}
        else{mat=stoneMat;type="stone";}

        const b=new THREE.Mesh(geo,mat);
        b.position.set(wx,y,wz);
        b.userData.type=type;

        group.add(b);
        blocks.push(b);
      }
    }
  }

  scene.add(group);
  chunks[key]=group;
}

// ================= CONTROLS =================
const keys={};
document.addEventListener("keydown",e=>{
  keys[e.key.toLowerCase()]=true;
  if(e.key==="F3") debug=!debug;
});
document.addEventListener("keyup",e=>keys[e.key.toLowerCase()]=false);

let yaw=0,pitch=0;
renderer.domElement.addEventListener("click",()=>renderer.domElement.requestPointerLock());
document.addEventListener("mousemove",e=>{
if(document.pointerLockElement===renderer.domElement){
  yaw-=e.movementX*0.002;
  pitch-=e.movementY*0.002;
  pitch=Math.max(-Math.PI/2,Math.min(Math.PI/2,pitch));
}
});

// ================= PLAYER =================
let velocityY=0;
let onGround=false;

function checkCollision(pos){

  for(let b of blocks){

    if(
      Math.abs(b.position.x-pos.x)<0.5+PLAYER_WIDTH &&
      Math.abs(b.position.z-pos.z)<0.5+PLAYER_WIDTH &&
      pos.y < b.position.y+1 &&
      pos.y+PLAYER_HEIGHT > b.position.y
    ){
      return b;
    }
  }
  return null;
}

// ================= DEBUG UI =================
let debug=false;
const ui=document.createElement("div");
ui.style.position="absolute";
ui.style.top="10px";
ui.style.left="10px";
ui.style.color="white";
ui.style.fontFamily="monospace";
document.body.appendChild(ui);

// ================= LOOP =================
function animate(){
requestAnimationFrame(animate);

const speed=0.15;

const forward=new THREE.Vector3(Math.sin(yaw),0,Math.cos(yaw));
const right=new THREE.Vector3(Math.cos(yaw),0,-Math.sin(yaw));

let next=camera.position.clone();

if(keys["w"]) next.add(forward.clone().multiplyScalar(speed));
if(keys["s"]) next.add(forward.clone().multiplyScalar(-speed));
if(keys["a"]) next.add(right.clone().multiplyScalar(-speed));
if(keys["d"]) next.add(right.clone().multiplyScalar(speed));

// XZ collision
let testPos=new THREE.Vector3(next.x,camera.position.y,next.z);
if(!checkCollision(testPos)){
  camera.position.x=next.x;
  camera.position.z=next.z;
}

// Gravity
velocityY-=0.02;
camera.position.y+=velocityY;

let below=new THREE.Vector3(camera.position.x,camera.position.y-0.1,camera.position.z);
let blockBelow=checkCollision(below);

if(blockBelow){
  camera.position.y=blockBelow.position.y+1;
  velocityY=0;
  onGround=true;
}else{
  onGround=false;
}

if(keys[" "] && onGround){
  velocityY=0.35;
}

// Day Night
let t=Date.now()*0.0002;
let intensity=(Math.sin(t)+1)/2;
sun.intensity=intensity;
scene.background=new THREE.Color().setHSL(0.6,0.5,intensity*0.6);

// Load chunks
const pcx=Math.floor(camera.position.x/CHUNK_SIZE);
const pcz=Math.floor(camera.position.z/CHUNK_SIZE);

for(let x=-RENDER_DISTANCE;x<=RENDER_DISTANCE;x++){
  for(let z=-RENDER_DISTANCE;z<=RENDER_DISTANCE;z++){
    generateChunk(pcx+x,pcz+z);
  }
}

// DEBUG
if(debug){
  ui.innerHTML=
  `XYZ: ${camera.position.x.toFixed(1)}, ${camera.position.y.toFixed(1)}, ${camera.position.z.toFixed(1)}<br>
   Chunk: ${pcx}, ${pcz}<br>
   Loaded Chunks: ${Object.keys(chunks).length}<br>
   Blocks: ${blocks.length}`;
}else{
  ui.innerHTML="";
}

camera.rotation.order="YXZ";
camera.rotation.y=yaw;
camera.rotation.x=pitch;

renderer.render(scene,camera);
}

animate();
