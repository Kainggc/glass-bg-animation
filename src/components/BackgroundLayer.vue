<script setup>
import { onMounted, onBeforeUnmount, ref, nextTick } from 'vue'
import * as THREE from 'three'

const container = ref(null)
const fallbackCanvas = ref(null)
let renderer, scene1, scene2, scene3, camera
let materialBalls, materialGlass, materialUI
let renderTarget, renderTarget2
let circles = []
const NUM_CIRCLES = 18 // 9 красных + 9 зелёных
const RENDER_SCALE = 1.0
const MAX_DROPS = 192
const BG_PASS_UPDATE_FPS = 34
const BG_PASS_UPDATE_INTERVAL = 1 / BG_PASS_UPDATE_FPS
const isWebGLSupported = ref(true)
const emit = defineEmits(['webgl-unsupported'])

// Явная проверка WebGL ДО создания рендерера
// Three.js иногда не бросает исключение при Mesa llvmpipe — просто пишет в консоль
const checkWebGLSupport = () => {
  try {
    const testCanvas = document.createElement('canvas')
    const ctx = testCanvas.getContext('webgl') || testCanvas.getContext('experimental-webgl')
    if (!ctx) return false
    // Проверяем что контекст реально работает
    const ext = ctx.getExtension('WEBGL_debug_renderer_info')
    // Принудительно пробуем создать буфер — если упадёт, WebGL нерабочий
    const buf = ctx.createBuffer()
    if (!buf) return false
    ctx.getError() // сбрасываем ошибки
    return true
  } catch (e) {
    return false
  }
}

// =============================================
// FALLBACK: Canvas 2D анимация (без WebGL)
// =============================================
let fallbackAnimId = null
const startFallbackAnimation = () => {
  const canvas = fallbackCanvas.value
  if (!canvas) return
  const ctx2d = canvas.getContext('2d')
  const resize = () => {
    canvas.width = window.innerWidth
    canvas.height = window.innerHeight
  }
  resize()
  window.addEventListener('resize', resize)

  // Простые движущиеся пятна для имитации метаболов
  const blobs = Array.from({ length: 6 }, () => ({
    x: Math.random() * window.innerWidth,
    y: Math.random() * window.innerHeight,
    vx: (Math.random() - 0.5) * 0.8,
    vy: (Math.random() - 0.5) * 0.8,
    r: 120 + Math.random() * 180,
    hue: Math.random() < 0.5 ? 0 : 140, // красный или зелёный
    phase: Math.random() * Math.PI * 2
  }))

  const draw = (time) => {
    const W = canvas.width, H = canvas.height
    ctx2d.clearRect(0, 0, W, H)
    ctx2d.fillStyle = '#050505'
    ctx2d.fillRect(0, 0, W, H)

    // Рисуем пятна с градиентом
    blobs.forEach((b, i) => {
      b.x += b.vx
      b.y += b.vy
      if (b.x < -b.r) b.x = W + b.r
      if (b.x > W + b.r) b.x = -b.r
      if (b.y < -b.r) b.y = H + b.r
      if (b.y > H + b.r) b.y = -b.r

      const brightness = 0.25 + 0.12 * Math.sin(time * 0.001 * (0.3 + i * 0.07) + b.phase)
      const alpha = brightness
      const g = ctx2d.createRadialGradient(b.x, b.y, 0, b.x, b.y, b.r)
      g.addColorStop(0, `hsla(${b.hue}, 100%, 45%, ${alpha})`)
      g.addColorStop(1, `hsla(${b.hue}, 100%, 20%, 0)`)
      ctx2d.beginPath()
      ctx2d.arc(b.x, b.y, b.r, 0, Math.PI * 2)
      ctx2d.fillStyle = g
      ctx2d.fill()
    })

    fallbackAnimId = requestAnimationFrame(draw)
  }
  fallbackAnimId = requestAnimationFrame(draw)

  return () => {
    window.removeEventListener('resize', resize)
    if (fallbackAnimId) cancelAnimationFrame(fallbackAnimId)
  }
}
let lastBackgroundPassTime = 0

// =============================================
// PASS 1: Шейдер метаболов (рендерится в текстуру)
// =============================================
const ballVertexShader = `
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
`

