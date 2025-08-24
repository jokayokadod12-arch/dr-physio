import React, { useEffect, useMemo, useState } from "react";
import { BrowserRouter as Router, Routes, Route, Link, useNavigate, useParams } from "react-router-dom";
import { motion, AnimatePresence } from "framer-motion";

// shadcn/ui components
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Switch } from "@/components/ui/switch";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Slider } from "@/components/ui/slider";
import { Label } from "@/components/ui/label";

// Icons
import { Plus, Trash2, ArrowUp, ArrowDown, Upload, Download, LayoutGrid, Save, RotateCcw, Settings, Palette, Image as ImageIcon, Type, List, FileText, Home, Edit3 } from "lucide-react";

/**
 * Multi-Page Editable Site
 * - Homepage with cards → each card links to its own page (/p/:slug)
 * - Control panel to edit: theme, hero, sections, cards, and per-page content blocks (text/gallery)
 * - Import/Export JSON, autosave to localStorage
 * - React + Tailwind + shadcn/ui + Framer Motion + React Router
 */

const THEMES = {
  sky: { name: "Sky", primary: "from-sky-400 to-blue-600", text: "text-slate-800", accent: "bg-sky-600 hover:bg-sky-700", ring: "ring-sky-500" },
  emerald: { name: "Emerald", primary: "from-emerald-400 to-green-600", text: "text-slate-800", accent: "bg-emerald-600 hover:bg-emerald-700", ring: "ring-emerald-500" },
  violet: { name: "Violet", primary: "from-violet-400 to-purple-600", text: "text-slate-800", accent: "bg-violet-600 hover:bg-violet-700", ring: "ring-violet-500" },
  rose: { name: "Rose", primary: "from-rose-400 to-pink-600", text: "text-slate-800", accent: "bg-rose-600 hover:bg-rose-700", ring: "ring-rose-500" },
};

const DEFAULT_DATA = {
  theme: "sky",
  maxWidth: 1100,
  siteTitle: "Physio Knowledge Hub",
  hero: {
    title: "مرجع العلاج الطبيعي للطلاب والأطباء",
    subtitle: "شروحات مُبسّطة، حالات عملية، وتمارين موثوقة — كل شيء في مكان واحد.",
    ctaText: "ابدأ التصفح",
  },
  sections: {
    about: { enabled: true, title: "عن الموقع", text: "منصة تعليمية لمراجعة التشريح، الحالات الإكلينيكية، وخطط العلاج مع مصادر موثوقة." },
    services: {
      enabled: true,
      title: "الأقسام",
      items: [
        { title: "تشريح (Anatomy)", desc: "ملفات وملخصات مع صور توضيحية." },
        { title: "حالات إكلينيكية", desc: "تقييم، تشخيص تفريقي، وخطة علاج عملية." },
        { title: "تمارين علاجية", desc: "بروتوكولات وتمارين موثوقة بالفيديو." },
      ],
    },
    gallery: { enabled: true, title: "معرض الصور", images: [
      "https://images.unsplash.com/photo-1599050751766-f8382b1b0cc1?q=80&w=1200&auto=format&fit=crop",
      "https://images.unsplash.com/photo-1581594693700-99c52be2e63b?q=80&w=1200&auto=format&fit=crop",
      "https://images.unsplash.com/photo-1561214072-2a00b1db1f4e?q=80&w=1200&auto=format&fit=crop"
    ] },
    contact: {
      enabled: true,
      title: "تواصل معنا",
      email: "info@physiohub.edu",
      phone: "+20 10 000 0000",
      address: "القاهرة، مصر",
      messagePlaceholder: "اكتب رسالتك هنا...",
    },
  },
  pages: {
    // key is slug
  },
};

function slugify(str) {
  return String(str || "").toLowerCase().trim()
    .replace(/[\s_]+/g, "-")
    .replace(/[^a-z0-9\-\u0600-\u06FF]/g, "")
    .replace(/-+/g, "-");
}

