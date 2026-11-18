---
name: pt-tek-beat-sync-animations
description: "Sincroniza animações do PT.TEK com o beat detector através do EventBus, um único AnimatorAgent e um único RAF. Respeita pausa, prefers-reduced-motion e o limite de três flashes por segundo."
metadata:
  version: '2.0'
  author: 'PT.TEK Underground Records'
  supersedes: '1.0 (2026-09-23)'
  language: 'Portuguese (European)'
  depends_on: ['EventBus.js', 'agents/BeatDetectorAgent.js', 'agents/AnimatorAgent.js', 'agents/AppAgent.js', 'AGENTS.md']
---

# PT.TEK | Sincronização beat ↔ animações (v2)

## Porque esta versão existe

A v1 usava `beatDetector.js`, `player.js` e `app.js`, que já não existem como tal. Também duplicava listeners, trocava `bgImage.src` a cada beat, usava `setTimeout` sem cancelamento e verificava uma propriedade inexistente (`audioElement.playing`). Esta versão corrige tudo isso.

> **Antes de implementar:** confirma os nomes exatos dos ficheiros e métodos em `agents/` e `EventBus.js`. Os nomes abaixo são os da arquitetura descrita no `AGENTS.md` e no `MELHORIAS_SITE.md`. Se diferirem, adapta os nomes, não a regra.

## 1. Regras inegociáveis

Estas regras vêm do `AGENTS.md` e não são opcionais.

1. **Um único caminho de eventos.** O beat sai do `BeatDetectorAgent` pelo `EventBus`. Nenhum outro módulo regista listeners de beat diretamente.
2. **Um único RAF.** Só o `AnimatorAgent` chama `requestAnimationFrame`. Nenhum `setTimeout` ou `setInterval` controla animações.
3. **Pausa para tudo.** Ao pausar, o RAF é cancelado e todas as transformações voltam ao repouso.
4. **Sem troca de imagem a cada beat.** O fundo biomecânico mantém-se. Os efeitos vivem em camadas transparentes sobre colunas e arestas.
5. **Limite de flashes.** Nunca mais de 3 variações de luminância por segundo (WCAG 2.3.1), mesmo que o BPM seja superior.
6. **`prefers-reduced-motion`.** Sem rotações nem escalas bruscas. Só variação suave de brilho.
7. **Sem simulação de FFT do SoundCloud.** A Widget API não expõe FFT do iframe.

## 2. Fluxo

```
AudioAgent (play/pause)
   │  EventBus: player:play / player:pause
   ▼
BeatDetectorAgent  ──►  EventBus: beat { index, intensity, bpm, confidence, time }
   │                              │
   │                              ▼
   │                      AnimatorAgent (um RAF)
   │                         ├─ pulso do logo (transform)
   │                         ├─ pulso das camadas de coluna e aresta (opacity/transform)
   │                         └─ indicador de BPM (textContent, sem animação)
   ▼
pausa → AnimatorAgent.stopAll() → cancelAnimationFrame + repouso
```

## 3. Contrato de eventos

| Evento | Emissor | Payload | Consumidor |
|---|---|---|---|
| `player:play` | AudioAgent | `{ trackId, time }` | BeatDetectorAgent.start(), AnimatorAgent.resume() |
| `player:pause` | AudioAgent | `{ time }` | BeatDetectorAgent.stop(), AnimatorAgent.stopAll() |
| `beat` | BeatDetectorAgent | `{ index, intensity (0–1), bpm, confidence (0–1), time }` | AnimatorAgent.onBeat() |
| `bpm:update` | BeatDetectorAgent | `{ bpm, confidence }` | UI do player (texto) |

O `time` vem de `audioContext.currentTime`. Não uses `performance.now()` para sincronizar com o áudio.

## 4. BeatDetectorAgent: limite de flashes e cooldown

O limite de flashes vive aqui, na origem. Assim nenhum consumidor consegue violá-lo.

