<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import BackgroundLayer from './components/BackgroundLayer.vue'

const backgroundLayer = ref(null)

let rafId = null
let lastTime = 0

const CYCLE = 60
const TWO_PI = Math.PI * 2
const RAIN_COUNT = 180
const RIDGE_COUNT = 14
const BASE_SPEED_MULT = 1.512

const LANE_PULL = 10.6
const LANE_DAMPING = 5.8
const LANE_SWITCH_HYST = 0.006
const FLOW_BIAS = 0.054
const FLOW_MICRO_AMPL = 0.24
const FLOW_MICRO_TIME = 0.148
const FLOW_ID_JITTER = 0.12
const FLOW_CURVE_LOOKAHEAD = 0.045
const FLOW_LANE_WAVE_AMPL = 0.11
const FLOW_RIDGE_CURVATURE_GAIN = 0.32
const FLOW_LANE_SWITCH_IMPULSE = 0.018
const LANE_SLOPE_COUPLING = 1.15
const MAX_LATERAL_SPEED = 0.11
const RIDGE_SWAY_SCALE = 2.2
const MASS_SPEED_GAIN = 1.35
const MASS_SPEED_EXP = 1.12
const MAX_SPEED_FACTOR = 6.6
const MAX_MASS_FACTOR = 8.0
const MERGE_DISTANCE = 1.05
const MERGE_RETENTION = 0.98
const MERGE_ANIM_MIN = 0.19
const MERGE_ANIM_MAX = 0.52
const MERGE_SUB_REST_MASS = 0.08
const MERGE_SPEED_BOOST_GAIN = 0.72
const MERGE_SPEED_BOOST_DECAY = 1.45
const MERGE_SPEED_BOOST_CAP = 1.35
const MERGE_BASE_SPEED_GAIN = 0.4
const MERGE_FINAL_VY_BOOST = 1.2
const MERGE_MIN_SPEED_UP = 0.12
const MIN_RENDER_RADIUS = 0.35
const MAIN_RADIUS_SCALE = 0.52
const DROP_STRETCH_MIN = 0.22
const DROP_STRETCH_SPEED = 0.19
const DROP_STRETCH_MASS = 0.12
const TRAIL_MAX_DROPS = 420
const TRAIL_SPAWN_RATE = 8.0
const TRAIL_MASS_MIN_RATIO = 0.055
const TRAIL_MASS_MAX_RATIO = 0.17
const TRAIL_RADIUS_SCALE = 0.95
const TRAIL_MIN_RENDER_RADIUS = 0.045
const TRAIL_LANE_PULL = 6.2
const TRAIL_MAX_LATERAL_SPEED = 0.023
const TRAIL_AFTERLIFE_RATIO = 0.8
const TRAIL_AFTERLIFE_SHRINK = 0.42
const RENDER_DROP_BUDGET = 192
const MAIN_RENDER_SHARE = 0.36
const MIN_TRAIL_RENDER_BUDGET = 92

const DEFS = []

const getFixedRandom = (i, seed) => {
  const x = Math.sin(i * 123.456 + seed * 789.012) * 10000
  return x - Math.floor(x)
}

const clamp = (v, a, b) => Math.max(a, Math.min(b, v))
const clampInt = (v, a, b) => Math.max(a, Math.min(b, v))
const randRange = (a, b) => a + (b - a) * Math.random()

function smoothstep(min, max, value) {
  const x = Math.max(0, Math.min(1, (value - min) / (max - min)))
  return x * x * (3 - 2 * x)
}

const getTargetVyForMass = (drop, mass, boost = 0) => {
  const massRatio = clamp(mass / drop.baseMass, 0.6, MAX_MASS_FACTOR)
  const speedFactor = 0.92 + MASS_SPEED_GAIN * Math.pow(massRatio, MASS_SPEED_EXP)
  return drop.baseSpeed * Math.min(speedFactor * (1 + boost), MAX_SPEED_FACTOR)
}

