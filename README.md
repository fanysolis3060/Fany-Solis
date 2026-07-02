https://es.wikipedia.org/wiki/Angel_Reese
<img width="1020" height="1372" alt="Angel_Reese_September_2024_(cropped)" src="https://github.com/user-attachments/assets/c7cb0e71-0b46-4795-b66c-e490d19089e0" />

import React, { useState, useEffect, useRef } from 'react';

// Definimos constantes estéticas para personificar al bot
const PERSONAS = {
  general: {
    id: 'general',
    name: 'Asistente General',
    icon: '🤖',
    desc: 'Un asistente amigable, inteligente y equilibrado para cualquier tarea.',
    systemInstruction: 'Actúa como un asistente virtual amigable, profesional y extremadamente inteligente. Brinda respuestas claras, bien estructuradas y detalladas en español.'
  },
  coder: {
    id: 'coder',
    name: 'Programador Experto',
    icon: '💻',
    desc: 'Especialista en desarrollo de software, depuración y arquitectura.',
    systemInstruction: 'Eres un ingeniero de software senior y experto en arquitectura de sistemas. Cuando escribas código, hazlo limpio, modular, bien documentado y optimizado. Explica tus decisiones técnicas brevemente.'
  },
  creative: {
    id: 'creative',
    name: 'Escritor Creativo',
    icon: '✍️',
    desc: 'Especializado en redacción persuasiva, historias y lluvia de ideas.',
    systemInstruction: 'Actúa como un novelista galardonado y redactor publicitario brillante. Usa metáforas sugerentes, estilo expresivo y vocabulario dinámico en tus respuestas.'
  },
  tutor: {
    id: 'tutor',
    name: 'Tutor de Idiomas',
    icon: '🗣️',
    desc: 'Te ayuda a practicar idiomas y corregir tu gramática.',
    systemInstruction: 'Eres un tutor de idiomas nativo y muy paciente. Corrige con amabilidad cualquier error de redacción que cometa el usuario, explícale el porqué de la corrección de forma sencilla y plantéale preguntas de seguimiento para continuar la conversación.'
  }
};

// Utilidades para convertir datos de audio en bruto (PCM 16-bit) a contenedores WAV legibles por el navegador
function writeString(view, offset, string) {
  for (let i = 0; i < string.length; i++) {
    view.setUint8(offset + i, string.charCodeAt(i));
  }
}

