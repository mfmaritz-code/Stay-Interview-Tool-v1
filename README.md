import React, { useState, useEffect } from 'react';
import { 
  User, 
  Target, 
  MessageSquare, 
  FileText, 
  PlusCircle, 
  ClipboardList, 
  RefreshCw, 
  ChevronRight, 
  ChevronLeft, 
  CheckCircle2, 
  AlertCircle,
  Clock,
  TrendingUp,
  BrainCircuit,
  Pencil,
  ExternalLink,
  BookOpen,
  Share2,
  ShieldAlert,
  ChevronDown,
  Download,
  Copy,
  Lock,
  MessagesSquare,
  Printer,
  Info,
  FileDown
} from 'lucide-react';

// --- AI Service with Exponential Backoff ---
const generateStayGuide = async (config) => {
  const apiKey = ""; 
  
  const finalTrigger = config.trigger === "Other" ? config.customTrigger : config.trigger;
  const finalPersona = config.persona === "Other" ? config.customPersona : config.persona;
  const finalSentiment = config.sentiment === "Other" ? config.customSentiment : config.sentiment;

  const systemPrompt = `
    Identity: You are the Thumbtack Stay Interview AI Coach. 
    Philosophy: Empower managers to conduct meaningful retention conversations through structured safety and tailored insight.
    
    Instruction: Generate exactly 7 tailored stay interview questions based on the provided context. 
    Also provide a Recommended Action Plan table with columns: Insight, Controllable Action, Uncontrollable Action.
    
    Format the response as JSON.
  `;

  const userPrompt = `
    Context:
    - Trigger: ${finalTrigger}
    - Type: ${config.type}
    - Persona: ${finalPersona}
    - Sentiment: ${finalSentiment}
  `;

  const payload = {
    contents: [{ parts: [{ text: userPrompt }] }],
    systemInstruction: { parts: [{ text: systemPrompt }] },
    generationConfig: { 
        responseMimeType: "application/json",
        responseSchema: {
            type: "OBJECT",
            properties: {
                openingScript: { type: "STRING" },
                tieredQuestions: {
                    type: "ARRAY",
                    items: {
                        type: "OBJECT",
                        properties: {
                            category: { type: "STRING" },
                            questions: { type: "ARRAY", items: { type: "STRING" } }
                        }
                    }
                },
                actionMatrix: {
                    type: "ARRAY",
                    items: {
                        type: "OBJECT",
                        properties: {
                            insight: { type: "STRING" },
                            controllable: { type: "STRING" },
                            uncontrollable: { type: "STRING" }
                        }
                    }
                }
            },
            required: ["openingScript", "tieredQuestions", "actionMatrix"]
        }
    }
  };

  const fetchWithRetry = async (url, options, retries = 5, delay = 1000) => {
    try {
      const response = await fetch(url, options);
      if (response.ok) return response;
      if (retries > 0 && (response.status === 429 || response.status >= 500)) {
        await new Promise(resolve => setTimeout(resolve, delay));
        return fetchWithRetry(url, options, retries - 1, delay * 2);
      }
      throw new Error(`API Request Failed with status ${response.status}`);
    } catch (error) {
      if (retries > 0) {
        await new Promise(resolve => setTimeout(resolve, delay));
        return fetchWithRetry(url, options, retries - 1, delay * 2);
      }
      throw error;
    }
  };

  try {
    const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;
    const response = await fetchWithRetry(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });

    const result = await response.json();
    const text = result.candidates?.[0]?.content?.parts?.[0]?.text;
    if (!text) throw new Error("Invalid API response format");
    
    return JSON.parse(text);
  } catch (error) {
    console.error("Generation error:", error);
    throw error;
  }
};

