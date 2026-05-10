// === params ===
const circleR    = 160;   // 円盤の半径（描画基準）
const canvasSize = 500;   // キャンバスの一辺
const margin     = 2;     // 円盤の内マージン

let frameMs   = 400;      // Play時の1コマ間隔（ms）

// --- 共通UI（参照を保持） ---
let fpsSlider, dotSizeSlider, lineWeightSlider;
let playBtn, nextBtn, prevBtn, resetBtn;
let addRingBtn, crossBtn, overflowBtn; // delRingBtnは撤去
let saveCircleBtn, saveSlitCircleBtn, slitPreviewBtn;
let stepDivSlider, stepDivInput;
let slitThicknessSlider, slitLengthSlider;
let ringContainer, ui;  // 右パネル参照

// --- 状態 ---
let playing = false;
let curAng = 0;              // ステップで回す全体角（rad）
let lastTick = 0;
let rings = [];              // 複数リングを格納

// ▼ 表示モード
let showCenterCross = false; // 中心十字の表示
let allowOverflow   = false; // 黒円からのはみ出し許可
let showSlitPreview = false; // スリット付き円盤のプレビュー

// ===== 便利関数（UI用） =====
function injectResponsiveLayoutCSS() {
  createElement('style', `
    body {
      margin: 0;
      overflow-x: hidden;
      background: #fff;
    }

    #layout {
      display: grid;
      grid-template-columns: minmax(320px, 500px) minmax(360px, 680px);
      gap: 20px;
      align-items: start;
      max-width: 1220px;
      margin: 18px auto;
      padding: 0 16px;
      box-sizing: border-box;
    }

    #canvasWrap,
    #uiWrap {
      min-width: 0;
    }

    #canvasWrap canvas {
      display: block;
      width: 100% !important;
      height: auto !important;
      max-width: 500px;
    }

    #controlPanel {
      box-sizing: border-box;
    }

    @media (max-width: 920px) {
      #layout {
        grid-template-columns: 1fr;
        max-width: 680px;
        gap: 12px;
        margin: 10px auto;
        padding: 0 10px;
      }

      #canvasWrap canvas {
        position: static !important;
        top: auto !important;
        max-width: 100%;
      }

      #controlPanel {
        max-height: none !important;
        overflow: visible !important;
        padding-right: 0 !important;
      }
    }
  `);
}

function createPanel(parent) {
  return createDiv()
    .parent(parent)
    .id('controlPanel')
    .style('width', '100%')
    .style('font-family', 'system-ui, -apple-system, Segoe UI, Roboto, sans-serif')
    .style('color', '#222')
    .style('line-height', '1.2')
    .style('user-select', 'none')
    // デスクトップでは右パネルだけ独立スクロール
    .style('max-height', 'calc(100vh - 36px)')
    .style('overflow', 'auto')
    .style('padding-right', '6px'); // スクロールバー分の余白
}

function section(parent, title, opts={}) {
  const box = createDiv().parent(parent)
    .style('border', '1px solid #cfcfcf')
    .style('border-radius', '10px')
    .style('padding', '12px')
    .style('margin', '8px 0')
    .style('background', '#fbfbfb');
  createDiv(title).parent(box)
    .style('font-weight', '700')
    .style('margin-bottom', '8px')
    .style('font-size', '14px');
  if (opts.subtitle) {
    createDiv(opts.subtitle).parent(box)
      .style('color', '#666').style('font-size', '12px').style('margin-top', '-4px');
  }
  return box;
}

function labeledSlider(parent, label, min, max, val, step, unit='', tooltip='') {
  const row = createDiv().parent(parent)
    .style('display', 'grid')
    .style('grid-template-columns', '110px 1fr 48px')
    .style('gap', '8px')
    .style('align-items', 'center')
    .style('margin', '6px 0');

  const lab = createSpan(label).parent(row).style('font-size', '13px');
  if (tooltip) lab.attribute('title', tooltip);

  const sld = createSlider(min, max, val, step).parent(row).style('width', '100%');
  const valBox = createSpan(String(val) + (unit||'')).parent(row)
    .style('text-align', 'right').style('color', '#444').style('font-variant-numeric','tabular-nums');

  sld.input(() => valBox.html(sld.value() + (unit||'')));
  return { slider: sld, valueEl: valBox, rowEl: row };
}

