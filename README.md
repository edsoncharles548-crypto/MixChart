<!DOCTYPE html>
<html lang="sw">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0" />
<title>Poa Chat — Ongea, Kila Mtu Aelewe</title>
<link rel="manifest" href="data:application/json;base64,eyJuYW1lIjoiUG9hIENoYXQiLCJzaG9ydF9uYW1lIjoiUG9hIENoYXQiLCJzdGFydF91cmwiOiIuIiwiZGlzcGxheSI6InN0YW5kYWxvbmUiLCJiYWNrZ3JvdW5kX2NvbG9yIjoiIzEyMDgyOCIsInRoZW1lX2NvbG9yIjoiIzZCNTNBMyJ9" />
<script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
<script src="https://cdn.tailwindcss.com"></script>
<style>
  :root {
    --tanzanite: #6B53A3;
    --tanzanite-dark: #4A3878;
    --tanzanite-light: #9A87C7;
    --gold: #D4A94C;
    --ink: #12141C;
    --paper: #F6F4F1;
  }
  * { -webkit-tap-highlight-color: transparent; }
  body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: var(--paper); overscroll-behavior-y: contain; }
  .bubble-sent { background: var(--tanzanite); color: white; border-radius: 18px 18px 4px 18px; }
  .bubble-recv { background: white; color: var(--ink); border-radius: 18px 18px 18px 4px; border: 1px solid #E8E4DE; }
  .mic-active { animation: pulse 1.2s infinite; }
  @keyframes pulse { 0%,100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(107,83,163,0.5); } 50% { transform: scale(1.08); box-shadow: 0 0 0 12px rgba(107,83,163,0); } }
  .scrollbar-none::-webkit-scrollbar { display: none; }
  .fade-in { animation: fadeIn 0.25s ease-out; }
  @keyframes fadeIn { from { opacity: 0; transform: translateY(6px); } to { opacity: 1; transform: translateY(0); } }
</style>
</head>
<body>
<div id="root"></div>

<script type="text/babel" data-presets="react">
const { useState, useEffect, useRef, useCallback } = React;

const SUPABASE_URL = "https://xypdslmizwcgmasruhti.supabase.co";
const SUPABASE_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Inh5cGRzbG1pendjZ21hc3J1aHRpIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODYwMDAyMDMsImV4cCI6MjEwMTU3NjIwM30.5hseQ0CWwhq-JZbG9qaBrQ0CsEXBYIvoIuVuQdPZWkQ";
const sb = window.supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

const LANGS = {
  sw: { label: "Kiswahili", flagText: "SW", speech: "sw-TZ" },
  en: { label: "English", flagText: "EN", speech: "en-US" },
  fr: { label: "Français", flagText: "FR", speech: "fr-FR" },
};

// ---------- Translation helper (MyMemory - free API) ----------
async function translateText(text, fromLang, toLang) {
  if (!text || fromLang === toLang) return text;
  try {
    const res = await fetch(
      `https://api.mymemory.translated.net/get?q=${encodeURIComponent(text)}&langpair=${fromLang}|${toLang}`
    );
    const data = await res.json();
    if (data && data.responseData && data.responseData.translatedText) {
      return data.responseData.translatedText;
    }
    return text;
  } catch (e) {
    console.error("Translation error", e);
    return text;
  }
}

function speak(text, langCode) {
  if (!("speechSynthesis" in window)) return;
  window.speechSynthesis.cancel();
  const utter = new SpeechSynthesisUtterance(text);
  utter.lang = (LANGS[langCode] || LANGS.en).speech;
  window.speechSynthesis.speak(utter);
}

function timeAgo(iso) {
  const d = new Date(iso);
  return d.toLocaleTimeString("sw-TZ", { hour: "2-digit", minute: "2-digit" });
}

