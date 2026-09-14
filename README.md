import React, { useState, useMemo, useRef, useEffect } from "react";
import {
  BarChart, Bar, LineChart, Line, PieChart, Pie, Cell,
  XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, Legend
} from "recharts";
import {
  BookOpen, MessageSquare, Brain, FileText, Calendar, LayoutDashboard,
  Info, Sun, Moon, Menu, X, Send, CheckCircle2, XCircle, Sparkles,
  Clock, Target, Flame, TrendingUp, ChevronRight, Plus, Trash2
} from "lucide-react";

/* ============================================================
   THEME TOKENS
   ============================================================ */
const palette = {
  light: {
    bg: "#FAFAF7",
    surface: "#FFFFFF",
    surfaceAlt: "#F1F0EA",
    ink: "#14131F",
    inkSoft: "#5B5A66",
    border: "#E4E2D9",
    primary: "#2954A6",
    primarySoft: "#E7EEFA",
    accent: "#F2B705",
    accentSoft: "#FDF3D2",
    success: "#3E8E5B",
    danger: "#C1443C",
  },
  dark: {
    bg: "#0F1016",
    surface: "#171825",
    surfaceAlt: "#1E202F",
    ink: "#F3F2EC",
    inkSoft: "#A6A5B4",
    border: "#2C2E40",
    primary: "#6D9BEB",
    primarySoft: "#1C2A45",
    accent: "#F2B705",
    accentSoft: "#3A2F0C",
    success: "#66C38C",
    danger: "#E27168",
  },
};

/* ============================================================
   MOCK / LOCAL "AI" ENGINE
   In production, swap the body of callAI() for a real fetch()
   to Claude / OpenAI. Everything downstream stays identical —
   that is the intended integration seam for AI/ML.
   ============================================================ */
const STOPWORDS = new Set("a an the is are was were be been being and or but if then else for of to in on at by with as it this that these those from into over under about your you i we he she they them his her their our not no do does did can could should would will shall may might must than so such very just also more most other some any each which who whom what when where why how".split(" "));

function extractiveSummary(text, maxSentences = 3, maxKeywords = 8) {
  const clean = text.replace(/\s+/g, " ").trim();
  if (!clean) return { summary: "", points: [], keywords: [] };
  const sentences = clean.match(/[^.!?]+[.!?]+|[^.!?]+$/g)?.map(s => s.trim()).filter(Boolean) || [clean];
  const freq = {};
  const words = clean.toLowerCase().match(/[a-z]{3,}/g) || [];
  words.forEach(w => { if (!STOPWORDS.has(w)) freq[w] = (freq[w] || 0) + 1; });

  const scored = sentences.map((s, idx) => {
    const sWords = s.toLowerCase().match(/[a-z]{3,}/g) || [];
    const score = sWords.reduce((acc, w) => acc + (freq[w] || 0), 0) / Math.sqrt(sWords.length || 1);
    return { s, idx, score };
  });

  const top = [...scored].sort((a, b) => b.score - a.score).slice(0, Math.min(maxSentences, sentences.length));
  const summary = top.sort((a, b) => a.idx - b.idx).map(t => t.s).join(" ");

  const pointsPool = [...scored].sort((a, b) => b.score - a.score).slice(0, Math.min(5, sentences.length));
  const points = pointsPool.sort((a, b) => a.idx - b.idx).map(t => t.s);

  const keywords = Object.entries(freq)
    .sort((a, b) => b[1] - a[1])
    .slice(0, maxKeywords)
    .map(([w]) => w);

  return { summary: summary || sentences[0], points, keywords };
}

function generateQuiz(topic, difficulty, count = 5) {
  const t = topic.trim() || "this topic";
  const difficultyTemplates = {
    Easy: [
      `What is the basic definition of ${t}?`,
      `Which of these is a core building block of ${t}?`,
      `${t} is most commonly used in which context?`,
      `Which statement about ${t} is true?`,
      `Which term is closely related to ${t}?`,
    ],
    Medium: [
      `Which of the following best explains how ${t} works?`,
      `In ${t}, what happens when a key assumption is violated?`,
      `Which scenario best demonstrates an application of ${t}?`,
      `What is a common limitation associated with ${t}?`,
      `Which factor most influences performance in ${t}?`,
    ],
    Hard: [
      `Which trade-off is most critical when applying ${t} at scale?`,
      `Given an edge case in ${t}, which approach is most robust?`,
      `Which advanced concept extends the core idea of ${t}?`,
      `How would you optimize a system built around ${t}?`,
      `Which statement correctly compares two approaches within ${t}?`,
    ],
  };
  const templates = difficultyTemplates[difficulty] || difficultyTemplates.Medium;
  const optionSets = [
    ["Option A", "Option B", "Option C", "Option D"],
  ];
  return Array.from({ length: count }).map((_, i) => {
    const q = templates[i % templates.length];
    const correctIdx = (i * 2 + 1) % 4;
    const labels = [
      `A foundational concept directly tied to ${t}`,
      `A concept unrelated to ${t}`,
      `A partially related idea, often confused with ${t}`,
      `An outdated approach no longer used in ${t}`,
    ];
    // rotate so correct answer position varies
    const rotated = [...labels.slice(correctIdx), ...labels.slice(0, correctIdx)];
    return {
      id: i,
      question: q,
      options: rotated,
      correctIndex: rotated.indexOf(labels[0]),
    };
  });
}