for (let i = 0; i < RAIN_COUNT; i++) {
  const type = getFixedRandom(i, 0) < 0.14 ? 0 : 1
  const rBase = type === 0 ? 7.2 : 4.7
  const rVar = type === 0 ? 2.2 : 1.8
  const radius = rBase + getFixedRandom(i, 1) * rVar
  const speed = ((0.5 + getFixedRandom(i, 8) * 2.5) / CYCLE) * BASE_SPEED_MULT
  const x = 0.02 + getFixedRandom(i, 2) * 0.96
  const y = getFixedRandom(i, 3)
  DEFS.push([x, y, speed, radius])
}

const ridgeDefs = Array.from({ length: RIDGE_COUNT }, (_, lane) => ({
  phase1: getFixedRandom(lane, 30),
  phase2: getFixedRandom(lane, 31),
  f1: 0.45 + getFixedRandom(lane, 32) * 0.65,
  f2: 1.1 + getFixedRandom(lane, 33) * 0.9,
  a1: ((0.08 + getFixedRandom(lane, 34) * 0.08) / RIDGE_COUNT) * RIDGE_SWAY_SCALE,
  a2: ((0.03 + getFixedRandom(lane, 35) * 0.05) / RIDGE_COUNT) * RIDGE_SWAY_SCALE
}))

const laneCenter = (lane, y) => {
  const p = ridgeDefs[lane]
  const base = (lane + 0.5) / RIDGE_COUNT
  const w1 = (y * p.f1 + p.phase1) * TWO_PI
  const w2 = (y * p.f2 + p.phase2) * TWO_PI
  return clamp(base + Math.sin(w1) * p.a1 + Math.sin(w2) * p.a2, 0.01, 0.99)
}

const laneSlope = (lane, y) => {
  const p = ridgeDefs[lane]
  const w1 = (y * p.f1 + p.phase1) * TWO_PI
  const w2 = (y * p.f2 + p.phase2) * TWO_PI
  return (
    Math.cos(w1) * (TWO_PI * p.f1) * p.a1 +
    Math.cos(w2) * (TWO_PI * p.f2) * p.a2
  )
}

const closestLane = (x, y, currentLane) => {
  let bestLane = currentLane
  let bestDist = Math.abs(x - laneCenter(currentLane, y))
  const start = Math.max(0, currentLane - 3)
  const end = Math.min(RIDGE_COUNT - 1, currentLane + 3)

  for (let lane = start; lane <= end; lane++) {
    const dist = Math.abs(x - laneCenter(lane, y))
    if (dist < bestDist) {
      bestDist = dist
      bestLane = lane
    }
  }

  return bestLane
}

const makeDrop = (d, index) => {
  const lane = clampInt(Math.floor(d[0] * RIDGE_COUNT), 0, RIDGE_COUNT - 1)
  const baseMass = d[3] * d[3]
  const xOffset = (getFixedRandom(index, 40) - 0.5) * (1 / RIDGE_COUNT) * 0.25
  const y = d[1]
  return {
    id: index,
    startLane: lane,
    lane,
    x: clamp(laneCenter(lane, y) + xOffset, 0.01, 0.99),
    y,
    vx: 0,
    vy: d[2] * (0.8 + getFixedRandom(index, 41) * 0.4),
    spawnBaseSpeed: d[2],
    baseSpeed: d[2],
    baseMass,
    mass: baseMass * (0.85 + getFixedRandom(index, 42) * 0.35),
    respawnCount: 0,
    mergePartner: -1,
    mergeSpeedBoost: 0,
    flowSeed: getFixedRandom(index, 90)
  }
}

const drops = DEFS.map(makeDrop)
const activeMerges = []
const trailDrops = []