const ballFragmentShader = `
precision highp float;
uniform vec2 uResolution;
uniform vec3 uCircles[18];
uniform vec3 uColors[18];
varying vec2 vUv;

void main() {
    vec2 uv = vUv;
    float aspect = uResolution.x / uResolution.y;
    uv.x *= aspect;

    vec3 finalColor = vec3(0.0);
    float totalWeight = 0.0;
    float energyField = 0.0;

    for (int i = 0; i < 18; i++) {
        vec2 pos = vec2(uCircles[i].x * aspect, uCircles[i].y);
        float radius = uCircles[i].z;
        vec3 col = uColors[i];
        float dist = length(uv - pos);
        float influence = (radius * radius) / (dist * dist + 0.0001);
        energyField += influence;
        finalColor += col * influence;
        totalWeight += influence;
    }

    float circleAlpha = smoothstep(3.0, 25.0, energyField);
    vec3 circleColor = finalColor / max(totalWeight, 0.001);

    // Яркость регулируется в JS через uColors (анимация затухания)
    gl_FragColor = vec4(circleColor * circleAlpha, 1.0);
}
`

// =============================================
// PASS 2: Шейдер рефракции фона (Worley стекло)
// =============================================
const glassVertexShader = `
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
`

const glassFragmentShader = `
precision highp float;
uniform sampler2D uTexture;
uniform vec2 uSize;
uniform float uScale;
uniform float uWarp;
uniform float uIntensity;
varying vec2 vUv;

vec2 hash22(vec2 p) {
    vec3 p3 = fract(vec3(p.xyx) * vec3(0.1031, 0.1030, 0.0973));
    p3 += dot(p3, p3.yzx + 33.33);
    return fract((p3.xx + p3.yz) * p3.zy);
}

float worleyF2MinusF1(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    float d1 = 1e9;
    float d2 = 1e9;
    for (int y = -1; y <= 1; y++) {
        for (int x = -1; x <= 1; x++) {
            vec2 g = vec2(float(x), float(y));
            vec2 o = hash22(i + g);
            vec2 r = g + o - f;
            float d = dot(r, r);
            if (d < d1) { d2 = d1; d1 = d; }
            else if (d < d2) { d2 = d; }
        }
    }
    return sqrt(d2) - sqrt(d1);
}

vec2 warpVec(vec2 p) {
    float e = worleyF2MinusF1(p);
    float ex = worleyF2MinusF1(p + vec2(0.35, 0.0));
    float ey = worleyF2MinusF1(p + vec2(0.0, 0.35));
    vec2 g = vec2(ex - e, ey - e);
    return g / (length(g) + 1e-3);
}

void main() {
    vec2 uv = vUv;
    vec2 fragCoord = vUv * uSize;

    float scalePx = max(uScale, 1.0);
    vec2 p = fragCoord / scalePx;

    // Domain Warping — умеренно волнистые ячейки
    p += vec2(sin(p.y * 0.4) * 1.4, cos(p.x * 0.4) * 1.4);

    vec2 dir = warpVec(p);
    vec2 perp = vec2(-dir.y, dir.x);

    vec3 base = texture2D(uTexture, uv).rgb;

    float edge = worleyF2MinusF1(p);
    float ridge = 1.0 - smoothstep(0.07, 0.22, edge);
    ridge = pow(clamp(ridge, 0.0, 1.0), 1.6);

    float intensity = clamp(uIntensity, 0.0, 1.0);
    vec2 invSize = 1.0 / max(uSize, vec2(1.0));

    vec2 baseOffset = dir * (uWarp * invSize);
    vec2 blurVec = dir * (uWarp * (0.55 + 1.65 * ridge)) * invSize;
    float blurLen = length(blurVec);

    vec2 uvW = clamp(uv + baseOffset, vec2(0.0), vec2(1.0));
    vec2 uv1 = clamp(uvW + blurVec * 0.60, vec2(0.0), vec2(1.0));
    vec2 uv2 = clamp(uvW - blurVec * 0.60, vec2(0.0), vec2(1.0));
    vec2 uv3 = clamp(uvW + perp * (blurLen * 0.55), vec2(0.0), vec2(1.0));
    vec2 uv4 = clamp(uvW - perp * (blurLen * 0.55), vec2(0.0), vec2(1.0));

    vec3 s0 = texture2D(uTexture, uvW).rgb;
    vec3 s1 = texture2D(uTexture, uv1).rgb;
    vec3 s2 = texture2D(uTexture, uv2).rgb;
    vec3 s3 = texture2D(uTexture, uv3).rgb;
    vec3 s4 = texture2D(uTexture, uv4).rgb;

    vec3 refracted = (s0 * 0.34) + (s1 * 0.16) + (s2 * 0.16) + (s3 * 0.17) + (s4 * 0.17);

    vec2 n2 = dir * 0.5 + 0.5;
    float light = clamp(
        dot(
            normalize(vec3(n2 * 2.0 - 1.0, 1.0)),
            normalize(vec3(0.35, -0.2, 1.0))
        ),
        0.0, 1.0
    );
    float spec = pow(light, 12.0) * ridge;

    vec3 shaded = refracted + vec3(spec) * 0.08;
    vec3 outColor = mix(base, shaded, intensity);

    gl_FragColor = vec4(outColor, 1.0);
}
`