function ensurePagesForServices(data) {
  const clone = structuredClone(data);
  clone.sections.services.items.forEach((it) => {
    const slug = slugify(it.title);
    if (!clone.pages[slug]) {
      clone.pages[slug] = {
        title: it.title,
        subtitle: it.desc || "",
        blocks: [
          { type: "text", title: "مقدمة", body: `${it.title} — هنا يمكنك كتابة مقدمة عامة عن القسم ومجالاته.` },
          { type: "gallery", title: "صور", images: [] },
        ],
      };
    }
  });
  return clone;
}

function useLocalState(key, initial) {
  const [state, setState] = useState(() => {
    try {
      const raw = localStorage.getItem(key);
      return raw ? ensurePagesForServices(JSON.parse(raw)) : ensurePagesForServices(initial);
    } catch {
      return ensurePagesForServices(initial);
    }
  });
  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(state)); } catch {}
  }, [key, state]);
  return [state, setState];
}

function SectionHeader({ title, subtitle, icon: Icon }) {
  return (
    <div className="flex items-center gap-3 mb-6">
      {Icon && <Icon className="w-6 h-6" />}
      <div>
        <h3 className="text-xl font-semibold">{title}</h3>
        {subtitle && <p className="text-sm text-slate-500">{subtitle}</p>}
      </div>
    </div>
  );
}

function ControlRow({ label, children }) {
  return (
    <div className="flex flex-col gap-2">
      <Label className="text-sm font-medium">{label}</Label>
      {children}
    </div>
  );
}

function SectionToggle({ label, checked, onCheckedChange }) {
  return (
    <div className="flex items-center justify-between py-1">
      <span className="text-sm">{label}</span>
      <Switch checked={checked} onCheckedChange={onCheckedChange} />
    </div>
  );
}

function BlocksEditor({ page, onChange }) {
  const updateBlock = (idx, block) => {
    const blocks = [...page.blocks];
    blocks[idx] = block;
    onChange({ ...page, blocks });
  };
  const addBlock = (type) => {
    const blocks = [...page.blocks];
    if (type === "text") blocks.push({ type: "text", title: "عنوان", body: "نص الفقرة..." });
    if (type === "gallery") blocks.push({ type: "gallery", title: "معرض", images: [] });
    onChange({ ...page, blocks });
  };
  const moveBlock = (i, dir) => {
    const target = i + dir;
    if (target < 0 || target >= page.blocks.length) return;
    const blocks = [...page.blocks];
    [blocks[i], blocks[target]] = [blocks[target], blocks[i]];
    onChange({ ...page, blocks });
  };
  const removeBlock = (i) => {
    const blocks = page.blocks.filter((_, idx) => idx !== i);
    onChange({ ...page, blocks });
  };

  return (
    <div className="space-y-3">
      {page.blocks.map((b, i) => (
        <Card key={i} className="border">
          <CardHeader className="pb-2">
            <CardTitle className="text-base flex items-center gap-2">
              {b.type === "text" ? <Type className="w-4 h-4"/> : <ImageIcon className="w-4 h-4"/>}
              {b.type === "text" ? "بلوك نص" : "بلوك صور"}
            </CardTitle>
          </CardHeader>
          <CardContent className="space-y-2">
            <ControlRow label="العنوان">
              <Input value={b.title || ""} onChange={(e)=>updateBlock(i, { ...b, title: e.target.value })} />
            </ControlRow>
            {b.type === "text" && (
              <ControlRow label="النص">
                <Textarea rows={4} value={b.body || ""} onChange={(e)=>updateBlock(i, { ...b, body: e.target.value })} />
              </ControlRow>
            )}
            {b.type === "gallery" && (
              <div className="space-y-2">
                {(b.images || []).map((src, idx) => (
                  <div key={idx} className="flex items-center gap-2">
                    <Input value={src} onChange={(e)=>{
                      const imgs = [...(b.images||[])]; imgs[idx] = e.target.value;
                      updateBlock(i, { ...b, images: imgs });
                    }} />
                    <Button variant="destructive" size="icon" onClick={()=>{
                      const imgs = (b.images||[]).filter((_,j)=>j!==idx);
                      updateBlock(i, { ...b, images: imgs });
                    }}><Trash2 className="w-4 h-4"/></Button>
                  </div>
                ))}
                <Button variant="secondary" onClick={()=>updateBlock(i, { ...b, images: [...(b.images||[]), "https://images.unsplash.com/photo-1520975922284-5f7f3e82b1b4?q=80&w=1200&auto=format&fit=crop"] })}>
                  <Plus className="w-4 h-4 mr-1"/> إضافة صورة
                </Button>
              </div>
            )}
            <div className="flex gap-2">
              <Button variant="secondary" onClick={()=>moveBlock(i,-1)}><ArrowUp className="w-4 h-4"/></Button>
              <Button variant="secondary" onClick={()=>moveBlock(i, 1)}><ArrowDown className="w-4 h-4"/></Button>
              <Button variant="destructive" onClick={()=>removeBlock(i)}><Trash2 className="w-4 h-4"/></Button>
            </div>
          </CardContent>
        </Card>
      ))}

      <div className="flex gap-2">
        <Button variant="secondary" onClick={()=>addBlock("text")}>
          <FileText className="w-4 h-4 mr-1"/> إضافة فقرة
        </Button>
        <Button variant="secondary" onClick={()=>addBlock("gallery")}>
          <ImageIcon className="w-4 h-4 mr-1"/> إضافة معرض
        </Button>
      </div>
    </div>
  );
}