```javascript
// agents/BeatDetectorAgent.js (excerto)
const MAX_FLASHES_PER_SECOND = 3;
const MIN_BEAT_COOLDOWN_MS = 100;

class BeatDetectorAgent {
  constructor(bus, audioAgent) {
    this.bus = bus;
    this.audio = audioAgent;
    this.index = 0;
    this.lastBeatMs = -Infinity;
    this.flashTimes = [];           // timestamps dos últimos flashes (ms)
  }

  // chamado a cada frame de análise FFT (não é RAF de animação)
  onAnalysis(bassEnergy, threshold, bpm, confidence, nowMs) {
    const canBeat = nowMs - this.lastBeatMs >= MIN_BEAT_COOLDOWN_MS;
    if (!canBeat || bassEnergy < threshold) return;

    this.lastBeatMs = nowMs;
    const intensity = Math.min(1, (bassEnergy - threshold) / threshold);

    // Flash = mudança de luminância. Limitar a 3 por segundo.
    const flashAllowed = this.consumeFlashSlot(nowMs);

    this.bus.emit('beat', {
      index: this.index++,
      intensity,
      bpm,
      confidence,
      time: this.audio.currentTime,
      flashAllowed
    });
  }

  consumeFlashSlot(nowMs) {
    this.flashTimes = this.flashTimes.filter(t => nowMs - t < 1000);
    if (this.flashTimes.length >= MAX_FLASHES_PER_SECOND) return false;
    this.flashTimes.push(nowMs);
    return true;
  }

  start() { /* liga a análise FFT */ }
  stop()  { this.flashTimes = []; /* desliga a análise FFT */ }
}
```

Nota: o `flashAllowed` é passado ao `AnimatorAgent`, que só altera luminância quando é `true`. A escala e a posição continuam a pulsar porque não são flashes.

## 5. AnimatorAgent: um único RAF, sem timers

```javascript
// agents/AnimatorAgent.js (excerto)
const REST = 'translate3d(0,0,0) scale(1)';

class AnimatorAgent {
  constructor(bus, targets) {
    // targets: { logo, columns, edges, bpmReadout } — elementos DOM
    this.bus = bus;
    this.targets = targets;
    this.reduced = window.matchMedia('(prefers-reduced-motion: reduce)');
    this.pulses = [];      // { start, duration, amplitude, kind }
    this.rafId = null;

    this.bus.on('beat', (b) => this.onBeat(b));
    this.bus.on('bpm:update', (b) => this.setBpm(b.bpm));
    this.bus.on('player:pause', () => this.stopAll());
    this.bus.on('player:play', () => this.resume());
  }

  // duração do pulso: no máximo 60% do intervalo entre beats, e nunca acima de 250 ms
  pulseDuration(bpm) {
    const beatMs = 60000 / Math.max(bpm, 60);
    return Math.min(0.6 * beatMs, 250);
  }

  onBeat({ intensity, bpm, flashAllowed, time }) {
    const duration = this.pulseDuration(bpm);
    this.pulses.push({
      start: time,                     // tempo do áudio, não do relógio do browser
      duration,
      amplitude: intensity,
      flash: flashAllowed && !this.reduced.matches
    });
    // mantém só os pulsos ativos
    this.pulses = this.pulses.filter(p => time - p.start < p.duration / 1000 + 0.05);
    this.ensureLoop();
  }

  ensureLoop() {
    if (this.rafId === null) this.rafId = requestAnimationFrame(() => this.frame());
  }

  frame() {
    const now = this.audioNow();
    let logoScale = 1, glow = 0;

    for (const p of this.pulses) {
      const t = Math.min(1, (now - p.start) * 1000 / p.duration);
      const e = 1 - Math.pow(1 - t, 3);          // ease-out cúbico
      const env = 1 - e;                          // pico no início, decai
      if (this.reduced.matches) {
        glow = Math.max(glow, env * p.amplitude * 0.6);   // só brilho suave
      } else {
        logoScale = Math.max(logoScale, 1 + env * p.amplitude * 0.25);
        if (p.flash) glow = Math.max(glow, env * p.amplitude);
      }
    }

    this.paint(logoScale, glow);

    this.pulses = this.pulses.filter(p => now - p.start < p.duration / 1000);
    if (this.pulses.length) {
      this.rafId = requestAnimationFrame(() => this.frame());
    } else {
      this.rafId = null;
      this.paint(1, 0);                           // repouso
    }
  }

  paint(scale, glow) {
    const { logo, columns, edges } = this.targets;
    if (logo) logo.style.transform = scale === 1 ? REST : `scale(${scale.toFixed(3)})`;
    // efeitos só em camadas transparentes, nunca no fundo
    if (columns) columns.style.opacity = String(0.25 + glow * 0.4);
    if (edges)   edges.style.opacity   = String(0.25 + glow * 0.4);
  }

  stopAll() {
    if (this.rafId !== null) cancelAnimationFrame(this.rafId);
    this.rafId = null;
    this.pulses = [];
    this.paint(1, 0);
  }

  resume() { /* o próximo beat reinicia o loop */ }

  setBpm(bpm) {
    if (this.targets.bpmReadout) this.targets.bpmReadout.textContent = Math.round(bpm);
  }

  // tempo do áudio, fornecido pelo AudioAgent
  audioNow() { return this.bus.audioTime(); }
}
```