// =============================================
// PASS 3: SDF-капли поверх фона (стеклянная линза)
// =============================================
const uiVertexShader = `
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
`

const uiFragmentShader = `
precision highp float;
uniform sampler2D uBackground;
uniform vec2 uResolution;
#define MAX_DROPS 192
uniform vec4 uDrops[MAX_DROPS];
uniform int uDropCount;
uniform float uDropRadius[MAX_DROPS];
uniform vec2 uDropAxis[MAX_DROPS];
uniform float uDropStretch[MAX_DROPS];

varying vec2 vUv;

// Функции шума для искажения структуры капли (синхронизировано со стеклом)
vec2 hash22(vec2 p) {
    vec3 p3 = fract(vec3(p.xyx) * vec3(0.1031, 0.1030, 0.0973));
    p3 += dot(p3, p3.yzx + 33.33);
    return fract((p3.xx + p3.yz) * p3.zy);
}

float worleyF2MinusF1(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    float d1 = 1e9;
    float d2 = 1e9;
    for (int y = -1; y <= 1; y++) {
        for (int x = -1; x <= 1; x++) {
            vec2 g = vec2(float(x), float(y));
            vec2 o = hash22(i + g);
            vec2 r = g + o - f;
            float d = dot(r, r);
            if (d < d1) { d2 = d1; d1 = d; }
            else if (d < d2) { d2 = d; }
        }
    }
    return sqrt(d2) - sqrt(d1);
}

// Smooth minimum для слипания соседних капель
float smin(float a, float b, float k) {
    float h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0);
    return mix(b, a, h) - k * h * (1.0 - h);
}

float sdTeardrop(vec2 p, float r, float stretch, float structureDistortion) {
    float s = 1.0 + stretch * 0.85;
    vec2 a = vec2(-r * (0.62 + 0.92 * s), 0.0);
    vec2 b = vec2( r * (0.28 + 0.10 * s), 0.0);
    float tailR = max(r * (0.17 - 0.03 * stretch), r * 0.10);
    float headR = r * (1.00 + 0.08 * stretch);

    vec2 ba = b - a;
    float denom = max(dot(ba, ba), 1e-5);
    float h = clamp(dot(p - a, ba) / denom, 0.0, 1.0);
    vec2 c = a + ba * h;
    float localR = mix(tailR, headR, pow(h, 0.74)) + structureDistortion;
    float dBody = length(p - c) - localR;

    // Заостряем хвост: чем дальше к хвосту, тем уже профиль
    float tailMask = smoothstep(0.34, 0.0, h);
    float tailWidth = (tailR + structureDistortion * 0.45) * (0.25 + 2.3 * h);
    float dTail = abs(p.y) - tailWidth;
    return max(dBody, mix(-1e3, dTail, tailMask));
}

float getSDF(vec2 uv, float aspect) {
    float d = 1e9;

    // Искажение структуры на основе шума стекла (Worley)
    // Масштаб 15.0 и ворп 0.4 синхронизированы с glassFragmentShader
    vec2 pNoise = uv * vec2(aspect, 1.0) * (uResolution.y / 15.0);
    pNoise += vec2(sin(pNoise.y * 0.4) * 1.4, cos(pNoise.x * 0.4) * 1.4);
    float noiseVal = worleyF2MinusF1(pNoise);

    for (int i = 0; i < MAX_DROPS; i++) {
        if (i >= uDropCount) break;

        vec2 dropCenter = uDrops[i].xy;
        vec2 p = uv - dropCenter;
        p.x *= aspect;

        float r = uDropRadius[i];
        vec2 axis = uDropAxis[i];
        float axisLen = length(axis);
        if (axisLen < 1e-5) axis = vec2(0.0, -1.0);
        else axis /= axisLen;
        vec2 perp = vec2(-axis.y, axis.x);
        vec2 pLocal = vec2(dot(p, axis), dot(p, perp));
        float stretch = clamp(uDropStretch[i], 0.0, 1.0);
        
        // Влияние структуры стекла на форму капли
        // Края капли становятся слегка неровными там, где есть "ребра" стекла
        float structureDistortion = noiseVal * 0.017;
        float dist = sdTeardrop(pLocal, r, stretch, structureDistortion);

        // Легкое сглаживание слияния соседних капель
        float localK = r * 0.33;
        d = (i == 0) ? dist : smin(d, dist, localK);
    }
    return d;
}

void main() {
    vec2 uv = vUv;
    float aspect = uResolution.x / uResolution.y;
    vec3 bg = texture2D(uBackground, uv).rgb;

    float d = getSDF(uv, aspect);

    // Далеко от капли — рисуем чистый фон
    if (d > 0.05) {
        gl_FragColor = vec4(bg, 1.0);
        return;
    }

    // --- Капля как выпуклая линза ---
#if USE_DERIVATIVES == 1
    vec2 n = normalize(vec2(dFdx(d), dFdy(d)));
#else
    float eps = 0.001;
    float dX = getSDF(uv + vec2(eps, 0.0), aspect) - getSDF(uv - vec2(eps, 0.0), aspect);
    float dY = getSDF(uv + vec2(0.0, eps), aspect) - getSDF(uv - vec2(0.0, eps), aspect);
    vec2 n = normalize(vec2(dX, dY));
#endif

    // Плавный профиль: максимум на краю, убывает к центру
    float distToEdge = smoothstep(0.05, -0.05, d);
    float centerLens = smoothstep(0.03, -0.06, d);
    float edgeLens = pow(distToEdge, 1.08);

    // Рефракция (усиленная линза + легкий внутренний сдвиг)
    vec2 tangent = vec2(-n.y, n.x);
    vec2 refrShift = n * (0.085 * edgeLens + 0.020 * centerLens);
    vec2 swirlShift = tangent * (0.008 * centerLens);
    vec2 refrUv = clamp(uv - refrShift + swirlShift, vec2(0.0), vec2(1.0));

    // Усиленная хроматика и мягкое внутреннее размытие
    float disp = (0.016 + 0.010 * centerLens) * edgeLens;
    vec2 ca = n * disp;
    vec2 blur = tangent * (disp * 0.50);

    vec3 c0 = texture2D(uBackground, refrUv).rgb;
    vec3 c1 = texture2D(uBackground, clamp(refrUv + ca * 0.80, vec2(0.0), vec2(1.0))).rgb;
    vec3 c2 = texture2D(uBackground, clamp(refrUv - ca * 0.80, vec2(0.0), vec2(1.0))).rgb;
    vec3 c3 = texture2D(uBackground, clamp(refrUv + blur, vec2(0.0), vec2(1.0))).rgb;
    vec3 c4 = texture2D(uBackground, clamp(refrUv - blur, vec2(0.0), vec2(1.0))).rgb;

    vec3 waterColor = vec3(c1.r, c0.g, c2.b);
    waterColor = waterColor * 0.62 + (c0 + c3 + c4) * (0.38 / 3.0);

    // Мягкий край
    float alpha = smoothstep(0.002, -0.002, d);
    gl_FragColor = vec4(mix(bg, waterColor, alpha), 1.0);
}
`

