import React, { useState, useRef, useEffect } from "react";

// Docerola Studio - Single-file React component
// - Usa Tailwind CSS para estilos (necesitas incluir Tailwind en tu proyecto)
// - Usa CDN para Tone.js, VexFlow y midiwriter-js (ver comentarios en README dentro)
// - Características implementadas (cliente):
//   1) Reproducción de tablaturas de guitarra (especial: docerola - guitarra de 12 cuerdas)
//   2) Generador simple de música a partir de letras (algoritmo heurístico)
//   3) Análisis de voz (pitch detection) y efecto de imitación simple (resíntesis / pitch-shift)
//   4) Todas las herramientas son gratuitas y funcionan en el navegador (no requiere servidores)
//   5) Descargar resultados: MIDI, WAV síntesis, tablatura (.txt)
// Notas: Para clonación/imitación de voz avanzada (calidad humana) necesitarías modelos de servidor.

export default function DocerolaStudio() {
  const [lyrics, setLyrics] = useState("Escribe aquí la letra de tu canción...\n(Soporta varias líneas)");
  const [generated, setGenerated] = useState({ notes: [], tabs: "" });
  const [isPlaying, setIsPlaying] = useState(false);
  const [scale, setScale] = useState("G_major");
  const [bpm, setBpm] = useState(90);
  const audioCtxRef = useRef(null);
  const mediaRef = useRef({});
  const [voiceSample, setVoiceSample] = useState(null);
  const [pitchHz, setPitchHz] = useState(null);
  const [downloadUrl, setDownloadUrl] = useState(null);

  useEffect(() => {
    // lazy-load AudioContext
    audioCtxRef.current = new (window.AudioContext || window.webkitAudioContext)();
  }, []);

  // ====== Utilities ======
  function syllablesFromText(text) {
    // heurística simple: dividir por sílabas usando vocales agrupadas (muy básica)
    // para este demo, separamos por palabras y luego por vocal groups
    return text
      .replace(/\n/g, " ")
      .split(/\s+/)
      .filter(Boolean)
      .map((w) => {
        // split into vowel groups
        const parts = w.match(/[aeiouáéíóúü]+/gi);
        if (!parts) return [w];
        return parts;
      })
      .flat();
  }

  const SCALES = {
    G_major: ["G3", "A3", "B3", "C4", "D4", "E4", "F#4", "G4"],
    E_minor: ["E3", "F#3", "G3", "A3", "B3", "C4", "D4", "E4"],
    A_minor: ["A3", "B3", "C4", "D4", "E4", "F4", "G4", "A4"],
  };

  function mapSyllablesToNotes(syllables, scaleName) {
    const scaleNotes = SCALES[scaleName] || SCALES["G_major"];
    const notes = syllables.map((s, i) => {
      const idx = i % scaleNotes.length;
      // vary rhythm depending on syllable length
      const dur = s.length > 2 ? "4n" : "8n";
      return { note: scaleNotes[idx], dur };
    });
    return notes;
  }

  function generateFromLyrics() {
    const syl = syllablesFromText(lyrics);
    const notes = mapSyllablesToNotes(syl, scale);
    // generate simple tablature for docerola (12 cuerdas) in standard tuning pairs
    const tabs = makeSimpleTabsFromNotes(notes);
    setGenerated({ notes, tabs });
  }

  function makeSimpleTabsFromNotes(notes) {
    // Para demo: crea líneas de tablatura con la cuerda 1-6 (duplicadas para docerola)
    // Afinación (pares): (E2/E3)-(A2/A3)-(D3/D4)-(G3/G4)-(B3/B4)-(E4/E5) (conceptual)
    const strings = ["e'","B","G","D","A","E"]; // simplified labels
    const measures = [];
    notes.forEach((n, i) => {
      const fret = Math.abs(n.note.length) % 12; // placeholder
      const line = strings.map((s) => (i % 4 === 0 ? "-" + fret + "-" : "--"));
      measures.push(line.join(" "));
    });
    return ["Tablatura (demo) for Docerola (12 cuerdas):", ...measures].join("\n");
  }

  // ===== Playback using Tone.js (CDN expected) =====
  async function playGenerated() {
    if (!generated.notes || generated.notes.length === 0) return;
    setIsPlaying(true);
    // Dynamic import Tone to keep bundle small
    const Tone = (await import("https://cdn.jsdelivr.net/npm/tone@14.7.77/build/Tone.js")).default;
    await Tone.start();
    Tone.Transport.bpm.value = bpm;
    const synth = new Tone.Synth().toDestination();
    let time = 0;
    generated.notes.forEach((n) => {
      synth.triggerAttackRelease(n.note, n.dur, Tone.now() + time);
      time += n.dur === "4n" ? (60 / bpm) : (60 / bpm) / 2; // rough durations
    });
    // Stop after sequence
    setTimeout(() => setIsPlaying(false), time * 1000 + 200);
  }

  // ===== Voice analysis (pitch detection) =====
  function handleVoiceFile(e) {
    const file = e.target.files[0];
    if (!file) return;
    setVoiceSample(file);
    const reader = new FileReader();
    reader.onload = async (ev) => {
      const arrayBuffer = ev.target.result;
      const audioCtx = audioCtxRef.current;
      const audioBuffer = await audioCtx.decodeAudioData(arrayBuffer);
      const floatData = audioBuffer.getChannelData(0);
      const pitch = autoCorrelate(floatData, audioCtx.sampleRate);
      setPitchHz(pitch ? Math.round(pitch) : "—");
    };
    reader.readAsArrayBuffer(file);
  }

  // Simple autocorrelation pitch detection (from Chris Wilson examples)
  function autoCorrelate(buf, sampleRate) {
    // Implementation simplified for demo
    let SIZE = buf.length;
    let rms = 0;
    for (let i = 0; i < SIZE; i++) {
      const val = buf[i];
      rms += val * val;
    }
    rms = Math.sqrt(rms / SIZE);
    if (rms < 0.01) return null; // too quiet
    let r1 = 0, r2 = SIZE - 1, thres = 0.2;
    for (let i = 0; i < SIZE / 2; i++) if (Math.abs(buf[i]) < thres) { r1 = i; break; }
    for (let i = 1; i < SIZE / 2; i++) if (Math.abs(buf[SIZE - i]) < thres) { r2 = SIZE - i; break; }
    buf = buf.slice(r1, r2);
    SIZE = buf.length;
    const c = new Array(SIZE).fill(0);
    for (let i = 0; i < SIZE; i++) for (let j = 0; j < SIZE - i; j++) c[i] = c[i] + buf[j] * buf[j + i];
    let d = 0; while (c[d] > c[d + 1]) d++;
    let maxval = -1, maxpos = -1;
    for (let i = d; i < SIZE; i++) if (c[i] > maxval) { maxval = c[i]; maxpos = i; }
    let T0 = maxpos;
    const pitch = sampleRate / T0;
    if (pitch > 1200 || pitch < 50) return null;
    return pitch;
  }

  // ===== Simple voice 'imitation' effect (pitch-shift + formant-ish filter) =====
  async function imitateVoice() {
    if (!voiceSample) return alert('Sube una muestra de voz primero.');
    // For demo: apply pitch shifting using OfflineAudioContext and play back
    const reader = new FileReader();
    reader.onload = async (ev) => {
      const arrayBuffer = ev.target.result;
      const audioCtx = audioCtxRef.current;
      const buffer = await audioCtx.decodeAudioData(arrayBuffer);
      // Create an OfflineAudioContext to render pitch-shifted audio (time-stretch naive)
      const offline = new OfflineAudioContext(buffer.numberOfChannels, buffer.length, buffer.sampleRate);
      const src = offline.createBufferSource();
      src.buffer = buffer;
      // simple pitch shift via playbackRate (not high-quality)
      src.playbackRate.value = 1.05; // small shift to imitate a different voice
      const biquad = offline.createBiquadFilter();
      biquad.type = 'peaking';
      biquad.frequency.value = 1200;
      biquad.gain.value = 4;
      src.connect(biquad).connect(offline.destination);
      src.start(0);
      const rendered = await offline.startRendering();
      // create blob and set download
      const wav = audioBufferToWav(rendered);
      const blob = new Blob([new DataView(wav)], { type: 'audio/wav' });
      const url = URL.createObjectURL(blob);
      setDownloadUrl(url);
      const a = new Audio(url);
      a.play();
    };
    reader.readAsArrayBuffer(voiceSample);
  }

  // helper: convert AudioBuffer to WAV (simple interleaved 16-bit)
  function audioBufferToWav(buffer) {
    const numOfChan = buffer.numberOfChannels;
    const length = buffer.length * numOfChan * 2 + 44;
    const buffer2 = new ArrayBuffer(length);
    const view = new DataView(buffer2);
    function writeString(view, offset, string) {
      for (let i = 0; i < string.length; i++) view.setUint8(offset + i, string.charCodeAt(i));
    }
    let offset = 0;
    writeString(view, offset, 'RIFF'); offset += 4;
    view.setUint32(offset, 36 + buffer.length * numOfChan * 2, true); offset += 4;
    writeString(view, offset, 'WAVE'); offset += 4;
    writeString(view, offset, 'fmt '); offset += 4;
    view.setUint32(offset, 16, true); offset += 4;
    view.setUint16(offset, 1, true); offset += 2;
    view.setUint16(offset, numOfChan, true); offset += 2;
    view.setUint32(offset, buffer.sampleRate, true); offset += 4;
    view.setUint32(offset, buffer.sampleRate * numOfChan * 2, true); offset += 4;
    view.setUint16(offset, numOfChan * 2, true); offset += 2;
    view.setUint16(offset, 16, true); offset += 2;
    writeString(view, offset, 'data'); offset += 4;
    view.setUint32(offset, buffer.length * numOfChan * 2, true); offset += 4;
    // write interleaved data
    const channels = [];
    for (let i = 0; i < numOfChan; i++) channels.push(buffer.getChannelData(i));
    let pos = 0;
    while (pos < buffer.length) {
      for (let i = 0; i < numOfChan; i++) {
        let sample = Math.max(-1, Math.min(1, channels[i][pos]));
        view.setInt16(offset, sample < 0 ? sample * 0x8000 : sample * 0x7fff, true);
        offset += 2;
      }
      pos++;
    }
    return buffer2;
  }

  // ===== Download helpers =====
  function downloadTabs() {
    const blob = new Blob([generated.tabs || "Sin tablatura generada"], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'tablatura_docerola.txt'; a.click();
  }

  async function downloadMidi() {
    if (!generated.notes || generated.notes.length === 0) return alert('Genera la música primero.');
    // Use midiwriter-js via CDN dynamic import
    const MidiWriter = (await import('https://cdn.jsdelivr.net/npm/midiwriter-js@2.1.0/build/index.umd.js')).default;
    const track = new MidiWriter.Track();
    generated.notes.forEach((n) => {
      const ev = new MidiWriter.NoteEvent({ pitch: [n.note], duration: n.dur === '4n' ? '4' : '8' });
      track.addEvent(ev);
    });
    const write = new MidiWriter.Writer([track]);
    const blob = new Blob([write.buildFile()], { type: 'audio/midi' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a'); a.href = url; a.download = 'docerola_song.mid'; a.click();
  }

  // ===== Render UI =====
  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-900 to-slate-800 text-white p-6">
      <div className="max-w-5xl mx-auto bg-slate-900/60 backdrop-blur rounded-2xl p-6 shadow-lg">
        <header className="flex items-center justify-between mb-4">
          <h1 className="text-3xl font-extrabold">Docerola Studio 🎸 (12 cuerdas)</h1>
          <div className="text-sm opacity-80">Todas las herramientas en el navegador — gratis</div>
        </header>

        <section className="grid md:grid-cols-2 gap-4">
          <div className="p-4 bg-slate-800/40 rounded-lg">
            <h2 className="font-semibold">1) Generador a partir de letras</h2>
            <textarea className="w-full h-40 mt-2 p-2 bg-slate-900 rounded" value={lyrics} onChange={e=>setLyrics(e.target.value)} />
            <div className="flex gap-2 mt-3">
              <select value={scale} onChange={e=>setScale(e.target.value)} className="p-2 rounded bg-slate-700">
                <option value="G_major">G mayor (sensible a docerola)</option>
                <option value="E_minor">E menor</option>
                <option value="A_minor">A menor</option>
              </select>
              <input type="number" className="w-24 p-2 rounded bg-slate-700" value={bpm} onChange={e=>setBpm(Number(e.target.value))} />
              <button className="px-3 py-2 bg-emerald-500 rounded text-slate-900 font-semibold" onClick={generateFromLyrics}>Generar</button>
              <button className="px-3 py-2 bg-blue-500 rounded" onClick={playGenerated} disabled={isPlaying}>{isPlaying? 'Reproduciendo...':'Reproducir'}</button>
            </div>
            <div className="mt-3">
              <h3 className="text-sm opacity-80">Vista previa de notas (array)</h3>
              <pre className="bg-slate-900 p-2 rounded h-28 overflow-auto">{JSON.stringify(generated.notes, null, 2)}</pre>
            </div>
          </div>

          <div className="p-4 bg-slate-800/40 rounded-lg">
            <h2 className="font-semibold">2) Tablatura & Descarga</h2>
            <p className="text-sm opacity-80">Especializado en docerola (guitarra de 12 cuerdas — pares). Genera una tablatura demostrativa.</p>
            <pre className="bg-slate-900 p-3 rounded h-48 overflow-auto mt-3">{generated.tabs || 'Aún no hay tablatura. Genera la música primero.'}</pre>
            <div className="flex gap-2 mt-3">
              <button className="px-3 py-2 bg-indigo-500 rounded" onClick={downloadTabs}>Descargar Tablatura</button>
              <button className="px-3 py-2 bg-fuchsia-500 rounded" onClick={downloadMidi}>Descargar MIDI</button>
            </div>
          </div>

          <div className="p-4 bg-slate-800/40 rounded-lg md:col-span-2">
            <h2 className="font-semibold">3) Análisis de voz y 'imitación' (demo)</h2>
            <p className="text-sm opacity-80">Sube una muestra de voz para analizar el pitch. La imitación aquí es una resíntesis pitch-shift simple (todo local).</p>
            <div className="flex gap-2 mt-2 items-center">
              <input type="file" accept="audio/*" onChange={handleVoiceFile} />
              <div className="text-sm">Pitch detectado: <strong>{pitchHz || '—'}</strong> Hz</div>
              <button className="px-3 py-2 bg-amber-500 rounded" onClick={imitateVoice}>Imitar (demo)</button>
              {downloadUrl && <a className="px-3 py-2 bg-green-600 rounded" href={downloadUrl} download="imitacion.wav">Descargar WAV</a>}
            </div>
            <div className="mt-3 text-xs opacity-75">
              <strong>Advertencia:</strong> La imitación avanzada de voz (clonación) puede implicar riesgos éticos. Usa muestras propias y pide permiso antes de publicar voces de otras personas.
            </div>
          </div>

          <div className="p-4 bg-slate-800/40 rounded-lg md:col-span-2">
            <h2 className="font-semibold">Extras & Cómo mejorar</h2>
            <ul className="list-disc ml-5 mt-2 text-sm opacity-90">
              <li>Integrar Magenta.js o un modelo de IA en servidor para melodías más musicales.</li>
              <li>Usar servicios open-source de clonación de voz (con consentimiento) en backend para imitación de alta calidad.</li>
              <li>Agregar editor de tablaturas interactivo con visualización de 12 cuerdas y exportador de PDF.</li>
            </ul>
          </div>
        </section>

        <footer className="mt-6 text-sm opacity-80">Docerola Studio • Demo cliente — Requiere conexión a CDN para Tone.js y midiwriter-js. "Docerola" = guitarra de 12 cuerdas (docerola = guitarra de doce cuerdas). Buscamos que todo funcione gratuitamente en el navegador.</footer>
      </div>
    </div>
  );
}

/*
README / Integración rápida:
- Añade Tailwind CSS a tu proyecto (o adapta estilos).
- Para mejores resultados, añade estas etiquetas <script> en index.html para servir las libs (si no usas imports dinámicos):
  <script src="https://cdn.jsdelivr.net/npm/tone@14.7.77/build/Tone.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/midiwriter-js@2.1.0/build/index.umd.js"></script>
- Este componente usa imports dinámicos a CDN para no aumentar el bundle.
- Extensiones posibles: render VexFlow (notación musical) para partituras, editor de tablatura visual, export a PDF.
*/