function generateStudyPlan(subjects, hoursPerDay, days) {
  const subs = subjects.split(",").map(s => s.trim()).filter(Boolean);
  if (subs.length === 0) return [];
  const slotsPerDay = Math.max(1, Math.round(hoursPerDay));
  const plan = [];
  let subjIdx = 0;
  for (let d = 1; d <= days; d++) {
    const daySlots = [];
    for (let s = 0; s < slotsPerDay; s++) {
      daySlots.push({
        subject: subs[subjIdx % subs.length],
        time: `${9 + s * 2}:00 - ${10 + s * 2}:00`,
      });
      subjIdx++;
    }
    plan.push({ day: d, slots: daySlots });
  }
  return plan;
}

function assistantReply(question) {
  const q = question.trim();
  const topic = q.replace(/^(what is|explain|how does|why|define)\s+/i, "").replace(/\?$/, "");
  return {
    text: `Here's a clear breakdown of "${topic}": at its core, this concept builds on a few underlying principles that connect to concrete examples you'd encounter in coursework. Think of it in three layers — the basic definition, the mechanism behind it, and a real-world example that ties it together. Once those three layers make sense, most exam-style questions on this topic become straightforward pattern matching.`,
    followUps: [
      `Can you give a real-world example of ${topic}?`,
      `What are common mistakes students make with ${topic}?`,
      `How does ${topic} connect to what I studied earlier?`,
    ],
  };
}

/* ============================================================
   DEMO DATA (for Dashboard)
   ============================================================ */
const demoScores = [
  { name: "W1", score: 62 }, { name: "W2", score: 68 }, { name: "W3", score: 74 },
  { name: "W4", score: 71 }, { name: "W5", score: 83 }, { name: "W6", score: 89 },
];
const demoStudyTime = [
  { day: "Mon", hrs: 1.5 }, { day: "Tue", hrs: 2 }, { day: "Wed", hrs: 1 },
  { day: "Thu", hrs: 2.5 }, { day: "Fri", hrs: 1.5 }, { day: "Sat", hrs: 3 }, { day: "Sun", hrs: 2 },
];
const demoTopics = [
  { name: "Completed", value: 18 }, { name: "In progress", value: 6 }, { name: "Not started", value: 4 },
];
const demoActivity = [
  { text: "Scored 89% on Machine Learning quiz", time: "2h ago" },
  { text: "Summarized notes on Neural Networks", time: "5h ago" },
  { text: "Completed study plan for Data Structures", time: "1d ago" },
  { text: "Asked AI Assistant about Gradient Descent", time: "1d ago" },
];

/* ============================================================
   SMALL UI PRIMITIVES
   ============================================================ */
const Section = ({ children, className = "" }) => (
  <section className={`max-w-6xl mx-auto px-6 md:px-10 ${className}`}>{children}</section>
);

function useTheme() {
  const [mode, setMode] = useState("light");
  const c = palette[mode];
  return { mode, setMode, c };
}

/* ============================================================
   PAGES
   ============================================================ */
