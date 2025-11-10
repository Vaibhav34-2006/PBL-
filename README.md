<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Autonomous Drone Flood Simulation — Algorithms Explained</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
  <style>
    body, html { height:100%; margin:0; font-family: Arial, sans-serif; }
    #map { height: 100vh; width: 72%; float:left; }
    #panel { width:28%; height:100vh; float:right; box-sizing:border-box; padding:12px; background:#f7f7f9; overflow:auto; }
    .section { margin-bottom:12px; }
    .title { font-weight:700; margin-bottom:6px; }
    .log { font-family:monospace; font-size:13px; background:#fff; border:1px solid #ddd; padding:8px; height:420px; overflow:auto; }
    button{padding:8px 12px; margin-right:6px; cursor:pointer;}
    .badge{display:inline-block;padding:4px 8px;border-radius:6px;background:#e2e8f0;margin-right:6px;}
    .stat { background:#fff; border:1px solid #ddd; padding:8px; margin-bottom:8px;}
  </style>
</head>
<body>

<div id="map"></div>
<div id="panel">
  <div class="section">
    <div class="title">Autonomous Drone Flood Simulation</div>
    <div>
      <button id="startBtn">Launch Drones</button>
      <button id="pauseBtn">Pause</button>
      <button id="resetBtn">Reset</button>
    </div>
    <small>Click on the map to set the flood center; drones will auto-detect people inside the flood area.</small>
  </div>

  <div class="section">
    <div class="title">Simulation Controls</div>
    <label>Number of Drones: <input id="numDrones" type="number" value="3" min="1" max="10" style="width:60px"></label><br/>
    <label>Detection density (approx people): <input id="density" type="range" min="5" max="40" value="12"></label> <span id="densityVal">12</span><br/>
    <label>Flood radius (meters): <input id="floodRadius" type="number" value="800" min="100" max="3000" style="width:80px"></label><br/>
    <label>Audio range (meters): <input id="audioRange" type="number" value="35" min="10" max="200" style="width:80px"></label>
  </div>

  <div class="section">
    <div class="title">Timeline (Algorithm events)</div>
    <div id="log" class="log"></div>
  </div>

  <div class="section">
    <div class="title">Live Stats</div>
    <div id="stats" class="stat">No simulation started.</div>
  </div>

  <div class="section">
    <div class="title">Legend</div>
    <div><span class="badge">●</span> Drone</div>
    <div><span style="color:#d9534f">●</span> Detected Human</div>
    <div><span style="color:#6c757d">Polygon</span> Voronoi region / team</div>
  </div>

  <div class="section">
    <div class="title">Notes</div>
    <small>
      - All code is client-side. Algorithms are clearly labeled inside the JS comments for your report.<br/>
      - Click the map to choose where the flood is (center). The flood area will be a circle (radius set above).<br/>
      - Press "Launch Drones" to start automatic detection and rescue.
    </small>
  </div>
</div>

<!-- Libraries -->
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@turf/turf@6/turf.min.js"></script>

<script>
/*
  Autonomous simulation implementing the 5 algorithms.
  Each algorithm section in code is clearly labeled with comments for explanation.
*/

// === Global map setup ===
let map = L.map('map').setView([18.5204, 73.8567], 13); // default Pune
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{ maxZoom:19 }).addTo(map);

// UI elements
const logEl = document.getElementById('log');
const statsEl = document.getElementById('stats');
const startBtn = document.getElementById('startBtn');
const pauseBtn = document.getElementById('pauseBtn');
const resetBtn = document.getElementById('resetBtn');
const numDInput = document.getElementById('numDrones');
const densityInput = document.getElementById('density');
const densityVal = document.getElementById('densityVal');
const floodRadiusInput = document.getElementById('floodRadius');
const audioRangeInput = document.getElementById('audioRange');

densityInput.oninput = () => densityVal.textContent = densityInput.value;

// Simple logger (timeline)
function log(msg){
  const t = new Date().toLocaleTimeString();
  logEl.innerHTML = `[${t}] ${msg}<br>` + logEl.innerHTML;
}

// Simulation state
let drones = [];    // { id, marker, seedLatLng, team, assignedRegionFeature, target, rescuedCount, speed }
let people = [];    // { id, marker, latlng, rescued }
let regionsLayer = null;
let floodCircle = null;
let floodCenter = null;
let running = false;
let simInterval = null;
let tickMs = 80; // simulation tick interval

// Keep track final summary readiness
let totalPeople = 0;

// Utility: generate unique IDs
let idCounter = 1;
function nextId(prefix){ return String(prefix) + (idCounter++); }

/* ======================================================
   ALGORITHM 1: Human Detection (Simulated YOLO)
   - Purpose: From the flood center and radius, simulate
     autonomous detection of people in the flooded area.
   - Implementation (simulation): Randomly place N points
     inside the flood circle. In real system, this would be
     YOLO inference on drone images producing coordinates.
   - We log detection events in timeline.
   ====================================================== */
function simulateDetection(){
  // determine density parameter -> number of people
  const dens = parseInt(densityInput.value || 12);
  // pick random between 80% and 120% of density to vary
  const n = Math.max(1, Math.round(dens * (0.8 + Math.random()*0.4)));
  totalPeople = n;
  log(`Algorithm 1 (Human Detection): Detected ${n} persons in flood area.`);

  // place persons randomly inside flood circle
  const center = floodCenter;
  const r = parseFloat(floodRadiusInput.value) || 800; // meters

  for(let i=0;i<n;i++){
    // random angle and distance
    const angle = Math.random() * Math.PI * 2;
    const radius = Math.sqrt(Math.random()) * r; // sqrt for uniform area
    // compute latlon offset using turf destination or simple approximation:
    const p = turf.destination([center.lng, center.lat], radius/1000, angle * 180/Math.PI, {units: 'kilometers'});
    const lat = p.geometry.coordinates[1], lng = p.geometry.coordinates[0];
    const marker = L.circleMarker([lat, lng], { radius:6, color:'#d9534f', fillColor:'#d9534f', fillOpacity:1 }).addTo(map);
  const pid = `P${i+1}`;
  marker.bindTooltip(pid, {permanent:false});
  people.push({ id: pid, marker, latlng: L.latLng(lat, lng), rescued:false });
    // Log each detection (could be batched in real system)
  log(`Detection: ${pid} at ${lat.toFixed(5)}, ${lng.toFixed(5)}.`);
  }
}

/* ======================================================
   ALGORITHM 2: Area Division (Voronoi-Based)
   - Purpose: Divide the flood region into subregions so each
     drone gets its own coverage zone (minimizes overlap).
   - Implementation: Use Turf.js to compute Voronoi of drone seeds
     and clip to a bounding box around the flood circle.
   - We display polygons and assign each polygon to a drone.
   ====================================================== */
function computeVoronoiAndAssign(){
  // prepare seeds for turf (lng,lat)
  const seeds = drones.map(d => [d.seedLatLng.lng, d.seedLatLng.lat]);
  const pts = turf.featureCollection(seeds.map(s => turf.point(s)));
  // build bbox slightly larger than flood circle
  const center = floodCenter;
  const radKm = parseFloat(floodRadiusInput.value || 800) / 1000;
  // bounding box: use a small square around center
  const bbox = [
    center.lng - radKm*0.03 - 0.02, // some margin
    center.lat - radKm*0.03 - 0.02,
    center.lng + radKm*0.03 + 0.02,
    center.lat + radKm*0.03 + 0.02
  ];
  const vor = turf.voronoi(pts, { bbox });
  if(!vor){
    log('Voronoi: failed to generate (not enough seeds).');
    return;
  }
  // draw polygons and attach to drones
  if(regionsLayer) { regionsLayer.clearLayers(); map.removeLayer(regionsLayer); regionsLayer = null; }
  regionsLayer = L.layerGroup().addTo(map);
  vor.features.forEach((feat, idx) => {
    // find nearest seed index to polygon centroid
  const centroid = turf.centroid(feat).geometry.coordinates; // [lng,lat]
    let bestIdx = 0, bestD = Infinity;
    seeds.forEach((s, i) => {
      const dx = s[0] - centroid[0], dy = s[1] - centroid[1];
      const dd = dx*dx + dy*dy;
      if(dd < bestD){ bestD = dd; bestIdx = i; }
    });
    // assign polygon to that drone
    const coords = feat.geometry.coordinates[0].map(c => [c[1], c[0]]);
    const poly = L.polygon(coords, { color:'#666', weight:1, fillColor: drones[bestIdx].color, fillOpacity:0.12 }).addTo(regionsLayer);
    poly.bindTooltip(drones[bestIdx].team, {permanent:false});
    drones[bestIdx].assignedRegionFeature = feat; // raw turf feature
    log(`Algorithm 2 (Area Division): Assigned ${drones[bestIdx].id} to ${drones[bestIdx].team}.`);
  });
}

/* ======================================================
   ALGORITHM 3: Swarm Coordination (Task Allocation & Path Planning)
   - Purpose: Decide which drone goes to which detected person.
   - Implementation (prototype):
       * Each drone focuses on people inside its assigned Voronoi region (local priority)
       * If none in its region, it can help globally.
       * For assignment inside region, greedy nearest-first is used.
       * Path planning here uses straight-line (waypoints are interpolated)
         — for demonstration. For a real system, A* / RRT would be used.
   - Each drone keeps its own target and rescued count.
   ====================================================== */
function assignTargetsToDrones(){
  // Build list of unrescued people
  const unrescued = people.filter(p => !p.rescued);
  if(unrescued.length === 0) return;

  drones.forEach(d => {
    // if drone already has a target and it's still unrescued, keep it
    if(d.target && !d.target.rescued) return;

    // prefer people inside drone's region
    let candidates = unrescued;
    if(d.assignedRegionFeature){
      candidates = unrescued.filter(p => {
        const pt = turf.point([p.latlng.lng, p.latlng.lat]);
        try { return turf.booleanPointInPolygon(pt, d.assignedRegionFeature); } catch(e) { return true; }
      });
      if(candidates.length === 0) candidates = unrescued; // fallback
    }

    if(candidates.length > 0){
      // nearest-first greedy
      let best = candidates[0], bestDist = map.distance(d.seedLatLng, candidates[0].latlng);
      for(let c of candidates){
        const dist = map.distance(d.seedLatLng, c.latlng); // using seed as drone current focal point
        if(dist < bestDist){ bestDist = dist; best = c; }
      }
      d.target = best;
  log(`Algorithm 3 (Swarm Coordination): ${d.id} assigned target ${best.id} (${Math.round(bestDist)} m).`);
    } else {
      d.target = null;
    }
  });
}

/* ======================================================
   ALGORITHM 4: Audio Guidance Trigger
   - Purpose: When drone is within 'audio range' of a detected human,
     play an audio instruction (text-to-speech) and log the event.
   - Implementation: Uses browser SpeechSynthesis for TTS.
   - Cooldown behavior: each human is rescued once; we log then mark rescued.
   ====================================================== */
function checkAudioTriggersAndRescue(){
  const audioRange = parseFloat(audioRangeInput.value || 35);
  drones.forEach(d => {
    if(!d.target || d.target.rescued) return;
    // get current drone marker location
    const dpos = d.marker.getLatLng();
    const ppos = d.target.latlng;
    const dist = map.distance(dpos, ppos);
    if(dist <= audioRange){
      // Trigger audio guidance
      const msg = `${d.team} to ${d.target.id}: Please move to higher ground. Help is approaching.`;
      speak(msg);
      log(`Algorithm 4 (Audio Guidance): ${d.id} near ${d.target.id} (~${Math.round(dist)} m). Message: "${msg}"`);
      // Simulate rescue completion and data routing
      d.target.rescued = true;
      d.rescuedCount += 1;
      d.target.marker.setStyle({ color:'#6c757d', fillColor:'#6c757d' }); // greyed as rescued
      // Route data to team (Algorithm 5)
      routeDataToTeam(d, d.target, Math.round(dist));
      // Clear target so drone will get new assignment
      d.target = null;
      // If all rescued => stop later
    }
  });
}

/* ======================================================
   ALGORITHM 5: Rescue Team Data Routing
   - Purpose: When a rescue event happens, send the collected
     data (drone id, person id, coords) to the assigned rescue team.
   - Implementation (simulation): Log the routing event in timeline
     and update per-drone / total counters. In the real system,
     this would publish to an MQTT topic or REST endpoint.
   ====================================================== */
function routeDataToTeam(drone, person, distanceMeter){
  const payload = {
    drone_id: drone.id,
    team: drone.team,
    person_id: person.id,
    person_location: [person.latlng.lat, person.latlng.lng],
    distance_m: distanceMeter,
    time: new Date().toISOString()
  };
  // Log as simulated publish
  log(`Algorithm 5 (Data Routing): Routed to ${drone.team}: ${JSON.stringify(payload)}`);
  updateStatsUI();
}

// Text-to-speech helper
function speak(text){
  try{
    if('speechSynthesis' in window){
      // cancel any ongoing speech to reduce overlap
      window.speechSynthesis.cancel();
      const u = new SpeechSynthesisUtterance(text);
      u.rate = 1.0;
      window.speechSynthesis.speak(u);
    }
  } catch(e){ console.warn('TTS error', e); }
}

/* ======================================================
   Drone movement and simulation tick
   - Move drones toward their current target using simple linear interpolation.
   - If no target, drone patrols (small random movement inside assigned region).
   - On each tick we run:
       1) assignTargetsToDrones()   (Algorithm 3 assignment)
       2) move drones step
       3) checkAudioTriggersAndRescue() (Algorithm 4 & 5 combined event)
   ====================================================== */
function simTick(){
  // 1) ensure assignments are fresh
  assignTargetsToDrones();

  // 2) move drones
  drones.forEach(d => {
    if(d.target && !d.target.rescued){
      // simple straight-line step towards target
      moveMarkerTowards(d.marker, d.target.latlng, d.speed);
    } else {
      // patrol behavior: small random jitter within assigned region or around seed
      const curr = d.marker.getLatLng();
      // jitter magnitude
      const jitter = 0.0004 * (1 + Math.random()*0.4);
      const newlat = curr.lat + (Math.random()-0.5)*jitter;
      const newlng = curr.lng + (Math.random()-0.5)*jitter;
      // if assigned region exists, try to keep inside its polygon; we simply set the marker
      d.marker.setLatLng([newlat, newlng]);
    }
  });

  // 3) check for audio triggers and rescue
  checkAudioTriggersAndRescue();

  // 4) check termination: all people rescued?
  const remaining = people.filter(p => !p.rescued).length;
  if(remaining === 0 && totalPeople > 0){
    log('All detected people have been rescued. Simulation complete.');
    stopSim();
    showSummary();
  }
}

// Move marker towards latlng by step (speed is fraction degrees per tick)
function moveMarkerTowards(marker, targetLatLng, speed){
  const cur = marker.getLatLng();
  const dx = targetLatLng.lng - cur.lng;
  const dy = targetLatLng.lat - cur.lat;
  const dist = Math.sqrt(dx*dx + dy*dy);
  if(dist < 0.0002){
    marker.setLatLng([targetLatLng.lat, targetLatLng.lng]);
    return;
  }
  const nx = cur.lng + (dx/dist) * speed;
  const ny = cur.lat + (dy/dist) * speed;
  marker.setLatLng([ny, nx]);
}

/* ======================================================
   UI and setup functions
   - click on map to set flood center (circle)
   - Launch Drones: creates drone markers, seeds, runs detection,
     computes Voronoi, and starts simulation loop.
   ====================================================== */

map.on('click', function(e){
  // Set flood center visually
  if(floodCircle) map.removeLayer(floodCircle);
  floodCenter = e.latlng;
  const r = parseFloat(floodRadiusInput.value) || 800;
  floodCircle = L.circle(e.latlng, { radius: r, color:'#0077be', fillColor:'#cceeff', fillOpacity:0.2 }).addTo(map);
  map.setView(e.latlng, 14);
  log(`Flood center set at ${e.latlng.lat.toFixed(5)}, ${e.latlng.lng.toFixed(5)} with radius ${r} m. Click Launch Drones to start.`);
  updateStatsUI();
});

// Create drones with seed positions around flood center
function createDrones(){
  const n = Math.max(1, parseInt(numDInput.value || 3));
  drones.forEach(d => { if(d.marker) map.removeLayer(d.marker); });
  drones = [];
  // distribute seeds in circle around flood center
  const center = floodCenter;
  const baseAngle = Math.random() * Math.PI*2;
  for(let i=0;i<n;i++){
    const angle = baseAngle + (i * 2*Math.PI / n);
    const ring = (parseFloat(floodRadiusInput.value || 800) / 1000) * (0.4 + Math.random()*0.3); // km offset from center
    const seedPt = turf.destination([center.lng, center.lat], ring, angle * 180/Math.PI, {units:'kilometers'});
    const lat = seedPt.geometry.coordinates[1], lng = seedPt.geometry.coordinates[0];
    const color = getColor(i);
    const marker = L.circleMarker([lat, lng], { radius:9, color:'#000', fillColor: color, fillOpacity:1 }).addTo(map);
  const id = `D${i+1}`;
  const team = `Team ${String.fromCharCode(65+i)}`; // Team A, B...
  marker.bindTooltip(id, {permanent:false});
    const drone = {
      id, team, seedLatLng: L.latLng(lat, lng), marker, color,
      assignedRegionFeature: null, target: null, rescuedCount: 0,
      speed: 0.0006 + Math.random()*0.0012 // degrees per tick (tunable)
    };
    drones.push(drone);
  log(`Drone created: ${id} at ${lat.toFixed(5)}, ${lng.toFixed(5)} assigned ${team}.`);
  }
  updateStatsUI();
}

// Start simulation sequence (autonomous)
function launchSimulation(){
  if(!floodCenter){ alert('Please click on the map to select a flood center before launching.'); return; }
  clearExistingPeopleAndRegions();
  createDrones();
  // === ALGORITHM 1: Human Detection (simulated) ===
  simulateDetection();

  // === ALGORITHM 2: Area Division (Voronoi) ===
  computeVoronoiAndAssign();

  // Start simulation ticks
  startSim();
  log('Simulation launched: algorithms 1→5 will run autonomously.');
}

// clear existing people and regions
function clearExistingPeopleAndRegions(){
  people.forEach(p => { if(p.marker) map.removeLayer(p.marker); });
  people = [];
  if(regionsLayer){ regionsLayer.clearLayers(); map.removeLayer(regionsLayer); regionsLayer = null; }
  if(simInterval){ clearInterval(simInterval); simInterval = null; }
  if(floodCircle) { /* keep flood circle */ }
  // reset counts
  totalPeople = 0;
  updateStatsUI();
}

// Start/stop controls
function startSim(){
  if(running) return;
  running = true;
  simInterval = setInterval(simTick, tickMs);
  log('Simulation running.');
}

function stopSim(){
  running = false;
  if(simInterval){ clearInterval(simInterval); simInterval = null; }
  log('Simulation paused.');
}

function resetAll(){
  // remove markers and reset everything
  drones.forEach(d => { if(d.marker) map.removeLayer(d.marker); });
  people.forEach(p => { if(p.marker) map.removeLayer(p.marker); });
  if(regionsLayer){ regionsLayer.clearLayers(); map.removeLayer(regionsLayer); regionsLayer = null; }
  if(floodCircle){ map.removeLayer(floodCircle); floodCircle = null; floodCenter = null; }
  drones = []; people = []; totalPeople = 0;
  logEl.innerHTML = '';
  updateStatsUI();
}

// Summary UI when done
function showSummary(){
  let html = `<b>Simulation Summary</b><br/>Total detected: ${totalPeople}<br/>`;
  let totalRescued = 0;
  drones.forEach(d => {
    html += `${d.id} (${d.team}): ${d.rescuedCount} rescued<br/>`;
    totalRescued += d.rescuedCount;
  });
  html += `<b>Total rescued: ${totalRescued}</b>`;
  statsEl.innerHTML = html;
  log('Summary generated and shown in Live Stats.');
}

// Update live stats box while running
function updateStatsUI(){
  let rem = people.filter(p => !p.rescued).length;
  let html = `<b>Flood center:</b> ${floodCenter ? floodCenter.lat.toFixed(5) + ', ' + floodCenter.lng.toFixed(5) : 'Not set'} <br/>`;
  html += `<b>Flood radius:</b> ${floodRadiusInput.value} m<br/>`;
  html += `<b>Detected persons:</b> ${totalPeople} <br/>`;
  html += `<b>Remaining:</b> ${rem} <br/>`;
  html += `<b>Drones:</b> ${drones.length} <br/>`;
  drones.forEach(d => { html += `${d.id}: ${d.rescuedCount} rescued<br/>`; });
  statsEl.innerHTML = html;
}

/* Helpers */
function getColor(i){
  const palette = ['#1f77b4','#ff7f0e','#2ca02c','#d62728','#9467bd','#8c564b','#e377c2','#7f7f7f'];
  return palette[i % palette.length];
}

/* ======================================================
   Event handlers for UI
   - Launch → begin whole autonomous pipeline
   - Pause → stop ticks
   - Reset → clear everything
   ====================================================== */
startBtn.onclick = () => launchSimulation();
pauseBtn.onclick = () => stopSim();
resetBtn.onclick = () => resetAll();

// initial log
log('Ready. Click on the map to set a flood center, then press "Launch Drones".');

</script>
</body>
</html>
