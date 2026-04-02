import React, { useState, useRef, useEffect, useCallback } from 'react';
import { 
  Pencil, 
  Eraser, 
  PaintBucket, 
  Trash2, 
  Play, 
  Brush,
  Undo2,
  Sparkles,
  Pause,
  LayoutTemplate,
  X,
  Wand2,
  Loader2,
  Download,
  AlertCircle
} from 'lucide-react';

// API Key is managed by the environment
const apiKey = ""; 

const DrawingApp = ({ onGoHome }) => {
  const canvasRef = useRef(null);
  const animCanvasRef = useRef(null);
  const contextRef = useRef(null);
  const requestRef = useRef(null);

  const [isDrawing, setIsDrawing] = useState(false);
  const [tool, setTool] = useState('pencil'); 
  const [color, setColor] = useState('#ff6b6b'); 
  const [lineWidth, setLineWidth] = useState(8);
  const [eraserSize, setEraserSize] = useState(40);
  const [isAnimating, setIsAnimating] = useState(false);
  const [history, setHistory] = useState([]);
  const [showTemplates, setShowTemplates] = useState(false);
  const [isGenerating, setIsGenerating] = useState(false);
  const [generationProgress, setGenerationProgress] = useState(0);
  const [activeAnimationType, setActiveAnimationType] = useState('generic');
  const [errorMessage, setErrorMessage] = useState(null);

  const colors = [
    '#000000', '#ff6b6b', '#4ecdc4', '#ffe66d', '#ff9ff3', 
    '#a29bfe', '#55efc4', '#fd79a8', '#74b9ff', '#ffffff'
  ];

  const templates = [
    { id: 'rocket', name: 'Roket', icon: '🚀' },
    { id: 'flower', name: 'Çiçek', icon: '🌻' },
    { id: 'cat', name: 'Kedi', icon: '🐱' },
    { id: 'bird', name: 'Kuş', icon: '🐦' },
    { id: 'rabbit', name: 'Tavşan', icon: '🐰' },
    { id: 'robot', name: 'Robot', icon: '🤖' },
    { id: 'star', name: 'Yıldız', icon: '⭐' },
    { id: 'apple', name: 'Elma', icon: '🍎' },
    { id: 'fish', name: 'Balık', icon: '🐟' }
  ];

  const saveToHistory = useCallback(() => {
    if (!contextRef.current || !canvasRef.current) return;
    try {
      const imageData = contextRef.current.getImageData(0, 0, canvasRef.current.width, canvasRef.current.height);
      setHistory(prev => [...prev, imageData].slice(-25));
    } catch (e) {
      console.warn("Geçmiş kaydedilemedi.");
    }
  }, []);

  useEffect(() => {
    const canvas = canvasRef.current;
    const animCanvas = animCanvasRef.current;
    
    const initCanvas = () => {
      const parent = canvas.parentElement;
      if (!parent) return;
      const rect = parent.getBoundingClientRect();
      if (rect.width === 0 || rect.height === 0) return; 
      
      const tempCanvas = document.createElement('canvas');
      tempCanvas.width = canvas.width;
      tempCanvas.height = canvas.height;
      if (canvas.width > 0 && canvas.height > 0) {
        const tempCtx = tempCanvas.getContext('2d');
        if (tempCtx) tempCtx.drawImage(canvas, 0, 0);
      }
      
      canvas.width = rect.width;
      canvas.height = rect.height;
      animCanvas.width = rect.width;
      animCanvas.height = rect.height;
      
      const context = canvas.getContext('2d', { willReadFrequently: true });
      context.lineCap = 'round';
      context.lineJoin = 'round';
      
      if (tempCanvas.width > 0 && tempCanvas.height > 0) {
        context.drawImage(tempCanvas, 0, 0, canvas.width, canvas.height);
      }
      
      contextRef.current = context;
      if (history.length === 0) saveToHistory();
    };

    initCanvas();
    window.addEventListener('resize', initCanvas);
    return () => window.removeEventListener('resize', initCanvas);
  }, [saveToHistory, history.length]);

  useEffect(() => {
    if (contextRef.current) {
      contextRef.current.strokeStyle = tool === 'eraser' ? '#ffffff' : color;
      contextRef.current.lineWidth = tool === 'eraser' ? eraserSize : lineWidth;
    }
  }, [color, lineWidth, eraserSize, tool]);

  const removeBackground = (ctx, width, height) => {
    if (width === 0 || height === 0) return;
    const imageData = ctx.getImageData(0, 0, width, height);
    const data = imageData.data;
    for (let i = 0; i < data.length; i += 4) {
      if (data[i] > 240 && data[i+1] > 240 && data[i+2] > 240) {
        data[i+3] = 0;
      }
    }
    ctx.putImageData(imageData, 0, 0);
  };

  const fetchWithRetry = async (url, options, retries = 5) => {
    let delay = 1000;
    for (let i = 0; i < retries; i++) {
      try {
        const response = await fetch(url, options);
        if (!response.ok) throw new Error(`Status: ${response.status}`);
        return await response.json();
      } catch (error) {
        if (i === retries - 1) throw error;
        await new Promise(res => setTimeout(res, delay));
        delay *= 2;
      }
    }
  };

  const enhanceImage = async () => {
    if (isGenerating || isAnimating) return;
    const canvas = canvasRef.current;
    if (canvas.width === 0 || canvas.height === 0) return;
    
    setIsGenerating(true);
    setGenerationProgress(10);
    setErrorMessage(null);

    try {
      const offscreen = document.createElement('canvas');
      offscreen.width = canvas.width;
      offscreen.height = canvas.height;
      const oCtx = offscreen.getContext('2d');
      oCtx.fillStyle = 'white';
      oCtx.fillRect(0, 0, offscreen.width, offscreen.height);
      oCtx.drawImage(canvas, 0, 0);
      const base64Data = offscreen.toDataURL('image/png').split(',')[1];

      setGenerationProgress(30);
      
      const identifyData = await fetchWithRetry(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{
            role: "user",
            parts: [
              { text: "Identify the main object in this sketch. Return only the object name in English. JSON format: { \"name\": \"object_name\" }" },
              { inlineData: { mimeType: "image/png", data: base64Data } }
            ]
          }],
          generationConfig: { responseMimeType: "application/json" }
        })
      });

      setGenerationProgress(50);
      
      let objName = "cute creature";
      try {
        const textResponse = identifyData.candidates?.[0]?.content?.parts?.[0]?.text;
        if (textResponse) {
          const parsed = JSON.parse(textResponse);
          objName = parsed.name || "cute creature";
        }
      } catch (e) { console.warn("Failed to parse object identification", e); }

      setGenerationProgress(70);

      const editData = await fetchWithRetry(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{
            parts: [
              { text: `Transform this sketch into an ultra-cute (Kawaii) 3D ${objName} character. Style: Squishmallow-like, soft plushy texture, rounded shapes, giant expressive eyes with sparkles, tiny blushing cheeks. Visual: Vibrant pastel colors, soft ambient lighting, Pixar-style charm. Background: Strictly pure white (#FFFFFF).` },
              { inlineData: { mimeType: "image/png", data: base64Data } }
            ]
          }],
          generationConfig: { responseModalities: ['TEXT', 'IMAGE'] }
        })
      });

      setGenerationProgress(90);

      const base64 = editData.candidates?.[0]?.content?.parts?.find(p => p.inlineData)?.inlineData?.data;

      if (base64) {
        const img = new Image();
        img.crossOrigin = "anonymous";
        img.onload = () => {
          const ctx = contextRef.current;
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          
          const temp = document.createElement('canvas');
          temp.width = canvas.width;
          temp.height = canvas.height;
          const tCtx = temp.getContext('2d');
          tCtx.drawImage(img, 0, 0, temp.width, temp.height);
          removeBackground(tCtx, temp.width, temp.height);
          
          ctx.drawImage(temp, 0, 0);
          setActiveAnimationType(objName.toLowerCase());
          saveToHistory();
          setGenerationProgress(100);
          setTimeout(() => {
            setIsGenerating(false);
            setGenerationProgress(0);
          }, 500);
        };
        img.src = `data:image/png;base64,${base64}`;
      } else {
        throw new Error("No image data returned from API.");
      }
    } catch (e) {
      console.error("Transformation Error:", e);
      setIsGenerating(false);
      setGenerationProgress(0);
      setErrorMessage("Karakterin dönüştürülürken bir hata oluştu. Lütfen tekrar dene.");
    }
  };

  const drawTemplate = (id) => {
    const ctx = contextRef.current;
    if (!ctx) return;
    const w = canvasRef.current.width, h = canvasRef.current.height, cx = w/2, cy = h/2;
    clearCanvas();
    ctx.strokeStyle = '#ffb8b8'; ctx.lineWidth = 12; ctx.beginPath();
    
    if (id === 'star') {
        for(let i=0; i<10; i++){
            const a = (i * 0.4 - 0.5) * Math.PI;
            const r = i % 2 === 0 ? 120 : 70;
            ctx.lineTo(cx + Math.cos(a)*r, cy + Math.sin(a)*r);
        }
        ctx.closePath();
    } else if (id === 'cat') {
        ctx.arc(cx, cy, 80, 0, Math.PI*2);
        ctx.moveTo(cx-50, cy-60); ctx.lineTo(cx-70, cy-120); ctx.lineTo(cx-20, cy-75);
        ctx.moveTo(cx+50, cy-60); ctx.lineTo(cx+70, cy-120); ctx.lineTo(cx+20, cy-75);
    } else if (id === 'rabbit') {
        ctx.ellipse(cx, cy, 70, 80, 0, 0, Math.PI*2);
        ctx.moveTo(cx-30, cy-70); ctx.quadraticCurveTo(cx-50, cy-180, cx-20, cy-180); ctx.quadraticCurveTo(cx+10, cy-180, cx, cy-70);
        ctx.moveTo(cx+30, cy-70); ctx.quadraticCurveTo(cx+50, cy-180, cx+20, cy-180); ctx.quadraticCurveTo(cx-10, cy-180, cx, cy-70);
    } else if (id === 'rocket') {
        ctx.moveTo(cx, cy-150); ctx.quadraticCurveTo(cx-60, cy-40, cx-60, cy+90);
        ctx.lineTo(cx+60, cy+90); ctx.quadraticCurveTo(cx+60, cy-40, cx, cy-150);
        ctx.moveTo(cx-60, cy+90); ctx.lineTo(cx-90, cy+130); ctx.lineTo(cx-40, cy+90);
        ctx.moveTo(cx+60, cy+90); ctx.lineTo(cx+90, cy+130); ctx.lineTo(cx+40, cy+90);
    } else if (id === 'flower') {
        ctx.arc(cx, cy-40, 30, 0, Math.PI*2);
        for(let i=0; i<6; i++){
            const a = (i*Math.PI*2)/6;
            ctx.moveTo(cx+Math.cos(a)*30, cy-40+Math.sin(a)*30);
            ctx.arc(cx+Math.cos(a)*80, cy-40+Math.sin(a)*80, 40, 0, Math.PI*2);
        }
        ctx.moveTo(cx, cy); ctx.lineTo(cx, cy+180);
    } else if (id === 'bird') {
        ctx.ellipse(cx, cy, 90, 60, 0, 0, Math.PI*2);
        ctx.moveTo(cx+80, cy-10); ctx.lineTo(cx+130, cy); ctx.lineTo(cx+80, cy+10);
        ctx.moveTo(cx-20, cy-20); ctx.quadraticCurveTo(cx-50, cy-80, cx-80, cy-20);
        ctx.arc(cx+40, cy-20, 8, 0, Math.PI*2);
    } else if (id === 'robot') {
        ctx.rect(cx-70, cy-80, 140, 160);
        ctx.moveTo(cx-70, cy); ctx.lineTo(cx-110, cy);
        ctx.moveTo(cx+70, cy); ctx.lineTo(cx+110, cy);
        ctx.rect(cx-40, cy-40, 20, 20);
        ctx.rect(cx+20, cy-40, 20, 20);
        ctx.moveTo(cx-30, cy+30); ctx.lineTo(cx+30, cy+30);
        ctx.moveTo(cx, cy-80); ctx.lineTo(cx, cy-120);
        ctx.moveTo(cx-10, cy-120); ctx.rect(cx-10, cy-130, 20, 10);
    } else if (id === 'apple') {
        ctx.moveTo(cx, cy-40);
        ctx.bezierCurveTo(cx+70, cy-80, cx+110, cy+50, cx, cy+90);
        ctx.bezierCurveTo(cx-110, cy+50, cx-70, cy-80, cx, cy-40);
        ctx.moveTo(cx, cy-40); ctx.quadraticCurveTo(cx+10, cy-70, cx+20, cy-90);
        ctx.moveTo(cx+15, cy-60); ctx.quadraticCurveTo(cx+50, cy-80, cx+70, cy-50); ctx.quadraticCurveTo(cx+40, cy-30, cx+15, cy-60);
    } else if (id === 'fish') {
        ctx.ellipse(cx, cy, 90, 60, 0, 0, Math.PI*2);
        ctx.moveTo(cx-90, cy); ctx.lineTo(cx-150, cy-50); ctx.lineTo(cx-150, cy+50); ctx.lineTo(cx-90, cy);
        ctx.moveTo(cx+40, cy-10); ctx.arc(cx+40, cy-10, 8, 0, Math.PI*2);
        ctx.moveTo(cx+90, cy+10); ctx.quadraticCurveTo(cx+60, cy+30, cx+30, cy+20);
        ctx.moveTo(cx-20, cy-60); ctx.lineTo(cx-40, cy-90); ctx.lineTo(cx-60, cy-50);
    }
    ctx.stroke();
    setActiveAnimationType(id);
    saveToHistory();
    setShowTemplates(false);
  };

  const getCoordinates = (e) => {
    const rect = canvasRef.current.getBoundingClientRect();
    const clientX = e.clientX || (e.touches && e.touches[0].clientX);
    const clientY = e.clientY || (e.touches && e.touches[0].clientY);
    if (clientX === undefined) return { x: 0, y: 0 };
    return {
      x: (clientX - rect.left) * (canvasRef.current.width / rect.width),
      y: (clientY - rect.top) * (canvasRef.current.height / rect.height)
    };
  };

  const handleStart = (e) => {
    if (isAnimating) setIsAnimating(false);
    const { x, y } = getCoordinates(e.nativeEvent || e);
    if (tool === 'bucket') {
      fillArea(Math.round(x), Math.round(y));
      saveToHistory();
      return;
    }
    contextRef.current.beginPath();
    contextRef.current.moveTo(x, y);
    setIsDrawing(true);
  };

  const handleMove = (e) => {
    if (!isDrawing) return;
    const { x, y } = getCoordinates(e.nativeEvent || e);
    contextRef.current.lineTo(x, y);
    contextRef.current.stroke();
  };

  const handleEnd = () => {
    if (isDrawing) {
      contextRef.current.closePath();
      setIsDrawing(false);
      saveToHistory();
    }
  };

  const fillArea = (startX, startY) => {
    const canvas = canvasRef.current, ctx = contextRef.current;
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const pixels = imageData.data;
    const startPos = (startY * canvas.width + startX) * 4;
    if (pixels[startPos + 3] > 200) return;

    const fillRGB = [
      parseInt(color.slice(1,3), 16), 
      parseInt(color.slice(3,5), 16), 
      parseInt(color.slice(5,7), 16)
    ];
    
    const stack = [[startX, startY]];
    while (stack.length > 0) {
      const [x, y] = stack.pop();
      if (x < 0 || x >= canvas.width || y < 0 || y >= canvas.height) continue;
      const pos = (y * canvas.width + x) * 4;
      if (pixels[pos + 3] <= 50) {
        pixels[pos] = fillRGB[0];
        pixels[pos+1] = fillRGB[1];
        pixels[pos+2] = fillRGB[2];
        pixels[pos+3] = 255;
        stack.push([x+1, y], [x-1, y], [x, y+1], [x, y-1]);
      }
    }
    ctx.putImageData(imageData, 0, 0);
  };

  const undo = () => {
    if (history.length <= 1) return;
    const newHistory = [...history];
    newHistory.pop();
    contextRef.current.putImageData(newHistory[newHistory.length - 1], 0, 0);
    setHistory(newHistory);
  };

  const clearCanvas = () => {
    contextRef.current.clearRect(0, 0, canvasRef.current.width, canvasRef.current.height);
    setIsAnimating(false);
    setActiveAnimationType('generic');
    saveToHistory();
  };

  const downloadImage = () => {
    try {
      const sourceCanvas = isAnimating ? animCanvasRef.current : canvasRef.current;
      const downloadCanvas = document.createElement('canvas');
      downloadCanvas.width = sourceCanvas.width;
      downloadCanvas.height = sourceCanvas.height;
      const dCtx = downloadCanvas.getContext('2d');
      
      dCtx.fillStyle = '#FFFFFF';
      dCtx.fillRect(0, 0, downloadCanvas.width, downloadCanvas.height);
      dCtx.drawImage(sourceCanvas, 0, 0);
      
      downloadCanvas.toBlob((blob) => {
        if (!blob) {
          setErrorMessage("Görsel dosyası oluşturulamadı.");
          return;
        }
        const url = URL.createObjectURL(blob);
        const link = document.createElement('a');
        link.download = `karakterim-${Date.now()}.jpg`;
        link.href = url;
        document.body.appendChild(link);
        link.click();
        setTimeout(() => {
          document.body.removeChild(link);
          URL.revokeObjectURL(url);
        }, 100);
      }, 'image/jpeg', 0.9);
    } catch (e) {
      console.error("Download Error:", e);
      setErrorMessage("Kaydetme sırasında bir sorun oluştu.");
    }
  };

  useEffect(() => {
    if (!isAnimating) return;
    const animCanvas = animCanvasRef.current;
    const animCtx = animCanvas.getContext('2d');
    const imageData = contextRef.current.getImageData(0,0, animCanvas.width, animCanvas.height);
    let minX = animCanvas.width, minY = animCanvas.height, maxX = 0, maxY = 0, found = false;
    
    for(let i=0; i<imageData.data.length; i+=4) {
      if(imageData.data[i+3] > 20) {
        const x = (i/4) % animCanvas.width;
        const y = Math.floor((i/4) / animCanvas.width);
        minX = Math.min(minX, x); minY = Math.min(minY, y);
        maxX = Math.max(maxX, x); maxY = Math.max(maxY, y);
        found = true;
      }
    }
    
    if(!found) { setIsAnimating(false); return; }
    
    const cx = (minX + maxX) / 2;
    const cy = (minY + maxY) / 2;
    const sprite = document.createElement('canvas');
    sprite.width = animCanvas.width; sprite.height = animCanvas.height;
    sprite.getContext('2d').putImageData(imageData, 0, 0);

    let startTime = Date.now();
    const animate = () => {
      if (!isAnimating) return;
      const time = (Date.now() - startTime) / 1000;
      animCtx.clearRect(0, 0, animCanvas.width, animCanvas.height);
      
      let tx = cx, ty = cy, rot = 0, sx = 1, sy = 1;
      const type = activeAnimationType.toLowerCase();
      const bounce = Math.abs(Math.sin(time * 4)) * 40;
      const squash = 1 + Math.sin(time * 4) * 0.12;
      
      if (type.includes('star') || type.includes('yıldız')) {
        rot = time * 0.8; ty -= bounce * 0.5; sx = sy = 1 + Math.sin(time * 6) * 0.1;
      } else if (type.includes('rocket') || type.includes('roket')) {
        ty -= bounce * 2; tx += Math.sin(time * 15) * 6;
      } else if (type.includes('bird') || type.includes('kuş')) {
        tx += Math.cos(time * 1.5) * 150; ty += Math.sin(time * 5) * 60;
        sy = 0.8 + Math.abs(Math.sin(time * 10)) * 0.4;
      } else if (type.includes('rabbit') || type.includes('tavşan')) {
        ty -= Math.max(0, Math.sin(time * 6)) * 100;
        sx = 1 + Math.max(0, -Math.sin(time * 6)) * 0.2;
      } else {
        ty -= bounce; sy = squash; sx = 1 / squash;
      }

      animCtx.save();
      animCtx.translate(tx, ty);
      animCtx.rotate(rot);
      animCtx.scale(sx, sy);
      animCtx.translate(-cx, -cy);
      animCtx.drawImage(sprite, 0, 0);
      animCtx.restore();
      requestRef.current = requestAnimationFrame(animate);
    };
    
    animate();
    return () => cancelAnimationFrame(requestRef.current);
  }, [isAnimating, activeAnimationType]);

  return (
    <div className="flex flex-col h-screen bg-[#fff7f7] font-sans select-none overflow-hidden text-slate-800">
      <nav className="h-20 bg-white/80 backdrop-blur-md border-b-2 border-pink-100 px-6 flex items-center justify-between shadow-sm z-50">
        <div className="flex items-center gap-5 cursor-pointer hover:opacity-80 transition-opacity" onClick={onGoHome} title="Ana Sayfaya Dön">
          <div className="bg-white rounded-2xl flex items-center justify-center shadow-lg border border-pink-50 p-2 h-14">
            <img 
              src="https://lh3.googleusercontent.com/d/1Juy34CkGmM9jU7tXy7Axq70bNmyChxG5"
              alt="Bilfen Logo" 
              className="h-full w-auto object-contain"
              onError={(e) => { e.currentTarget.src = "https://placehold.co/100x40/pink/white?text=BILFEN"; }}
            />
          </div>
          <div className="hidden sm:block">
            <h1 className="text-xl font-black tracking-tight text-slate-800 uppercase">BİLFEN ÇİZİM ATÖLYESİ</h1>
            <p className="text-[10px] font-bold text-pink-400 uppercase tracking-widest flex items-center gap-1">
              <Sparkles size={12} className="fill-pink-400" /> Hayal Et ve Canlandır
            </p>
          </div>
        </div>

        <div className="flex items-center gap-2">
          <NavBtn onClick={enhanceImage} disabled={isGenerating} icon={isGenerating ? <Loader2 size={18} className="animate-spin" /> : <Wand2 size={18} />} label="Sihirli Dokunuş" variant="magic" />
          <NavBtn onClick={() => setShowTemplates(true)} icon={<LayoutTemplate size={18} />} label="Taslaklar" />
          <NavBtn onClick={undo} disabled={history.length <= 1} icon={<Undo2 size={18} />} label="Geri" />
          <NavBtn onClick={() => setIsAnimating(!isAnimating)} active={isAnimating} icon={isAnimating ? <Pause size={18} fill="currentColor" /> : <Play size={18} fill="currentColor" />} label={isAnimating ? "Durdur" : "Canlandır"} variant="play" />
          <NavBtn onClick={downloadImage} icon={<Download size={18} />} label=".JPG Kaydet" variant="success" />
          <NavBtn onClick={clearCanvas} icon={<Trash2 size={18} />} label="Sil" variant="danger" />
        </div>
      </nav>

      <div className="flex flex-1 overflow-hidden">
        <aside className="w-20 bg-white/50 backdrop-blur-sm border-r border-pink-50 flex flex-col items-center py-6 gap-6 z-10">
          <div className="flex flex-col gap-2">
            <ToolIcon active={tool === 'pencil'} onClick={() => setTool('pencil')} icon={<Pencil size={22} />} tooltip="Kalem" />
            <ToolIcon active={tool === 'brush'} onClick={() => setTool('brush')} icon={<Brush size={22} />} tooltip="Fırça" />
            <ToolIcon active={tool === 'eraser'} onClick={() => setTool('eraser')} icon={<Eraser size={22} />} tooltip="Silgi" />
            <ToolIcon active={tool === 'bucket'} onClick={() => setTool('bucket')} icon={<PaintBucket size={22} />} tooltip="Boya" />
          </div>
          <div className="w-8 h-0.5 bg-pink-100 rounded-full" />
          <div className="flex flex-col gap-2 overflow-y-auto pb-4 [&::-webkit-scrollbar]:hidden [-ms-overflow-style:none] [scrollbar-width:none]">
            {colors.map(c => (
              <button 
                key={c} 
                onClick={() => setColor(c)} 
                className={`w-9 h-9 flex-shrink-0 rounded-xl border-4 transition-all ${color === c ? 'border-white scale-110 shadow-md ring-2 ring-pink-200' : 'border-transparent hover:scale-105'}`}
                style={{ backgroundColor: c }}
              />
            ))}
          </div>
          <div className="mt-auto flex flex-col items-center gap-2">
            <span className="text-[10px] font-black text-pink-400">{tool === 'eraser' ? eraserSize : lineWidth}</span>
            <input 
              type="range" min="2" max="120" 
              value={tool === 'eraser' ? eraserSize : lineWidth} 
              onChange={e => tool === 'eraser' ? setEraserSize(Number(e.target.value)) : setLineWidth(Number(e.target.value))} 
              className="accent-pink-500 h-24 w-1 appearance-none bg-pink-100 rounded-full cursor-pointer"
              style={{ writingMode: 'bt-lr', appearance: 'slider-vertical' }}
            />
          </div>
        </aside>

        <main className="flex-1 bg-[#fffafb] p-4 relative flex items-center justify-center">
          <div className="w-full h-full max-w-5xl aspect-[4/3] bg-white rounded-[2.5rem] shadow-2xl relative overflow-hidden border-[10px] border-white">
            <canvas
              ref={canvasRef} 
              onMouseDown={handleStart} onMouseMove={handleMove} onMouseUp={handleEnd} onMouseLeave={handleEnd}
              onTouchStart={handleStart} onTouchMove={handleMove} onTouchEnd={handleEnd}
              className={`w-full h-full cursor-crosshair touch-none transition-all duration-700 ${isAnimating ? 'opacity-0' : 'opacity-100'}`}
            />
            <canvas
              ref={animCanvasRef} 
              className={`absolute inset-0 pointer-events-none transition-all duration-700 ${isAnimating ? 'opacity-100 scale-100' : 'opacity-0 scale-110'}`}
            />
            
            {/* Yükleme Ekranı */}
            {isGenerating && (
              <div className="absolute inset-0 bg-white/90 backdrop-blur-md flex flex-col items-center justify-center z-[60] animate-in fade-in duration-500">
                <div className="relative mb-8 flex flex-col items-center w-full max-w-md px-10">
                  {/* RO-Bİ Görseli */}
                  <img 
                    src="https://lh3.googleusercontent.com/d/114MpGMQyDUk0JlIx89MdLVIgQgnCpdt4" 
                    alt="RO-Bİ Yükleniyor" 
                    className="w-56 h-56 object-contain animate-bounce mb-8 drop-shadow-xl" 
                  />
                  
                  {/* Yükleme Çubuğu Alanı */}
                  <div className="w-full h-4 bg-pink-50 rounded-full border border-pink-100 overflow-hidden shadow-inner relative">
                    <div 
                      className="h-full bg-gradient-to-r from-pink-400 to-pink-600 transition-all duration-500 ease-out rounded-full"
                      style={{ width: `${generationProgress}%` }}
                    />
                    <div className="absolute inset-0 bg-[linear-gradient(45deg,rgba(255,255,255,.2)_25%,transparent_25%,transparent_50%,rgba(255,255,255,.2)_50%,rgba(255,255,255,.2)_75%,transparent_75%,transparent)] bg-[length:20px_20px] animate-[pulse_2s_linear_infinite] opacity-30" />
                  </div>
                  
                  <div className="mt-4 flex justify-between w-full px-2">
                    <span className="text-pink-600 font-black text-xs uppercase tracking-widest animate-pulse">
                      {generationProgress < 30 ? "Hazırlanıyor..." : 
                       generationProgress < 60 ? "Çizim İnceleniyor..." : 
                       generationProgress < 90 ? "Karakter Oluşturuluyor..." : "Tamamlanıyor!"}
                    </span>
                    <span className="text-pink-600 font-black text-xs">%{generationProgress}</span>
                  </div>
                </div>
                <h3 className="text-3xl font-black text-slate-800 mt-2">RO-Bİ senin için çalışıyor...</h3>
                <p className="text-pink-400 font-bold mt-2">Yapay zeka harikalar yaratıyor</p>
              </div>
            )}
          </div>
        </main>
      </div>

      {showTemplates && (
        <div className="fixed inset-0 bg-pink-900/30 backdrop-blur-sm z-[100] flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-2xl rounded-[3rem] p-10 shadow-2xl relative animate-in zoom-in duration-300">
            <button onClick={() => setShowTemplates(false)} className="absolute top-8 right-8 p-2 hover:bg-pink-100 rounded-full text-pink-400 transition-colors">
              <X size={32} />
            </button>
            <h2 className="text-3xl font-black text-slate-800 mb-6">Taslak Seç</h2>
            <div className="grid grid-cols-2 sm:grid-cols-3 gap-5">
              {templates.map(t => (
                <button key={t.id} onClick={() => drawTemplate(t.id)} className="flex flex-col items-center gap-4 p-6 rounded-[2rem] bg-white border-2 border-slate-50 hover:border-pink-200 hover:bg-pink-50 transition-all group">
                  <span className="text-5xl group-hover:scale-110 transition-transform">{t.icon}</span>
                  <span className="font-black text-xs text-slate-400 uppercase tracking-widest">{t.name}</span>
                </button>
              ))}
            </div>
          </div>
        </div>
      )}

      {errorMessage && (
        <div className="fixed top-24 left-1/2 -translate-x-1/2 z-[150] animate-in slide-in-from-top duration-300">
          <div className="bg-white rounded-2xl shadow-xl border-2 border-red-100 p-4 flex items-center gap-4">
            <div className="w-10 h-10 bg-red-100 rounded-full flex items-center justify-center text-red-500">
              <AlertCircle size={24} />
            </div>
            <div>
              <p className="font-black text-sm text-slate-800">{errorMessage}</p>
              <button onClick={() => setErrorMessage(null)} className="text-xs font-bold text-red-400 uppercase tracking-widest hover:text-red-600">Tamam</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

const App = () => {
  const [view, setView] = useState('home');

  if (view === 'home') {
    return (
      <div className="flex flex-col h-screen bg-[#fff7f7] font-sans select-none overflow-hidden text-slate-800 relative">
        <div className="absolute top-20 left-10 w-72 h-72 bg-pink-200/40 rounded-full mix-blend-multiply filter blur-3xl animate-pulse"></div>
        <div className="absolute bottom-20 right-10 w-80 h-80 bg-cyan-200/40 rounded-full mix-blend-multiply filter blur-3xl animate-pulse" style={{ animationDelay: '2s' }}></div>

        <nav className="h-20 bg-transparent px-6 flex items-center shadow-none z-50 mt-4">
          <div className="flex items-center gap-5">
            <div className="bg-white rounded-2xl flex items-center justify-center shadow-lg border border-pink-50 p-2 h-14">
              <img 
                src="https://lh3.googleusercontent.com/d/1Juy34CkGmM9jU7tXy7Axq70bNmyChxG5"
                alt="Bilfen Logo" 
                className="h-full w-auto object-contain"
                onError={(e) => { e.currentTarget.src = "https://placehold.co/100x40/pink/white?text=BILFEN"; }}
              />
            </div>
            <div>
              <h1 className="text-xl sm:text-2xl font-black tracking-tight text-slate-800 uppercase">BİLFEN ÇİZİM ATÖLYESİ</h1>
              <p className="text-[10px] sm:text-[12px] font-bold text-pink-400 uppercase tracking-widest flex items-center gap-1 mt-1">
                <Sparkles size={14} className="fill-pink-400" /> Hayal Et ve Canlandır
              </p>
            </div>
          </div>
        </nav>

        <main className="flex-1 flex flex-col items-center justify-center relative z-10 p-6 text-center">
          <div className="relative mb-8 group cursor-pointer" onClick={() => setView('drawing')}>
            <div className="absolute inset-0 bg-pink-300 rounded-full blur-3xl opacity-30 group-hover:opacity-50 transition-opacity duration-500"></div>
            <img src="https://lh3.googleusercontent.com/d/1MFFuXd6_8fKzyXu-hDbpe12SL-FUqSLp" alt="RO-Bİ" className="w-64 h-64 md:w-96 md:h-96 relative z-10 drop-shadow-2xl group-hover:scale-110 transition-transform duration-500 object-contain" />
          </div>
          
          <h2 className="text-4xl md:text-6xl font-black text-slate-800 mb-6 tracking-tight">Kendi Karakterini Yarat!</h2>
          
          <button 
            onClick={() => setView('drawing')}
            className="bg-pink-500 hover:bg-pink-600 text-white text-2xl md:text-3xl font-black py-5 px-12 md:px-16 rounded-full shadow-[0_10px_40px_-10px_rgba(236,72,153,0.8)] hover:shadow-[0_10px_40px_-5px_rgba(236,72,153,1)] hover:-translate-y-2 transition-all duration-300 flex items-center gap-4 group"
          >
            <Brush className="group-hover:rotate-12 transition-transform" size={32} />
            Çizime Başla
          </button>
        </main>
      </div>
    );
  }

  return <DrawingApp onGoHome={() => setView('home')} />;
};

const NavBtn = ({ onClick, icon, label, variant, disabled, active }) => {
  const base = "flex items-center gap-2 px-4 py-2 rounded-xl font-black text-[10px] uppercase tracking-widest transition-all active:scale-95 disabled:opacity-30 border-b-4";
  const styles = {
    magic: "bg-pink-500 text-white border-pink-700 shadow-md",
    play: active ? "bg-cyan-500 text-white border-cyan-700" : "bg-cyan-50 text-cyan-600 border-cyan-100",
    success: "bg-emerald-500 text-white border-emerald-700 shadow-md hover:bg-emerald-600",
    danger: "bg-red-50 text-red-600 border-red-100",
    default: "bg-slate-50 text-slate-600 border-slate-200"
  };
  return (
    <button onClick={onClick} disabled={disabled} className={`${base} ${styles[variant || 'default']}`}>
      {icon}
      <span className="hidden xl:inline">{label}</span>
    </button>
  );
};

const ToolIcon = ({ active, onClick, icon, tooltip }) => (
  <button onClick={onClick} title={tooltip} className={`w-12 h-12 rounded-xl flex items-center justify-center transition-all border-b-4 relative group ${active ? 'bg-pink-500 text-white border-pink-700' : 'bg-white text-slate-300 border-slate-100 hover:bg-pink-50'}`}>
    {icon}
  </button>
);

export default App;