function Landing({ c, go }) {
  return (
    <div>
      <Section className="pt-16 pb-20 grid md:grid-cols-2 gap-12 items-center">
        <div>
          <p style={{ color: c.primary }} className="font-medium mb-3 tracking-tight">AI StudyMate</p>
          <h1 className="font-serif text-4xl md:text-5xl leading-tight mb-5" style={{ color: c.ink }}>
            A study desk that explains, quizzes, and plans — so you don't have to do it alone.
          </h1>
          <p className="text-lg mb-8 leading-relaxed" style={{ color: c.inkSoft }}>
            AI StudyMate combines a study assistant, quiz generator, notes summarizer and
            planner into one workspace, built as a BCA final-year project exploring practical
            applications of AI/ML in education.
          </p>
          <div className="flex flex-wrap gap-3">
            <button onClick={() => go("assistant")} className="px-6 py-3 rounded-md font-medium transition"
              style={{ background: c.primary, color: "#fff" }}>
              Try the Study Assistant
            </button>
            <button onClick={() => go("about")} className="px-6 py-3 rounded-md font-medium border transition"
              style={{ borderColor: c.border, color: c.ink }}>
              Read the project brief
            </button>
          </div>
        </div>

        {/* Characteristic hero element: a live mock chat, since chat is the core feature */}
        <div className="rounded-xl border overflow-hidden" style={{ borderColor: c.border, background: c.surface }}>
          <div className="px-4 py-3 border-b flex items-center gap-2" style={{ borderColor: c.border }}>
            <MessageSquare size={16} style={{ color: c.primary }} />
            <span className="text-sm font-medium" style={{ color: c.ink }}>Study Assistant</span>
          </div>
          <div className="p-4 space-y-3 text-sm">
            <div className="ml-auto max-w-[80%] rounded-lg px-3 py-2" style={{ background: c.primarySoft, color: c.ink }}>
              What is gradient descent?
            </div>
            <div className="max-w-[85%] rounded-lg px-3 py-2" style={{ background: c.surfaceAlt, color: c.ink }}>
              It's an optimization method that nudges model parameters in the direction that
              most reduces error, one small step at a time — like walking downhill in fog,
              feeling for the steepest slope underfoot.
            </div>
            <div className="flex flex-wrap gap-2 pt-1">
              {["Show a real example", "How is it used in neural nets?"].map((f, i) => (
                <span key={i} className="text-xs px-2.5 py-1 rounded-full border" style={{ borderColor: c.border, color: c.inkSoft }}>{f}</span>
              ))}
            </div>
          </div>
        </div>
      </Section>

      {/* How it works */}
      <Section className="py-16">
        <h2 className="font-serif text-2xl mb-8" style={{ color: c.ink }}>How it works</h2>
        <div className="grid md:grid-cols-4 gap-6">
          {[
            { icon: MessageSquare, title: "Ask", desc: "Ask a question or paste a topic into the Study Assistant." },
            { icon: Brain, title: "Understand", desc: "Get a layered explanation, plus follow-up questions to go deeper." },
            { icon: FileText, title: "Practice", desc: "Generate a quiz or summarize notes to reinforce the concept." },
            { icon: Calendar, title: "Plan", desc: "Fit it into a study schedule the planner builds around your time." },
          ].map((step, i) => (
            <div key={i} className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
              <step.icon size={20} style={{ color: c.primary }} className="mb-3" />
              <p className="font-medium mb-1" style={{ color: c.ink }}>{step.title}</p>
              <p className="text-sm" style={{ color: c.inkSoft }}>{step.desc}</p>
            </div>
          ))}
        </div>
      </Section>

      {/* Features */}
      <Section className="py-16">
        <h2 className="font-serif text-2xl mb-8" style={{ color: c.ink }}>Everything in one workspace</h2>
        <div className="grid md:grid-cols-3 gap-6">
          {[
            { icon: MessageSquare, title: "AI Study Assistant", desc: "Chat-style explanations with suggested follow-ups.", page: "assistant" },
            { icon: Brain, title: "Quiz Generator", desc: "Topic + difficulty in, scored MCQ quiz out.", page: "quiz" },
            { icon: FileText, title: "Notes Summarizer", desc: "Paste notes, get a summary, key points and keywords.", page: "notes" },
            { icon: Calendar, title: "Study Planner", desc: "Turns subjects and available hours into a schedule.", page: "planner" },
            { icon: LayoutDashboard, title: "Dashboard", desc: "Track quiz scores, study time and topic progress.", page: "dashboard" },
            { icon: Brain, title: "AI/ML Explained", desc: "The concepts and models behind the project.", page: "aiml" },
          ].map((f, i) => (
            <button key={i} onClick={() => go(f.page)} className="text-left p-5 rounded-lg border transition hover:shadow-sm"
              style={{ borderColor: c.border, background: c.surface }}>
              <f.icon size={20} style={{ color: c.accent }} className="mb-3" />
              <p className="font-medium mb-1" style={{ color: c.ink }}>{f.title}</p>
              <p className="text-sm" style={{ color: c.inkSoft }}>{f.desc}</p>
            </button>
          ))}
        </div>
      </Section>

      {/* Benefits + CTA */}
      <Section className="py-16">
        <div className="rounded-xl p-8 md:p-10 grid md:grid-cols-3 gap-8" style={{ background: c.surfaceAlt }}>
          <div className="md:col-span-2">
            <h3 className="font-serif text-2xl mb-3" style={{ color: c.ink }}>Built to reduce the gap between studying and understanding</h3>
            <p style={{ color: c.inkSoft }} className="leading-relaxed">
              Most students juggle separate tools for notes, quizzes and planning. AI StudyMate
              keeps that loop in one place, so revising a topic and testing yourself on it happen
              back to back instead of across five different apps.
            </p>
          </div>
          <div className="flex items-center">
            <button onClick={() => go("assistant")} className="w-full px-6 py-3 rounded-md font-medium"
              style={{ background: c.accent, color: "#14131F" }}>
              Start studying now
            </button>
          </div>
        </div>
      </Section>
    </div>
  );
}