// Генерация шаров: 9 красных + 9 зелёных
// Детерминированные параметры метаболов — 60-секундный цикл через синусоиды
// [cx, cy, ax, ay, fx, fy, φx, φy, радиус, частота_яркости, фаза_яркости, color]
// cx/cy: центр орбиты, ax/ay: амплитуда, fx/fy: целые множители
const CIRCLE_DEFS = [
  // ------- 9 красных -------
  [0.25, 0.30, 0.20, 0.25, 1, 2, 0.00, 0.52, 0.55, 1, 0.0,  'red'],
  [0.70, 0.25, 0.18, 0.30, 2, 1, 1.05, 0.00, 0.65, 2, 1.0,  'red'],
  [0.50, 0.60, 0.30, 0.20, 1, 3, 2.09, 1.57, 0.48, 3, 2.1,  'red'],
  [0.80, 0.70, 0.15, 0.25, 3, 2, 0.52, 2.09, 0.70, 1, 0.5,  'red'],
  [0.15, 0.75, 0.20, 0.15, 2, 3, 1.57, 0.84, 0.58, 2, 3.2,  'red'],
  [0.60, 0.40, 0.25, 0.28, 3, 1, 3.14, 1.26, 0.42, 3, 1.8,  'red'],
  [0.35, 0.50, 0.22, 0.20, 1, 2, 0.31, 2.62, 0.62, 1, 2.5,  'red'],
  [0.85, 0.45, 0.12, 0.22, 2, 1, 1.88, 0.31, 0.50, 2, 0.3,  'red'],
  [0.08, 0.58, 0.13, 0.18, 2, 3, 2.74, 1.02, 0.52, 2, 2.2,  'red'],
  // ------- 9 зелёных -------
  [0.40, 0.20, 0.28, 0.22, 1, 3, 2.41, 1.78, 0.60, 3, 1.5,  'green'],
  [0.65, 0.80, 0.20, 0.18, 3, 2, 0.84, 3.14, 0.72, 1, 2.9,  'green'],
  [0.20, 0.55, 0.18, 0.30, 2, 1, 2.93, 0.63, 0.45, 2, 0.7,  'green'],
  [0.75, 0.60, 0.25, 0.20, 1, 2, 1.26, 2.30, 0.68, 3, 3.5,  'green'],
  [0.50, 0.35, 0.22, 0.32, 3, 3, 0.63, 1.57, 0.53, 1, 1.2,  'green'],
  [0.30, 0.80, 0.30, 0.15, 2, 1, 3.67, 0.21, 0.78, 2, 0.0,  'green'],
  [0.88, 0.30, 0.10, 0.28, 1, 2, 1.05, 2.62, 0.40, 3, 2.7,  'green'],
  [0.12, 0.40, 0.15, 0.20, 3, 1, 2.09, 0.84, 0.57, 1, 1.9,  'green'],
  [0.92, 0.66, 0.11, 0.19, 2, 3, 0.42, 2.48, 0.49, 2, 1.1,  'green'],
]