// --- Constants ---
const TRIGGERS = ["90-Day Milestone", "Post-Promotion", "Post-Reorganization", "Annual Check-In", "At-Risk Signal", "Other"];
const PERSONAS = ["High Performer / IC", "Recently Promoted / New to Role", "Tenured / Steady Performer", "At-Risk / Showing Disengagement", "New Hire (under 6 months)", "Other"];
const SENTIMENTS = ["Energized but stretched", "Quiet/withdrawn recently", "Frustrated with project outcomes", "Unclear on growth path", "Thriving, want to keep momentum", "Other"];
const INTERVIEW_TYPES = [
  { id: 'standard', label: 'Standard (Manager-led)', icon: User },
  { id: 'skip', label: 'Skip-Level (Boss\'s Boss)', icon: TrendingUp }
];

// --- Components ---

const StepIndicator = ({ currentStep }) => (
  <div className="flex items-center justify-between mb-8 print:hidden">
    {[1, 2, 3].map((step) => (
      <div key={step} className="flex items-center flex-1 text-center">
        <div className={`w-8 h-8 rounded-full mx-auto flex items-center justify-center font-bold text-xs transition-colors ${
          currentStep >= step ? 'bg-blue-600 text-white' : 'bg-gray-200 text-gray-400'
        }`}>
          {step}
        </div>
        {step < 3 && <div className={`h-0.5 flex-1 mx-2 ${currentStep > step ? 'bg-blue-600' : 'bg-gray-200'}`} />}
      </div>
    ))}
  </div>
);

const ConfidentialityNotice = () => (
  <div className="bg-amber-50 border border-amber-200 rounded-2xl p-4 flex items-start gap-3 shadow-sm mb-8 animate-in fade-in slide-in-from-top-4 duration-500 print:hidden">
    <ShieldAlert size={20} className="text-amber-600 flex-shrink-0 mt-0.5" />
    <div className="space-y-1">
        <h4 className="text-sm font-bold text-amber-900 leading-none">Confidentiality Notice</h4>
        <p className="text-[11px] text-amber-800 font-medium leading-relaxed">
            This conversation is private. Escalate immediately if legal or safety concerns are raised.
        </p>
    </div>
  </div>
);