function Assistant({ c }) {
  const [messages, setMessages] = useState([
    { role: "assistant", text: "Ask me to explain any topic — I'll break it down and suggest what to explore next.", followUps: [] }
  ]);
  const [input, setInput] = useState("");
  const endRef = useRef(null);

  useEffect(() => { endRef.current?.scrollIntoView({ behavior: "smooth" }); }, [messages]);

  const ask = (text) => {
    const q = text ?? input;
    if (!q.trim()) return;
    const reply = assistantReply(q);
    setMessages(m => [...m, { role: "user", text: q }, { role: "assistant", text: reply.text, followUps: reply.followUps }]);
    setInput("");
  };

  return (
    <Section className="py-12">
      <h1 className="font-serif text-3xl mb-2" style={{ color: c.ink }}>AI Study Assistant</h1>
      <p className="mb-8" style={{ color: c.inkSoft }}>Ask a question or paste a topic to get an explanation.</p>

      <div className="rounded-xl border flex flex-col" style={{ borderColor: c.border, background: c.surface, height: "60vh" }}>
        <div className="flex-1 overflow-y-auto p-5 space-y-4">
          {messages.map((m, i) => (
            <div key={i} className={m.role === "user" ? "flex justify-end" : "flex justify-start"}>
              <div className="max-w-[80%]">
                <div className="rounded-lg px-4 py-2.5 text-sm leading-relaxed"
                  style={{ background: m.role === "user" ? c.primarySoft : c.surfaceAlt, color: c.ink }}>
                  {m.text}
                </div>
                {m.followUps?.length > 0 && (
                  <div className="flex flex-wrap gap-2 mt-2">
                    {m.followUps.map((f, j) => (
                      <button key={j} onClick={() => ask(f)} className="text-xs px-2.5 py-1 rounded-full border transition hover:opacity-80"
                        style={{ borderColor: c.border, color: c.inkSoft }}>
                        {f}
                      </button>
                    ))}
                  </div>
                )}
              </div>
            </div>
          ))}
          <div ref={endRef} />
        </div>
        <div className="p-3 border-t flex gap-2" style={{ borderColor: c.border }}>
          <input value={input} onChange={e => setInput(e.target.value)}
            onKeyDown={e => e.key === "Enter" && ask()}
            placeholder="e.g. What is supervised learning?"
            className="flex-1 px-4 py-2.5 rounded-md border outline-none text-sm"
            style={{ borderColor: c.border, background: c.bg, color: c.ink }} />
          <button onClick={() => ask()} className="px-4 py-2.5 rounded-md" style={{ background: c.primary, color: "#fff" }}>
            <Send size={16} />
          </button>
        </div>
      </div>
    </Section>
  );
}

function Quiz({ c }) {
  const [topic, setTopic] = useState("");
  const [difficulty, setDifficulty] = useState("Medium");
  const [quiz, setQuiz] = useState(null);
  const [answers, setAnswers] = useState({});
  const [submitted, setSubmitted] = useState(false);

  const start = () => {
    setQuiz(generateQuiz(topic || "Data Structures", difficulty));
    setAnswers({});
    setSubmitted(false);
  };

  const score = useMemo(() => {
    if (!quiz) return 0;
    return quiz.filter(q => answers[q.id] === q.correctIndex).length;
  }, [quiz, answers, submitted]);

  return (
    <Section className="py-12">
      <h1 className="font-serif text-3xl mb-2" style={{ color: c.ink }}>AI Quiz Generator</h1>
      <p className="mb-8" style={{ color: c.inkSoft }}>Enter a subject and difficulty to generate a scored quiz.</p>

      <div className="flex flex-wrap gap-3 mb-8">
        <input value={topic} onChange={e => setTopic(e.target.value)} placeholder="Subject / topic (e.g. Operating Systems)"
          className="flex-1 min-w-[220px] px-4 py-2.5 rounded-md border text-sm outline-none"
          style={{ borderColor: c.border, background: c.surface, color: c.ink }} />
        <select value={difficulty} onChange={e => setDifficulty(e.target.value)}
          className="px-4 py-2.5 rounded-md border text-sm outline-none" style={{ borderColor: c.border, background: c.surface, color: c.ink }}>
          <option>Easy</option><option>Medium</option><option>Hard</option>
        </select>
        <button onClick={start} className="px-5 py-2.5 rounded-md font-medium" style={{ background: c.primary, color: "#fff" }}>
          Generate quiz
        </button>
      </div>

      {quiz && (
        <div className="space-y-5">
          {quiz.map((q, i) => (
            <div key={q.id} className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
              <p className="font-medium mb-3" style={{ color: c.ink }}>{i + 1}. {q.question}</p>
              <div className="grid sm:grid-cols-2 gap-2">
                {q.options.map((opt, idx) => {
                  const chosen = answers[q.id] === idx;
                  const isCorrect = submitted && idx === q.correctIndex;
                  const isWrongChoice = submitted && chosen && idx !== q.correctIndex;
                  return (
                    <button key={idx} disabled={submitted}
                      onClick={() => setAnswers(a => ({ ...a, [q.id]: idx }))}
                      className="text-left px-3 py-2 rounded-md border text-sm flex items-center justify-between gap-2"
                      style={{
                        borderColor: isCorrect ? c.success : isWrongChoice ? c.danger : chosen ? c.primary : c.border,
                        background: isCorrect ? c.accentSoft : chosen ? c.primarySoft : "transparent",
                        color: c.ink,
                      }}>
                      <span>{opt}</span>
                      {isCorrect && <CheckCircle2 size={16} style={{ color: c.success }} />}
                      {isWrongChoice && <XCircle size={16} style={{ color: c.danger }} />}
                    </button>
                  );
                })}
              </div>
            </div>
          ))}

          {!submitted ? (
            <button onClick={() => setSubmitted(true)} className="px-6 py-3 rounded-md font-medium" style={{ background: c.accent, color: "#14131F" }}>
              Submit quiz
            </button>
          ) : (
            <div className="p-5 rounded-lg border font-medium" style={{ borderColor: c.border, background: c.surfaceAlt, color: c.ink }}>
              Score: {score} / {quiz.length} ({Math.round((score / quiz.length) * 100)}%)
            </div>
          )}
        </div>
      )}
    </Section>
  );
}

