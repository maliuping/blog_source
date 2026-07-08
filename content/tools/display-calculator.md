+++
date = '2026-07-01T13:40:00+08:00'
draft = false
title = 'Display Calculator'
description = '实时计算显示链路带宽、Lane Rate、时钟、UI 与 DSI video timing。'
ShowToc = false
+++

{{< rawhtml >}}
<section class="display-calculator">
  <div class="dc-hero">
    <div>
      <p class="dc-eyebrow">Display Link Budget</p>
      <h2>Display Calculator</h2>
      <p class="dc-intro">
        输入像素时钟、BPP、Lane 数、PPI Width 和水平时序参数，页面会实时计算总吞吐、每 Lane 速率、相关时钟参数以及 DSI video timing。
      </p>
    </div>
    <div class="dc-hero-note">
      <span>Live</span>
      <strong>Auto Refresh</strong>
      <p>所有结果会在输入变化时立即更新。</p>
    </div>
  </div>

  <div class="dc-grid">
    <section class="dc-panel dc-inputs">
      <div class="dc-panel-head">
        <h3>Inputs</h3>
        <p>左侧输入参数</p>
      </div>

      <label class="dc-field">
        <span>Pixel Clock (MHz)</span>
        <input id="pixel-clock" type="number" min="0" step="0.01" value="148.5" />
      </label>

      <label class="dc-field">
        <span>Bits Per Pixel</span>
        <input id="bits-per-pixel" type="number" min="1" step="1" value="24" />
      </label>

      <label class="dc-field">
        <span>Lane Number</span>
        <input id="lane-number" type="number" min="1" step="1" value="4" />
      </label>

      <label class="dc-field">
        <span>PPI Width</span>
        <input id="ppi-width" type="number" min="1" step="1" value="8" />
      </label>

      <label class="dc-field">
        <span>Margin (%)</span>
        <input id="margin-percent" type="number" min="0" step="0.1" value="20" />
      </label>

      <div class="dc-input-group">
        <h4>Horizontal Timing</h4>
        <p>单位均为 pixel</p>
      </div>

      <label class="dc-field">
        <span>H Active</span>
        <input id="hactive" type="number" min="1" step="1" value="1920" />
      </label>

      <label class="dc-field">
        <span>H Sync Width</span>
        <input id="hsync-width" type="number" min="0" step="1" value="44" />
      </label>

      <label class="dc-field">
        <span>H Back Porch</span>
        <input id="hback-porch" type="number" min="0" step="1" value="148" />
      </label>

      <label class="dc-field">
        <span>H Front Porch</span>
        <input id="hfront-porch" type="number" min="0" step="1" value="88" />
      </label>
    </section>

    <section class="dc-panel dc-results">
      <div class="dc-panel-head">
        <h3>Outputs</h3>
        <p>右侧实时结果</p>
      </div>

      <div class="dc-result-grid">
        <article class="dc-result-card">
          <span>Total Throughput</span>
          <strong id="total-throughput">0 Mbps</strong>
        </article>

        <article class="dc-result-card">
          <span>Per Lane Rate</span>
          <strong id="per-lane-rate">0 Mbps</strong>
        </article>

        <article class="dc-result-card dc-result-card-accent">
          <span>Recommended Lane Rate</span>
          <strong id="recommended-lane-rate">0 Mbps</strong>
        </article>

        <article class="dc-result-card">
          <span>DDR Clock</span>
          <strong id="ddr-clock">0 MHz</strong>
        </article>

        <article class="dc-result-card">
          <span>Line Byte Clock</span>
          <strong id="byte-clock">0 MHz</strong>
        </article>

        <article class="dc-result-card">
          <span>UI (ns)</span>
          <strong id="ui-value">0 ns</strong>
        </article>

        <article class="dc-result-card">
          <span>HSA Time</span>
          <strong id="hsa-time">0 cycles</strong>
        </article>

        <article class="dc-result-card">
          <span>HBP Time</span>
          <strong id="hbp-time">0 cycles</strong>
        </article>

        <article class="dc-result-card">
          <span>HACT Time</span>
          <strong id="hact-time">0 cycles</strong>
        </article>

        <article class="dc-result-card dc-result-card-accent">
          <span>HLINE Time</span>
          <strong id="hline-time">0 cycles</strong>
        </article>
      </div>
    </section>
  </div>

  <section class="dc-panel dc-formulas">
    <div class="dc-panel-head">
      <h3>Generated Formulas</h3>
      <p>下面自动生成公式与代入结果</p>
    </div>

    <div class="dc-formula-list">
      <div class="dc-formula-item">
        <code id="formula-total">TotalThroughput = PixelClock × BPP</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-lane-rate">LaneRate = PixelClock × BPP / Lane</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-recommended">RecommendedLaneRate = LaneRate × (1 + Margin / 100)</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-byte-clock">LineByteClock = LaneRate / PPIWidth</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-ddr-clock">DDRClock = LaneRate / 2</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-ui">UI = 1000 / LaneRate</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-hsa-time">HsaTime = floor(HSyncWidth × LineByteClock / PixelClock)</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-hbp-time">HbpTime = floor(HBackPorch × LineByteClock / PixelClock)</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-hact-time">HactTime = floor(HActive × LineByteClock / PixelClock)</code>
      </div>
      <div class="dc-formula-item">
        <code id="formula-hline-time">HlineTime = ceil((HSyncWidth + HBackPorch + HActive + HFrontPorch) × LineByteClock / PixelClock)</code>
      </div>
    </div>

    <p class="dc-footnote">
      注：<code>DDR Clock</code>、<code>Line Byte Clock</code> 和 <code>UI</code> 按公式使用 <code>Per Lane Rate</code> 计算；
      <code>HSA/HBP/HACT/HLINE Time</code> 以 <code>Line Byte Clock</code> 换算为 byte clock cycles；
      DSI D-PHY 下 <code>PPI Width</code> 通常取 <code>8</code>；
      <code>Recommended Lane Rate</code> 额外用于留出 Margin。
    </p>
  </section>