function PageRenderer({ data, theme }) {
  const { slug } = useParams();
  const page = data.pages[slug];
  const containerStyle = useMemo(() => ({ maxWidth: data.maxWidth + "px" }), [data.maxWidth]);
  const navigate = useNavigate();

  if (!page) {
    return (
      <div className="max-w-screen-2xl mx-auto p-4">
        <Card className="rounded-3xl border-white/60 bg-white/80 backdrop-blur">
          <CardContent className="p-8">
            <p className="text-slate-700">الصفحة غير موجودة.</p>
            <div className="mt-4 flex gap-2">
              <Button onClick={()=>navigate("/")}>عودة للرئيسية</Button>
            </div>
          </CardContent>
        </Card>
      </div>
    );
  }

  return (
    <div className={`min-h-screen ${theme.text} bg-gradient-to-br ${theme.primary} pb-20`}>
      <div className="sticky top-0 z-30 backdrop-blur supports-[backdrop-filter]:bg-white/40 bg-white/60 border-b border-white/50">
        <div className="max-w-screen-2xl mx-auto px-4 py-2 flex items-center justify-between">
          <Link to="/" className="flex items-center gap-2"><Home className="w-5 h-5"/><span className="font-semibold">{data.siteTitle}</span></Link>
          <Link to={`/_/edit/${slug}`} className="text-sm underline flex items-center gap-1"><Edit3 className="w-4 h-4"/>تحرير هذه الصفحة</Link>
        </div>
      </div>

      <div className="max-w-screen-2xl mx-auto p-4">
        <section className="rounded-3xl overflow-hidden shadow-xl">
          <div className="bg-white/80 p-10 md:p-14">
            <div className="mx-auto" style={containerStyle}>
              <motion.h1 initial={{opacity:0,y:20}} animate={{opacity:1,y:0}} className="text-3xl md:text-5xl font-extrabold leading-tight">{page.title}</motion.h1>
              {page.subtitle && <motion.p initial={{opacity:0,y:10}} animate={{opacity:1,y:0}} className="mt-3 text-lg md:text-xl text-slate-600">{page.subtitle}</motion.p>}
            </div>
          </div>
        </section>

        <div className="mt-6 space-y-6" style={containerStyle}>
          {page.blocks.map((b, i) => (
            <AnimatePresence key={i}>
              {b.type === "text" && (
                <motion.section initial={{opacity:0, y:20}} animate={{opacity:1, y:0}} className="">
                  <Card className="rounded-3xl border-white/60 bg-white/80 backdrop-blur">
                    <CardHeader>{b.title && <CardTitle className="text-2xl">{b.title}</CardTitle>}</CardHeader>
                    <CardContent>
                      <p className="text-slate-700 leading-relaxed whitespace-pre-wrap">{b.body}</p>
                    </CardContent>
                  </Card>
                </motion.section>
              )}
              {b.type === "gallery" && (
                <motion.section initial={{opacity:0, y:20}} animate={{opacity:1, y:0}} className="">
                  <Card className="rounded-3xl border-white/60 bg-white/80 backdrop-blur">
                    <CardHeader className="flex-row items-center justify-between">
                      <CardTitle className="text-2xl flex items-center gap-2"><ImageIcon className="w-5 h-5"/>{b.title || "معرض"}</CardTitle>
                    </CardHeader>
                    <CardContent>
                      {(!b.images || b.images.length===0) && (
                        <p className="text-slate-500 text-sm">لا توجد صور بعد.</p>
                      )}
                      <div className="grid sm:grid-cols-2 md:grid-cols-3 gap-3">
                        {(b.images||[]).map((src, idx)=> (
                          <img key={idx} src={src} alt={`img-${idx}`} className="w-full h-48 object-cover rounded-2xl border" />
                        ))}
                      </div>
                    </CardContent>
                  </Card>
                </motion.section>
              )}
            </AnimatePresence>
          ))}
        </div>
      </div>

      <footer className="mt-10 text-center text-white/90">
        <p className="text-sm">© {new Date().getFullYear()} — {data.siteTitle}</p>
      </footer>
    </div>
  );
}