function Notes({ c }) {
  const [text, setText] = useState("");
  const [result, setResult] = useState(null);

  return (
    <Section className="py-12">
      <h1 className="font-serif text-3xl mb-2" style={{ color: c.ink }}>AI Notes Summarizer</h1>
      <p className="mb-8" style={{ color: c.inkSoft }}>
        Paste study material below — this runs real extractive summarization (word-frequency scoring) in your browser.
      </p>

      <textarea value={text} onChange={e => setText(e.target.value)} rows={8}
        placeholder="Paste a paragraph or two of study notes here..."
        className="w-full px-4 py-3 rounded-md border text-sm outline-none mb-3"
        style={{ borderColor: c.border, background: c.surface, color: c.ink }} />
      <button onClick={() => setResult(extractiveSummary(text))} className="px-5 py-2.5 rounded-md font-medium mb-8"
        style={{ background: c.primary, color: "#fff" }}>
        Summarize
      </button>

      {result && (
        <div className="grid md:grid-cols-2 gap-6">
          <div className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
            <p className="font-medium mb-2" style={{ color: c.ink }}>Summary</p>
            <p className="text-sm leading-relaxed" style={{ color: c.inkSoft }}>{result.summary || "Add some text above to generate a summary."}</p>
          </div>
          <div className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
            <p className="font-medium mb-2" style={{ color: c.ink }}>Important points</p>
            <ul className="text-sm space-y-1.5 list-disc pl-4" style={{ color: c.inkSoft }}>
              {result.points.map((p, i) => <li key={i}>{p}</li>)}
            </ul>
          </div>
          <div className="p-5 rounded-lg border md:col-span-2" style={{ borderColor: c.border, background: c.surface }}>
            <p className="font-medium mb-3" style={{ color: c.ink }}>Keywords</p>
            <div className="flex flex-wrap gap-2">
              {result.keywords.map((k, i) => (
                <span key={i} className="text-xs px-2.5 py-1 rounded-full" style={{ background: c.accentSoft, color: c.ink }}>{k}</span>
              ))}
            </div>
          </div>
        </div>
      )}
    </Section>
  );
}

function Planner({ c }) {
  const [subjects, setSubjects] = useState("Data Structures, DBMS, Machine Learning");
  const [hours, setHours] = useState(3);
  const [days, setDays] = useState(5);
  const [plan, setPlan] = useState(null);
  const [activeDay, setActiveDay] = useState(0);

  const build = () => {
    const p = generateStudyPlan(subjects, hours, days);
    setPlan(p);
    setActiveDay(0);
  };

  return (
    <Section className="py-12">
      <h1 className="font-serif text-3xl mb-2" style={{ color: c.ink }}>Smart Study Planner</h1>
      <p className="mb-8" style={{ color: c.inkSoft }}>List your subjects and available time to generate a day-by-day schedule.</p>

      <div className="grid md:grid-cols-3 gap-3 mb-8">
        <input value={subjects} onChange={e => setSubjects(e.target.value)} placeholder="Subjects, comma separated"
          className="md:col-span-1 px-4 py-2.5 rounded-md border text-sm outline-none" style={{ borderColor: c.border, background: c.surface, color: c.ink }} />
        <div className="flex items-center gap-2">
          <label className="text-sm shrink-0" style={{ color: c.inkSoft }}>Hours/day</label>
          <input type="number" min={1} max={10} value={hours} onChange={e => setHours(Number(e.target.value))}
            className="w-full px-4 py-2.5 rounded-md border text-sm outline-none" style={{ borderColor: c.border, background: c.surface, color: c.ink }} />
        </div>
        <div className="flex items-center gap-2">
          <label className="text-sm shrink-0" style={{ color: c.inkSoft }}>Days</label>
          <input type="number" min={1} max={14} value={days} onChange={e => setDays(Number(e.target.value))}
            className="w-full px-4 py-2.5 rounded-md border text-sm outline-none" style={{ borderColor: c.border, background: c.surface, color: c.ink }} />
        </div>
      </div>
      <button onClick={build} className="px-5 py-2.5 rounded-md font-medium mb-8" style={{ background: c.primary, color: "#fff" }}>
        Generate schedule
      </button>

      {plan && (
        <div>
          <div className="flex gap-2 overflow-x-auto mb-5 pb-1">
            {plan.map((d, i) => (
              <button key={i} onClick={() => setActiveDay(i)} className="px-4 py-2 rounded-md text-sm whitespace-nowrap border"
                style={{
                  background: activeDay === i ? c.primary : "transparent",
                  color: activeDay === i ? "#fff" : c.ink,
                  borderColor: c.border,
                }}>
                Day {d.day}
              </button>
            ))}
          </div>
          <div className="space-y-3">
            {plan[activeDay].slots.map((s, i) => (
              <div key={i} className="flex items-center gap-4 p-4 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
                <Clock size={16} style={{ color: c.primary }} />
                <span className="text-sm font-medium w-32 shrink-0" style={{ color: c.ink }}>{s.time}</span>
                <span className="text-sm" style={{ color: c.inkSoft }}>{s.subject}</span>
              </div>
            ))}
          </div>
        </div>
      )}
    </Section>
  );
}