</section>

<style>
  .display-calculator {
    --dc-border: color-mix(in srgb, var(--border) 70%, transparent);
    --dc-soft: color-mix(in srgb, var(--theme) 72%, #f6efe4);
    --dc-accent: #c46528;
    --dc-surface: color-mix(in srgb, var(--entry) 88%, var(--theme));
    --dc-surface-strong: color-mix(in srgb, var(--entry) 96%, var(--theme));
    --dc-surface-accent: color-mix(in srgb, var(--entry) 78%, rgba(196, 101, 40, 0.16));
    --dc-text-soft: var(--secondary);
    color: var(--primary);
    display: grid;
    gap: 1.5rem;
    margin-top: 1rem;
  }

  .display-calculator h2,
  .display-calculator h3,
  .display-calculator p {
    margin: 0;
    color: var(--primary);
  }

  .display-calculator strong,
  .display-calculator span,
  .display-calculator label,
  .display-calculator code,
  .display-calculator input {
    color: inherit;
  }

  .dc-hero,
  .dc-panel {
    border: 1px solid var(--dc-border);
    background:
      radial-gradient(circle at top right, rgba(196, 101, 40, 0.12), transparent 32%),
      linear-gradient(135deg, var(--dc-soft), var(--dc-surface));
    border-radius: 24px;
    box-shadow: 0 18px 48px rgba(15, 23, 42, 0.08);
  }

  .dc-hero {
    display: grid;
    grid-template-columns: minmax(0, 1fr) 220px;
    gap: 1rem;
    padding: 1.75rem;
    align-items: end;
  }

  .dc-eyebrow {
    display: inline-flex;
    padding: 0.35rem 0.7rem;
    margin-bottom: 0.85rem;
    border-radius: 999px;
    background: rgba(196, 101, 40, 0.12);
    color: var(--dc-accent);
    font-size: 0.76rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
  }

  .dc-hero h2 {
    font-size: clamp(2rem, 3vw, 2.75rem);
    line-height: 1;
    margin-bottom: 0.75rem;
  }

  .dc-intro {
    max-width: 58ch;
    color: var(--dc-text-soft);
  }

  .dc-hero-note {
    padding: 1rem;
    border-radius: 20px;
    background: var(--dc-surface-strong);
    border: 1px solid rgba(196, 101, 40, 0.16);
    backdrop-filter: blur(10px);
  }

  .dc-hero-note span {
    display: inline-block;
    margin-bottom: 0.4rem;
    color: var(--dc-accent);
    font-size: 0.78rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
  }

  .dc-hero-note strong {
    display: block;
    font-size: 1.15rem;
    margin-bottom: 0.45rem;
  }

  .dc-hero-note p {
    color: var(--dc-text-soft);
    font-size: 0.95rem;
  }

  .dc-grid {
    display: grid;
    grid-template-columns: minmax(280px, 360px) minmax(0, 1fr);
    gap: 1.5rem;
  }

  .dc-panel {
    padding: 1.4rem;
  }

  .dc-panel-head {
    margin-bottom: 1.1rem;
  }

  .dc-panel-head h3 {
    font-size: 1.15rem;
    margin-bottom: 0.3rem;
  }

  .dc-panel-head p {
    color: var(--dc-text-soft);
    font-size: 0.95rem;
  }

  .dc-input-group {
    margin-top: 0.35rem;
    padding-top: 0.35rem;
    border-top: 1px solid var(--dc-border);
  }

  .dc-input-group h4,
  .dc-input-group p {
    margin: 0;
  }

  .dc-input-group h4 {
    font-size: 0.98rem;
    margin-bottom: 0.25rem;
  }

  .dc-input-group p {
    color: var(--dc-text-soft);
    font-size: 0.88rem;
  }

  .dc-inputs {
    display: grid;
    gap: 0.95rem;
    align-content: start;
  }

  .dc-field {
    display: grid;
    gap: 0.45rem;
  }

  .dc-field span {
    font-size: 0.95rem;
    font-weight: 600;
  }

  .dc-field input {
    width: 100%;
    border: 1px solid var(--dc-border);
    border-radius: 14px;
    padding: 0.9rem 1rem;
    background: var(--dc-surface-strong);
    color: var(--primary);
    caret-color: var(--dc-accent);
    font-size: 1rem;
    transition: border-color 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;
  }

  .dc-field input::placeholder {
    color: var(--secondary);
  }

  .dc-field input:focus {
    outline: none;
    border-color: rgba(196, 101, 40, 0.55);
    box-shadow: 0 0 0 4px rgba(196, 101, 40, 0.12);
    transform: translateY(-1px);
  }

  .dc-result-grid {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 1rem;
  }

  .dc-result-card {
    padding: 1rem;
    border-radius: 18px;
    border: 1px solid var(--dc-border);
    background: var(--dc-surface-strong);
    min-height: 120px;
    display: grid;
    align-content: space-between;
    gap: 0.8rem;
  }

  .dc-result-card span {
    color: var(--dc-text-soft);
    font-size: 0.92rem;
  }

  .dc-result-card strong {
    font-size: clamp(1.2rem, 2vw, 1.9rem);
    line-height: 1.1;
    word-break: break-word;
  }

  .dc-result-card-accent {
    background: linear-gradient(145deg, rgba(196, 101, 40, 0.14), var(--dc-surface-accent));
    border-color: rgba(196, 101, 40, 0.28);
  }

  .dc-formula-list {
    display: grid;
    gap: 0.8rem;
  }

  .dc-formula-item {
    padding: 0.95rem 1rem;
    border-radius: 16px;
    border: 1px solid var(--dc-border);
    background: var(--dc-surface-strong);
    overflow-x: auto;
  }

  .dc-formula-item code {
    display: block;
    font-size: 0.96rem;
    line-height: 1.6;
    color: var(--primary);
    white-space: nowrap;
  }

  .dc-footnote {
    margin-top: 1rem;
    color: var(--dc-text-soft);
    font-size: 0.92rem;
  }

  @media (max-width: 900px) {
    .dc-hero,
    .dc-grid {
      grid-template-columns: 1fr;
    }

    .dc-result-grid {
      grid-template-columns: 1fr;
    }
  }
</style>

<script>
  (() => {
    const inputs = {
      pixelClock: document.getElementById("pixel-clock"),
      bitsPerPixel: document.getElementById("bits-per-pixel"),
      laneNumber: document.getElementById("lane-number"),
      ppiWidth: document.getElementById("ppi-width"),
      marginPercent: document.getElementById("margin-percent"),
      hactive: document.getElementById("hactive"),
      hsyncWidth: document.getElementById("hsync-width"),
      hbackPorch: document.getElementById("hback-porch"),
      hfrontPorch: document.getElementById("hfront-porch"),
    };

    const outputs = {
      totalThroughput: document.getElementById("total-throughput"),
      perLaneRate: document.getElementById("per-lane-rate"),
      recommendedLaneRate: document.getElementById("recommended-lane-rate"),
      ddrClock: document.getElementById("ddr-clock"),
      byteClock: document.getElementById("byte-clock"),
      uiValue: document.getElementById("ui-value"),
      hsaTime: document.getElementById("hsa-time"),
      hbpTime: document.getElementById("hbp-time"),
      hactTime: document.getElementById("hact-time"),
      hlineTime: document.getElementById("hline-time"),
      formulaTotal: document.getElementById("formula-total"),
      formulaLaneRate: document.getElementById("formula-lane-rate"),
      formulaRecommended: document.getElementById("formula-recommended"),
      formulaLineByteClock: document.getElementById("formula-byte-clock"),
      formulaDdrClock: document.getElementById("formula-ddr-clock"),
      formulaUi: document.getElementById("formula-ui"),
      formulaHsaTime: document.getElementById("formula-hsa-time"),
      formulaHbpTime: document.getElementById("formula-hbp-time"),
      formulaHactTime: document.getElementById("formula-hact-time"),
      formulaHlineTime: document.getElementById("formula-hline-time"),
    };

    const readValue = (element) => {
      const value = Number.parseFloat(element.value);
      return Number.isFinite(value) ? value : 0;
    };

    const safeDiv = (numerator, denominator) => (denominator > 0 ? numerator / denominator : 0);

    const formatNumber = (value, digits = 2) =>
      new Intl.NumberFormat("en-US", {
        minimumFractionDigits: digits,
        maximumFractionDigits: digits,
      }).format(value);

    const formatRate = (value) => `${formatNumber(value)} Mbps`;
    const formatClock = (value) => `${formatNumber(value)} MHz`;
    const formatUi = (value) => `${formatNumber(value, 4)} ns`;
    const formatCycles = (value) => `${formatNumber(value, 0)} cycles`;

    const update = () => {
      const pixelClock = readValue(inputs.pixelClock);
      const bitsPerPixel = readValue(inputs.bitsPerPixel);
      const laneNumber = readValue(inputs.laneNumber);
      const ppiWidth = readValue(inputs.ppiWidth);
      const marginPercent = readValue(inputs.marginPercent);
      const hactive = readValue(inputs.hactive);
      const hsyncWidth = readValue(inputs.hsyncWidth);
      const hbackPorch = readValue(inputs.hbackPorch);
      const hfrontPorch = readValue(inputs.hfrontPorch);

      const totalThroughput = pixelClock * bitsPerPixel;
      const laneRate = safeDiv(totalThroughput, laneNumber);
      const recommendedLaneRate = laneRate * (1 + marginPercent / 100);
      const byteClock = safeDiv(laneRate, ppiWidth);
      const ddrClock = laneRate / 2;
      const ui = laneRate > 0 ? 1000 / laneRate : 0;
      const htotal = hsyncWidth + hbackPorch + hactive + hfrontPorch;
      const hsaTime = Math.trunc(safeDiv(hsyncWidth * byteClock, pixelClock));
      const hbpTime = Math.trunc(safeDiv(hbackPorch * byteClock, pixelClock));
      const hactTime = Math.trunc(safeDiv(hactive * byteClock, pixelClock));
      const hlineTime = Math.ceil(safeDiv(htotal * byteClock, pixelClock));

      outputs.totalThroughput.textContent = formatRate(totalThroughput);
      outputs.perLaneRate.textContent = formatRate(laneRate);
      outputs.recommendedLaneRate.textContent = formatRate(recommendedLaneRate);
      outputs.byteClock.textContent = formatClock(byteClock);
      outputs.ddrClock.textContent = formatClock(ddrClock);
      outputs.uiValue.textContent = formatUi(ui);
      outputs.hsaTime.textContent = formatCycles(hsaTime);
      outputs.hbpTime.textContent = formatCycles(hbpTime);
      outputs.hactTime.textContent = formatCycles(hactTime);
      outputs.hlineTime.textContent = formatCycles(hlineTime);

      outputs.formulaTotal.textContent =
        `TotalThroughput = ${formatNumber(pixelClock)} × ${formatNumber(bitsPerPixel)} = ${formatNumber(totalThroughput)} Mbps`;
      outputs.formulaLaneRate.textContent =
        `LaneRate = ${formatNumber(pixelClock)} × ${formatNumber(bitsPerPixel)} / ${formatNumber(laneNumber)} = ${formatNumber(laneRate)} Mbps`;
      outputs.formulaRecommended.textContent =
        `RecommendedLaneRate = ${formatNumber(laneRate)} × (1 + ${formatNumber(marginPercent)} / 100) = ${formatNumber(recommendedLaneRate)} Mbps`;
      outputs.formulaLineByteClock.textContent =
        `LineByteClock = ${formatNumber(laneRate)} / ${formatNumber(ppiWidth)} = ${formatNumber(byteClock)} MHz`;
      outputs.formulaDdrClock.textContent =
        `DDRClock = ${formatNumber(laneRate)} / 2 = ${formatNumber(ddrClock)} MHz`;
      outputs.formulaUi.textContent =
        `UI = 1000 / ${formatNumber(laneRate)} = ${formatNumber(ui, 4)} ns`;
      outputs.formulaHsaTime.textContent =
        `HsaTime = floor(${formatNumber(hsyncWidth, 0)} × ${formatNumber(byteClock)} / ${formatNumber(pixelClock)}) = ${formatNumber(hsaTime, 0)} cycles`;
      outputs.formulaHbpTime.textContent =
        `HbpTime = floor(${formatNumber(hbackPorch, 0)} × ${formatNumber(byteClock)} / ${formatNumber(pixelClock)}) = ${formatNumber(hbpTime, 0)} cycles`;
      outputs.formulaHactTime.textContent =
        `HactTime = floor(${formatNumber(hactive, 0)} × ${formatNumber(byteClock)} / ${formatNumber(pixelClock)}) = ${formatNumber(hactTime, 0)} cycles`;
      outputs.formulaHlineTime.textContent =
        `HlineTime = ceil((${formatNumber(hsyncWidth, 0)} + ${formatNumber(hbackPorch, 0)} + ${formatNumber(hactive, 0)} + ${formatNumber(hfrontPorch, 0)}) × ${formatNumber(byteClock)} / ${formatNumber(pixelClock)}) = ${formatNumber(hlineTime, 0)} cycles`;
    };

    Object.values(inputs).forEach((input) => {
      input.addEventListener("input", update);
    });

    update();
  })();
</script>
{{< /rawhtml >}}