const ManagerForm = ({ config, setConfig, onGenerate, loading }) => {
  const [step, setStep] = useState(1);

  const isFormValid = () => {
    if (step === 1) return config.trigger && (config.trigger !== "Other" || config.customTrigger.trim()) && config.type;
    if (step === 2) return config.persona && (config.persona !== "Other" || config.customPersona.trim());
    if (step === 3) return config.sentiment !== "";
    return true;
  };

  return (
    <div className="bg-white rounded-3xl shadow-sm p-8 md:p-10 border border-gray-100 max-w-2xl mx-auto">
      <StepIndicator currentStep={step} />
      
      {step === 1 && (
        <div className="space-y-6 animate-in slide-in-from-right-4 duration-300">
          <div>
            <label className="block text-sm font-bold text-gray-700 mb-4 flex items-center gap-2 uppercase tracking-wider">
              <Clock size={16} className="text-blue-600" /> 1. Select Trigger
            </label>
            <div className="grid grid-cols-2 gap-3">
              {TRIGGERS.map(t => (
                <button
                  key={t}
                  onClick={() => setConfig({ ...config, trigger: t })}
                  className={`px-4 py-4 text-sm rounded-xl border text-left transition-all ${
                    config.trigger === t ? 'border-blue-600 bg-blue-50 text-blue-700 font-bold' : 'border-gray-200 hover:border-blue-200 text-gray-600'
                  }`}
                >
                  {t}
                </button>
              ))}
            </div>
            {config.trigger === "Other" && (
              <input 
                type="text"
                placeholder="Describe trigger..."
                value={config.customTrigger}
                onChange={(e) => setConfig({...config, customTrigger: e.target.value})}
                className="w-full mt-3 px-4 py-2 rounded-lg border border-blue-200 focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm font-medium"
              />
            )}
          </div>
          <div>
            <label className="block text-sm font-bold text-gray-700 mb-4 uppercase tracking-wider">Interview Type</label>
            <div className="grid grid-cols-2 gap-3">
              {INTERVIEW_TYPES.map(type => (
                <button
                  key={type.id}
                  onClick={() => setConfig({ ...config, type: type.label })}
                  className={`flex flex-col items-center justify-center p-4 rounded-xl border transition-all ${
                    config.type === type.label ? 'border-blue-600 bg-blue-50 text-blue-700 font-bold' : 'border-gray-200 hover:border-blue-200 text-gray-500'
                  }`}
                >
                  <type.icon className="mb-2" size={20} />
                  <span className="text-xs font-semibold">{type.label}</span>
                </button>
              ))}
            </div>
          </div>
        </div>
      )}

      {step === 2 && (
        <div className="space-y-6 animate-in slide-in-from-right-4 duration-300">
          <div>
            <label className="block text-sm font-bold text-gray-700 mb-4 flex items-center gap-2 uppercase tracking-wider">
              <User size={16} className="text-blue-600" /> 2. Employee Persona
            </label>
            <div className="grid grid-cols-1 gap-2">
              {PERSONAS.map(p => (
                <button
                  key={p}
                  onClick={() => setConfig({ ...config, persona: p })}
                  className={`px-4 py-4 text-sm rounded-xl border text-left transition-all ${
                    config.persona === p ? 'border-blue-600 bg-blue-50 text-blue-700 font-bold' : 'border-gray-200 hover:border-blue-200 text-gray-600'
                  }`}
                >
                  {p}
                </button>
              ))}
            </div>
            {config.persona === "Other" && (
              <input 
                type="text"
                placeholder="Describe role/profile..."
                value={config.customPersona}
                onChange={(e) => setConfig({...config, customPersona: e.target.value})}
                className="w-full mt-3 px-4 py-2 rounded-lg border border-blue-200 focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm font-medium"
              />
            )}
          </div>
        </div>
      )}

      {step === 3 && (
        <div className="space-y-6 animate-in slide-in-from-right-4 duration-300">
          <div>
            <label className="block text-sm font-bold text-gray-700 mb-4 flex items-center gap-2 uppercase tracking-wider">
              <MessageSquare size={16} className="text-blue-600" /> 3. Current Sentiment
            </label>
            <select 
              value={config.sentiment}
              onChange={(e) => setConfig({...config, sentiment: e.target.value})}
              className="w-full px-4 py-4 rounded-xl border border-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm appearance-none bg-white shadow-sm font-semibold"
            >
              <option value="">Select current vibe...</option>
              {SENTIMENTS.map(s => <option key={s} value={s}>{s}</option>)}
            </select>
          </div>
        </div>
      )}

      <div className="mt-10 flex justify-between gap-4">
        {step > 1 ? (
          <button onClick={() => setStep(step - 1)} className="text-gray-500 font-bold text-sm px-4 hover:text-gray-900 transition-colors">Back</button>
        ) : <div />}

        {step < 3 ? (
          <button 
            disabled={!isFormValid()}
            onClick={() => setStep(step + 1)}
            className="bg-blue-600 hover:bg-blue-700 text-white font-bold px-8 py-3 rounded-xl transition-all shadow-lg shadow-blue-100 disabled:opacity-50 uppercase tracking-widest text-xs"
          >
            Continue
          </button>
        ) : (
          <button 
            onClick={onGenerate}
            disabled={loading || !config.sentiment}
            className="bg-blue-600 hover:bg-blue-700 text-white font-bold px-10 py-3 rounded-xl transition-all shadow-lg shadow-blue-100 disabled:bg-gray-300 flex items-center gap-2 uppercase tracking-widest text-xs"
          >
            {loading ? <RefreshCw className="animate-spin" size={18} /> : <MessagesSquare size={18} />}
            {loading ? 'Building Toolkit...' : 'Generate Toolkit'}
          </button>
        )}
      </div>
    </div>
  );
};