export default function App() {
  const [data, setData] = useLocalState("editable-multipage-site-v1", DEFAULT_DATA);
  const theme = THEMES[data.theme] ?? THEMES.sky;
  const containerStyle = useMemo(() => ({ maxWidth: data.maxWidth + "px" }), [data.maxWidth]);

  // helpers
  const update = (path, value) => {
    setData((old) => {
      const clone = structuredClone(old);
      const parts = path.split(".");
      let obj = clone;
      while (parts.length > 1) {
        const p = parts.shift();
        obj[p] = obj[p] ?? {};
        obj = obj[p];
      }
      obj[parts[0]] = value;
      return ensurePagesForServices(clone);
    });
  };

  const exportJSON = () => {
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url; a.download = "site-data.json"; a.click(); URL.revokeObjectURL(url);
  };

  const importJSON = (file) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      try { setData(ensurePagesForServices(JSON.parse(String(e.target?.result || "{}")))); }
      catch { alert("ملف غير صالح"); }
    };
    reader.readAsText(file);
  };

  // services controls
  const addService = () => update("sections.services.items", [ ...data.sections.services.items, { title: "قسم جديد", desc: "وصف مختصر" } ]);
  const moveService = (i, dir) => {
    const items = [...data.sections.services.items];
    const t = i + dir; if (t<0 || t>=items.length) return; [items[i], items[t]] = [items[t], items[i]]; update("sections.services.items", items);
  };
  const removeService = (i) => {
    const items = data.sections.services.items.filter((_, idx)=> idx!==i);
    update("sections.services.items", items);
  };

  return (
    <Router>
      <Routes>
        {/* PAGE VIEW */}
        <Route path="/p/:slug" element={<PageRenderer data={data} theme={theme} />} />

        {/* PAGE EDITOR */}
        <Route path="/_/edit/:slug" element={
          <div className={`min-h-screen ${theme.text} bg-gradient-to-br ${theme.primary} pb-20`}>
            <div className="sticky top-0 z-30 backdrop-blur supports-[backdrop-filter]:bg-white/40 bg-white/60 border-b border-white/50">
              <div className="max-w-screen-2xl mx-auto px-4 py-2 flex items-center justify-between">
                <Link to="/" className="flex items-center gap-2"><Home className="w-5 h-5"/><span className="font-semibold">{data.siteTitle}</span></Link>
              </div>
            </div>
            <EditPage data={data} setData={setData} />
          </div>
        } />

        {/* HOME + CONTROL PANEL */}
        <Route path="/" element={
          <div className={`min-h-screen ${theme.text} bg-gradient-to-br ${theme.primary} pb-20`}>
            {/* Top Bar */}
            <div className="sticky top-0 z-30 backdrop-blur supports-[backdrop-filter]:bg-white/40 bg-white/60 border-b border-white/50">
              <div className="max-w-screen-2xl mx-auto px-4 py-2 flex items-center justify-between">
                <div className="flex items-center gap-2">
                  <LayoutGrid className="w-5 h-5" />
                  <span className="font-semibold">{data.siteTitle}</span>
                </div>
                <div className="flex items-center gap-2">
                  <Button size="sm" className={`${theme.accent} text-white`} onClick={() => document.getElementById("hero")?.scrollIntoView({ behavior: "smooth" })}>
                    انتقل للأعلى
                  </Button>
                  <Button size="sm" variant="secondary" onClick={exportJSON}>
                    <Download className="w-4 h-4 mr-1" /> تصدير JSON
                  </Button>
                </div>
              </div>
            </div>

            <div className="max-w-screen-2xl mx-auto p-4 grid md:grid-cols-[1fr_380px] gap-4">
              {/* MAIN */}
              <main>
                {/* HERO */}
                <section id="hero" className="rounded-3xl overflow-hidden shadow-xl">
                  <div className="bg-white/80 p-10 md:p-14">
                    <div className="mx-auto" style={containerStyle}>
                      <motion.h1 initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.5 }} className="text-3xl md:text-5xl font-extrabold leading-tight">
                        {data.hero.title}
                      </motion.h1>
                      <motion.p initial={{ opacity: 0, y: 10 }} animate={{ opacity: 1, y: 0 }} transition={{ delay: 0.1, duration: 0.5 }} className="mt-3 text-lg md:text-xl text-slate-600">
                        {data.hero.subtitle}
                      </motion.p>
                      <Button className={`mt-6 ${theme.accent} text-white`}>{data.hero.ctaText}</Button>
                    </div>
                  </div>
                </section>

                {/* ABOUT */}
                <AnimatePresence>
                  {data.sections.about.enabled && (
                    <motion.section initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }} exit={{ opacity: 0, y: -10 }} className="mt-6">
                      <Card className="rounded-3xl border-white/60 bg-white/80 backdrop-blur">
                        <CardHeader>
                          <CardTitle className="text-2xl">{data.sections.about.title}</CardTitle>
                        </CardHeader>
                        <CardContent>
                          <p className="text-slate-700 leading-relaxed" style={containerStyle}>{data.sections.about.text}</p>
                        </CardContent>
                      </Card>
                    </motion.section>
                  )}
                </AnimatePresence>

                {/* SERVICES */}
                <AnimatePresence>
                  {data.sections.services.enabled && (
                    <motion.section initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }} exit={{ opacity: 0, y: -10 }} className="mt-6">
                      <Card className="rounded-3xl border-white/60 bg-white/80 backdrop-blur">
                        <CardHeader>
                          <CardTitle className="text-2xl">{data.sections.services.title}</CardTitle>
                        </CardHeader>
                        <CardContent>
                          <div className="grid md:grid-cols-3 gap-4" style={containerStyle}>
                            {data.sections.services.items.map((it, i) => {
                              const slug = slugify(it.title);
                              return (
                                <Link key={i} to={`/p/${slug}`} className="rounded-2xl border p-4 bg-white/70 block hover:shadow-md transition-shadow">
                                  <h4 className="font-semibold text-lg">{it.title}</h4>
                                  <p className="text-slate-600 text-sm mt-1">{it.desc}</p>
                                  <div className="text-xs mt-3 underline">افتح الصفحة</div>
                                </Link>
                              );
                            })}
                          </div>
                        </CardContent>
                      </Card>
                    </motion.section>
                  )}
                </AnimatePresence>

                {/* GALLERY */}
                <AnimatePresence>
                  {data.sections.gallery.enabled && (
                    <motion.section initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }} exit={{ opacity: 0, y: -10 }} className="mt-6">
                      <Card className="rounded-3xl border-white/60 bg-white/80 backdrop-blur">
                        <CardHeader>
                          <CardTitle className="text-2xl flex items-center gap-2"><ImageIcon className="w-5 h-5" /> {data.sections.gallery.title}</CardTitle>
                        </CardHeader>
                        <CardContent>
                          <div className="grid sm:grid-cols-2 md:grid-cols-3 gap-3" style={containerStyle}>
                            {data.sections.gallery.images.map((src, idx) => (
                              <img key={idx} src={src} alt={`img-${idx}`} className="w-full h-48 object-cover rounded-2xl border" />
                            ))}
                          </div>
                        </CardContent>
                      </Card>
                    </motion.section>
                  )}
                </AnimatePresence>
              </main>

              {/* CONTROL PANEL */}
              <aside className="space-y-4">
                <Card className="rounded-3xl border-white/60 bg-white/90 backdrop-blur">
                  <CardHeader className="pb-2">
                    <CardTitle className="flex items-center gap-2 text-base"><Settings className="w-4 h-4"/> لوحة التحكم</CardTitle>
                  </CardHeader>
                  <CardContent className="space-y-4">
                    <Tabs defaultValue="general" className="w-full">
                      <TabsList className="grid grid-cols-3">
                        <TabsTrigger value="general">عام</TabsTrigger>
                        <TabsTrigger value="sections">الأقسام</TabsTrigger>
                        <TabsTrigger value="data">البيانات</TabsTrigger>
                      </TabsList>

                      {/* GENERAL */}
                      <TabsContent value="general" className="space-y-4 mt-4">
                        <ControlRow label="عنوان الموقع">
                          <Input value={data.siteTitle} onChange={(e)=>update("siteTitle", e.target.value)} />
                        </ControlRow>

                        <div className="grid grid-cols-2 gap-4">
                          <ControlRow label="السمة (Theme)">
                            <div className="grid grid-cols-2 gap-2">
                              {Object.entries(THEMES).map(([k, v]) => (
                                <Button key={k} variant={data.theme===k?"default":"secondary"} onClick={()=>update("theme", k)} className={data.theme===k?`ring-2 ${THEMES[data.theme].ring}`:undefined}>
                                  <Palette className="w-4 h-4 mr-2" /> {v.name}
                                </Button>
                              ))}
                            </div>
                          </ControlRow>
                          <ControlRow label={`عرض المحتوى (${data.maxWidth}px)`}>
                            <Slider value={[data.maxWidth]} min={700} max={1400} step={10} onValueChange={(v)=>update("maxWidth", v[0])} />
                          </ControlRow>
                        </div>

                        <ControlRow label="نص العنوان الرئيسي">
                          <Textarea value={data.hero.title} onChange={(e)=>update("hero.title", e.target.value)} />
                        </ControlRow>
                        <ControlRow label="نص الوصف تحت العنوان">
                          <Textarea value={data.hero.subtitle} onChange={(e)=>update("hero.subtitle", e.target.value)} />
                        </ControlRow>
                        <ControlRow label="نص زر الدعوة (CTA)">
                          <Input value={data.hero.ctaText} onChange={(e)=>update("hero.ctaText", e.target.value)} />
                        </ControlRow>

                        <div className="flex items-center gap-2">
                          <Button variant="secondary" onClick={()=>setData(DEFAULT_DATA)}><RotateCcw className="w-4 h-4 mr-1"/>إعادة الافتراضي</Button>
                          <Button onClick={exportJSON}><Save className="w-4 h-4 mr-1"/>تصدير</Button>
                          <label className="inline-flex items-center gap-2 cursor-pointer">
                            <Upload className="w-4 h-4"/>
                            <span>استيراد JSON</span>
                            <input type="file" accept="application/json" className="hidden" onChange={(e)=> e.target.files && importJSON(e.target.files[0])} />
                          </label>
                        </div>
                      </TabsContent>

                      {/* SECTIONS */}
                      <TabsContent value="sections" className="space-y-6 mt-4">
                        <Card className="border">
                          <CardHeader className="pb-2"><CardTitle className="text-base">تفعيل/تعطيل الأقسام</CardTitle></CardHeader>
                          <CardContent className="space-y-2">
                            <SectionToggle label="عن الموقع" checked={data.sections.about.enabled} onCheckedChange={(v)=>update("sections.about.enabled", v)} />
                            <SectionToggle label="الأقسام" checked={data.sections.services.enabled} onCheckedChange={(v)=>update("sections.services.enabled", v)} />
                            <SectionToggle label="المعرض" checked={data.sections.gallery.enabled} onCheckedChange={(v)=>update("sections.gallery.enabled", v)} />
                          </CardContent>
                        </Card>

                        {/* ABOUT editor */}
                        <Card className="border">
                          <CardHeader className="pb-2"><CardTitle className="text-base">عن الموقع</CardTitle></CardHeader>
                          <CardContent className="space-y-3">
                            <ControlRow label="العنوان">
                              <Input value={data.sections.about.title} onChange={(e)=>update("sections.about.title", e.target.value)} />
                            </ControlRow>
                            <ControlRow label="النص">
                              <Textarea value={data.sections.about.text} onChange={(e)=>update("sections.about.text", e.target.value)} rows={3} />
                            </ControlRow>
                          </CardContent>
                        </Card>

                        {/* SERVICES editor */}
                        <Card className="border">
                          <CardHeader className="pb-2"><CardTitle className="text-base">الأقسام (الكروت)</CardTitle></CardHeader>
                          <CardContent className="space-y-3">
                            <div className="flex justify-between items-center">
                              <Label>عنوان مجموعة الأقسام</Label>
                              <Button size="sm" onClick={addService}><Plus className="w-4 h-4 mr-1"/> إضافة كارت</Button>
                            </div>
                            <Input value={data.sections.services.title} onChange={(e)=>update("sections.services.title", e.target.value)} />

                            <div className="space-y-3">
                              {data.sections.services.items.map((it, i)=> {
                                const slug = slugify(it.title);
                                return (
                                  <div key={i} className="rounded-xl border p-3 bg-white/60">
                                    <div className="grid grid-cols-1 md:grid-cols-[1fr_auto] gap-2 items-start">
                                      <div className="space-y-2">
                                        <Input value={it.title} onChange={(e)=>{
                                          const items = [...data.sections.services.items];
                                          items[i] = { ...it, title: e.target.value };
                                          update("sections.services.items", items);
                                        }} placeholder="عنوان" />
                                        <Textarea value={it.desc} onChange={(e)=>{
                                          const items = [...data.sections.services.items];
                                          items[i] = { ...it, desc: e.target.value };
                                          update("sections.services.items", items);
                                        }} rows={2} placeholder="وصف" />
                                        <div className="text-xs text-slate-600">الرابط: <code className="bg-slate-100 px-1 rounded">/p/{slug}</code> — <Link to={`/_/edit/${slug}`} className="underline">تحرير صفحة هذا القسم</Link></div>
                                      </div>
                                      <div className="flex md:flex-col gap-2 justify-end">
                                        <Button variant="secondary" onClick={()=>moveService(i, -1)}><ArrowUp className="w-4 h-4"/></Button>
                                        <Button variant="secondary" onClick={()=>moveService(i, 1)}><ArrowDown className="w-4 h-4"/></Button>
                                        <Button variant="destructive" onClick={()=>removeService(i)}><Trash2 className="w-4 h-4"/></Button>
                                      </div>
                                    </div>
                                  </div>
                                );
                              })}
                            </div>
                          </CardContent>
                        </Card>

                        {/* GALLERY editor */}
                        <Card className="border">
                          <CardHeader className="pb-2"><CardTitle className="text-base">المعرض (روابط صور)</CardTitle></CardHeader>
                          <CardContent className="space-y-3">
                            {data.sections.gallery.images.map((src, idx)=> (
                              <div key={idx} className="flex items-center gap-2">
                                <Input value={src} onChange={(e)=>{
                                  const imgs = [...data.sections.gallery.images];
                                  imgs[idx] = e.target.value;
                                  update("sections.gallery.images", imgs);
                                }} />
                                <Button variant="destructive" size="icon" onClick={()=>{
                                  const imgs = data.sections.gallery.images.filter((_,i)=>i!==idx);
                                  update("sections.gallery.images", imgs);
                                }}><Trash2 className="w-4 h-4"/></Button>
                              </div>
                            ))}
                            <Button variant="secondary" onClick={()=> update("sections.gallery.images", [...data.sections.gallery.images, "https://images.unsplash.com/photo-1520975922284-5f7f3e82b1b4?q=80&w=1200&auto=format&fit=crop"]) }>
                              <Plus className="w-4 h-4 mr-1"/> إضافة صورة
                            </Button>
                          </CardContent>
                        </Card>
                      </TabsContent>

                      {/* DATA */}
                      <TabsContent value="data" className="space-y-3 mt-4">
                        <p className="text-sm text-slate-600">يمكنك نسخ/لصق بيانات الموقع كـ JSON للتخزين أو النقل.</p>
                        <Textarea rows={14} value={JSON.stringify(data, null, 2)} onChange={(e)=>{
                          try { setData(ensurePagesForServices(JSON.parse(e.target.value))); } catch {}
                        }} className="font-mono text-xs" />
                      </TabsContent>
                    </Tabs>
                  </CardContent>
                </Card>
              </aside>
            </div>

            <footer className="mt-10 text-center text-white/90">
              <p className="text-sm">© {new Date().getFullYear()} — صفحة متعددة الصفحات قابلة للتحكم الكامل.</p>
            </footer>
          </div>
        } />
      </Routes>
    </Router>
  );
}