function Dashboard({ c }) {
  const COLORS = [c.primary, c.accent, c.border];
  const stats = [
    { label: "Study hours (7d)", value: "13.5", icon: Clock },
    { label: "Avg quiz score", value: "78%", icon: Target },
    { label: "Topics completed", value: "18", icon: CheckCircle2 },
    { label: "Study streak", value: "6 days", icon: Flame },
  ];
  return (
    <Section className="py-12">
      <h1 className="font-serif text-3xl mb-2" style={{ color: c.ink }}>Student Dashboard</h1>
      <p className="mb-8" style={{ color: c.inkSoft }}>Demo data shown below for presentation purposes.</p>

      <div className="grid sm:grid-cols-2 md:grid-cols-4 gap-4 mb-8">
        {stats.map((s, i) => (
          <div key={i} className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
            <s.icon size={18} style={{ color: c.primary }} className="mb-3" />
            <p className="text-2xl font-semibold" style={{ color: c.ink }}>{s.value}</p>
            <p className="text-xs mt-1" style={{ color: c.inkSoft }}>{s.label}</p>
          </div>
        ))}
      </div>

      <div className="grid md:grid-cols-2 gap-6 mb-6">
        <div className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
          <p className="font-medium mb-4" style={{ color: c.ink }}>Quiz scores over time</p>
          <ResponsiveContainer width="100%" height={220}>
            <LineChart data={demoScores}>
              <CartesianGrid strokeDasharray="3 3" stroke={c.border} />
              <XAxis dataKey="name" stroke={c.inkSoft} fontSize={12} />
              <YAxis stroke={c.inkSoft} fontSize={12} />
              <Tooltip contentStyle={{ background: c.surface, border: `1px solid ${c.border}` }} />
              <Line type="monotone" dataKey="score" stroke={c.primary} strokeWidth={2} dot={{ r: 3 }} />
            </LineChart>
          </ResponsiveContainer>
        </div>
        <div className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
          <p className="font-medium mb-4" style={{ color: c.ink }}>Study time this week</p>
          <ResponsiveContainer width="100%" height={220}>
            <BarChart data={demoStudyTime}>
              <CartesianGrid strokeDasharray="3 3" stroke={c.border} />
              <XAxis dataKey="day" stroke={c.inkSoft} fontSize={12} />
              <YAxis stroke={c.inkSoft} fontSize={12} />
              <Tooltip contentStyle={{ background: c.surface, border: `1px solid ${c.border}` }} />
              <Bar dataKey="hrs" fill={c.accent} radius={[4, 4, 0, 0]} />
            </BarChart>
          </ResponsiveContainer>
        </div>
      </div>

      <div className="grid md:grid-cols-2 gap-6">
        <div className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
          <p className="font-medium mb-4" style={{ color: c.ink }}>Topic progress</p>
          <ResponsiveContainer width="100%" height={220}>
            <PieChart>
              <Pie data={demoTopics} dataKey="value" nameKey="name" innerRadius={50} outerRadius={80} paddingAngle={3}>
                {demoTopics.map((_, i) => <Cell key={i} fill={COLORS[i % COLORS.length]} />)}
              </Pie>
              <Legend verticalAlign="bottom" height={30} wrapperStyle={{ fontSize: 12, color: c.inkSoft }} />
              <Tooltip contentStyle={{ background: c.surface, border: `1px solid ${c.border}` }} />
            </PieChart>
          </ResponsiveContainer>
        </div>
        <div className="p-5 rounded-lg border" style={{ borderColor: c.border, background: c.surface }}>
          <p className="font-medium mb-4" style={{ color: c.ink }}>Recent activity</p>
          <div className="space-y-3">
            {demoActivity.map((a, i) => (
              <div key={i} className="flex items-start gap-3 text-sm">
                <TrendingUp size={14} style={{ color: c.primary }} className="mt-1 shrink-0" />
                <div>
                  <p style={{ color: c.ink }}>{a.text}</p>
                  <p className="text-xs" style={{ color: c.inkSoft }}>{a.time}</p>
                </div>
              </div>
            ))}
          </div>
        </div>
      </div>
    </Section>
  );
}