function pillButton(label) {
  return createButton(label)
    .style('padding', '8px 12px')
    .style('border-radius', '10px')
    .style('border', '1px solid #cfcfcf')
    .style('background', '#fff')
    .style('cursor', 'pointer');
}

function smallHint(parent, text) {
  createDiv(text).parent(parent).style('color','#666').style('font-size','12px').style('margin-top','4px');
}

// ====== p5.js ======
function setup() {
  injectResponsiveLayoutCSS();

  const layout = createDiv().id('layout');
  const canvasWrap = createDiv().id('canvasWrap').parent(layout);
  const uiWrap = createDiv().id('uiWrap').parent(layout);

  // キャンバス作成（デスクトップでは左固定＝sticky）
  const cnv = createCanvas(canvasSize, canvasSize);
  cnv.parent(canvasWrap);
  cnv.style('position', 'sticky');
  cnv.style('top', '16px');
  cnv.style('z-index', '5');
  cnv.style('box-shadow', '0 2px 6px rgba(0,0,0,0.08)');

  angleMode(RADIANS);
  textFont('monospace');

  // 右パネル（デスクトップでは独立スクロール、スマホでは縦積み）
  ui = createPanel(uiWrap);

  // ---- 基本設定 ----
  const secBasic = section(ui, '基本設定');
  fpsSlider = labeledSlider(
    secBasic, 'FPS', 1, 50, 20, 1, '',
    '毎秒のコマ数。高いほど滑らか（負荷↑）'
  ).slider;

  dotSizeSlider = labeledSlider(
    secBasic, 'ドット半径', 1, 20, 4, 1, 'px',
    '点モードの点の大きさ'
  ).slider;

  lineWeightSlider = labeledSlider(
    secBasic, '線の太さ', 1, 10, 2, 1, 'px',
    '線モードの線幅'
  ).slider;

  const secStep = section(ui, '回転ステップ');
  createDiv('「1回転＝360°を何分割するか」。数が大きいほど1コマの回転が小さくなる。')
    .parent(secStep).style('font-size','12px').style('color','#555').style('margin','4px 0 8px');

  const stepRow = createDiv().parent(secStep)
    .style('display','grid')
    .style('grid-template-columns','110px 1fr 60px 48px')
    .style('gap','8px')
    .style('align-items','center');

  createSpan('分割数').parent(stepRow).style('font-size','13px');
  stepDivSlider = createSlider(1, 60, 12, 1).parent(stepRow).style('width','100%');
  stepDivInput  = createInput('12','number').parent(stepRow).style('width','100%');
  stepDivInput.attribute('min','1'); stepDivInput.attribute('max','360');
  createSpan('等分').parent(stepRow).style('text-align','right').style('color','#444');

  stepDivSlider.input(() => stepDivInput.value(String(stepDivSlider.value())));
  stepDivInput.input(() => {
    let v = parseInt(stepDivInput.value(), 10);
    if (!Number.isFinite(v)) v = 12;
    v = Math.max(1, Math.min(360, v));
    stepDivInput.value(String(v));
    stepDivSlider.value(v);
  });
  smallHint(secStep, '例）12等分 → 30°ずつ回転 / 24等分 → 15°ずつ回転');

  // ---- 再生/コマ送り ----
  const secPlay = section(ui, '再生 / コマ送り');
  const btnRow1 = createDiv().parent(secPlay)
    .style('display','grid')
    .style('grid-template-columns','repeat(3, minmax(120px, 1fr))')
    .style('gap','8px');

  playBtn  = pillButton('▶︎ 再生').parent(btnRow1);
  nextBtn  = pillButton('⤴︎ 次のコマ').parent(btnRow1);
  prevBtn  = pillButton('⤵︎ 前のコマ').parent(btnRow1);

  const btnRow2 = createDiv().parent(secPlay)
    .style('display','grid')
    .style('grid-template-columns','repeat(3, minmax(120px, 1fr))')
    .style('gap','8px').style('margin-top','6px');

  resetBtn = pillButton('↺ リセット').parent(btnRow2);
  crossBtn = pillButton('✚ 中心十字 切替').parent(btnRow2);
  overflowBtn = pillButton('⛶ はみ出し許可: OFF').parent(btnRow2);

  playBtn.mousePressed(() => {
    playing = !playing;
    playBtn.html(playing ? '⏸ 停止' : '▶︎ 再生');
    lastTick = millis();
  });
  nextBtn.mousePressed(() => stepOnce(+1, getStepRad()));
  prevBtn.mousePressed(() => stepOnce(-1, getStepRad()));
  resetBtn.mousePressed(() => { playing = false; playBtn.html('▶︎ 再生'); curAng = 0; });
  crossBtn.mousePressed(() => { showCenterCross = !showCenterCross; });
  overflowBtn.mousePressed(() => {
    allowOverflow = !allowOverflow;
    overflowBtn.html(allowOverflow ? '⛶ はみ出し許可: ON' : '⛶ はみ出し許可: OFF');
  });

  // ---- 保存 ----
  const secSave = section(ui, '保存', {subtitle:'スリット数は「回転ステップ」の分割数に自動同期する。'});

  const saveBtnRow = createDiv().parent(secSave)
    .style('display','grid')
    .style('grid-template-columns','repeat(3, minmax(120px, 1fr))')
    .style('gap','8px');

  saveCircleBtn = pillButton('💾 円保存').parent(saveBtnRow);
  saveSlitCircleBtn = pillButton('💾 スリット＋円保存').parent(saveBtnRow);
  slitPreviewBtn = pillButton('👁 スリットプレビュー: OFF').parent(saveBtnRow);

  saveCircleBtn.attribute('title','中央の黒円領域だけを正方形に切り出してPNG保存します。');
  saveSlitCircleBtn.attribute('title','外周にスリットを付けた円盤全体を正方形に切り出してPNG保存します。');
  slitPreviewBtn.attribute('title','保存前に、スリット付き円盤の見た目をキャンバス上で確認します。');

  saveCircleBtn.mousePressed(saveCircleImageHiRes);
  saveSlitCircleBtn.mousePressed(saveSlitCircleImageHiRes);
  slitPreviewBtn.mousePressed(() => {
    showSlitPreview = !showSlitPreview;
    updateSlitPreviewButton();
  });

  slitThicknessSlider = labeledSlider(
    secSave, 'スリット太さ', 2, 30, 8, 1, 'px',
    '白いスリットの太さ。円周方向の幅。'
  ).slider;

  slitLengthSlider = labeledSlider(
    secSave, 'スリット長さ', 8, 140, 42, 1, 'px',
    '白いスリットの長さ。外周方向に長くなり、外側の黒円の半径も大きくなる。'
  ).slider;

  smallHint(secSave, 'プレビューON時、大きな円盤が画面に収まるように表示だけ自動縮小する。保存画像は指定サイズで出力される。');
  updateSlitPreviewButton();

  // ---- リング設定 ----
  const secRings = section(ui, 'リング設定', {subtitle:'各リングをカードで編集（削除はカード右上の🗑）'});
  const headRow = createDiv().parent(secRings)
    .style('display','flex').style('gap','8px').style('flex-wrap','wrap');

  addRingBtn = pillButton('＋ リング追加').parent(headRow);
  addRingBtn.attribute('title','新しいリング（図形群）を追加します');

  ringContainer = createDiv().parent(secRings)
    .style('border','1px dashed #cfcfcf')
    .style('border-radius','10px')
    .style('padding','8px')
    .style('background','#fff')
    .style('margin-top','8px');

  addRingBtn.mousePressed(addRing);

  // 最初のリングを1つ用意
  addRing();
}