function EditPage({ data, setData }) {
  const { slug } = useParams();
  const page = data.pages[slug];
  const theme = THEMES[data.theme] ?? THEMES.sky;
  const navigate = useNavigate();

  if (!page) {
    return (
      <div className="max-w-screen-2xl mx-auto p-4">
        <Card className="rounded-3xl border-white/60 bg-white/80 backdrop-blur">
          <CardContent className="p-8">
            <p className="text-slate-700">لا توجد صفحة لهذا القسم بعد.</p>
            <div className="mt-4 flex gap-2">
              <Button onClick={()=>navigate("/")}>عودة للرئيسية</Button>
            </div>
          </CardContent>
        </Card>
      </div>
    );
  }

  const updatePage = (next) => {
    setData((old) => ({ ...old, pages: { ...old.pages, [slug]: next }}));
  };

  return (
    <div className={`max-w-screen-2xl mx-auto p-4 ${theme.text}`}>
      <div className="grid md:grid-cols-[1fr_380px] gap-4">
        {/* PREVIEW */}
        <main>
          <section className="rounded-3xl overflow-hidden shadow-xl">
            <div className="bg-white/80 p-10 md:p-14">
              <motion.h1 initial={{opacity:0,y:20}} animate={{opacity:1,y:0}} className="text-3xl md:text-5xl font-extrabold leading-tight">{page.title}</motion.h1>
              {page.subtitle && <motion.p initial={{opacity:0,y:10}} animate={{opacity:1,y:0}} className="mt-3 text-lg md:text-xl text-slate-600">{page.subtitle}</motion.p>}
            </div>
          </section>
          <div className="mt-6">
            {page.blocks.map((b, i) => (
              <Card key={i} className="rounded-3xl border-white/60 bg-white/80 backdrop-blur mb-4">
                <CardHeader>
                  <CardTitle className="text-2xl">{b.title || (b.type === 'text' ? 'فقرة' : 'معرض')}</CardTitle>
                </CardHeader>
                <CardContent>
                  {b.type === 'text' ? (
                    <p className="text-slate-700 whitespace-pre-wrap">{b.body}</p>
                  ) : (
                    <div className="grid sm:grid-cols-2 md:grid-cols-3 gap-3">
                      {(b.images||[]).map((src, idx)=> (
                        <img key={idx} src={src} alt={`img-${idx}`} className="w-full h-48 object-cover rounded-2xl border" />
                      ))}
                    </div>
                  )}
                </CardContent>
              </Card>
            ))}
          </div>
        </main>

        {/* EDITOR */}
        <aside className="space-y-4">
          <Card className="rounded-3xl border-white/60 bg-white/90 backdrop-blur">
            <CardHeader className="pb-2"><CardTitle className="text-base">تحرير الصفحة</CardTitle></CardHeader>
            <CardContent className="space-y-4">
              <ControlRow label="عنوان الصفحة">
                <Input value={page.title} onChange={(e)=>updatePage({ ...page, title: e.target.value })} />
              </ControlRow>
              <ControlRow label="وصف مختصر">
                <Textarea rows={2} value={page.subtitle} onChange={(e)=>updatePage({ ...page, subtitle: e.target.value })} />
              </ControlRow>

              <SectionHeader title="البلوكات" subtitle="أضف فقرات أو معارض صور" icon={List} />
              <BlocksEditor page={page} onChange={updatePage} />

              <div className="flex gap-2">
                <Button onClick={()=>navigate(`/p/${slug}`)}>عرض الصفحة</Button>
                <Button variant="secondary" onClick={()=>navigate("/")}>عودة للرئيسية</Button>
              </div>
            </CardContent>
          </Card>
        </aside>
      </div>
    </div>
  );
}