function AIML({ c }) {
  const block = (title, children) => (
    <div className="mb-10">
      <h2 className="font-serif text-2xl mb-3" style={{ color: c.ink }}>{title}</h2>
      <div className="text-sm leading-relaxed space-y-2" style={{ color: c.inkSoft }}>{children}</div>
    </div>
  );
  return (
    <Section className="py-12 max-w-3xl">
      <h1 className="font-serif text-3xl mb-8" style={{ color: c.ink }}>AI & Machine Learning in this project</h1>

      {block("What is AI?", <p>Artificial Intelligence is the broad field of building systems that perform tasks which normally need human judgment — understanding language, recognizing patterns, or making decisions from data.</p>)}

      {block("What is Machine Learning?", <p>Machine Learning is a subset of AI where a system improves at a task by learning patterns from data, rather than following hand-written rules for every case.</p>)}

      {block("How this project uses AI/ML concepts", <ul className="list-disc pl-5 space-y-1">
        <li>The Notes Summarizer uses <strong>extractive summarization</strong>: word-frequency scoring ranks sentences by how much "informative" vocabulary they contain, then the top-ranked sentences form the summary — a classic NLP technique.</li>
        <li>The Quiz Generator and Study Assistant currently use structured templates. They're built around a single <code>callAI()</code> integration point, so swapping in a real large language model (Claude/GPT) later requires no architectural change.</li>
        <li>The Dashboard is designed to eventually feed real usage data into a simple classifier that flags weak topics based on quiz performance.</li>
      </ul>)}

      {block("Possible ML models & algorithms for future versions", <ul className="list-disc pl-5 space-y-1">
        <li>TF-IDF / embeddings-based summarization for more accurate notes summaries</li>
        <li>A transformer-based LLM (via API) for open-ended question answering</li>
        <li>A classification model (e.g. logistic regression or a small neural net) to predict weak topics from quiz history</li>
        <li>Collaborative filtering to recommend what to study next based on similar students' patterns</li>
      </ul>)}

      {block("Future scope", <ul className="list-disc pl-5 space-y-1">
        <li>Connect a real LLM API for the Assistant and Quiz Generator</li>
        <li>Add a backend + database to persist real quiz scores and study history</li>
        <li>Personalized difficulty adjustment based on past performance</li>
        <li>Voice-based question input</li>
      </ul>)}
    </Section>
  );
}

function About({ c }) {
  const block = (title, children) => (
    <div className="mb-8">
      <h2 className="font-medium text-lg mb-2" style={{ color: c.ink }}>{title}</h2>
      <div className="text-sm leading-relaxed" style={{ color: c.inkSoft }}>{children}</div>
    </div>
  );
  return (
    <Section className="py-12 max-w-3xl">
      <h1 className="font-serif text-3xl mb-8" style={{ color: c.ink }}>About this project</h1>

      {block("Problem statement", "Students rely on disconnected tools — a chat app for doubts, a separate app for quizzes, a notebook for summarizing, and a planner drawn by hand. Switching between them breaks focus and makes it harder to track progress.")}
      {block("Proposed solution", "AI StudyMate brings explanation, self-testing, summarization and planning into a single workspace, with a dashboard that ties usage back to visible progress.")}
      {block("Objectives", <ul className="list-disc pl-5 space-y-1">
        <li>Build a working, presentable demo of an AI-assisted study workflow</li>
        <li>Apply a real NLP technique (extractive summarization) rather than only mock UI</li>
        <li>Design the system so a real LLM API can be plugged in without restructuring</li>
        <li>Practice full front-end architecture: componentization, state management, responsive design</li>
      </ul>)}
      {block("Technologies used", "React (component architecture, hooks), Tailwind CSS (styling), Recharts (dashboard visualizations), lucide-react (icons). Designed to connect to a Node/Express backend and a database (MongoDB or PostgreSQL) plus a real LLM API in a production version.")}
      {block("Advantages", <ul className="list-disc pl-5 space-y-1">
        <li>One workspace instead of several disconnected tools</li>
        <li>Immediate self-testing right after reading an explanation</li>
        <li>Extractive summarizer works instantly, with no API cost or key</li>
        <li>Clear seam for adding a real AI API later</li>
      </ul>)}
      {block("Limitations", <ul className="list-disc pl-5 space-y-1">
        <li>Study Assistant and Quiz Generator currently use templated responses, not a live LLM</li>
        <li>No backend/database yet — data resets on refresh</li>
        <li>Summarizer works best on well-structured paragraph text</li>
      </ul>)}
      {block("Future scope", "Real LLM integration, persistent accounts with a database, adaptive quiz difficulty, and analytics that recommend what to revise next.")}
    </Section>
  );
}