function addRing() {
  const card = createDiv().parent(ringContainer)
    .style('border','1px solid #ddd')
    .style('border-radius','12px')
    .style('padding','10px')
    .style('margin','8px 0')
    .style('background','#fcfcfc')
    .style('box-shadow','0 1px 0 rgba(0,0,0,0.02)');

  // ヘッダー行：タイトル＋削除ボタン
  const titleRow = createDiv().parent(card)
    .style('display','flex').style('justify-content','space-between').style('align-items','center');
  createSpan('リング').parent(titleRow).style('font-weight','700');
  const delBtn = pillButton('🗑 削除').parent(titleRow);
  delBtn.style('background','#fff8f8').style('border-color','#f2caca');

  // グリッド領域
  const grid = createDiv().parent(card)
    .style('display','grid')
    .style('grid-template-columns','1fr')
    .style('gap','6px');

  // コントロール群
  const size     = labeledSlider(grid, '図形の大きさ', 1, 180, 30, 1, 'px', '図形の外接円半径');
  const cgR      = labeledSlider(grid, '図形の位置', 0, circleR - 10, 100, 1, 'px', '円盤中心からリング中心までの距離');
  const count    = labeledSlider(grid, '個数', 3, 60, 12, 1, '個', 'リング上に配置する図形の数');
  const sides    = labeledSlider(grid, '角数', 3, 12, 3, 1, '角', '多角形の頂点数');
  const phaseRow = labeledSlider(grid, '放射位置', -180, 180, 0, 1, '°', 'リング全体の配置角（中心からの方位）');

  // 表示モード & 反転ボタン
  const modeRow = createDiv().parent(card)
    .style('display','flex').style('gap','8px').style('margin-top','6px');
  const modeBtn   = pillButton('⇄ 表示：線').parent(modeRow);
  const flipBtn   = pillButton('↻ 図形を反転: OFF').parent(modeRow);

  const ring = {
    sizeSlider: size.slider,
    cgRSlider:  cgR.slider,
    countSlider: count.slider,
    sidesSlider: sides.slider,
    radialPhaseSlider: phaseRow.slider,
    modeBtn,
    showDots: false,
    // 反転フラグ（+1:通常, -1:反転）
    orientSign: 1,
    container: card
  };

  modeBtn.mousePressed(() => {
    ring.showDots = !ring.showDots;
    modeBtn.html(ring.showDots ? '⇄ 表示：点' : '⇄ 表示：線');
  });

  flipBtn.mousePressed(() => {
    ring.orientSign *= -1;
    flipBtn.html(ring.orientSign === -1 ? '↻ 図形を反転: ON' : '↻ 図形を反転: OFF');
  });

  delBtn.mousePressed(() => {
    // 自分を配列から取り除き、カードを消す
    const idx = rings.indexOf(ring);
    if (idx >= 0) rings.splice(idx, 1);
    ring.container.remove();
  });

  rings.push(ring);
}