for (let i = 0; i < NUM_CIRCLES; i++) {
  const d = CIRCLE_DEFS[i]
  const isRed = d[11] === 'red'
  const color = isRed
    ? new THREE.Vector3(1.0, 0.0, 0.0)
    : new THREE.Vector3(0.0, 1.0, 0.0)
  circles.push({
    cx: d[0], cy: d[1], ax: d[2], ay: d[3],
    fx: d[4], fy: d[5], φx: d[6], φy: d[7],
    radius: d[8],
    brightFreq: d[9], brightPhase: d[10],
    color: color,
    baseColor: color.clone()
  })
}

const init = () => {
  console.log('[BG] init()')

  // Сначала явно проверяем поддержку WebGL
  if (!checkWebGLSupport()) {
    console.warn('[BG] WebGL недоступен — запускаем CSS/Canvas2D fallback')
    isWebGLSupported.value = false
    emit('webgl-unsupported')
    return
  }

  if (!container.value) return

  const w = Math.floor(window.innerWidth * RENDER_SCALE)
  const h = Math.floor(window.innerHeight * RENDER_SCALE)
  console.log('[BG] Canvas:', w, 'x', h)
    // === Init Renderer ===
    try {
      renderer = new THREE.WebGLRenderer({
        powerPreference: 'high-performance',
        antialias: false,
        stencil: false,
        depth: false
      })
    } catch (e) {
      console.error('[BG] WebGL Context Error:', e)
      isWebGLSupported.value = false
      emit('webgl-unsupported')
      return
    }

    // Дополнительная проверка: контекст реально создан?
    if (!renderer || !renderer.getContext || !renderer.getContext()) {
      console.warn('[BG] Three.js WebGLRenderer вернул невалидный контекст')
      if (renderer) renderer.dispose()
      renderer = null
      isWebGLSupported.value = false
      emit('webgl-unsupported')
      return
    }

    renderer.setSize(w, h)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    renderer.setClearColor(0x000000, 1)

    renderer.domElement.style.width = '100%'
    renderer.domElement.style.height = '100%'
    renderer.domElement.style.imageRendering = 'auto'
    renderer.domElement.style.willChange = 'transform'

    container.value.appendChild(renderer.domElement)

  camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1)

  // === RenderTarget для Pass 1 (метаболы) ===
  renderTarget = new THREE.WebGLRenderTarget(w, h, {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat
  })

  // === RenderTarget для Pass 2 (рефракция фона) ===
  renderTarget2 = new THREE.WebGLRenderTarget(w, h, {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat
  })

  // === Scene 1: Метаболы ===
  scene1 = new THREE.Scene()
  const circleUniforms = circles.map(c => new THREE.Vector3(c.x, c.y, c.radius))
  const colorUniforms = circles.map(c => c.color)

  materialBalls = new THREE.ShaderMaterial({
    vertexShader: ballVertexShader,
    fragmentShader: ballFragmentShader,
    uniforms: {
      uResolution: { value: new THREE.Vector2(w, h) },
      uCircles: { value: circleUniforms },
      uColors: { value: colorUniforms }
    }
  })

  const geo1 = new THREE.PlaneGeometry(2, 2)
  scene1.add(new THREE.Mesh(geo1, materialBalls))

  // === Scene 2: Рефракция стекла (Worley) ===
  scene2 = new THREE.Scene()

  materialGlass = new THREE.ShaderMaterial({
    vertexShader: glassVertexShader,
    fragmentShader: glassFragmentShader,
    uniforms: {
      uTexture: { value: renderTarget.texture },
      uSize: { value: new THREE.Vector2(w, h) },
      uScale: { value: 15.0 },
      uWarp: { value: 45.0 },
      uIntensity: { value: 1.0 }
    }
  })

  const geo2 = new THREE.PlaneGeometry(2, 2)
  scene2.add(new THREE.Mesh(geo2, materialGlass))

  // === Scene 3: SDF-капли ===
  scene3 = new THREE.Scene()

  const dropUniforms = []
  for (let i = 0; i < MAX_DROPS; i++) {
      dropUniforms.push(new THREE.Vector4(0, 0, 0, 0))
  }
  const dropAxisUniforms = []
  for (let i = 0; i < MAX_DROPS; i++) {
      dropAxisUniforms.push(new THREE.Vector2(0, -1))
  }

  const canUseDerivatives = renderer.capabilities.isWebGL2 || renderer.extensions.has('OES_standard_derivatives')

  materialUI = new THREE.ShaderMaterial({
    vertexShader: uiVertexShader,
    fragmentShader: uiFragmentShader,
    defines: {
      USE_DERIVATIVES: canUseDerivatives ? 1 : 0
    },
    extensions: {
      derivatives: !renderer.capabilities.isWebGL2 && canUseDerivatives
    },
    uniforms: {
      uBackground: { value: renderTarget2.texture },
      uResolution: { value: new THREE.Vector2(w, h) },
      uDrops: { value: dropUniforms },
      uDropCount: { value: 0 },
      uDropRadius: { value: new Array(MAX_DROPS).fill(0.001) },
      uDropAxis: { value: dropAxisUniforms },
      uDropStretch: { value: new Array(MAX_DROPS).fill(0.0) }
    }
  })

  const geo3 = new THREE.PlaneGeometry(2, 2)
  scene3.add(new THREE.Mesh(geo3, materialUI))

  // Первый кадр (3 прохода: метаболы, Worley-фон, капли)
  renderBackgroundPasses(performance.now() / 1000, true)
  renderer.setRenderTarget(null)
  renderer.render(scene3, camera)
  console.log('[BG] First frame OK (three-pass)')
  animate()
}