const spawnTrailDrops = (w, h, dt, now) => {
  if (trailDrops.length >= TRAIL_MAX_DROPS) return
  const invH = 1 / Math.max(h, 1)

  for (let i = 0; i < drops.length; i++) {
    if (trailDrops.length >= TRAIL_MAX_DROPS) break

    const drop = drops[i]
    if (drop.mergePartner !== -1) continue
    if (drop.y < 0.02 || drop.y > 1.02) continue

    const speedRatio = clamp(drop.vy / Math.max(drop.baseSpeed, 1e-5), 0.0, MAX_SPEED_FACTOR)
    const massRatio = clamp(drop.mass / Math.max(drop.baseMass, 1e-5), 0.6, MAX_MASS_FACTOR)
    const spawnRate = TRAIL_SPAWN_RATE *
      clamp(speedRatio - 0.55, 0.0, 3.0) *
      (0.9 + 0.5 * Math.min(Math.sqrt(massRatio) / 2.5, 1.0))
    if (Math.random() > spawnRate * dt) continue

    const modeRoll = Math.random()
    const mode = modeRoll < 0.4 ? 0 : (modeRoll < 0.95 ? 1 : 2)

    const vyMain = Math.max(drop.vy, drop.baseSpeed * 0.45)
    const dirLen = Math.hypot(drop.vx, vyMain)
    const dirX = dirLen > 1e-5 ? drop.vx / dirLen : 0
    const dirY = dirLen > 1e-5 ? vyMain / dirLen : 1

    const mainRadiusNorm = (Math.sqrt(drop.mass) * MAIN_RADIUS_SCALE) * invH
    const backOffset = mainRadiusNorm * randRange(1.2, 3.8)
    const jitter = mainRadiusNorm * randRange(-0.45, 0.45)

    const baseMass = Math.max(
      drop.mass * randRange(TRAIL_MASS_MIN_RATIO, TRAIL_MASS_MAX_RATIO),
      0.2
    )
    const life =
      mode === 0 ? randRange(6.8, 12.8) :
      mode === 1 ? randRange(6.4, 13.6) :
      randRange(4.0, 7.6)

    const vy =
      mode === 0 ? drop.vy * randRange(0.32, 0.58) :
      mode === 1 ? drop.vy * randRange(0.1, 0.24) :
      drop.vy * randRange(0.14, 0.35)
    const vx = drop.vx * randRange(0.35, 0.72) + randRange(-0.0052, 0.0052)

    trailDrops.push({
      x: clamp(drop.x - dirX * backOffset + jitter, 0.01, 0.99),
      y: drop.y - dirY * backOffset - mainRadiusNorm * randRange(0.04, 0.28),
      vx,
      vy,
      baseSpeed: drop.baseSpeed * randRange(0.55, 0.95),
      mass: baseMass,
      baseMass,
      life,
      lifeMax: life,
      mode,
      drag: mode === 0 ? randRange(1.4, 2.7) : randRange(2.1, 4.2),
      shrink: mode === 0 ? randRange(0.09, 0.18) : mode === 1 ? randRange(0.12, 0.24) : randRange(0.32, 0.62),
      targetVy: mode === 0 ? drop.baseSpeed * randRange(0.2, 0.38) : 0,
      laneHint: drop.lane,
      seed: now * 0.11 + i * 0.37 + randRange(0, 1)
    })
  }
}

const updateTrailDrops = (dt, now) => {
  for (let i = trailDrops.length - 1; i >= 0; i--) {
    const t = trailDrops[i]
    t.life -= dt
    const afterLife = t.lifeMax * TRAIL_AFTERLIFE_RATIO
    if (t.life <= -afterLife || t.y > 1.09 || t.mass < 0.0035) {
      trailDrops.splice(i, 1)
      continue
    }

    t.laneHint = closestLane(t.x, t.y, clampInt(t.laneHint || 0, 0, RIDGE_COUNT - 1))
    const center = laneCenter(t.laneHint, t.y)
    const centerAhead = laneCenter(t.laneHint, t.y + 0.02)
    const grooveBend = centerAhead - center
    const slope = laneSlope(t.laneHint, t.y)
    const slopeAhead = laneSlope(t.laneHint, t.y + 0.02)
    const ridgeImpact = (slopeAhead - slope) * 1.9
    const flowNoise =
      Math.sin((t.y * (1.5 + t.laneHint * 0.032) + now * 0.038 + t.seed) * TWO_PI) * 0.17 +
      Math.sin((t.y * (2.3 + t.laneHint * 0.021) + now * 0.061 + t.seed * 1.7) * TWO_PI) * 0.07
    t.vx += (center - t.x) * TRAIL_LANE_PULL * dt + grooveBend * 0.34 + ridgeImpact * 0.12 + flowNoise * dt

    if (t.mode === 0) {
      t.vy += (t.targetVy - t.vy) * (1 - Math.exp(-2.9 * dt))
      t.mass *= Math.exp(-t.shrink * dt)
    } else if (t.mode === 1) {
      t.vy += (0 - t.vy) * (1 - Math.exp(-6.4 * dt))
      const lifeK = clamp(t.life / Math.max(t.lifeMax, 1e-5), 0.0, 1.0)
      const extraShrink = lifeK < 0.42 ? 0.18 : 0.0
      t.mass *= Math.exp(-(t.shrink + extraShrink) * dt)
    } else {
      t.vy *= Math.exp(-2.35 * dt)
      t.mass *= Math.exp(-(t.shrink + 0.5) * dt)
    }

    if (t.life < 0) {
      const tailK = clamp(-t.life / Math.max(afterLife, 1e-5), 0.0, 1.0)
      t.vy *= Math.exp(-(0.75 + tailK * 0.95) * dt)
      t.mass *= Math.exp(-(TRAIL_AFTERLIFE_SHRINK + tailK * 0.55) * dt)
    }

    t.vx *= Math.exp(-t.drag * dt)
    t.vx = clamp(t.vx, -TRAIL_MAX_LATERAL_SPEED, TRAIL_MAX_LATERAL_SPEED)
    t.vy = Math.max(t.vy, 0)

    t.x = clamp(t.x + t.vx * dt, 0.01, 0.99)
    t.y += t.vy * dt
  }
}