const ResultViewer = ({ guide, onReset, config }) => {
  const [activeTab, setActiveTab] = useState('questions');
  const [copied, setCopied] = useState(false);

  const displayTrigger = config.trigger === "Other" ? config.customTrigger : config.trigger;
  const displayPersona = config.persona === "Other" ? config.customPersona : config.persona;
  const allQuestions = guide?.tieredQuestions?.flatMap(cat => cat.questions || []) || [];

  const handlePrint = () => {
    window.print();
  };

  const downloadStandaloneDoc = () => {
    const questionsHtml = allQuestions.map((q, i) => `
        <div style="margin-bottom: 20px;">
            <p style="font-weight: 600; font-size: 16px; margin: 0; color: #1e293b;">${i + 1}. ${q}</p>
        </div>
    `).join('');

    const matrixRows = (guide.actionMatrix || []).map(row => `
        <tr>
            <td style="border: 1px solid #cbd5e1; padding: 12px; font-weight: 700;">${row.insight}</td>
            <td style="border: 1px solid #cbd5e1; padding: 12px; color: #334155;">${row.controllable}</td>
            <td style="border: 1px solid #cbd5e1; padding: 12px; font-style: italic; color: #64748b;">${row.uncontrollable}</td>
        </tr>
    `).join('');

    const fullHtml = `
      <!DOCTYPE html>
      <html>
      <head>
        <meta charset="utf-8">
        <title>Stay Interview Reference - ${displayTrigger}</title>
        <link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;500;600;700;900&display=swap" rel="stylesheet">
        <style>
            body { font-family: 'Montserrat', sans-serif; line-height: 1.6; color: #0f172a; padding: 40px; max-width: 850px; margin: 0 auto; }
            .header { border-bottom: 4px solid #2563eb; padding-bottom: 20px; margin-bottom: 30px; }
            h1 { font-weight: 900; font-size: 28px; margin: 0; color: #1e293b; text-transform: uppercase; }
            .meta { display: flex; justify-content: space-between; font-weight: 700; color: #64748b; font-size: 12px; margin-top: 10px; text-transform: uppercase; }
            .script-box { background: #eff6ff; border-left: 6px solid #2563eb; padding: 20px; margin: 30px 0; border-radius: 4px; }
            h2 { font-weight: 800; color: #2563eb; border-bottom: 1px solid #e2e8f0; padding-bottom: 8px; margin-top: 40px; }
            table { width: 100%; border-collapse: collapse; margin-top: 20px; }
            th { background: #f1f5f9; padding: 12px; text-align: left; font-weight: 900; font-size: 11px; text-transform: uppercase; color: #475569; border: 1px solid #cbd5e1; }
            .footer { margin-top: 60px; padding-top: 20px; border-top: 1px solid #e2e8f0; text-align: center; font-size: 10px; color: #94a3b8; font-weight: 700; }
            @media print { .no-print { display: none; } }
        </style>
      </head>
      <body>
        <div class="no-print" style="background: #1e293b; color: white; padding: 15px; border-radius: 8px; margin-bottom: 30px; text-align: center;">
            <p style="margin: 0; font-weight: 700; font-size: 14px;">Coaching Guide Ready! Press <b>Ctrl + P</b> (or Cmd + P) to Save as PDF.</p>
        </div>
        <div class="header">
            <h1>Stay Interview Toolkit</h1>
            <div class="meta">
                <span>Trigger: ${displayTrigger}</span>
                <span>Persona: ${displayPersona}</span>
            </div>
        </div>
        <div class="script-box">
            <div style="font-size: 11px; font-weight: 900; color: #2563eb; margin-bottom: 8px; text-transform: uppercase;">Suggested Opening Script</div>
            <p style="margin: 0; font-weight: 500; font-size: 18px; color: #1e3a8a;">"${guide.openingScript}"</p>
        </div>
        <h2>Interview Reference Questions</h2>
        <p style="font-size: 12px; color: #64748b; margin-bottom: 20px;">Use this list as a guide during your conversation. Log all notes and feedback in the official Google Form tracker.</p>
        ${questionsHtml}
        <div style="page-break-before: always;">
            <h2>Recommended Action Plan</h2>
            <table>
                <thead>
                    <tr>
                        <th>Insight Uncovered</th>
                        <th>Manager-led Action</th>
                        <th>Escalation Point</th>
                    </tr>
                </thead>
                <tbody>${matrixRows}</tbody>
            </table>
        </div>
        <div class="footer">Thumbtack Manager Toolkit • Internal Use Only</div>
      </body>
      </html>
    `;

    const blob = new Blob([fullHtml], { type: 'text/html' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `Stay_Interview_Reference_${displayTrigger.replace(/\s+/g, '_')}.html`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  const copyToClipboard = () => {
    const questionsText = allQuestions.map((q, i) => `${i + 1}. ${q}\n`).join('');
    const matrixText = (guide.actionMatrix || []).map(row => `- Insight: ${row.insight}\n  Fix: ${row.controllable}\n  Escalate: ${row.uncontrollable}`).join('\n\n');
    const text = `STAY INTERVIEW TOOLKIT\n\nPrepared for: ${displayPersona}\nTrigger: ${displayTrigger}\n\n1. OPENING SCRIPT\n"${guide.openingScript}"\n\n2. QUESTIONS\n${questionsText}\n\n3. RECOMMENDED ACTION PLAN\n${matrixText}`;
    
    const el = document.createElement('textarea');
    el.value = text;
    document.body.appendChild(el);
    el.select();
    document.execCommand('copy');
    document.body.removeChild(el);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  const getPrefilledUrl = () => {
    const baseUrl = "https://docs.google.com/forms/d/e/1FAIpQLSf0ZYrWzdNensiO56FpXgnIVFHuKDz57zZGoayviBeTsdK4CQ/viewform";
    const TYPE_ID = "entry.264105683";     
    const PERSONA_ID = "entry.359219502";  
    const QUESTION_IDS = ["entry.1957147871", "entry.1390220802", "entry.435531527", "entry.1635334784", "entry.1081351633", "entry.1090088368", "entry.741519336"];

    const params = new URLSearchParams();
    params.append("usp", "pp_url");
    params.append(TYPE_ID, displayTrigger);
    params.append(PERSONA_ID, displayPersona);
    
    allQuestions.forEach((q, index) => {
        if (QUESTION_IDS[index]) { 
            params.append(QUESTION_IDS[index], `${index + 1}. ${q}\n\n\n`); 
        }
    });
    return `${baseUrl}?${params.toString()}`;
  };

  const generatedUrl = getPrefilledUrl();

  return (
    <div className="bg-white rounded-3xl shadow-xl border border-gray-100 overflow-hidden animate-in fade-in zoom-in-95 duration-500 font-sans relative">
      
      {/* EXPORT VIEW (Shown in Print) - CLEAN LIST ONLY */}
      <div id="pdf-export-content" className="hidden print:block p-12 text-slate-900 bg-white min-h-screen">
          <div className="border-b-4 border-blue-600 pb-6 mb-8">
              <h1 className="text-3xl font-black text-slate-900 mb-2">Stay Interview Reference</h1>
              <div className="flex justify-between text-sm font-bold text-slate-500 uppercase tracking-widest">
                  <span>Persona: {displayPersona}</span>
                  <span>Trigger: {displayTrigger}</span>
                  <span>Date: {new Date().toLocaleDateString()}</span>
              </div>
          </div>
          
          <div className="mb-10 bg-slate-50 border border-slate-200 p-8 rounded-xl">
              <h3 className="text-xs font-black uppercase tracking-widest text-blue-600 mb-3">Opening Script</h3>
              <p className="text-xl font-medium leading-relaxed italic text-slate-700">"{guide.openingScript}"</p>
          </div>

          <h2 className="text-xl font-bold text-blue-600 mb-4 border-b border-slate-100 pb-2">Targeted Retention Questions</h2>
          <p className="text-xs text-slate-400 font-medium mb-6">Conduct the interview using these questions as a guide. Log responses in your Google Form tracker.</p>
          
          <div className="space-y-4 mb-12">
              {allQuestions.map((q, i) => (
                  <div key={i} className="pb-4 border-b border-slate-50 last:border-0">
                      <p className="text-lg font-semibold text-slate-800">{i + 1}. {q}</p>
                  </div>
              ))}
          </div>

          <div className="page-break-before">
              <h2 className="text-xl font-bold text-blue-600 mb-6 border-b border-slate-100 pb-2">Recommended Action Plan</h2>
              <table className="w-full border-collapse">
                  <thead>
                      <tr className="bg-slate-100">
                          <th className="border border-slate-300 p-4 text-left text-xs font-black uppercase tracking-wider">Possible Insight</th>
                          <th className="border border-slate-300 p-4 text-left text-xs font-black uppercase tracking-wider">Manager-led Action</th>
                          <th className="border border-slate-300 p-4 text-left text-xs font-black uppercase tracking-wider">Escalation Point</th>
                      </tr>
                  </thead>
                  <tbody>
                      {(guide.actionMatrix || []).map((row, idx) => (
                          <tr key={idx}>
                              <td className="border border-slate-300 p-4 font-bold text-slate-800 align-top">{row.insight}</td>
                              <td className="border border-slate-300 p-4 text-slate-600 align-top">{row.controllable}</td>
                              <td className="border border-slate-300 p-4 text-slate-400 italic align-top">{row.uncontrollable}</td>
                          </tr>
                      ))}
                  </tbody>
              </table>
          </div>
          
          <div className="mt-12 text-[10px] font-bold text-slate-400 uppercase text-center border-t pt-6">
              Reference Only • Use Google Form for documentation • Thumbtack Manager Toolkit
          </div>
      </div>

      {/* WEB VIEW */}
      <div className="print:hidden">
        <ConfidentialityNotice />
        <div className="p-8 border-b bg-gray-50/50 flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
            <div>
            <h2 className="text-2xl font-bold text-gray-900 leading-tight">Framework Ready</h2>
            <p className="text-xs text-gray-500 font-bold uppercase tracking-widest mt-1">{displayTrigger} • {displayPersona}</p>
            </div>
            <div className="flex flex-wrap gap-2 w-full md:w-auto">
            <button 
                onClick={downloadStandaloneDoc} 
                className="flex-1 md:flex-none flex items-center justify-center gap-2 bg-blue-600 text-white px-5 py-2.5 rounded-xl text-xs font-bold uppercase tracking-widest hover:bg-blue-700 transition-all shadow-sm group"
            >
                <FileDown size={16} /> Save Reference List
            </button>
            <button onClick={copyToClipboard} className="flex-1 md:flex-none flex items-center justify-center gap-2 bg-white border border-gray-200 px-5 py-2.5 rounded-xl text-xs font-bold uppercase tracking-widest hover:bg-gray-50 transition-all shadow-sm">
                {copied ? <CheckCircle2 size={16} className="text-green-600" /> : <Copy size={16} />}
                {copied ? 'Copied!' : 'Copy to Clipboard'}
            </button>
            <button onClick={onReset} title="Restart" className="p-2.5 bg-gray-200 text-gray-600 rounded-xl hover:bg-gray-300 transition-colors flex items-center gap-2 px-4">
                <RefreshCw size={18} /> <span className="text-xs font-bold uppercase tracking-widest">Reset</span>
            </button>
            </div>
        </div>

        <div className="flex border-b text-[10px] font-black uppercase tracking-[0.2em] bg-white">
            {['questions', 'matrix'].map(tab => (
            <button key={tab} onClick={() => setActiveTab(tab)} className={`flex-1 py-5 border-b-2 transition-all ${activeTab === tab ? 'border-blue-600 text-blue-600 bg-blue-50/30' : 'border-transparent text-gray-400 hover:text-gray-600'}`}>
                {tab === 'questions' ? 'Coaching Guide' : 'Recommended Action Plan'}
            </button>
            ))}
        </div>

        <div className="p-8 max-h-[45vh] overflow-y-auto bg-[#fafafb]">
            {activeTab === 'questions' && (
            <div className="space-y-6">
                <div className="p-6 bg-blue-50 rounded-2xl border-l-4 border-blue-500 text-sm text-gray-700 mb-8 leading-relaxed shadow-sm font-medium">"{guide.openingScript}"</div>
                {guide.tieredQuestions.map((cat, i) => (
                <div key={i} className="border border-gray-100 rounded-2xl overflow-hidden shadow-sm bg-white">
                    <div className="bg-gray-50/80 px-5 py-3 border-b text-[10px] font-black text-gray-400 uppercase tracking-widest">{cat.category}</div>
                    <div className="divide-y divide-gray-50">
                    {cat.questions.map((q, j) => (
                        <div key={j} className="p-5 flex items-start gap-4 hover:bg-gray-50/30 transition-colors group">
                        <div className="mt-1.5 w-2 h-2 rounded-full bg-blue-300 group-hover:scale-125 transition-transform" />
                        <p className="text-sm font-semibold text-gray-800 leading-relaxed">{q}</p>
                        </div>
                    ))}
                    </div>
                </div>
                ))}
            </div>
            )}

            {activeTab === 'matrix' && (
            <div className="space-y-6">
                <div className="overflow-x-auto rounded-2xl border border-gray-100 shadow-sm bg-white">
                <table className="w-full text-left text-sm">
                    <thead className="bg-gray-50 border-b border-gray-100">
                    <tr>
                        <th className="px-6 py-4 font-black text-gray-500 uppercase text-[10px] tracking-widest">Possible Insight</th>
                        <th className="px-6 py-4 font-black text-blue-600 uppercase text-[10px] tracking-widest">Controllable Action</th>
                        <th className="px-6 py-4 font-black text-red-400 uppercase text-[10px] tracking-widest">Uncontrollable (Escalate)</th>
                    </tr>
                    </thead>
                    <tbody className="divide-y divide-gray-50">
                    {(guide.actionMatrix || []).map((row, idx) => (
                        <tr key={idx} className="hover:bg-gray-50/50 transition-colors">
                        <td className="px-6 py-5 font-bold text-gray-800">{row.insight}</td>
                        <td className="px-6 py-5 text-gray-600 leading-relaxed">{row.controllable}</td>
                        <td className="px-6 py-5 text-gray-400 italic text-xs leading-relaxed">{row.uncontrollable}</td>
                        </tr>
                    ))}
                    </tbody>
                </table>
                </div>
            </div>
            )}
        </div>

        <div className="p-8 bg-gray-900 text-white border-t border-gray-800">
            <div className="flex items-center gap-3 mb-6">
            <div className="p-2 bg-blue-500/20 rounded-lg"><CheckCircle2 className="text-blue-400" size={24} /></div>
            <h3 className="text-lg font-bold">Log Your Session</h3>
            </div>
            <div className="space-y-4 mb-8">
                <p className="text-lg text-gray-300 leading-relaxed font-medium">I've prepared your framework! Click below to:</p>
                <div className="flex flex-col gap-6">
                    <a href={generatedUrl} target="_blank" rel="noopener noreferrer" className="inline-flex w-full md:w-auto items-center justify-center gap-3 bg-blue-600 hover:bg-blue-500 text-white px-10 py-6 rounded-2xl font-black text-sm uppercase tracking-widest transition-all shadow-2xl shadow-blue-900/40 group active:scale-95">
                        Open Tracker & Log Feedback <ExternalLink size={18} className="group-hover:translate-x-1 group-hover:-translate-y-1 transition-transform" />
                    </a>
                    <p className="text-sm text-gray-400 font-medium">I have pre-filled the interview type and the milestone questions so you are ready to go.</p>
                </div>
            </div>
            <div className="flex items-start gap-3 text-[11px] text-gray-500 bg-black/20 p-4 rounded-lg">
            <ShieldAlert size={14} className="flex-shrink-0 mt-0.5" />
            <p>Note: Each question sent to the form now includes space underneath for you to type feedback directly.</p>
            </div>
        </div>
      </div>
    </div>
  );
};

export default function App() {
  const [config, setConfig] = useState({ trigger: '', customTrigger: '', type: 'Standard (Manager-led)', persona: '', customPersona: '', sentiment: '', goals: ['Retention Factors', 'Growth Alignment'] });
  const [loading, setLoading] = useState(false);
  const [guide, setGuide] = useState(null);
  const [error, setError] = useState(null);

  const handleGenerate = async () => {
    setLoading(true);
    setError(null);
    try {
        const result = await generateStayGuide(config);
        if (result) setGuide(result);
        else setError("Connection error. Please check your network.");
    } catch (e) {
        setError("AI Coach is temporarily unavailable. Please refresh.");
    }
    setLoading(false);
  };

  return (
    <div className="min-h-screen bg-[#fcfcfd] text-gray-900 font-sans selection:bg-blue-100">
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@400;500;600;700;800;900&display=swap');
        .font-sans { font-family: 'Montserrat', sans-serif !important; }
        
        @media print {
            @page { size: portrait; margin: 15mm; }
            body { background: white !important; margin: 0 !important; padding: 0 !important; }
            .print\\:hidden { display: none !important; }
            .print\\:block { display: block !important; }
            #pdf-export-content { display: block !important; visibility: visible !important; position: relative !important; width: 100% !important; }
            .page-break-before { page-break-before: always; margin-top: 20pt; }
        }
      `}</style>
      
      {!guide && (
        <div className="max-w-6xl mx-auto pt-6 px-4 animate-in fade-in slide-in-from-top-4 duration-500">
          <ConfidentialityNotice />
        </div>
      )}
      <header className="max-w-6xl mx-auto py-12 px-4 flex justify-between items-end print:hidden">
        <div>
          <div className="flex items-center gap-3 mb-3">
            <div className="w-12 h-12 bg-blue-600 rounded-2xl flex items-center justify-center shadow-2xl shadow-blue-200 rotate-3 transition-transform hover:rotate-0">
              <MessagesSquare className="text-white" size={28} />
            </div>
            <h1 className="text-3xl font-bold tracking-tight text-gray-900 leading-tight">Stay Interview Toolkit</h1>
          </div>
          <p className="text-gray-500 font-medium text-sm border-l-4 border-blue-600 pl-4 ml-1 max-w-2xl">
            Conduct meaningful retention conversations at the moments that matter most.
          </p>
        </div>
      </header>
      <main className="max-w-6xl mx-auto px-4 pb-24 print:p-0">
        {!guide ? (
          <div className="animate-in fade-in slide-in-from-bottom-8 duration-700">
            <div className="text-center mb-10 max-w-2xl mx-auto space-y-4">
              <h2 className="text-2xl font-bold text-gray-900">Get Your Coaching Framework</h2>
              <p className="text-gray-500 text-sm leading-relaxed font-medium">
                Enter employee context and receive milestone-specific questions for effective stay interviews.
              </p>
            </div>
            <ManagerForm config={config} setConfig={setConfig} onGenerate={handleGenerate} loading={loading} />
            {error && <div className="max-w-2xl mx-auto mt-6 p-4 bg-red-50 border border-red-200 text-red-600 rounded-xl text-sm font-bold flex items-center gap-2 animate-in slide-in-from-top-2"><AlertCircle size={18} /> {error}</div>}
          </div>
        ) : (
          <ResultViewer guide={guide} config={config} onReset={() => setGuide(null)} />
        )}
      </main>
      <footer className="max-w-6xl mx-auto pb-12 px-4 border-t border-gray-100 pt-8 flex justify-between items-center opacity-40 grayscale print:hidden">
        <div className="text-[10px] font-black uppercase tracking-widest text-blue-900">Thumbtack Manager Toolkit</div>
        <div className="text-[10px] font-bold text-gray-400">© 2025 THUMBTACK INTERNAL TOOL</div>
      </footer>
    </div>
  );
}