// ---------- Auth Screen ----------
function AuthScreen({ onLogin }) {
  const [mode, setMode] = useState("login"); // login | signup
  const [name, setName] = useState("");
  const [pin, setPin] = useState("");
  const [lang, setLang] = useState("sw");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError("");
    if (!name.trim() || pin.length < 4) {
      setError("Jaza jina na PIN (angalau namba 4)");
      return;
    }
    setLoading(true);
    try {
      if (mode === "signup") {
        const { data, error } = await sb.rpc("signup_user", {
          p_name: name.trim(),
          p_pin: pin,
          p_language: lang,
        });
        setLoading(false);
        if (error) { setError("Hitilafu: " + error.message); return; }
        const row = data && data[0];
        if (!row || !row.success) { setError(row?.error_message || "Imeshindikana kujisajili"); return; }
        const user = { id: row.user_id, name: name.trim(), preferred_language: lang };
        localStorage.setItem("poachat_user", JSON.stringify(user));
        onLogin(user);
      } else {
        const { data, error } = await sb.rpc("verify_user_pin", {
          p_name: name.trim(),
          p_pin: pin,
        });
        setLoading(false);
        if (error) { setError("Hitilafu: " + error.message); return; }
        const row = data && data[0];
        if (!row || !row.valid) { setError("Jina au PIN si sahihi"); return; }
        const user = { id: row.user_id, name: row.name, preferred_language: row.preferred_language };
        localStorage.setItem("poachat_user", JSON.stringify(user));
        onLogin(user);
      }
    } catch (err) {
      setLoading(false);
      setError("Hitilafu ya mtandao. Jaribu tena.");
    }
  };

  return (
    <div className="min-h-screen flex flex-col justify-center px-6" style={{background: `linear-gradient(180deg, var(--tanzanite-dark), var(--ink))`}}>
      <div className="max-w-sm mx-auto w-full fade-in">
        <div className="text-center mb-8">
          <div className="inline-flex items-center justify-center w-16 h-16 rounded-2xl mb-4" style={{background: "var(--gold)"}}>
            <span className="text-2xl font-bold" style={{color: "var(--ink)"}}>PC</span>
          </div>
          <h1 className="text-2xl font-bold text-white">Poa Chat</h1>
          <p className="text-sm mt-1" style={{color: "var(--tanzanite-light)"}}>Ongea kwa lugha yako, wasikilizaji waelewe kwa yao.</p>
        </div>

        <div className="bg-white rounded-2xl p-5 shadow-xl">
          <div className="flex mb-5 rounded-xl overflow-hidden border" style={{borderColor: "#E8E4DE"}}>
            <button onClick={() => setMode("login")} className={`flex-1 py-2 text-sm font-semibold ${mode==="login" ? "text-white" : "text-gray-500"}`} style={mode==="login" ? {background:"var(--tanzanite)"} : {}}>Ingia</button>
            <button onClick={() => setMode("signup")} className={`flex-1 py-2 text-sm font-semibold ${mode==="signup" ? "text-white" : "text-gray-500"}`} style={mode==="signup" ? {background:"var(--tanzanite)"} : {}}>Jisajili</button>
          </div>

          <form onSubmit={handleSubmit} className="space-y-3">
            <div>
              <label className="text-xs font-medium text-gray-500">Jina</label>
              <input value={name} onChange={e=>setName(e.target.value)} placeholder="mfano: Ptronilla"
                className="w-full mt-1 px-3 py-2.5 rounded-xl border border-gray-200 focus:outline-none focus:ring-2" style={{"--tw-ring-color":"var(--tanzanite-light)"}} />
            </div>
            <div>
              <label className="text-xs font-medium text-gray-500">PIN (namba 4+)</label>
              <input value={pin} onChange={e=>setPin(e.target.value.replace(/\D/g,""))} type="password" inputMode="numeric" placeholder="••••"
                className="w-full mt-1 px-3 py-2.5 rounded-xl border border-gray-200 focus:outline-none focus:ring-2" style={{"--tw-ring-color":"var(--tanzanite-light)"}} />
            </div>

            {mode === "signup" && (
              <div>
                <label className="text-xs font-medium text-gray-500">Lugha yako</label>
                <div className="grid grid-cols-3 gap-2 mt-1">
                  {Object.entries(LANGS).map(([code, l]) => (
                    <button type="button" key={code} onClick={() => setLang(code)}
                      className={`py-2 rounded-xl text-sm font-semibold border ${lang===code ? "text-white" : "text-gray-600 border-gray-200"}`}
                      style={lang===code ? {background:"var(--tanzanite)", borderColor:"var(--tanzanite)"} : {}}>
                      {l.label}
                    </button>
                  ))}
                </div>
              </div>
            )}

            {error && <p className="text-red-500 text-sm">{error}</p>}

            <button type="submit" disabled={loading}
              className="w-full py-3 rounded-xl text-white font-semibold mt-2 disabled:opacity-60" style={{background:"var(--gold)", color:"var(--ink)"}}>
              {loading ? "Inasubiri..." : mode === "signup" ? "Jisajili" : "Ingia"}
            </button>
          </form>
        </div>
      </div>
    </div>
  );
}