function base64ToArrayBuffer(base64) {
  const binaryString = window.atob(base64);
  const len = binaryString.length;
  const bytes = new Uint8Array(len);
  for (let i = 0; i < len; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes.buffer;
}

function pcmToWav(pcm16Array, sampleRate) {
  const buffer = new ArrayBuffer(44 + pcm16Array.length * 2);
  const view = new DataView(buffer);

  // Identificador "RIFF"
  writeString(view, 0, 'RIFF');
  // Longitud del archivo
  view.setUint32(4, 36 + pcm16Array.length * 2, true);
  // Tipo "WAVE"
  writeString(view, 8, 'WAVE');
  // Formato del bloque de información
  writeString(view, 12, 'fmt ');
  // Tamaño del bloque de formato (16 bytes)
  view.setUint32(16, 16, true);
  // Formato de muestra: PCM (1)
  view.setUint16(20, 1, true);
  // Número de canales: Mono (1)
  view.setUint16(22, 1, true);
  // Tasa de muestreo (ej. 24000)
  view.setUint32(24, sampleRate, true);
  // Tasa de bytes (sampleRate * blockAlign)
  view.setUint32(28, sampleRate * 2, true);
  // Alineación de bloque (canales * bits/8)
  view.setUint16(32, 2, true);
  // Bits por muestra: 16-bit
  view.setUint16(34, 16, true);
  // Identificador del bloque de datos
  writeString(view, 36, 'data');
  // Longitud del bloque de datos
  view.setUint32(40, pcm16Array.length * 2, true);

  // Escribimos los datos PCM
  for (let i = 0; i < pcm16Array.length; i++) {
    view.setInt16(44 + i * 2, pcm16Array[i], true);
  }

  return new Blob([buffer], { type: 'audio/wav' });
}

export default function App() {
  const [messages, setMessages] = useState([
    {
      id: 'welcome',
      role: 'assistant',
      content: '¡Hola! Bienvenido a tu nuevo espacio inteligente. Soy tu chatbot potenciado con Gemini. ¿En qué te puedo asistir hoy?',
      timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }),
    }
  ]);
  const [input, setInput] = useState('');
  const [isGenerating, setIsGenerating] = useState(false);
  const [useSearch, setUseSearch] = useState(false);
  const [selectedPersona, setSelectedPersona] = useState('general');
  const [temperature, setTemperature] = useState(0.7);
  const [selectedImage, setSelectedImage] = useState(null); // { base64, mimeType, name }
  const [playingAudioId, setPlayingAudioId] = useState(null);
  const [audioElements, setAudioElements] = useState({}); // Stores instances of Audio objects
  const [sidebarOpen, setSidebarOpen] = useState(false);
  const [toastMessage, setToastMessage] = useState(null);

  // Referencias UI
  const messagesEndRef = useRef(null);
  const fileInputRef = useRef(null);

  useEffect(() => {
    scrollToBottom();
  }, [messages]);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  const showToast = (msg) => {
    setToastMessage(msg);
    setTimeout(() => {
      setToastMessage(null);
    }, 4000);
  };

  const handleImageUpload = (e) => {
    const file = e.target.files[0];
    if (!file) return;

    if (!file.type.startsWith('image/')) {
      showToast('Por favor, selecciona solo archivos de imagen (PNG, JPG, WEBP).');
      return;
    }

    const reader = new FileReader();
    reader.onload = () => {
      const base64Data = reader.result.split(',')[1];
      setSelectedImage({
        base64: base64Data,
        mimeType: file.type,
        name: file.name,
        preview: reader.result
      });
    };
    reader.readAsDataURL(file);
  };

  const removeImage = () => {
    setSelectedImage(null);
    if (fileInputRef.current) {
      fileInputRef.current.value = '';
    }
  };

  const handlePlayTTS = async (messageId, text) => {
    // Si ya se está reproduciendo este audio, pausar
    if (playingAudioId === messageId) {
      audioElements[messageId]?.pause();
      setPlayingAudioId(null);
      return;
    }

    // Si ya existe un elemento cargado previamente, reproducirlo de nuevo
    if (audioElements[messageId]) {
      audioElements[messageId].play();
      setPlayingAudioId(messageId);
      return;
    }

    // Detener cualquier otra reproducción activa
    if (playingAudioId) {
      audioElements[playingAudioId]?.pause();
    }

    try {
      setPlayingAudioId(`${messageId}-loading`); // Estado temporal de cargando
      
      const payload = {
        contents: [{
          parts: [{ text: `Lee en voz alta de manera clara y natural: ${text.slice(0, 300)}` }] // Límite corto por seguridad y fluidez
        }],
        generationConfig: {
          responseModalities: ["AUDIO"],
          speechConfig: {
            voiceConfig: {
              prebuiltVoiceConfig: { voiceName: "Aoede" } // Aoede es dulce y clara en español
            }
          }
        },
        model: "gemini-2.5-flash-preview-tts"
      };

      const apiKey = ""; // API Key implícita a nivel de Canvas
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}`;

      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      if (!response.ok) throw new Error('Error al generar la voz de Gemini.');

      const result = await response.json();
      const part = result?.candidates?.[0]?.content?.parts?.[0];
      const audioData = part?.inlineData?.data;
      const mimeType = part?.inlineData?.mimeType;

      if (audioData && mimeType && mimeType.startsWith("audio/")) {
        // Obtenemos la tasa de muestreo del MIME Type (por ejemplo: audio/L16;rate=24000)
        const matchRate = mimeType.match(/rate=(\d+)/);
        const sampleRate = matchRate ? parseInt(matchRate[1], 10) : 24000;

        const pcmBuffer = base64ToArrayBuffer(audioData);
        const pcm16 = new Int16Array(pcmBuffer);
        const wavBlob = pcmToWav(pcm16, sampleRate);
        const audioUrl = URL.createObjectURL(wavBlob);

        const newAudio = new Audio(audioUrl);
        newAudio.play();
        
        newAudio.onended = () => {
          setPlayingAudioId(null);
        };

        setAudioElements(prev => ({ ...prev, [messageId]: newAudio }));
        setPlayingAudioId(messageId);
      } else {
        throw new Error('No se recibió audio válido de la API.');
      }
    } catch (err) {
      console.error(err);
      showToast('Error al procesar el lector de voz de Gemini.');
      setPlayingAudioId(null);
    }
  };

  const handleSendMessage = async (e) => {
    e.preventDefault();
    if (!input.trim() && !selectedImage) return;

    const userMessageContent = input;
    const currentImg = selectedImage;
    
    // Crear el mensaje del usuario para la interfaz
    const userMessageId = 'msg-' + Date.now();
    const newUserMessage = {
      id: userMessageId,
      role: 'user',
      content: userMessageContent,
      image: currentImg ? currentImg.preview : null,
      timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
    };

    setMessages(prev => [...prev, newUserMessage]);
    setInput('');
    removeImage();
    setIsGenerating(true);

    try {
      // Preparamos el historial para enviarlo a Gemini
      // Formateamos las respuestas previas omitiendo la de bienvenida básica si tiene id de sistema
      const chatHistory = messages
        .filter(msg => msg.id !== 'welcome')
        .map(msg => ({
          role: msg.role === 'assistant' ? 'model' : 'user',
          parts: [{ text: msg.content }]
        }));

      // Añadimos la nueva consulta del usuario al payload
      const currentParts = [];
      if (currentImg) {
        currentParts.push({
          inlineData: {
            mimeType: currentImg.mimeType,
            data: currentImg.base64
          }
        });
      }
      currentParts.push({ text: userMessageContent || "Describe esta imagen de forma detallada." });

      chatHistory.push({
        role: 'user',
        parts: currentParts
      });

      // Construcción del Payload
      const payload = {
        contents: chatHistory,
        generationConfig: {
          temperature: temperature,
        },
        systemInstruction: {
          parts: [{ text: PERSONAS[selectedPersona].systemInstruction }]
        }
      };

      // Si está habilitada la búsqueda de Google, la añadimos como herramienta
      if (useSearch) {
        payload.tools = [{ "google_search": {} }];
      }

      const apiKey = ""; // API Key provista por el entorno Canvas
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key=${apiKey}`;

      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      if (!response.ok) {
        throw new Error('Problema de respuesta con el servidor de Gemini.');
      }

      const data = await response.json();
      const candidate = data.candidates?.[0];
      const modelText = candidate?.content?.parts?.[0]?.text || "No logré procesar tu solicitud.";

      // Extraemos fuentes/citas si se activó la búsqueda en Google
      let sources = [];
      const groundingMetadata = candidate?.groundingMetadata;
      if (groundingMetadata && groundingMetadata.groundingAttributions) {
        sources = groundingMetadata.groundingAttributions
          .map(attr => ({
            uri: attr.web?.uri,
            title: attr.web?.title
          }))
          .filter(src => src.uri && src.title);
      }

      // Añadimos la respuesta del bot a la conversación
      setMessages(prev => [
        ...prev,
        {
          id: 'bot-' + Date.now(),
          role: 'assistant',
          content: modelText,
          sources: sources,
          timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
        }
      ]);

    } catch (err) {
      console.error(err);
      setMessages(prev => [
        ...prev,
        {
          id: 'err-' + Date.now(),
          role: 'assistant',
          content: 'Lo siento, ocurrió un error al conectarme con la inteligencia artificial de Gemini. Verifica que tu conexión a internet sea estable e inténtalo nuevamente.',
          timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
        }
      ]);
    } finally {
      setIsGenerating(false);
    }
  };

  const clearChat = () => {
    setMessages([
      {
        id: 'welcome',
        role: 'assistant',
        content: `He reiniciado el chat con el perfil de **${PERSONAS[selectedPersona].name}**. ¿En qué más te puedo asistir hoy?`,
        timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
      }
    ]);
    setSelectedImage(null);
    setAudioElements({});
    setPlayingAudioId(null);
    showToast('Historial de chat borrado con éxito.');
  };

  return (
    <div className="flex h-screen bg-slate-950 text-slate-100 font-sans overflow-hidden">
      
      {/* Toast Notification */}
      {toastMessage && (
        <div className="fixed top-4 right-4 z-50 bg-slate-800 border border-indigo-500 text-indigo-200 px-4 py-3 rounded-xl shadow-2xl transition-all flex items-center gap-2">
          <svg className="w-5 h-5 text-indigo-400 animate-pulse" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
          </svg>
          <span className="text-sm font-medium">{toastMessage}</span>
        </div>
      )}

      {}
      {/* Sidebar - Settings & Customization */}
      <aside className={`fixed inset-y-0 left-0 z-40 w-80 bg-slate-900 border-r border-slate-800 transform transition-transform duration-300 ease-in-out md:relative md:translate-x-0 flex flex-col justify-between ${sidebarOpen ? 'translate-x-0' : '-translate-x-full'}`}>
        <div className="p-6 overflow-y-auto space-y-6 flex-1">
          
          {/* Logo */}
          <div className="flex items-center gap-3 border-b border-slate-800 pb-5">
            <div className="w-10 h-10 bg-indigo-600 rounded-xl flex items-center justify-center font-bold text-white text-lg shadow-lg shadow-indigo-600/30">
              G2
            </div>
            <div>
              <h1 className="font-bold text-lg tracking-wide bg-gradient-to-r from-indigo-400 to-purple-400 bg-clip-text text-transparent">Gemini Playground</h1>
              <p className="text-xs text-slate-400">Asistente Conversacional</p>
            </div>
          </div>

          {/* New Chat Button */}
          <button 
            onClick={clearChat}
            className="w-full flex items-center justify-center gap-2 bg-gradient-to-r from-indigo-600 to-purple-600 hover:from-indigo-500 hover:to-purple-500 text-white font-medium py-3 px-4 rounded-xl shadow-md shadow-indigo-500/10 transition-all duration-200 hover:scale-[1.01]"
          >
            <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 4v16m8-8H4" /></svg>
            Nuevo Chat
          </button>

          {/* Personas Selector */}
          <div className="space-y-3">
            <h3 className="text-xs font-semibold text-slate-400 uppercase tracking-wider">Perfil del Asistente</h3>
            <div className="space-y-2">
              {Object.values(PERSONAS).map((p) => (
                <button
                  key={p.id}
                  onClick={() => {
                    setSelectedPersona(p.id);
                    showToast(`Rol cambiado a: ${p.name}`);
                  }}
                  className={`w-full text-left p-3 rounded-xl border transition-all flex items-start gap-3 ${selectedPersona === p.id ? 'bg-indigo-600/10 border-indigo-500 text-white' : 'bg-slate-800/50 border-transparent text-slate-300 hover:bg-slate-800'}`}
                >
                  <span className="text-2xl mt-0.5">{p.icon}</span>
                  <div>
                    <div className="font-semibold text-sm">{p.name}</div>
                    <div className="text-xs text-slate-400 line-clamp-1 mt-0.5">{p.desc}</div>
                  </div>
                </button>
              ))}
            </div>
          </div>

          {/* Configuration Settings */}
          <div className="space-y-4 pt-4 border-t border-slate-800">
            <h3 className="text-xs font-semibold text-slate-400 uppercase tracking-wider">Ajustes del Modelo</h3>
            
            {/* Google Search Grounding Switch */}
            <div className="flex items-center justify-between p-3 bg-slate-800/30 rounded-xl border border-slate-800">
              <div className="flex flex-col">
                <span className="text-xs font-semibold text-slate-200 flex items-center gap-1.5">
                  <svg className="w-3.5 h-3.5 text-blue-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2.5} d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
                  </svg>
                  Búsqueda Google
                </span>
                <span className="text-[10px] text-slate-400">Grounding en tiempo real</span>
              </div>
              <label className="relative inline-flex items-center cursor-pointer">
                <input 
                  type="checkbox" 
                  checked={useSearch} 
                  onChange={(e) => setUseSearch(e.target.checked)} 
                  className="sr-only peer" 
                />
                <div className="w-11 h-6 bg-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all peer-checked:bg-indigo-600"></div>
              </label>
            </div>

            {/* Temperature Slider */}
            <div className="space-y-2 p-3 bg-slate-800/30 rounded-xl border border-slate-800">
              <div className="flex justify-between text-xs font-semibold text-slate-200">
                <span>Creatividad (Temp.)</span>
                <span className="text-indigo-400">{temperature}</span>
              </div>
              <input 
                type="range" 
                min="0.1" 
                max="1.0" 
                step="0.1"
                value={temperature}
                onChange={(e) => setTemperature(parseFloat(e.target.value))}
                className="w-full h-1 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-indigo-500"
              />
              <div className="flex justify-between text-[9px] text-slate-500">
                <span>Más preciso</span>
                <span>Más creativo</span>
              </div>
            </div>

          </div>

        </div>

        {/* Brand / Footer */}
        <div className="p-4 border-t border-slate-800 bg-slate-950/40 text-center text-xs text-slate-500">
          Potenciado con Gemini 3 Flash
        </div>
      </aside>

      {/* Overlay to close sidebar on mobile */}
      {sidebarOpen && (
        <div 
          onClick={() => setSidebarOpen(false)} 
          className="fixed inset-0 z-30 bg-black/60 md:hidden"
        />
      )}

      {}
      {/* Main Chat Workspace */}
      <main className="flex-1 flex flex-col h-full bg-slate-950 relative">
        
        {/* Header */}
        <header className="bg-slate-900/80 backdrop-blur-md border-b border-slate-800 p-4 flex items-center justify-between sticky top-0 z-20">
          <div className="flex items-center gap-3">
            {/* Sidebar toggle for mobile */}
            <button 
              onClick={() => setSidebarOpen(true)}
              className="p-2 -ml-2 text-slate-400 hover:text-slate-100 md:hidden transition-colors"
            >
              <svg className="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 6h16M4 12h16M4 18h16" /></svg>
            </button>
            <div className="text-2xl">{PERSONAS[selectedPersona].icon}</div>
            <div>
              <h2 className="font-bold text-slate-100 flex items-center gap-2">
                {PERSONAS[selectedPersona].name}
                {useSearch && (
                  <span className="inline-flex items-center gap-1 bg-blue-500/10 text-blue-400 text-[10px] px-2 py-0.5 rounded-full font-semibold border border-blue-500/20 animate-pulse">
                    Web Grounding
                  </span>
                )}
              </h2>
              <p className="text-xs text-slate-400 hidden sm:block">{PERSONAS[selectedPersona].desc}</p>
            </div>
          </div>

          <div className="flex items-center gap-2">
            <button 
              onClick={clearChat}
              title="Borrar chat"
              className="p-2 text-slate-400 hover:text-red-400 hover:bg-slate-800/80 rounded-xl transition-all"
            >
              <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
            </button>
          </div>
        </header>

        {}
        {/* Messages Log Panel */}
        <div className="flex-1 overflow-y-auto p-4 md:p-6 space-y-6">
          {messages.map((message) => {
            const isUser = message.role === 'user';
            const isAudioLoading = playingAudioId === `${message.id}-loading`;
            const isAudioPlaying = playingAudioId === message.id;

            return (
              <div 
                key={message.id}
                className={`flex gap-4 ${isUser ? 'justify-end' : 'justify-start'} animate-fade-in`}
              >
                {/* Bot Avatar */}
                {!isUser && (
                  <div className="w-9 h-9 bg-slate-800 border border-slate-700 rounded-xl flex items-center justify-center text-lg shadow-sm flex-shrink-0">
                    {PERSONAS[selectedPersona].icon}
                  </div>
                )}

                {/* Message Body */}
                <div className={`max-w-[85%] md:max-w-[70%] space-y-2 flex flex-col ${isUser ? 'items-end' : 'items-start'}`}>
                  
                  {/* Uploaded User Image inside user message */}
                  {message.image && (
                    <div className="border border-slate-700 rounded-xl overflow-hidden shadow-md max-w-sm">
                      <img src={message.image} alt="Imagen adjunta" className="w-full h-auto object-cover max-h-60" />
                    </div>
                  )}

                  <div className={`p-4 rounded-2xl text-sm leading-relaxed whitespace-pre-wrap ${isUser ? 'bg-indigo-600 text-white shadow-lg shadow-indigo-600/10 rounded-tr-none' : 'bg-slate-900 border border-slate-800 text-slate-200 rounded-tl-none'}`}>
                    {message.content}

                    {/* Grounding Sources Panel */}
                    {!isUser && message.sources && message.sources.length > 0 && (
                      <div className="mt-4 pt-3 border-t border-slate-800 space-y-1.5">
                        <div className="text-[10px] font-bold uppercase tracking-wider text-slate-400 flex items-center gap-1">
                          <svg className="w-3 h-3 text-blue-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M13.828 10.172a4 4 0 00-5.656 0l-4 4a4 4 0 105.656 5.656l1.102-1.101m-.758-4.899a4 4 0 005.656 0l4-4a4 4 0 00-5.656-5.656l-1.1 1.1" />
                          </svg>
                          Fuentes Consultadas:
                        </div>
                        <div className="flex flex-wrap gap-2 pt-1">
                          {message.sources.slice(0, 3).map((src, idx) => (
                            <a 
                              key={idx} 
                              href={src.uri} 
                              target="_blank" 
                              rel="noopener noreferrer"
                              className="inline-flex items-center gap-1 bg-slate-800 hover:bg-slate-700 px-2 py-1 rounded text-[11px] text-blue-400 hover:text-blue-300 font-medium border border-slate-700/50 truncate max-w-[200px]"
                            >
                              {src.title || 'Web Page'}
                            </a>
                          ))}
                        </div>
                      </div>
                    )}
                  </div>

                  {/* Message Actions / Meta (Time & Sound Readout) */}
                  <div className="flex items-center gap-2 text-[10px] text-slate-500 px-1">
                    <span>{message.timestamp}</span>
                    
                    {!isUser && message.id !== 'welcome' && (
                      <button 
                        onClick={() => handlePlayTTS(message.id, message.content)}
                        className={`flex items-center gap-1 py-0.5 px-2 rounded-full border transition-all ${isAudioPlaying ? 'bg-indigo-500/10 border-indigo-500 text-indigo-400' : 'bg-slate-900/40 border-slate-800 hover:border-slate-700 text-slate-400 hover:text-slate-300'}`}
                        disabled={isAudioLoading}
                      >
                        {isAudioLoading ? (
                          <>
                            <div className="w-3 h-3 border-2 border-indigo-400 border-t-transparent rounded-full animate-spin"></div>
                            <span>Cargando Audio...</span>
                          </>
                        ) : isAudioPlaying ? (
                          <>
                            <svg className="w-3.5 h-3.5 text-indigo-400 animate-bounce" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M15.536 8.464a5 5 0 010 7.072m2.828-9.9a9 9 0 010 12.728M5.586 15H4a1 1 0 01-1-1v-4a1 1 0 011-1h1.586l4.707-4.707C10.923 3.663 12 4.109 12 5v14c0 .891-1.077 1.337-1.707.707L5.586 15z" />
                            </svg>
                            <span className="font-semibold">Detener Lectura</span>
                          </>
                        ) : (
                          <>
                            <svg className="w-3.5 h-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M15.536 8.464a5 5 0 010 7.072m2.828-9.9a9 9 0 010 12.728M5.586 15H4a1 1 0 01-1-1v-4a1 1 0 011-1h1.586l4.707-4.707C10.923 3.663 12 4.109 12 5v14c0 .891-1.077 1.337-1.707.707L5.586 15z" />
                            </svg>
                            <span>Escuchar Respuesta</span>
                          </>
                        )}
                      </button>
                    )}
                  </div>

                </div>

                {/* User Avatar */}
                {isUser && (
                  <div className="w-9 h-9 bg-indigo-600 text-white rounded-xl flex items-center justify-center font-bold text-sm shadow-sm flex-shrink-0">
                    U
                  </div>
                )}
              </div>
            );
          })}

          {/* Assistant Generation Loader */}
          {isGenerating && (
            <div className="flex gap-4 justify-start">
              <div className="w-9 h-9 bg-slate-800 border border-slate-700 rounded-xl flex items-center justify-center text-lg shadow-sm flex-shrink-0">
                {PERSONAS[selectedPersona].icon}
              </div>
              <div className="space-y-2 max-w-[80%]">
                <div className="bg-slate-900 border border-slate-800 p-4 rounded-2xl rounded-tl-none flex items-center gap-3">
                  <div className="flex space-x-1.5 items-center">
                    <span className="block w-2.5 h-2.5 bg-indigo-500 rounded-full animate-bounce" style={{ animationDelay: '0ms' }}></span>
                    <span className="block w-2.5 h-2.5 bg-indigo-500 rounded-full animate-bounce" style={{ animationDelay: '150ms' }}></span>
                    <span className="block w-2.5 h-2.5 bg-indigo-500 rounded-full animate-bounce" style={{ animationDelay: '300ms' }}></span>
                  </div>
                  <span className="text-sm text-slate-400 font-medium">
                    {useSearch ? 'Buscando fuentes y formulando respuesta...' : 'Gemini está pensando...'}
                  </span>
                </div>
              </div>
            </div>
          )}

          <div ref={messagesEndRef} />
        </div>

        {}
        {/* Bottom Input Workspace */}
        <div className="p-4 bg-slate-900/60 backdrop-blur-md border-t border-slate-800 space-y-3 sticky bottom-0 z-20">
          
          {/* Selected Image Thumbnail Preview */}
          {selectedImage && (
            <div className="flex items-center gap-3 bg-slate-800/80 p-2 rounded-xl border border-slate-700/50 max-w-sm ml-1">
              <img src={selectedImage.preview} alt="Visualización" className="w-12 h-12 object-cover rounded-lg border border-slate-700" />
              <div className="flex-1 min-w-0">
                <div className="text-xs font-semibold text-slate-200 truncate">{selectedImage.name}</div>
                <div className="text-[10px] text-slate-400">Imagen lista para analizar</div>
              </div>
              <button 
                onClick={removeImage}
                className="p-1.5 text-slate-400 hover:text-red-400 hover:bg-slate-700 rounded-lg transition-colors"
              >
                <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" /></svg>
              </button>
            </div>
          )}

          {/* Form */}
          <form onSubmit={handleSendMessage} className="flex gap-2 items-center">
            
            {/* Attachment Button */}
            <input 
              type="file" 
              accept="image/*" 
              ref={fileInputRef} 
              onChange={handleImageUpload} 
              className="hidden" 
            />
            <button
              type="button"
              onClick={() => fileInputRef.current?.click()}
              className="p-3 bg-slate-800 border border-slate-700/80 hover:bg-slate-700 hover:border-slate-600 text-slate-300 hover:text-white rounded-xl transition-all shadow-md flex-shrink-0"
              title="Adjuntar imagen"
            >
              <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
              </svg>
            </button>

            {/* Input Text Box */}
            <input
              type="text"
              value={input}
              onChange={(e) => setInput(e.target.value)}
              placeholder="Escribe un mensaje, haz una pregunta o adjunta una imagen..."
              className="flex-1 bg-slate-800 border border-slate-700/80 text-slate-100 placeholder-slate-400 text-sm px-4 py-3 rounded-xl focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition-all"
              disabled={isGenerating}
            />

            {/* Submit Send Button */}
            <button
              type="submit"
              disabled={isGenerating || (!input.trim() && !selectedImage)}
              className="p-3 bg-indigo-600 hover:bg-indigo-500 text-white rounded-xl shadow-md shadow-indigo-600/20 hover:scale-[1.03] transition-all disabled:opacity-50 disabled:hover:scale-100 flex-shrink-0"
            >
              <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8" />
              </svg>
            </button>
          </form>

        </div>

      </main>

    </div>
  );
}
