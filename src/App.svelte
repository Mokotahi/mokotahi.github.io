<script>
  import { onMount } from "svelte";
  import * as THREE from "three";

  let heroContainer;
  let cloudsContainer;
  let topoCanvas;
  let topoCtx;

  let currentShape = $state("triangle");
  let currentPalette = $state("mustard");
  let isLoading = $state(true);
  let showControls = $state(false);

  const PALETTES = {
    mustard: { primary: "#d4a817", name: "Mustard" },
    cyan: { primary: "#17d4d4", name: "Cyan" },
    magenta: { primary: "#d417a8", name: "Magenta" },
    lime: { primary: "#a8d417", name: "Lime" },
    coral: { primary: "#d46717", name: "Coral" },
    lavender: { primary: "#8b17d4", name: "Lavender" },
  };

  const SHAPES = {
    square: 0,
    circle: 1,
    triangle: 2,
    diamond: 3,
    hexagon: 4,
    star: 5,
    pentagon: 6,
    octagon: 7,
  };

  const mousePos = $state({ x: 0.5, y: 0.5 });

  // Simple 2D noise function for topographic map
  function noise2D(x, y, seed = 0) {
    const n = Math.sin(x * 12.9898 + y * 78.233 + seed) * 43758.5453;
    return n - Math.floor(n);
  }

  function smoothNoise(x, y, scale, seed = 0) {
    const X = Math.floor(x / scale);
    const Y = Math.floor(y / scale);
    const fractX = x / scale - X;
    const fractY = y / scale - Y;

    const v1 = noise2D(X, Y, seed);
    const v2 = noise2D(X + 1, Y, seed);
    const v3 = noise2D(X, Y + 1, seed);
    const v4 = noise2D(X + 1, Y + 1, seed);

    const i1 = v1 * (1 - fractX) + v2 * fractX;
    const i2 = v3 * (1 - fractX) + v4 * fractX;

    return i1 * (1 - fractY) + i2 * fractY;
  }

  function fbmNoise(x, y, octaves = 4, time = 0) {
    let value = 0;
    let amplitude = 1;
    let frequency = 1;
    let maxValue = 0;

    for (let i = 0; i < octaves; i++) {
      value +=
        smoothNoise(
          x * frequency + time * 0.5,
          y * frequency + time * 0.3,
          50 / frequency,
          i * 100,
        ) * amplitude;
      maxValue += amplitude;
      amplitude *= 0.5;
      frequency *= 2;
    }

    return value / maxValue;
  }

  function createClouds() {
    const clouds = [];
    const numClouds = 8;

    for (let i = 0; i < numClouds; i++) {
      const size = Math.random() * 200 + 100;
      clouds.push({
        x: Math.random() * window.innerWidth,
        y: Math.random() * window.innerHeight,
        size: size,
        speedX: (Math.random() - 0.5) * 0.2,
        speedY: (Math.random() - 0.5) * 0.2,
        opacity: Math.random() * 0.08 + 0.02,
      });
    }

    return clouds;
  }

  onMount(() => {
    // Setup topographic canvas
    const topoCanvasEl = document.createElement("canvas");
    topoCanvasEl.width = window.innerWidth;
    topoCanvasEl.height = window.innerHeight;
    topoCanvasEl.style.position = "absolute";
    topoCanvasEl.style.inset = "0";
    topoCanvasEl.style.pointerEvents = "none";
    topoCanvasEl.style.zIndex = "-1";
    topoCanvas = topoCanvasEl;
    topoCtx = topoCanvasEl.getContext("2d");
    document.body.insertBefore(topoCanvasEl, document.body.firstChild);

    // Setup floating clouds
    const clouds = createClouds();
    cloudsContainer.innerHTML = "";

    clouds.forEach((cloud) => {
      const cloudEl = document.createElement("div");
      cloudEl.className = "cloud";
      cloudEl.style.left = `${cloud.x}px`;
      cloudEl.style.top = `${cloud.y}px`;
      cloudEl.style.width = `${cloud.size}px`;
      cloudEl.style.height = `${cloud.size * 0.6}px`;
      cloudEl.style.opacity = cloud.opacity;
      cloudsContainer.appendChild(cloudEl);

      function animateCloud() {
        cloud.x += cloud.speedX;
        cloud.y += cloud.speedY;

        if (cloud.x < -cloud.size) cloud.x = window.innerWidth + cloud.size;
        if (cloud.x > window.innerWidth + cloud.size) cloud.x = -cloud.size;
        if (cloud.y < -cloud.size) cloud.y = window.innerHeight + cloud.size;
        if (cloud.y > window.innerHeight + cloud.size) cloud.y = -cloud.size;

        cloudEl.style.left = `${cloud.x}px`;
        cloudEl.style.top = `${cloud.y}px`;
        cloudEl.style.background = PALETTES[currentPalette].primary;

        requestAnimationFrame(animateCloud);
      }

      animateCloud();
    });

    // Advanced shader setup
    const vertexShaderMain = `void main() {
      gl_Position = vec4(position, 1.0);
    }`;

    const fragmentShaderMain = `
precision highp float;

uniform vec3  uColor;
uniform vec2  uResolution;
uniform float uTime;
uniform float uPixelSize;
uniform int   uShapeType;
uniform vec2  uMouse;

const int SHAPE_SQUARE   = 0;
const int SHAPE_CIRCLE   = 1;
const int SHAPE_TRIANGLE = 2;
const int SHAPE_DIAMOND  = 3;
const int SHAPE_HEXAGON  = 4;
const int SHAPE_STAR     = 5;
const int SHAPE_PENTAGON = 6;
const int SHAPE_OCTAGON  = 7;

const int MAX_CLICKS = 10;

uniform vec2  uClickPos  [MAX_CLICKS];
uniform float uClickTimes[MAX_CLICKS];

out vec4 fragColor;

// Bayer matrix helpers
float Bayer2(vec2 a) {
    a = floor(a);
    return fract(a.x / 2. + a.y * a.y * .75);
}

#define Bayer4(a) (Bayer2(.5*(a))*0.25 + Bayer2(a))
#define Bayer8(a) (Bayer4(.5*(a))*0.25 + Bayer2(a))

#define FBM_OCTAVES     5
#define FBM_LACUNARITY  1.25
#define FBM_GAIN        1.
#define FBM_SCALE       4.0

// Hash and noise
float hash11(float n) {
    return fract(sin(n)*43758.5453);
}

float vnoise(vec3 p) {
    vec3 ip = floor(p);
    vec3 fp = fract(p);

    float n000 = hash11(dot(ip + vec3(0.0,0.0,0.0), vec3(1.0,57.0,113.0)));
    float n100 = hash11(dot(ip + vec3(1.0,0.0,0.0), vec3(1.0,57.0,113.0)));
    float n010 = hash11(dot(ip + vec3(0.0,1.0,0.0), vec3(1.0,57.0,113.0)));
    float n110 = hash11(dot(ip + vec3(1.0,1.0,0.0), vec3(1.0,57.0,113.0)));
    float n001 = hash11(dot(ip + vec3(0.0,0.0,1.0), vec3(1.0,57.0,113.0)));
    float n101 = hash11(dot(ip + vec3(1.0,0.0,1.0), vec3(1.0,57.0,113.0)));
    float n011 = hash11(dot(ip + vec3(0.0,1.0,1.0), vec3(1.0,57.0,113.0)));
    float n111 = hash11(dot(ip + vec3(1.0,1.0,1.0), vec3(1.0,57.0,113.0)));

    vec3 w = fp*fp*fp*(fp*(fp*6.0-15.0)+10.0);

    float x00 = mix(n000, n100, w.x);
    float x10 = mix(n010, n110, w.x);
    float x01 = mix(n001, n101, w.x);
    float x11 = mix(n011, n111, w.x);

    float y0  = mix(x00, x10, w.y);
    float y1  = mix(x01, x11, w.y);

    return mix(y0, y1, w.z) * 2.0 - 1.0;
}

float fbm2(vec2 uv, float t, vec2 mouseInfluence) {
    vec3 p   = vec3(uv * FBM_SCALE + mouseInfluence * 0.3, t);
    float amp  = 1.;
    float freq = 1.;
    float sum  = 1.;

    for (int i = 0; i < FBM_OCTAVES; ++i) {
        sum  += amp * vnoise(p * freq);
        freq *= FBM_LACUNARITY;
        amp  *= FBM_GAIN;
    }

    return sum * 0.5 + 0.5;
}

float maskCircle(vec2 p, float cov) {
    float r = sqrt(cov) * .25;
    float d = length(p - 0.5) - r;
    float aa = 0.5 * fwidth(d);
    return cov * (1.0 - smoothstep(-aa, aa, d * 2.));
}

float maskTriangle(vec2 p, vec2 id, float cov) {
    bool flip = mod(id.x + id.y, 2.0) > 0.5;
    if (flip) p.x = 1.0 - p.x;
    float r = sqrt(cov);
    float d  = p.y - r*(1.0 - p.x);
    float aa = fwidth(d);
    return cov * clamp(0.5 - d/aa, 0.0, 1.0);
}

float maskDiamond(vec2 p, float cov) {
    float r = sqrt(cov) * 0.564;
    return step(abs(p.x - 0.49) + abs(p.y - 0.49), r);
}

float maskHexagon(vec2 p, float cov) {
    p = (p - 0.5) * 2.0;
    float r = sqrt(cov) * 0.5;
    const vec3 k = vec3(-0.866025404, 0.5, 0.577350269);
    p = abs(p);
    p -= 2.0 * min(dot(k.xy, p), 0.0) * k.xy;
    p -= vec2(clamp(p.x, -k.z*r, k.z*r), r);
    float d = length(p) * sign(p.y);
    float aa = fwidth(d);
    return cov * clamp(0.5 - d/aa, 0.0, 1.0);
}

float maskStar(vec2 p, float cov) {
    p = (p - 0.5) * 2.0;
    float r = sqrt(cov) * 0.5;
    const float an = 3.141593 / 5.0;
    const float en = 3.141593 / 10.0;
    const vec2 acs = vec2(cos(an), sin(an));
    const vec2 ecs = vec2(cos(en), sin(en));
    float bn = mod(atan(p.x, p.y), 2.0*an) - an;
    p = length(p) * vec2(cos(bn), abs(sin(bn)));
    p -= r * acs;
    p += ecs * clamp(-dot(p, ecs), 0.0, r*acs.y/ecs.y);
    float d = length(p) * sign(p.x);
    float aa = fwidth(d);
    return cov * clamp(0.5 - d/aa, 0.0, 1.0);
}

float maskPentagon(vec2 p, float cov) {
    p = (p - 0.5) * 2.0;
    float r = sqrt(cov) * 0.5;
    const vec3 k = vec3(0.809016994, 0.587785252, 0.726542528);
    p.x = abs(p.x);
    p -= 2.0 * min(dot(vec2(-k.x, k.y), p), 0.0) * vec2(-k.x, k.y);
    p -= 2.0 * min(dot(vec2(k.x, k.y), p), 0.0) * vec2(k.x, k.y);
    p -= vec2(clamp(p.x, -r*k.z, r*k.z), r);
    float d = length(p) * sign(p.y);
    float aa = fwidth(d);
    return cov * clamp(0.5 - d/aa, 0.0, 1.0);
}

float maskOctagon(vec2 p, float cov) {
    p = (p - 0.5) * 2.0;
    float r = sqrt(cov) * 0.5;
    const vec3 k = vec3(-0.9238795325, 0.3826834324, 0.4142135624);
    p = abs(p);
    p -= 2.0 * min(dot(vec2(k.x, k.y), p), 0.0) * vec2(k.x, k.y);
    p -= 2.0 * min(dot(vec2(-k.x, k.y), p), 0.0) * vec2(-k.x, k.y);
    p -= vec2(clamp(p.x, -k.z*r, k.z*r), r);
    float d = length(p) * sign(p.y);
    float aa = fwidth(d);
    return cov * clamp(0.5 - d/aa, 0.0, 1.0);
}

void main() {
    float pixelSize = uPixelSize;
    vec2 fragCoord = gl_FragCoord.xy - uResolution * .5;

    float aspectRatio = uResolution.x / uResolution.y;

    vec2 pixelId = floor(fragCoord / pixelSize);
    vec2 pixelUV = fract(fragCoord / pixelSize);

    float cellPixelSize =  8. * pixelSize;
    vec2 cellId = floor(fragCoord / cellPixelSize);

    vec2 cellCoord = cellId * cellPixelSize;
    vec2 uv = cellCoord / uResolution * vec2(aspectRatio, 1.0);

    // Mouse influence
    vec2 mouseInfluence = (uMouse - 0.5) * 2.0;

    // Animated fbm feed with mouse influence
    float feed = fbm2(uv, uTime * 0.05, mouseInfluence);
    feed = feed * 0.5 - 0.65;

    // Ripple clicks
    const float speed     = 0.30;
    const float thickness = 0.10;
    const float dampT     = 1.0;
    const float dampR     = 10.0;

    for (int i = 0; i < MAX_CLICKS; ++i) {
        vec2 pos = uClickPos[i];
        if (pos.x < 0.0) continue;

        vec2 cuv = (((pos - uResolution * .5 - cellPixelSize * .5) / (uResolution) )) * vec2(aspectRatio, 1.0);

        float t = max(uTime - uClickTimes[i], 0.0);
        float r = distance(uv, cuv);

        float waveR = speed * t;
        float ring  = exp(-pow((r - waveR) / thickness, 2.0));
        float atten = exp(-dampT * t) * exp(-dampR * r);
        feed = max(feed, ring * atten);
    }

    float bayer = Bayer8(fragCoord / uPixelSize) - 0.5;
    float bw    = step(0.5, feed + bayer);

    // Shape masking
    float coverage = bw;
    float M;
    if      (uShapeType == SHAPE_CIRCLE)   M = maskCircle(pixelUV, coverage);
    else if (uShapeType == SHAPE_TRIANGLE) M = maskTriangle(pixelUV, pixelId, coverage);
    else if (uShapeType == SHAPE_DIAMOND)  M = maskDiamond(pixelUV, coverage);
    else if (uShapeType == SHAPE_HEXAGON)  M = maskHexagon(pixelUV, coverage);
    else if (uShapeType == SHAPE_STAR)     M = maskStar(pixelUV, coverage);
    else if (uShapeType == SHAPE_PENTAGON) M = maskPentagon(pixelUV, coverage);
    else if (uShapeType == SHAPE_OCTAGON)  M = maskOctagon(pixelUV, coverage);
    else                                   M = coverage;

    vec3 color = uColor;
    fragColor = vec4(color, M);
}`;

    const canvas = document.createElement("canvas");
    const gl = canvas.getContext("webgl2");
    const renderer = new THREE.WebGLRenderer({
      canvas,
      context: gl,
      antialias: true,
      alpha: true,
    });
    heroContainer.appendChild(canvas);

    const scene = new THREE.Scene();
    const camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
    const MAX_CLICKS = 10;

    const uniforms = {
      uResolution: { value: new THREE.Vector2() },
      uTime: { value: 0 },
      uColor: { value: new THREE.Color(PALETTES[currentPalette].primary) },
      uClickPos: {
        value: Array.from(
          { length: MAX_CLICKS },
          () => new THREE.Vector2(-1, -1),
        ),
      },
      uClickTimes: { value: new Float32Array(MAX_CLICKS) },
      uShapeType: { value: SHAPES[currentShape] },
      uPixelSize: { value: 4.0 },
      uMouse: { value: new THREE.Vector2(0.5, 0.5) },
    };

    const materialMain = new THREE.ShaderMaterial({
      vertexShader: vertexShaderMain,
      fragmentShader: fragmentShaderMain,
      uniforms,
      glslVersion: THREE.GLSL3,
      transparent: true,
    });

    const quad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), materialMain);
    scene.add(quad);

    let clickIndex = 0;
    const clock = new THREE.Clock();
    let lastTopoUpdate = 0;

    canvas.addEventListener("pointerdown", (e) => {
      const rect = canvas.getBoundingClientRect();
      const cssX = e.clientX - rect.left;
      const cssY = e.clientY - rect.top;

      const fragX = (cssX * canvas.width) / rect.width;
      const fragY = ((rect.height - cssY) * canvas.height) / rect.height;

      uniforms.uClickPos.value[clickIndex].set(fragX, fragY);
      uniforms.uClickTimes.value[clickIndex] = uniforms.uTime.value;
      clickIndex = (clickIndex + 1) % MAX_CLICKS;
    });

    // Mouse tracking for influence
    window.addEventListener("mousemove", (e) => {
      mousePos.x = e.clientX / window.innerWidth;
      mousePos.y = 1.0 - e.clientY / window.innerHeight;
    });

    // Keyboard shortcuts
    window.addEventListener("keydown", (e) => {
      const shapes = Object.keys(SHAPES);

      if (e.key === "/") {
        e.preventDefault();
        showControls = !showControls;
      } else if (e.code === "Space") {
        e.preventDefault();
        cycleShape();
      } else if (e.key === "c" || e.key === "C") {
        cyclePalette();
      } else if (e.key >= "1" && e.key <= "8") {
        const index = parseInt(e.key, 10) - 1;
        if (index < shapes.length) {
          currentShape = shapes[index];
        }
      }
    });

    // Touch gestures for mobile
    let touchStartX = 0;
    let touchStartY = 0;

    canvas.addEventListener(
      "touchstart",
      (e) => {
        touchStartX = e.touches[0].clientX;
        touchStartY = e.touches[0].clientY;
      },
      { passive: true },
    );

    canvas.addEventListener(
      "touchmove",
      (e) => {
        const touch = e.touches[0];
        mousePos.x = touch.clientX / window.innerWidth;
        mousePos.y = 1.0 - touch.clientY / window.innerHeight;
      },
      { passive: true },
    );

    canvas.addEventListener(
      "touchend",
      (e) => {
        const touchEndX = e.changedTouches[0].clientX;
        const touchEndY = e.changedTouches[0].clientY;

        const deltaX = touchEndX - touchStartX;
        const deltaY = touchEndY - touchStartY;

        // Detect swipe (horizontal swipe > 50px)
        if (Math.abs(deltaX) > 50 && Math.abs(deltaX) > Math.abs(deltaY)) {
          if (deltaX > 0) {
            // Swipe right - next shape
            cycleShape();
          } else {
            // Swipe left - previous shape
            const shapes = Object.keys(SHAPES);
            const currentIndex = shapes.indexOf(currentShape);
            const prevIndex =
              (currentIndex - 1 + shapes.length) % shapes.length;
            currentShape = shapes[prevIndex];
          }
        }
      },
      { passive: true },
    );

    function resize() {
      const width = window.innerWidth;
      const height = window.innerHeight;
      renderer.setSize(width, height, false);
      uniforms.uResolution.value.set(width, height);

      if (topoCanvas) {
        topoCanvas.width = width;
        topoCanvas.height = height;
        const time = clock ? clock.getElapsedTime() : 0;
        drawTopographicMap(time);
      }
    }

    window.addEventListener("resize", resize);
    resize();

    function drawTopographicMap(time) {
      if (!topoCtx) return;

      const width = topoCanvas.width;
      const height = topoCanvas.height;

      topoCtx.fillStyle = "#000";
      topoCtx.fillRect(0, 0, width, height);

      const color = PALETTES[currentPalette].primary;
      topoCtx.strokeStyle = color;
      topoCtx.lineWidth = 1.5;
      topoCtx.lineCap = "round";
      topoCtx.lineJoin = "round";
      topoCtx.globalAlpha = 0.2;

      const spacing = 20; // Grid spacing for sampling (reduced for smoother lines)
      const elevationStep = 0.1; // Contour interval

      // Create elevation grid
      const gridW = Math.ceil(width / spacing) + 1;
      const gridH = Math.ceil(height / spacing) + 1;
      const elevationGrid = [];

      for (let y = 0; y < gridH; y++) {
        elevationGrid[y] = [];
        for (let x = 0; x < gridW; x++) {
          const worldX = x * spacing;
          const worldY = y * spacing;
          elevationGrid[y][x] = fbmNoise(worldX, worldY, 4, time * 0.1);
        }
      }

      // Draw contour lines using marching squares-like approach
      const labelPositions = [];

      for (let level = 0; level < 1; level += elevationStep) {
        topoCtx.beginPath();
        const contourPoints = [];

        for (let y = 0; y < gridH - 1; y++) {
          for (let x = 0; x < gridW - 1; x++) {
            const x0 = x * spacing;
            const y0 = y * spacing;
            const x1 = (x + 1) * spacing;
            const y1 = (y + 1) * spacing;

            const v00 = elevationGrid[y][x];
            const v10 = elevationGrid[y][x + 1];
            const v01 = elevationGrid[y + 1][x];
            const v11 = elevationGrid[y + 1][x + 1];

            // Check for contour crossing on each edge
            const edges = [];

            // Top edge
            if (
              (v00 < level && v10 >= level) ||
              (v00 >= level && v10 < level)
            ) {
              const t = (level - v00) / (v10 - v00);
              edges.push({ x: x0 + (x1 - x0) * t, y: y0 });
            }

            // Right edge
            if (
              (v10 < level && v11 >= level) ||
              (v10 >= level && v11 < level)
            ) {
              const t = (level - v10) / (v11 - v10);
              edges.push({ x: x1, y: y0 + (y1 - y0) * t });
            }

            // Bottom edge
            if (
              (v01 < level && v11 >= level) ||
              (v01 >= level && v11 < level)
            ) {
              const t = (level - v01) / (v11 - v01);
              edges.push({ x: x0 + (x1 - x0) * t, y: y1 });
            }

            // Left edge
            if (
              (v00 < level && v01 >= level) ||
              (v00 >= level && v01 < level)
            ) {
              const t = (level - v00) / (v01 - v00);
              edges.push({ x: x0, y: y0 + (y1 - y0) * t });
            }

            // Draw line segment if we have exactly 2 crossings
            if (edges.length === 2) {
              topoCtx.moveTo(edges[0].x, edges[0].y);
              topoCtx.lineTo(edges[1].x, edges[1].y);
              contourPoints.push({ x: edges[0].x, y: edges[0].y, level });
            }
          }
        }

        topoCtx.stroke();

        // Add label positions for this contour (every 3rd contour line)
        const levelIndex = Math.round(level / elevationStep);
        if (levelIndex % 3 === 0 && contourPoints.length > 0) {
          // Use fixed position based on level hash to prevent jumping
          const hash = (level * 99999) % 1;
          const angle = hash * Math.PI * 2;
          const radius = 0.3 * Math.min(width, height);

          labelPositions.push({
            x: width / 2 + Math.cos(angle) * radius,
            y: height / 2 + Math.sin(angle) * radius,
            elevation: Math.round(level * 1000),
          });
        }
      }

      // Draw elevation labels
      topoCtx.font = "11px 'Courier New', monospace";
      topoCtx.fillStyle = color;
      topoCtx.textAlign = "center";
      topoCtx.textBaseline = "middle";
      topoCtx.globalAlpha = 0.5;

      labelPositions.forEach((label) => {
        // Draw background box for better readability
        topoCtx.globalAlpha = 0.8;
        topoCtx.fillStyle = "#000";
        const text = label.elevation.toString();
        const textWidth = topoCtx.measureText(text).width;
        topoCtx.fillRect(
          label.x - textWidth / 2 - 3,
          label.y - 7,
          textWidth + 6,
          14,
        );

        // Draw text
        topoCtx.fillStyle = color;
        topoCtx.globalAlpha = 0.7;
        topoCtx.fillText(text, label.x, label.y);
      });

      topoCtx.globalAlpha = 1.0;
    }

    function render() {
      const time = clock.getElapsedTime();
      uniforms.uTime.value = time;
      uniforms.uShapeType.value = SHAPES[currentShape];
      uniforms.uColor.value.set(PALETTES[currentPalette].primary);
      uniforms.uMouse.value.set(mousePos.x, mousePos.y);

      // Update topographic map every 100ms for smooth animation
      if (time - lastTopoUpdate > 0.1) {
        drawTopographicMap(time);
        lastTopoUpdate = time;
      }

      renderer.render(scene, camera);
      requestAnimationFrame(render);
    }

    // Draw initial topographic map
    drawTopographicMap(0);

    render();

    // Hide loader after everything is ready
    setTimeout(() => {
      isLoading = false;
    }, 500);

    return () => {
      window.removeEventListener("resize", resize);
      renderer.dispose();
    };
  });

  function cycleShape() {
    const shapes = Object.keys(SHAPES);
    const currentIndex = shapes.indexOf(currentShape);
    const nextIndex = (currentIndex + 1) % shapes.length;
    currentShape = shapes[nextIndex];
  }

  function cyclePalette() {
    const palettes = Object.keys(PALETTES);
    const currentIndex = palettes.indexOf(currentPalette);
    const nextIndex = (currentIndex + 1) % palettes.length;
    currentPalette = palettes[nextIndex];
  }