Porque não há `setTimeout`: o fim de cada pulso é calculado a partir de `audioContext.currentTime` em cada frame. Assim, pausar corta tudo de imediato e não há timers órfãos.

## 6. Ligação no AppAgent

```javascript
// agents/AppAgent.js (excerto)
const animator = new AnimatorAgent(bus, {
  logo:       document.querySelector('[data-fx="logo"]'),
  columns:    document.querySelector('[data-fx="columns"]'),
  edges:      document.querySelector('[data-fx="edges"]'),
  bpmReadout: document.querySelector('[data-fx="bpm"]')
});
```

Regras de marcação no HTML:
- `data-fx="logo"`, `data-fx="columns"`, `data-fx="edges"` e `data-fx="bpm"` identificam os alvos.
- Sem `data-fx`, o `AnimatorAgent` ignora o alvo. O código não quebra.
- Não há classes de animação CSS com `animation:` infinita, porque não podem ser pausadas pelo JavaScript.

## 7. CSS mínimo de suporte

```css
/* camadas de efeito: transparentes, por cima do fundo, sem trocar imagens */
[data-fx="columns"],
[data-fx="edges"] {
  pointer-events: none;
  will-change: opacity;
  opacity: 0.25;
}

[data-fx="logo"] {
  will-change: transform;
  transform-origin: center;
}

/* o CSS não anima nada por si só: quem anima é o AnimatorAgent */
@media (prefers-reduced-motion: reduce) {
  [data-fx="logo"] { transform: none !important; }
}
```

## 8. Testes obrigatórios

Ficheiro: `tests/beat-sync.test.cjs`, executado com `node --test tests/*.test.cjs`.

| Teste | O que prova |
|---|---|
| Um único listener de `beat` por `AnimatorAgent` | Sem duplicação |
| `pulseDuration(180)` ≤ 250 ms e ≤ 60% de 333 ms | Sem sobreposição a 180 BPM |
| Com 10 beats em 1 s, no máximo 3 com `flashAllowed: true` | Limite de três flashes |
| Com `prefers-reduced-motion`, `logoScale` permanece 1 | Sem escala com reduced motion |
| Após `player:pause`, `rafId === null` e `paint(1, 0)` foi chamado | Pausa para tudo |
| Nenhum uso de `setTimeout` em `AnimatorAgent.js` e `BeatDetectorAgent.js` | Regra de timers |
| Nenhuma escrita em `src` de imagem de fundo durante `beat` | Fundo biomecânico intacto |

Os testes de navegador (desktop e mobile) só se declaram executados se tiverem sido executados.

## 9. Checklist de integração

- [ ] Nomes de ficheiros e métodos confirmados em `agents/` e `EventBus.js`
- [ ] `BeatDetectorAgent` emite `beat` com `flashAllowed` e `time`
- [ ] `AnimatorAgent` é o único consumidor de `beat` para animação
- [ ] `player:pause` chama `stopAll()` e cancela o RAF
- [ ] Atributos `data-fx` presentes no HTML
- [ ] Testes do ponto 8 verdes
- [ ] Validação visual com `prefers-reduced-motion` ativo e inativo, desktop e mobile

## 10. Resolução de problemas

| Sintoma | Causa provável | Ação |
|---|---|---|
| Efeitos dobrados | Listener registado em dois sítios | Procurar `on('beat'` e manter só um |
| Efeitos continuam após pausa | RAF não cancelado | Confirmar `stopAll()` no evento `player:pause` |
| Logo não se mexe | Falta `data-fx="logo"` | Acrescentar o atributo ao HTML |
| Brilho pisca demasiado | `flashAllowed` ignorado | Confirmar que o `AnimatorAgent` lê `p.flash` |
| Sem efeito com reduced motion | Esperado | Só o brilho suave se mantém |

## 11. Changelog

- **2.0:** arquitetura alinhada com `agents/` e `EventBus`; um único RAF; pausa total; limite de três flashes no `BeatDetectorAgent`; `prefers-reduced-motion`; fundo sem troca de imagem; duração de pulso limitada a 60% do intervalo entre beats; timers removidos; referência a `audioContext.currentTime`.
- **1.0 (2026-09-23):** versão original, substituída.