function draw() {
  drawScene(this, 1, { withSlits: showSlitPreview, fitToCanvas: showSlitPreview });

  // --- コマ撮り更新 ---
  const fps = fpsSlider.value();
  frameMs = 1000 / fps;
  if (playing) {
    const now = millis();
    if (now - lastTick >= frameMs) {
      stepOnce(+1, getStepRad());
      lastTick = now;
    }
  }

  // --- 左上のオーバーレイ ---
  fill(0); noStroke();
  const divs = getStepDivisions();
  const stepDegDisp = 360 / divs;
  const deg = degrees((curAng % TWO_PI + TWO_PI) % TWO_PI);
  text(
    `angle:${nf(deg,1,1)}°  step:${nf(stepDegDisp,1,2)}°  div:${divs}  fps:${fps}  dotR:${dotSizeSlider.value()}  lineW:${lineWeightSlider.value()}  rings:${rings.length}`,
    10, 20
  );
}

// --- 共通描画（pg: p5.Graphics or this, scale: 倍率）---
function drawScene(pg, scale, opts = {}){
  const withSlits = !!opts.withSlits;
  const fitToCanvas = !!opts.fitToCanvas;

  pg.push();
  pg.background(255);
  pg.translate(pg.width/2, pg.height/2);

  // プレビュー時だけ、外側のスリット円盤がキャンバスに収まるよう表示を縮小する
  if (withSlits && fitToCanvas) {
    const cfgForFit = getSlitConfig(scale);
    const availableR = Math.min(pg.width, pg.height) / 2 - 12 * scale;
    const fit = Math.min(1, availableR / cfgForFit.outerR);
    pg.scale(fit);
  }

  // 全体回転
  pg.push();
  pg.rotate(curAng);

  if (withSlits) {
    const slitCfg = getSlitConfig(scale);
    drawSlitDisk(pg, slitCfg);
  }

  // 黒円
  pg.noStroke(); pg.fill(0);
  pg.circle(0, 0, circleR * 2 * scale);

  // 中心の赤い十字（長さ3px、線幅2px）
  if (showCenterCross) {
    pg.stroke(255, 0, 0); pg.strokeWeight(2 * scale);
    pg.line(-3*scale, 0, 3*scale, 0);
    pg.line(0, -3*scale, 0, 3*scale);
  }

  // 各リング
  const dotR = dotSizeSlider.value()   * scale;
  const lw   = lineWeightSlider.value()* scale;

  for (let ring of rings) {
    const desiredR = ring.sizeSlider.value()   * scale;
    const cgR      = ring.cgRSlider.value()    * scale;
    const n        = ring.countSlider.value();
    const k        = ring.sidesSlider.value();
    const phase    = radians(ring.radialPhaseSlider.value() || 0);

    const maxR = (circleR * scale) - (margin * scale);
    const Rv = allowOverflow ? desiredR : Math.max(0, Math.min(desiredR, maxR - cgR));

    // ここが回転ずれ。orientSign（+1 / -1）で反転
    const totalTwistDeg = 360 / k;              // 1図形の「向き」周期
    const stepTwistDeg  = (totalTwistDeg / n) * ring.orientSign;
    const radialStep    = TWO_PI / n;

    for (let i = 0; i < n; i++) {
      const radial   = i * radialStep;
      const twistDeg = (i * stepTwistDeg);      // 符号そのまま使う
      const twist    = radians(twistDeg);

      const centerAng = radial + phase;
      const cx = cgR * cos(centerAng);
      const cy = cgR * sin(centerAng);

      pg.push();
      pg.translate(cx, cy);
      pg.rotate(radial + twist);

      if (ring.showDots) {
        pg.noStroke(); pg.fill(255);
        for (let j = 0; j < k; j++) {
          const ang = j * TWO_PI / k;
          const vx = Rv * cos(ang);
          const vy = Rv * sin(ang);
          pg.circle(vx, vy, dotR * 2);
        }
      } else {
        pg.noFill(); pg.stroke(255);
        pg.strokeWeight(lw);
        pg.beginShape();
        for (let j = 0; j < k; j++) {
          const ang = j * TWO_PI / k;
          pg.vertex(Rv * cos(ang), Rv * sin(ang));
        }
        pg.endShape(CLOSE);
      }
      pg.pop();
    }
  }

  pg.pop(); // 全体回転
  pg.pop();
}