// drops: Array of { x, y, width, height, flowX, flowY, stretch }
const setDrops = (drops) => {
    if (!materialUI) return

    const count = Math.min(drops.length, MAX_DROPS)
    materialUI.uniforms.uDropCount.value = count

    const w = window.innerWidth
    const h = window.innerHeight

    for (let i = 0; i < count; i++) {
        const d = drops[i]

        // CSS: начало сверху-слева, Y вниз. GL/UV: начало снизу-слева, Y вверх.
        const cx = d.x + d.width / 2
        const cy = d.y + d.height / 2

        const nx = cx / w
        const ny = 1.0 - (cy / h)

        materialUI.uniforms.uDrops.value[i].set(nx, ny, d.width / w, d.height / h)

        // Радиус нормируется по высоте вьюпорта
        const radius = d.radius || Math.min(d.width, d.height) / 2
        materialUI.uniforms.uDropRadius.value[i] = radius / h

        const flowX = Number.isFinite(d.flowX) ? d.flowX : 0
        const flowY = Number.isFinite(d.flowY) ? d.flowY : -1
        const flowLen = Math.hypot(flowX, flowY)
        if (flowLen > 1e-5) {
            materialUI.uniforms.uDropAxis.value[i].set(flowX / flowLen, flowY / flowLen)
        } else {
            materialUI.uniforms.uDropAxis.value[i].set(0, -1)
        }
        materialUI.uniforms.uDropStretch.value[i] = Math.max(0, Math.min(1, d.stretch || 0))
    }
}