// ---------- Chat List Screen ----------
function ChatListScreen({ user, onOpenChat, onLogout }) {
  const [conversations, setConversations] = useState([]);
  const [search, setSearch] = useState("");
  const [searchResults, setSearchResults] = useState([]);
  const [searching, setSearching] = useState(false);

  const [rows, setRows] = useState([]);
  const refresh = useCallback(async () => {
    const { data: convos } = await sb
      .from("conversations")
      .select("*")
      .or(`user1_id.eq.${user.id},user2_id.eq.${user.id}`)
      .order("created_at", { ascending: false });
    if (!convos) { setRows([]); return; }
    const enriched = await Promise.all(convos.map(async (c) => {
      const otherId = c.user1_id === user.id ? c.user2_id : c.user1_id;
      const { data: otherUser } = await sb.from("users").select("id,name,preferred_language").eq("id", otherId).single();
      const { data: lastMsg } = await sb.from("messages").select("*").eq("conversation_id", c.id).order("created_at", { ascending: false }).limit(1);
      return { conversation: c, other: otherUser, last: lastMsg && lastMsg[0] };
    }));
    setRows(enriched);
  }, [user.id]);

  useEffect(() => { refresh(); }, [refresh]);

  useEffect(() => {
    const channel = sb.channel("convo-list-" + user.id)
      .on("postgres_changes", { event: "INSERT", schema: "public", table: "messages" }, () => refresh())
      .on("postgres_changes", { event: "INSERT", schema: "public", table: "conversations" }, () => refresh())
      .subscribe();
    return () => sb.removeChannel(channel);
  }, [user.id, refresh]);

  const doSearch = async (q) => {
    setSearch(q);
    if (!q.trim()) { setSearchResults([]); return; }
    setSearching(true);
    const { data } = await sb.from("users").select("id,name,preferred_language").ilike("name", `%${q.trim()}%`).neq("id", user.id).limit(8);
    setSearchResults(data || []);
    setSearching(false);
  };

  const startChat = async (otherUser) => {
    const [a, b] = [user.id, otherUser.id].sort();
    let { data: existing } = await sb.from("conversations").select("*").eq("user1_id", a).eq("user2_id", b).maybeSingle();
    if (!existing) {
      const { data: created } = await sb.from("conversations").insert({ user1_id: a, user2_id: b }).select().single();
      existing = created;
    }
    setSearch(""); setSearchResults([]);
    onOpenChat(existing, otherUser);
  };

  return (
    <div className="min-h-screen flex flex-col" style={{background: "var(--paper)"}}>
      <div className="text-white px-4 pt-5 pb-4 rounded-b-2xl shadow-md" style={{background: "var(--tanzanite)"}}>
        <div className="flex justify-between items-center mb-3">
          <div>
            <h1 className="text-lg font-bold">Poa Chat</h1>
            <p className="text-xs opacity-80">Habari, {user.name} · {LANGS[user.preferred_language]?.label}</p>
          </div>
          <button onClick={onLogout} className="text-xs bg-white/20 px-3 py-1.5 rounded-full">Toka</button>
        </div>
        <input
          value={search}
          onChange={e => doSearch(e.target.value)}
          placeholder="Tafuta jina la mtu kuanza mazungumzo..."
          className="w-full px-3 py-2.5 rounded-xl text-sm text-gray-800 focus:outline-none"
        />
      </div>

      {search.trim() && (
        <div className="px-4 py-3 fade-in">
          <p className="text-xs text-gray-400 mb-2">{searching ? "Inatafuta..." : `Matokeo (${searchResults.length})`}</p>
          {searchResults.map(u => (
            <button key={u.id} onClick={() => startChat(u)} className="w-full flex items-center gap-3 bg-white p-3 rounded-xl mb-2 shadow-sm text-left">
              <div className="w-10 h-10 rounded-full flex items-center justify-center text-white font-bold" style={{background:"var(--tanzanite-light)"}}>{u.name[0]?.toUpperCase()}</div>
              <div>
                <p className="font-semibold text-sm">{u.name}</p>
                <p className="text-xs text-gray-400">{LANGS[u.preferred_language]?.label}</p>
              </div>
            </button>
          ))}
          {!searching && searchResults.length === 0 && <p className="text-sm text-gray-400">Hakuna mtu mwenye jina hilo.</p>}
        </div>
      )}

      {!search.trim() && (
        <div className="flex-1 overflow-y-auto px-4 py-3 scrollbar-none">
          {rows.length === 0 && (
            <div className="text-center mt-16 px-6">
              <p className="text-gray-400 text-sm">Bado hujaanza mazungumzo yoyote.</p>
              <p className="text-gray-400 text-xs mt-1">Tumia utafutaji juu kumpata rafiki yako.</p>
            </div>
          )}
          {rows.map(({ conversation, other, last }) => other && (
            <button key={conversation.id} onClick={() => onOpenChat(conversation, other)}
              className="w-full flex items-center gap-3 bg-white p-3 rounded-xl mb-2 shadow-sm text-left fade-in">
              <div className="w-11 h-11 rounded-full flex items-center justify-center text-white font-bold flex-shrink-0" style={{background:"var(--tanzanite)"}}>{other.name[0]?.toUpperCase()}</div>
              <div className="flex-1 min-w-0">
                <div className="flex justify-between items-baseline">
                  <p className="font-semibold text-sm truncate">{other.name}</p>
                  {last && <p className="text-[10px] text-gray-400 flex-shrink-0">{timeAgo(last.created_at)}</p>}
                </div>
                <p className="text-xs text-gray-400 truncate">
                  {last ? (last.msg_type === "voice" ? "🎤 Ujumbe wa sauti" : last.original_text) : "Anza mazungumzo"}
                </p>
              </div>
            </button>
          ))}
        </div>
      )}
    </div>
  );
}

