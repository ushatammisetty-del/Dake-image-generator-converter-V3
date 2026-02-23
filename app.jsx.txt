import React, { useState, useRef, useEffect } from 'react';

// --- CONFIGURATION ---
const apiKey = ""; 
const imageModelId = "imagen-4.0-generate-001";
const editModelId = "gemini-2.5-flash-image-preview";

export default function App() {
  const [prompt, setPrompt] = useState("");
  const [mode, setMode] = useState("GENERATE"); 
  const [logs, setLogs] = useState([]);
  const [isProcessing, setIsProcessing] = useState(false);
  const [inputMedia, setInputMedia] = useState(null); 
  const [outputMedia, setOutputMedia] = useState(null);
  const [isDrawing, setIsDrawing] = useState(false);
  
  const canvasRef = useRef(null);
  const logEndRef = useRef(null);

  useEffect(() => {
    logEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [logs]);

  const addLog = (msg, type = 'info') => {
    setLogs(prev => [...prev, { msg, type, time: new Date().toLocaleTimeString().split(' ')[0] }].slice(-50));
  };

  const handleFileUpload = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (event) => {
      setInputMedia(event.target.result);
      addLog(`File: ${file.name}`, "success");
      const img = new Image();
      img.onload = () => {
        const canvas = canvasRef.current;
        if (!canvas) return;
        const ctx = canvas.getContext('2d');
        canvas.width = img.width;
        canvas.height = img.height;
        ctx.drawImage(img, 0, 0);
      };
      img.src = event.target.result;
    };
    reader.readAsDataURL(file);
  };

  const draw = (e) => {
    if (!isDrawing || mode === "GENERATE" || !canvasRef.current) return;
    const canvas = canvasRef.current;
    const rect = canvas.getBoundingClientRect();
    const ctx = canvas.getContext('2d');
    
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const clientY = e.touches ? e.touches[0].clientY : e.clientY;
    
    const x = (clientX - rect.left) * (canvas.width / rect.width);
    const y = (clientY - rect.top) * (canvas.height / rect.height);

    ctx.lineWidth = 50 * (canvas.width / 1024);
    ctx.lineCap = 'round';
    ctx.strokeStyle = 'rgba(6, 182, 212, 0.5)'; 
    ctx.lineTo(x, y);
    ctx.stroke();
    ctx.beginPath();
    ctx.moveTo(x, y);
  };

  const handleDownload = () => {
    if (!outputMedia) return;
    const link = document.createElement('a');
    link.href = outputMedia.content;
    link.download = `DAKE_V3_${Date.now()}.png`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  const handleExecute = async () => {
    if (mode !== 'GENERATE' && !inputMedia) return addLog("Matrix Required", "error");
    setIsProcessing(true);
    addLog(`Running ${mode}...`, "warn");

    try {
      let url, body;
      if (mode === "GENERATE") {
        url = `https://generativelanguage.googleapis.com/v1beta/models/${imageModelId}:predict?key=${apiKey}`;
        body = { instances: [{ prompt }], parameters: { sampleCount: 1 } };
      } else {
        url = `https://generativelanguage.googleapis.com/v1beta/models/${editModelId}:generateContent?key=${apiKey}`;
        const b64 = canvasRef.current.toDataURL('image/png').split(',')[1];
        body = {
          contents: [{ parts: [{ text: `${mode}: ${prompt}` }, { inlineData: { mimeType: "image/png", data: b64 } }] }],
          generationConfig: { responseModalities: ['TEXT', 'IMAGE'] }
        };
      }

      const res = await fetch(url, { method: 'POST', body: JSON.stringify(body) });
      const data = await res.json();
      const b64Data = mode === "GENERATE" ? data.predictions[0].bytesBase64Encoded : data.candidates[0].content.parts.find(p => p.inlineData).inlineData.data;
      
      setOutputMedia({ content: `data:image/png;base64,${b64Data}` });
      addLog("Success", "success");
    } catch (err) {
      addLog("API Error", "error");
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <div className="min-h-screen bg-[#050505] text-zinc-100 p-4 font-sans select-none">
      <div className="max-w-5xl mx-auto">
        <header className="flex flex-col items-center mb-8 gap-4">
          <h1 className="text-3xl font-black italic tracking-tighter">DAKE <span className="text-cyan-500">V3</span></h1>
          <div className="flex bg-zinc-900 p-1 rounded-2xl border border-zinc-800">
            {['GENERATE', 'ENHANCE', 'REMOVE'].map(m => (
              <button key={m} onClick={() => setMode(m)} className={`px-5 py-2 text-[11px] font-bold rounded-xl transition-all ${mode === m ? 'bg-white text-black' : 'text-zinc-500'}`}>{m}</button>
            ))}
          </div>
        </header>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
          <div className="space-y-4">
            <input type="file" onChange={handleFileUpload} className="hidden" id="f-up" accept="image/*" />
            <label htmlFor="f-up" className="block w-full border-2 border-dashed border-zinc-800 rounded-3xl p-10 text-center text-xs text-zinc-500 cursor-pointer active:scale-95 transition-transform uppercase font-black">Load Matrix</label>
            <textarea value={prompt} onChange={e => setPrompt(e.target.value)} className="w-full h-24 bg-zinc-900 rounded-2xl p-4 text-sm border border-zinc-800 focus:border-cyan-500/50 outline-none" placeholder="Directives..." />
            <button onClick={handleExecute} disabled={isProcessing} className="w-full py-5 bg-cyan-500 text-black rounded-3xl font-black text-sm tracking-widest active:scale-95 transition-all">{isProcessing ? "PROCESSING..." : "RUN ENGINE"}</button>
            <div className="bg-black p-4 rounded-3xl h-24 overflow-y-auto font-mono text-[9px] text-zinc-600 border border-zinc-900">
              {logs.map((l, i) => <div key={i}>{l.time} - {l.msg}</div>)}
              <div ref={logEndRef} />
            </div>
          </div>

          <div className="space-y-4">
            <div className="aspect-square bg-black rounded-[40px] border border-zinc-900 overflow-hidden relative shadow-2xl">
              <canvas ref={canvasRef} onMouseDown={() => setIsDrawing(true)} onMouseMove={draw} onMouseUp={() => setIsDrawing(false)} onTouchStart={(e) => { e.preventDefault(); setIsDrawing(true); }} onTouchMove={(e) => { e.preventDefault(); draw(e); }} onTouchEnd={() => setIsDrawing(false)} className="w-full h-full object-contain touch-none" />
            </div>
            <div className="aspect-square bg-zinc-900 rounded-[40px] border border-zinc-900 overflow-hidden relative flex items-center justify-center shadow-2xl">
              {outputMedia ? (
                <>
                  <img src={outputMedia.content} className="w-full h-full object-contain animate-in fade-in duration-500" alt="Output" />
                  <button onClick={handleDownload} className="absolute bottom-6 right-6 bg-white text-black p-5 rounded-full shadow-2xl active:scale-90 transition-transform">
                    <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="3"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/></svg>
                  </button>
                </>
              ) : (
                <span className="text-zinc-800 font-black text-xs tracking-widest animate-pulse italic">AWAITING SIGNAL</span>
              )}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}

