<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Docerola Studio</title>
  <!-- Tailwind CDN (rápido para demo). Para producción conviene compilar Tailwind o usar CSS propio -->
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="min-h-screen bg-slate-900 text-white">
  <div id="root"></div>

  <!-- Si tu JS exporta como componente React, necesitas React + ReactDOM -->
  <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>

  <!-- Libraries CDN (opcionales si tu minificado ya hace import dinámico) -->
  <script src="https://cdn.jsdelivr.net/npm/tone@14.7.77/build/Tone.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/midiwriter-js@2.1.0/build/index.umd.js"></script>

  <!-- Tu código minificado (asegúrate que DocerolaStudio.min.js exporte/monte la app en #root) -->
  <script src="DocerolaStudio.min.js"></script>

  <!-- Si tu bundle exporta un componente, usa este pequeño bootstrapper -->
  <script>
    // Si tu minificado deja un componente global, por ejemplo window.DocerolaStudioMount,
    // llama aquí a la función de montaje. Si tu minificado ya lo monta, puedes eliminar este bloque.
    if (window.DocerolaStudioMount && document.getElementById('root')) {
      window.DocerolaStudioMount(document.getElementById('root'));
    } else if (window.DocerolaStudio) {
      // ejemplo: si exportaste un componente React como window.DocerolaStudio
      const e = React.createElement;
      ReactDOM.createRoot(document.getElementById('root')).render(e(window.DocerolaStudio));
    }
  </script>
</body>
</html>
<script src="DocerolaStudio.min.js"></script>
importReact,{useState,useRef,useEffect}from"react";//DocerolaStudio-Single-fileReactcomponent(corregido)//-UsaTailwindCSSparaestilos(necesitasincluirTailwindentuproyecto)//-UsaCDNparaTone.jsymidiwriter-js(siprefieres,añade<script>enindex.htmlcomoindicamos)
</script><scriptsrc="https://cdn.jsdelivr.net/npm/midiwriter-js@2.1.0/build/index.umd.js"></script>-Mejorasfuturas:calculartrastesrealesapartirdenotaMIDI,mostrarVexFlowparapartituras,editordetablaturasvisual.*/
<script src="DocerolaStudio.min.js"></script>
https://cdn.jsdelivr.net/npm/midiwriter-js@2.1.0/build/index.umd.jsDocerolaStudio.min.js