/* ============================================================
   APP SHELL
   ============================================================ */
const NAV = [
  { key: "landing", label: "Home", icon: BookOpen },
  { key: "assistant", label: "Assistant", icon: MessageSquare },
  { key: "quiz", label: "Quiz", icon: Brain },
  { key: "notes", label: "Summarizer", icon: FileText },
  { key: "planner", label: "Planner", icon: Calendar },
  { key: "dashboard", label: "Dashboard", icon: LayoutDashboard },
  { key: "aiml", label: "AI/ML", icon: Sparkles },
  { key: "about", label: "About", icon: Info },
];

export default function App() {
  const { mode, setMode, c } = useTheme();
  const [page, setPage] = useState("landing");
  const [menuOpen, setMenuOpen] = useState(false);

  const go = (p) => { setPage(p); setMenuOpen(false); window.scrollTo(0, 0); };

  const pages = {
    landing: <Landing c={c} go={go} />,
    assistant: <Assistant c={c} />,
    quiz: <Quiz c={c} />,
    notes: <Notes c={c} />,
    planner: <Planner c={c} />,
    dashboard: <Dashboard c={c} />,
    aiml: <AIML c={c} />,
    about: <About c={c} />,
  };

  return (
    <div style={{ background: c.bg, minHeight: "100vh", fontFamily: "ui-sans-serif, system-ui, sans-serif" }}>
      <style>{`.font-serif { font-family: Georgia, 'Times New Roman', serif; } code { background: ${c.surfaceAlt}; padding: 1px 5px; border-radius: 4px; }`}</style>

      {/* Navbar */}
      <header className="sticky top-0 z-20 border-b" style={{ borderColor: c.border, background: c.bg + "F2", backdropFilter: "blur(6px)" }}>
        <Section className="flex items-center justify-between py-4">
          <button onClick={() => go("landing")} className="flex items-center gap-2">
            <BookOpen size={20} style={{ color: c.primary }} />
            <span className="font-serif text-lg" style={{ color: c.ink }}>AI StudyMate</span>
          </button>

          <nav className="hidden lg:flex items-center gap-1">
            {NAV.map(n => (
              <button key={n.key} onClick={() => go(n.key)}
                className="px-3 py-2 rounded-md text-sm transition"
                style={{ color: page === n.key ? c.primary : c.inkSoft, background: page === n.key ? c.primarySoft : "transparent" }}>
                {n.label}
              </button>
            ))}
          </nav>

          <div className="flex items-center gap-2">
            <button onClick={() => setMode(m => m === "light" ? "dark" : "light")}
              className="p-2 rounded-md border" style={{ borderColor: c.border }}>
              {mode === "light" ? <Moon size={16} color={c.ink} /> : <Sun size={16} color={c.ink} />}
            </button>
            <button onClick={() => setMenuOpen(o => !o)} className="p-2 rounded-md border lg:hidden" style={{ borderColor: c.border }}>
              {menuOpen ? <X size={16} color={c.ink} /> : <Menu size={16} color={c.ink} />}
            </button>
          </div>
        </Section>

        {menuOpen && (
          <div className="lg:hidden border-t px-6 py-3 flex flex-col gap-1" style={{ borderColor: c.border }}>
            {NAV.map(n => (
              <button key={n.key} onClick={() => go(n.key)} className="text-left px-3 py-2 rounded-md text-sm"
                style={{ color: page === n.key ? c.primary : c.inkSoft, background: page === n.key ? c.primarySoft : "transparent" }}>
                {n.label}
              </button>
            ))}
          </div>
        )}
      </header>

      <main>{pages[page]}</main>

      <footer className="border-t mt-10" style={{ borderColor: c.border }}>
        <Section className="py-10 grid md:grid-cols-3 gap-8 text-sm">
          <div>
            <div className="flex items-center gap-2 mb-3">
              <BookOpen size={18} style={{ color: c.primary }} />
              <span className="font-serif text-base" style={{ color: c.ink }}>AI StudyMate</span>
            </div>
            <p style={{ color: c.inkSoft }}>A BCA final-year project exploring AI/ML in education.</p>
          </div>
          <div>
            <p className="font-medium mb-2" style={{ color: c.ink }}>Explore</p>
            <div className="flex flex-col gap-1.5">
              {NAV.slice(1).map(n => (
                <button key={n.key} onClick={() => go(n.key)} className="text-left" style={{ color: c.inkSoft }}>{n.label}</button>
              ))}
            </div>
          </div>
          <div>
            <p className="font-medium mb-2" style={{ color: c.ink }}>Project</p>
            <p style={{ color: c.inkSoft }}>Built with React, Tailwind CSS and Recharts. Designed for future integration with a real LLM API and backend.</p>
          </div>
        </Section>
      </footer>
    </div>
  );
}