const respawnDrop = (drop, index, fromMerge = false) => {
  drop.respawnCount += 1
  const salt = drop.respawnCount * 19 + (fromMerge ? 500 : 0)

  let lane = Math.floor(getFixedRandom(index, 60 + salt) * RIDGE_COUNT)
  if (!fromMerge && getFixedRandom(index, 61 + salt) < 0.7) {
    lane = drop.startLane
  }
  lane = clampInt(lane, 0, RIDGE_COUNT - 1)

  drop.lane = lane
  drop.y = -getFixedRandom(index, 62 + salt) * (fromMerge ? 0.25 : 0.6)
  drop.vx = 0
  drop.baseSpeed = drop.spawnBaseSpeed * (0.96 + getFixedRandom(index, 66 + salt) * 0.08)
  drop.vy = drop.baseSpeed * (0.72 + getFixedRandom(index, 63 + salt) * 0.44)

  const massScale = fromMerge ? 0.78 : 1.0
  drop.mass = drop.baseMass * massScale * (0.84 + getFixedRandom(index, 64 + salt) * 0.4)

  const xOffset = (getFixedRandom(index, 65 + salt) - 0.5) * (1 / RIDGE_COUNT) * 0.22
  drop.x = clamp(laneCenter(lane, drop.y) + xOffset, 0.01, 0.99)
  drop.mergePartner = -1
  drop.mergeSpeedBoost = 0
}