// --- ステップ分割の取得（1〜360の整数にクランプ）
function getStepDivisions() {
  let v = parseInt(stepDivInput.value(), 10);
  if (!Number.isFinite(v)) v = stepDivSlider.value();
  v = Math.max(1, Math.min(360, Math.floor(v)));
  if (v !== stepDivSlider.value()) stepDivSlider.value(v);
  if (String(v) !== stepDivInput.value()) stepDivInput.value(String(v));
  return v;
}

// --- 1コマの回転角（rad）を計算（分割数に依存）
function getStepRad() {
  const divs = getStepDivisions();
  return radians(360 / divs);
}

function updateSlitPreviewButton() {
  if (!slitPreviewBtn) return;
  slitPreviewBtn.html(showSlitPreview ? '👁 スリットプレビュー: ON' : '👁 スリットプレビュー: OFF');
  slitPreviewBtn.style('background', showSlitPreview ? '#eef6ff' : '#fff');
  slitPreviewBtn.style('border-color', showSlitPreview ? '#8bbcff' : '#cfcfcf');
}

function askTargetPx(defaultPx = 2000, label = '保存する直径(px) を入力（例: 2000）') {
  const promptStr = prompt(label, String(Math.round(defaultPx)));
  if (promptStr === null) return null; // キャンセル
  let targetD = parseInt(promptStr, 10);
  if (!Number.isFinite(targetD) || targetD <= 0) targetD = defaultPx;
  return Math.max(1, Math.round(targetD));
}