// ---------- Chat Screen ----------
function ChatScreen({ user, conversation, other, onBack }) {
  const [messages, setMessages] = useState([]);
  const [text, setText] = useState("");
  const [recording, setRecording] = useState(false);
  const [showOriginal, setShowOriginal] = useState({});
  const [sending, setSending] = useState(false);
  const scrollRef = useRef(null);
  const recognitionRef = useRef(null);

  const myLang = user.preferred_language;
  const theirLang = other.preferred_language;

  const loadMessages = useCallback(async () => {
    const { data } = await sb.from("messages").select("*").eq("conversation_id", conversation.id).order("created_at", { ascending: true });
    setMessages(data || []);
  }, [conversation.id]);

  useEffect(() => { loadMessages(); }, [loadMessages]);

  useEffect(() => {
    const channel = sb.channel("chat-" + conversation.id)
      .on("postgres_changes", { event: "INSERT", schema: "public", table: "messages", filter: `conversation_id=eq.${conversation.id}` },
        (payload) => setMessages(prev => [...prev, payload.new]))
      .subscribe();
    return () => sb.removeChannel(channel);
  }, [conversation.id]);

  useEffect(() => {
    if (scrollRef.current) scrollRef.current.scrollTop = scrollRef.current.scrollHeight;
  }, [messages]);

  const sendMessage = async (content, msgType = "text") => {
    if (!content.trim()) return;
    setSending(true);
    const translated = await translateText(content, myLang, theirLang);
    const translations = {};
    translations[myLang] = content;
    translations[theirLang] = translated;
    await sb.from("messages").insert({
      conversation_id: conversation.id,
      sender_id: user.id,
      msg_type: msgType,
      original_text: content,
      original_language: myLang,
      translations,
    });
    setText("");
    setSending(false);
  };

  const startRecording = () => {
    const SpeechRec = window.SpeechRecognition || window.webkitSpeechRecognition;
    if (!SpeechRec) {
      alert("Simu/browser yako haiungi mkono kurekodi sauti. Tumia Chrome kwenye Android.");
      return;
    }
    const rec = new SpeechRec();
    rec.lang = LANGS[myLang].speech;
    rec.interimResults = false;
    rec.maxAlternatives = 1;
    rec.onresult = (e) => {
      const transcript = e.results[0][0].transcript;
      sendMessage(transcript, "voice");
    };
    rec.onerror = () => setRecording(false);
    rec.onend = () => setRecording(false);
    recognitionRef.current = rec;
    setRecording(true);
    rec.start();
  };

  const stopRecording = () => {
    if (recognitionRef.current) recognitionRef.current.stop();
    setRecording(false);
  };

  const toggleOriginal = (id) => setShowOriginal(prev => ({ ...prev, [id]: !prev[id] }));

  return (
    <div className="min-h-screen flex flex-col" style={{background: "var(--paper)"}}>
      <div className="text-white px-4 py-3 flex items-center gap-3 shadow-md flex-shrink-0" style={{background: "var(--tanzanite)"}}>
        <button onClick={onBack} className="text-xl">←</button>
        <div className="w-9 h-9 rounded-full flex items-center justify-center font-bold" style={{background:"var(--tanzanite-light)"}}>{other.name[0]?.toUpperCase()}</div>
        <div>
          <p className="font-semibold text-sm">{other.name}</p>
          <p className="text-[11px] opacity-80">{LANGS[theirLang]?.label} · wewe: {LANGS[myLang]?.label}</p>
        </div>
      </div>

      <div ref={scrollRef} className="flex-1 overflow-y-auto px-3 py-4 space-y-2 scrollbar-none">
        {messages.map((m) => {
          const isMine = m.sender_id === user.id;
          const mainText = (m.translations && m.translations[myLang]) || m.original_text;
          const original = m.original_text;
          const isShowingOriginal = showOriginal[m.id];
          return (
            <div key={m.id} className={`flex ${isMine ? "justify-end" : "justify-start"} fade-in`}>
              <div className={`max-w-[75%] px-3.5 py-2.5 ${isMine ? "bubble-sent" : "bubble-recv"}`}>
                {m.msg_type === "voice" && <p className="text-[10px] opacity-70 mb-0.5">🎤 Ujumbe wa sauti</p>}
                <p className="text-sm leading-snug">{isShowingOriginal ? original : mainText}</p>
                {m.original_language !== myLang && (
                  <button onClick={() => toggleOriginal(m.id)} className={`text-[10px] mt-1 underline opacity-70`}>
                    {isShowingOriginal ? `Onyesha ${LANGS[myLang]?.label}` : `Ona ${LANGS[m.original_language]?.label} halisi`}
                  </button>
                )}
                <div className="flex items-center justify-between mt-1">
                  <span className="text-[9px] opacity-60">{timeAgo(m.created_at)}</span>
                  <button onClick={() => speak(isShowingOriginal ? original : mainText, isShowingOriginal ? m.original_language : myLang)} className="text-[11px] opacity-70 ml-2">🔊</button>
                </div>
              </div>
            </div>
          );
        })}
        {messages.length === 0 && (
          <p className="text-center text-gray-400 text-sm mt-10">Tuma ujumbe wa kwanza kwa {other.name} 👋</p>
        )}
      </div>

      <div className="flex items-center gap-2 p-3 bg-white border-t border-gray-100 flex-shrink-0">
        <input
          value={text}
          onChange={e => setText(e.target.value)}
          onKeyDown={e => e.key === "Enter" && sendMessage(text)}
          placeholder={`Andika kwa ${LANGS[myLang]?.label}...`}
          className="flex-1 px-4 py-2.5 rounded-full border border-gray-200 text-sm focus:outline-none"
        />
        <button
          onClick={recording ? stopRecording : startRecording}
          className={`w-11 h-11 rounded-full flex items-center justify-center text-white flex-shrink-0 ${recording ? "mic-active" : ""}`}
          style={{background: recording ? "#D64545" : "var(--gold)", color: "var(--ink)"}}>
          {recording ? "⏹" : "🎤"}
        </button>
        <button
          onClick={() => sendMessage(text)}
          disabled={sending || !text.trim()}
          className="w-11 h-11 rounded-full flex items-center justify-center text-white flex-shrink-0 disabled:opacity-40"
          style={{background: "var(--tanzanite)"}}>
          ➤
        </button>
      </div>
    </div>
  );
}

// ---------- Root App ----------
function App() {
  const [user, setUser] = useState(() => {
    try { return JSON.parse(localStorage.getItem("poachat_user")); } catch { return null; }
  });
  const [activeChat, setActiveChat] = useState(null); // { conversation, other }

  const handleLogout = () => {
    localStorage.removeItem("poachat_user");
    setUser(null);
    setActiveChat(null);
  };

  if (!user) return <AuthScreen onLogin={setUser} />;

  if (activeChat) {
    return <ChatScreen user={user} conversation={activeChat.conversation} other={activeChat.other} onBack={() => setActiveChat(null)} />;
  }

  return <ChatListScreen user={user} onOpenChat={(conversation, other) => setActiveChat({ conversation, other })} onLogout={handleLogout} />;
}

ReactDOM.createRoot(document.getElementById("root")).render(<App />);
</script>
</body>
</html>