const updateDrop = (drop, dt, now) => {
  if (drop.mergePartner !== -1) return

  const prevLane = drop.lane
  const aheadYForLane = drop.y + Math.max(0.015, Math.min(0.075, drop.vy * 0.95))
  const bestLane = closestLane(drop.x, aheadYForLane, drop.lane)
  if (bestLane !== drop.lane) {
    const currentDist = Math.abs(drop.x - laneCenter(drop.lane, aheadYForLane))
    const bestDist = Math.abs(drop.x - laneCenter(bestLane, aheadYForLane))
    if (bestDist + LANE_SWITCH_HYST < currentDist) {
      drop.lane = bestLane
    }
  }
  if (drop.lane !== prevLane) {
    const laneDelta = drop.lane - prevLane
    const speedRatio = clamp(drop.vy / Math.max(drop.baseSpeed, 1e-5), 0.8, 3.8)
    drop.vx += laneDelta * FLOW_LANE_SWITCH_IMPULSE * speedRatio
  }

  const center = laneCenter(drop.lane, drop.y)
  const lookAheadY = drop.y + FLOW_CURVE_LOOKAHEAD + Math.min(0.045, Math.max(0.0, drop.vy * 0.65))
  const centerAhead = laneCenter(drop.lane, lookAheadY)
  const grooveBend = centerAhead - center
  const slope = laneSlope(drop.lane, drop.y)
  const slopeAhead = laneSlope(drop.lane, lookAheadY)
  const curvature = (slopeAhead - slope) / Math.max(lookAheadY - drop.y, 1e-5)
  const lanePhase = ridgeDefs[drop.lane].phase2
  const baseFlow = Math.sin(
    (drop.y * (1.15 + drop.lane * 0.028) + lanePhase) * TWO_PI
  ) * 0.32
  const microFlow = Math.sin(
    (drop.y * (1.95 + drop.lane * 0.041) + lanePhase * 1.7 + now * FLOW_MICRO_TIME) * TWO_PI
  ) * FLOW_MICRO_AMPL
  const idJitter = Math.sin((now * (0.19 + (drop.id % 7) * 0.018) + drop.id * 0.23 + drop.y * 1.35) * TWO_PI) * FLOW_ID_JITTER
  const laneWave = Math.sin((drop.y * (1.42 + drop.lane * 0.022) + drop.flowSeed * 0.71 + now * 0.054) * TWO_PI) * FLOW_LANE_WAVE_AMPL
  const localFlow = baseFlow + microFlow + idJitter + laneWave
  const ridgeImpact = curvature * Math.max(drop.vy, drop.baseSpeed * 0.65) * FLOW_RIDGE_CURVATURE_GAIN
  const targetX = center + localFlow * (1 / RIDGE_COUNT) * FLOW_BIAS + grooveBend * 1.35

  drop.vx += (targetX - drop.x) * LANE_PULL * dt
  const speedDrive = Math.max(drop.vy, drop.baseSpeed * 0.65)
  const patternPush = slope * (0.11 + speedDrive * 1.8) * LANE_SLOPE_COUPLING
  const groovePush = grooveBend * (1.2 + 0.9 * Math.abs(localFlow))
  const curvaturePush = ridgeImpact * (1.2 + 0.8 * Math.abs(localFlow))
  const targetVx =
    patternPush +
    groovePush +
    curvaturePush
  drop.vx += (targetVx - drop.vx) * (1 - Math.exp(-10.0 * dt))
  drop.vx += (localFlow * 0.07 + grooveBend * 2.8) * dt
  drop.vx += ridgeImpact * (0.95 + 0.55 * Math.abs(localFlow)) * dt
  drop.vx *= Math.exp(-LANE_DAMPING * dt)
  drop.vx = clamp(drop.vx, -MAX_LATERAL_SPEED, MAX_LATERAL_SPEED)

  const targetVy = getTargetVyForMass(drop, drop.mass, drop.mergeSpeedBoost)
  drop.vy += (targetVy - drop.vy) * (1 - Math.exp(-4.8 * dt))
  drop.vy = Math.max(drop.vy, drop.baseSpeed * 0.5)
  drop.mergeSpeedBoost = Math.max(0, drop.mergeSpeedBoost - MERGE_SPEED_BOOST_DECAY * dt)

  drop.x = clamp(drop.x + drop.vx * dt, 0.01, 0.99)
  drop.y += drop.vy * dt

  if (drop.y > 1.08) {
    respawnDrop(drop, drop.id, false)
  }
}

const chooseMergeRoles = (i, j) => {
  const massI = drops[i].mass
  const massJ = drops[j].mass
  const diff = Math.abs(massI - massJ) / Math.max(massI, massJ, 1e-6)

  if (diff < 0.12) {
    return drops[i].y >= drops[j].y
      ? { mainIndex: i, subIndex: j }
      : { mainIndex: j, subIndex: i }
  }

  return massI >= massJ
    ? { mainIndex: i, subIndex: j }
    : { mainIndex: j, subIndex: i }
}

const startMerge = (i, j) => {
  const { mainIndex, subIndex } = chooseMergeRoles(i, j)
  const main = drops[mainIndex]
  const sub = drops[subIndex]

  if (main.mergePartner !== -1 || sub.mergePartner !== -1) return

  const sizeFactor = clamp((Math.sqrt(main.mass) + Math.sqrt(sub.mass)) / 26.0, 0.0, 1.0)
  const duration = MERGE_ANIM_MAX - (MERGE_ANIM_MAX - MERGE_ANIM_MIN) * sizeFactor

  main.mergePartner = subIndex
  sub.mergePartner = mainIndex

  activeMerges.push({
    mainIndex,
    subIndex,
    t: 0,
    duration,
    mainMass0: main.mass,
    subMass0: sub.mass,
    mainVy0: main.vy,
    mainBaseSpeed0: main.baseSpeed,
    subBaseSpeed0: sub.baseSpeed
  })
}