function getSlitConfig(scale = 1) {
  const slitThickness = ((slitThicknessSlider ? slitThicknessSlider.value() : 8) || 8) * scale;
  const slitLength    = ((slitLengthSlider ? slitLengthSlider.value() : 42) || 42) * scale;

  // 中央円とスリットの間の余白
  const slitGap = 10 * scale;

  // 外周黒円の余白。スリットが外縁に接しすぎないようにする
  const rimPadding = Math.max(8 * scale, slitThickness * 0.8);

  const slitInnerR = circleR * scale + slitGap;
  const slitOuterR = slitInnerR + slitLength;
  const outerR = slitOuterR + rimPadding;
  const slitCount = getStepDivisions();

  return {
    slitThickness,
    slitLength,
    slitGap,
    rimPadding,
    slitInnerR,
    slitOuterR,
    outerR,
    slitCount
  };
}

function drawSlitDisk(pg, cfg) {
  // 外側の大きな黒円
  pg.noStroke();
  pg.fill(0);
  pg.circle(0, 0, cfg.outerR * 2);

  // 白いスリット。各スリットは放射方向に長い長方形
  const slitCenterR = (cfg.slitInnerR + cfg.slitOuterR) / 2;
  pg.push();
  pg.rectMode(CENTER);
  pg.noStroke();
  pg.fill(255);

  for (let i = 0; i < cfg.slitCount; i++) {
    const ang = i * TWO_PI / cfg.slitCount;
    const cx = slitCenterR * cos(ang);
    const cy = slitCenterR * sin(ang);

    pg.push();
    pg.translate(cx, cy);
    pg.rotate(ang);
    pg.rect(0, 0, cfg.slitLength, cfg.slitThickness, cfg.slitThickness * 0.35);
    pg.pop();
  }

  pg.pop();
}

// --- 高解像度で中央の黒円部分を保存（背景=白） ---
function saveCircleImageHiRes() {
  const targetD = askTargetPx(2000, '円保存：保存する直径(px) を入力（例: 2000）');
  if (targetD === null) return;

  const scale = targetD / (circleR * 2);
  const W = Math.round(canvasSize * scale);
  const H = Math.round(canvasSize * scale);
  const pg = createGraphics(W, H);
  pg.pixelDensity(1);
  drawScene(pg, scale);

  const x = Math.round(W/2 - targetD/2);
  const y = Math.round(H/2 - targetD/2);
  const img = pg.get(x, y, targetD, targetD);

  save(img, `circle_${targetD}px.png`);
  pg.remove();
}

// --- 高解像度でスリット付き円盤全体を保存（背景=白） ---
function saveSlitCircleImageHiRes() {
  const baseCfg = getSlitConfig(1);
  const baseD = baseCfg.outerR * 2;
  const defaultD = Math.max(2000, Math.round(baseD * 4));

  const targetD = askTargetPx(defaultD, 'スリット＋円保存：外側の円盤全体の直径(px) を入力（例: 2400）');
  if (targetD === null) return;

  const scale = targetD / baseD;
  const padding = 24 * scale;
  const W = Math.round(targetD + padding * 2);
  const H = Math.round(targetD + padding * 2);

  const pg = createGraphics(W, H);
  pg.pixelDensity(1);
  drawScene(pg, scale, { withSlits: true, fitToCanvas: false });

  const x = Math.round(W/2 - targetD/2);
  const y = Math.round(H/2 - targetD/2);
  const img = pg.get(x, y, targetD, targetD);

  save(img, `slit_circle_${targetD}px.png`);
  pg.remove();
}

// --- 1コマ進める（または戻す） ---
function stepOnce(dir, stepRad) {
  curAng += stepRad * dir;
  curAng = ((curAng % TWO_PI) + TWO_PI) % TWO_PI; // 0..360°に正規化
}