const CYCLE_BG = 60 // цикл фона тоже 60 секунд

const updateCircles = (timeSec) => {
  const t = timeSec % CYCLE_BG
  const k = (2 * Math.PI) / CYCLE_BG

  circles.forEach((c, i) => {
    // Позиция по синусоидальным орбитам
    const x = c.cx + c.ax * Math.sin(k * c.fx * t + c.φx)
    const y = c.cy + c.ay * Math.sin(k * c.fy * t + c.φy)
    materialBalls.uniforms.uCircles.value[i].set(x, y, c.radius)

    // Яркость тоже зациклена
    const brightness = 0.25 + 0.15 * Math.sin(k * c.brightFreq * t + c.brightPhase)
    c.color.copy(c.baseColor).multiplyScalar(brightness)
  })
}

const renderBackgroundPasses = (timeSec, force = false) => {
  if (!renderer || !scene1 || !scene2 || !camera) return
  if (!force && (timeSec - lastBackgroundPassTime) < BG_PASS_UPDATE_INTERVAL) return

  updateCircles(timeSec)
  renderer.setRenderTarget(renderTarget)
  renderer.render(scene1, camera)
  renderer.setRenderTarget(renderTarget2)
  renderer.render(scene2, camera)
  lastBackgroundPassTime = timeSec
}

const animate = () => {
  requestAnimationFrame(animate)
  if (!materialBalls || !materialGlass || !materialUI) return

  const nowSec = performance.now() / 1000
  renderBackgroundPasses(nowSec)

  // Pass 3: SDF-капли поверх фона
  materialUI.uniforms.uBackground.value = renderTarget2.texture
  renderer.setRenderTarget(null)
  renderer.render(scene3, camera)
}

const onResize = () => {
  if (!renderer || !materialBalls) return
  const w = Math.floor(window.innerWidth * RENDER_SCALE)
  const h = Math.floor(window.innerHeight * RENDER_SCALE)
  renderer.setSize(w, h)
  renderer.domElement.style.width = '100%'
  renderer.domElement.style.height = '100%'
  renderTarget.setSize(w, h)
  renderTarget2.setSize(w, h)

  materialBalls.uniforms.uResolution.value.set(w, h)
  materialGlass.uniforms.uSize.value.set(w, h)
  materialUI.uniforms.uResolution.value.set(w, h)
  lastBackgroundPassTime = 0
}

// Expose to parent
defineExpose({ setDrops, isWebGLSupported })

let stopFallback = null
onMounted(() => {
  init()
  if (isWebGLSupported.value) {
    window.addEventListener('resize', onResize)
  } else {
    // Ждём следующего тика — Vue должен перерисовать DOM
    // чтобы fallbackCanvas.value стал доступен в шаблоне
    nextTick(() => {
      stopFallback = startFallbackAnimation()
    })
  }
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize)
  if (renderTarget) renderTarget.dispose()
  if (renderTarget2) renderTarget2.dispose()
  if (renderer) renderer.dispose()
  if (stopFallback) stopFallback()
})
</script>

<template>
  <div v-if="isWebGLSupported" ref="container" class="background-container"></div>
  <div v-else class="fallback-bg">
    <canvas ref="fallbackCanvas" class="fallback-canvas"></canvas>
  </div>
</template>

<style scoped>
.background-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
  background-color: black;
  overflow: hidden;
  pointer-events: none;
}
.fallback-bg {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  background: #050505; z-index: 0; overflow: hidden;
}
.fallback-canvas {
  position: absolute; top: 0; left: 0;
  width: 100%; height: 100%;
  display: block;
}
</style>