const detectMergeCandidates = (w, h) => {
  const order = drops.map((_, index) => index).sort((a, b) => drops[a].y - drops[b].y)
  const aspect = w / Math.max(h, 1)

  for (let ai = 0; ai < order.length; ai++) {
    const i = order[ai]
    if (drops[i].mergePartner !== -1) continue

    for (let bi = ai + 1; bi < order.length; bi++) {
      const j = order[bi]
      if (drops[j].mergePartner !== -1) continue

      const dyRaw = drops[j].y - drops[i].y
      if (dyRaw > 0.09) break
      if (Math.abs(drops[i].lane - drops[j].lane) > 1) continue

      const radiusI = Math.sqrt(drops[i].mass) / h
      const radiusJ = Math.sqrt(drops[j].mass) / h
      const mergeDist = (radiusI + radiusJ) * MERGE_DISTANCE

      const dx = (drops[i].x - drops[j].x) * aspect
      const dy = drops[i].y - drops[j].y
      if (dx * dx + dy * dy > mergeDist * mergeDist) continue

      startMerge(i, j)
      break
    }
  }
}

const updateMerges = (dt) => {
  for (let i = activeMerges.length - 1; i >= 0; i--) {
    const merge = activeMerges[i]
    const main = drops[merge.mainIndex]
    const sub = drops[merge.subIndex]

    if (main.mergePartner !== sub.id || sub.mergePartner !== main.id) {
      activeMerges.splice(i, 1)
      continue
    }

    merge.t = Math.min(1.0, merge.t + dt / merge.duration)
    const k = smoothstep(0.0, 1.0, merge.t)

    const mainTargetMass = Math.min(
      merge.mainMass0 + merge.subMass0 * MERGE_RETENTION,
      main.baseMass * MAX_MASS_FACTOR
    )
    const massGainRatio = clamp(
      (mainTargetMass - merge.mainMass0) / Math.max(merge.mainMass0, 1e-6),
      0.0,
      3.0
    )
    const subTargetMass = Math.max(
      merge.subMass0 * (1.0 - k),
      sub.baseMass * MERGE_SUB_REST_MASS
    )
    main.mass = merge.mainMass0 + (mainTargetMass - merge.mainMass0) * k
    sub.mass = subTargetMass

    const total = Math.max(main.mass + sub.mass, 1e-6)
    const centerX = clamp((main.x * main.mass + sub.x * sub.mass) / total, 0.01, 0.99)
    const centerY = (main.y * main.mass + sub.y * sub.mass) / total

    main.x += (centerX - main.x) * (0.2 + 0.35 * k)
    sub.x += (centerX - sub.x) * (0.45 + 0.4 * k)
    main.y += (centerY - main.y) * (0.14 + 0.22 * k)
    sub.y += (centerY - sub.y) * (0.35 + 0.32 * k)

    const commonVx = (main.vx * main.mass + sub.vx * sub.mass) / total
    const commonVy = Math.max(
      (main.vy * main.mass + sub.vy * sub.mass) / total,
      main.baseSpeed * 0.7
    )
    const mergePhaseBoost = massGainRatio * 0.28 * k
    const targetVyMain = getTargetVyForMass(
      main,
      main.mass,
      main.mergeSpeedBoost + mergePhaseBoost
    )
    const targetVySub = getTargetVyForMass(sub, sub.mass)
    const guidedVyMain = Math.max(
      commonVy,
      targetVyMain * (1.0 + 0.2 * massGainRatio * k)
    )
    const guidedVySub = Math.max(commonVy * (0.82 - 0.25 * k), targetVySub * 0.6)

    main.vx += (commonVx - main.vx) * (0.2 + 0.35 * k)
    sub.vx += (commonVx - sub.vx) * (0.38 + 0.42 * k)
    const mainVyDelta = (guidedVyMain - main.vy) * (0.42 + 0.5 * k)
    if (mainVyDelta > 0) main.vy += mainVyDelta
    sub.vy += (guidedVySub - sub.vy) * (0.42 + 0.4 * k)

    main.y += main.vy * dt * (0.96 + 0.18 * k)
    sub.y += sub.vy * dt * (0.88 + 0.14 * k)
    main.x = clamp(main.x, 0.01, 0.99)
    sub.x = clamp(sub.x, 0.01, 0.99)

    main.lane = closestLane(main.x, main.y, main.lane)
    sub.lane = closestLane(sub.x, sub.y, sub.lane)

    if (merge.t >= 1.0 || main.y > 1.08 || sub.y > 1.08) {
      const blendedTotal = Math.max(main.mass + sub.mass, 1e-6)
      main.x = clamp((main.x * main.mass + sub.x * sub.mass) / blendedTotal, 0.01, 0.99)
      main.y = (main.y * main.mass + sub.y * sub.mass) / blendedTotal
      main.vx = (main.vx * main.mass + sub.vx * sub.mass) / blendedTotal
      const carriedMainMass = main.mass
      const carriedSubMass = sub.mass
      const finalTotal = Math.max(carriedMainMass + carriedSubMass, 1e-6)
      const mergeBaseMassTotal = Math.max(merge.mainMass0 + merge.subMass0, 1e-6)
      const mergedBaseSpeed =
        (merge.mainBaseSpeed0 * merge.mainMass0 + merge.subBaseSpeed0 * merge.subMass0) /
        mergeBaseMassTotal
      main.mass = mainTargetMass
      main.baseSpeed = clamp(
        mergedBaseSpeed * (1 + massGainRatio * MERGE_BASE_SPEED_GAIN),
        main.spawnBaseSpeed * 0.85,
        main.spawnBaseSpeed * 2.8
      )
      const mergeBoost = clamp(
        0.22 + massGainRatio * MERGE_SPEED_BOOST_GAIN,
        0.0,
        MERGE_SPEED_BOOST_CAP
      )
      main.mergeSpeedBoost = Math.max(main.mergeSpeedBoost, mergeBoost)
      const finalTargetVy = getTargetVyForMass(main, main.mass, main.mergeSpeedBoost)
      const finalImpulseVy = main.baseSpeed * (0.28 + massGainRatio * MERGE_FINAL_VY_BOOST)
      const mergeBaselineVy = merge.mainVy0 * (1 + MERGE_MIN_SPEED_UP + 0.18 * massGainRatio)
      const finalVyFloor = Math.min(
        Math.max(finalTargetVy * (1 + 0.08 + 0.12 * massGainRatio), mergeBaselineVy),
        finalTargetVy * 1.8
      )
      main.vy = Math.max(
        (main.vy * carriedMainMass + sub.vy * carriedSubMass) / finalTotal + finalImpulseVy,
        finalVyFloor
      )
      main.mergePartner = -1
      sub.mergePartner = -1

      if (main.y > 1.08) {
        respawnDrop(main, main.id, false)
      }
      respawnDrop(sub, sub.id, true)

      activeMerges.splice(i, 1)
    }
  }
}