</script>

{#if isLoading}
  <div class="loader">
    <div class="loader-spinner"></div>
    <div class="loader-text">Loading...</div>
  </div>
{/if}

<div class="clouds" bind:this={cloudsContainer}></div>
<div id="hero_bg" bind:this={heroContainer}></div>

{#if showControls}
  <div class="controls">
    <div class="shape-shifter">
      <button onclick={cycleShape} class="control-button">
        {currentShape}
      </button>
    </div>

    <div class="palette-switcher">
      <button
        onclick={cyclePalette}
        class="control-button"
        style="color: {PALETTES[currentPalette]
          .primary}; border-color: {PALETTES[currentPalette]
          .primary}; box-shadow: 0 0 10px {PALETTES[currentPalette].primary}33;"
      >
        {PALETTES[currentPalette].name}
      </button>
    </div>
  </div>
{/if}

<div class="label">Mokotahi</div>

<style>
  .loader {
    position: fixed;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    background: #000;
    z-index: 9999;
    animation: fadeOut 0.5s ease-out 0.5s forwards;
  }

  @keyframes fadeOut {
    to {
      opacity: 0;
      pointer-events: none;
    }
  }

  .loader-spinner {
    width: 60px;
    height: 60px;
    border: 3px solid rgba(212, 168, 23, 0.1);
    border-top-color: #d4a817;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .loader-text {
    margin-top: 1.5rem;
    color: #d4a817;
    font-family: "Courier New", monospace;
    font-size: 0.9rem;
    letter-spacing: 0.2em;
    text-transform: uppercase;
  }

  .clouds {
    position: absolute;
    inset: 0;
    pointer-events: none;
    z-index: 0;
  }

  :global(.cloud) {
    position: absolute;
    border-radius: 50%;
    background: #d4a817;
    filter: blur(40px);
    will-change: left, top;
  }

  #hero_bg {
    position: absolute;
    inset: 0;
    z-index: 2;
  }

  #hero_bg :global(canvas) {
    width: 100%;
    height: 100%;
    display: block;
  }

  .controls {
    position: absolute;
    top: 2rem;
    right: 2rem;
    z-index: 4;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    animation: fadeIn 0.3s ease-in;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateX(20px);
    }
    to {
      opacity: 1;
      transform: translateX(0);
    }
  }

  .shape-shifter,
  .palette-switcher {
    display: flex;
    justify-content: flex-end;
  }

  .control-button {
    background: none;
    border: 2px solid currentColor;
    color: inherit;
    padding: 0.75rem 1.5rem;
    font-family: "Courier New", monospace;
    font-size: 0.9rem;
    cursor: pointer;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    transition: all 0.2s;
    min-width: 140px;
  }

  .control-button:hover {
    background: currentColor;
    color: #000;
    transform: scale(1.05);
  }

  .label {
    position: absolute;
    inset: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: clamp(3rem, 10vw, 10rem);
    letter-spacing: -0.085em;
    mix-blend-mode: difference;
    pointer-events: none;
    font-weight: 600;
    z-index: 3;
  }

  @media (max-width: 768px) {
    .controls {
      top: 1rem;
      right: 1rem;
      gap: 0.75rem;
    }

    .control-button {
      padding: 0.5rem 1rem;
      font-size: 0.8rem;
      min-width: 100px;
    }
  }
</style>