const buildRenderDrops = (w, h) => {
  const mainDrops = []
  const trailDrops = []

  for (let i = 0; i < drops.length; i++) {
    const drop = drops[i]
    const topFade = smoothstep(-0.04, 0.03, drop.y)
    const bottomFade = 1.0 - smoothstep(0.93, 1.05, drop.y)
    const edgeFade = clamp(topFade * bottomFade, 0.0, 1.0)
    const radius = Math.sqrt(drop.mass) * edgeFade * MAIN_RADIUS_SCALE
    if (radius < MIN_RENDER_RADIUS * MAIN_RADIUS_SCALE) continue

    const speedRatio = clamp(drop.vy / Math.max(drop.baseSpeed, 1e-5), 0.5, MAX_SPEED_FACTOR)
    const massRatio = clamp(drop.mass / Math.max(drop.baseMass, 1e-5), 0.6, MAX_MASS_FACTOR)
    const stretch = clamp(
      DROP_STRETCH_MIN +
      (speedRatio - 1.0) * DROP_STRETCH_SPEED +
      (massRatio - 1.0) * DROP_STRETCH_MASS,
      0.0,
      1.0
    )

    mainDrops.push({
      x: drop.x * w - radius,
      y: drop.y * h - radius,
      width: radius * 2,
      height: radius * 2,
      radius,
      flowX: drop.vx,
      flowY: -drop.vy,
      stretch,
      priority: radius
    })
  }

  for (let i = 0; i < trailDrops.length; i++) {
    const t = trailDrops[i]
    const topFade = smoothstep(-0.05, 0.03, t.y)
    const bottomFade = 1.0 - smoothstep(0.93, 1.05, t.y)
    const lifeSpan = Math.max(t.lifeMax, 1e-5)
    const afterLife = lifeSpan * TRAIL_AFTERLIFE_RATIO
    const lifeNorm = clamp((t.life + afterLife) / (lifeSpan + afterLife), 0.0, 1.0)
    const lifeFade = 0.72 + 0.28 * smoothstep(0.0, 1.0, Math.pow(lifeNorm, 0.6))
    const edgeFade = clamp(topFade * bottomFade, 0.0, 1.0) * lifeFade
    const radius = Math.sqrt(t.mass) * TRAIL_RADIUS_SCALE * edgeFade
    if (radius < TRAIL_MIN_RENDER_RADIUS) continue

    const stretch = clamp(0.08 + (t.vy / Math.max(t.baseSpeed, 1e-5)) * 0.22, 0.05, 0.45)

    trailDrops.push({
      x: t.x * w - radius,
      y: t.y * h - radius,
      width: radius * 2,
      height: radius * 2,
      radius,
      flowX: t.vx,
      flowY: -Math.max(t.vy, t.baseSpeed * 0.08),
      stretch,
      priority: radius * (0.7 + 0.3 * lifeFade)
    })
  }

  mainDrops.sort((a, b) => b.priority - a.priority)
  trailDrops.sort((a, b) => b.priority - a.priority)

  let mainBudget = Math.min(
    mainDrops.length,
    Math.floor(RENDER_DROP_BUDGET * MAIN_RENDER_SHARE)
  )
  let trailBudget = Math.min(
    trailDrops.length,
    RENDER_DROP_BUDGET - mainBudget
  )

  if (trailDrops.length > 0 && trailBudget < MIN_TRAIL_RENDER_BUDGET) {
    const minTrailBudget = Math.min(MIN_TRAIL_RENDER_BUDGET, trailDrops.length, RENDER_DROP_BUDGET)
    const moved = Math.min(mainBudget, minTrailBudget - trailBudget)
    mainBudget -= moved
    trailBudget += moved
  }

  const result = []
  for (let i = 0; i < mainBudget; i++) {
    const b = mainDrops[i]
    b.priority = undefined
    result.push(b)
  }
  for (let i = 0; i < trailBudget; i++) {
    const b = trailDrops[i]
    b.priority = undefined
    result.push(b)
  }

  return result
}

const animate = () => {
  rafId = requestAnimationFrame(animate)
  if (!backgroundLayer.value) return

  const w = window.innerWidth
  const h = window.innerHeight
  const now = performance.now() / 1000
  const dt = lastTime ? Math.min(0.033, now - lastTime) : 0.016
  lastTime = now

  for (let i = 0; i < drops.length; i++) {
    updateDrop(drops[i], dt, now)
  }

  detectMergeCandidates(w, h)
  updateMerges(dt)
  spawnTrailDrops(w, h, dt, now)
  updateTrailDrops(dt, now)
  backgroundLayer.value.setDrops(buildRenderDrops(w, h))
}

onMounted(() => {
  lastTime = 0
  animate()
})

onBeforeUnmount(() => {
  if (rafId) cancelAnimationFrame(rafId)
})
</script>

<template>
  <BackgroundLayer ref="backgroundLayer" />
</template>

<style>
*, *::before, *::after { margin: 0; padding: 0; box-sizing: border-box; }
html, body, #app { width: 100%; height: 100%; overflow: hidden; background: #000; }
</style>
