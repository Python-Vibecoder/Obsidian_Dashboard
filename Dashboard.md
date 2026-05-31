```dataviewjs
// ═══════════════════════════════════════════════════
//  CONFIG
// ═══════════════════════════════════════════════════
const CFG = {
  name:            "Rob",
  inboxFolder:     "Inbox To Sort",
  projectsFolder:  "Projects",
  readingFolder:   "01. Personal/Entertainment/Books/Book List",
  moviesFolder:    "01. Personal/Entertainment/Movies/Movie List",
  workTasksFile:   "Work Tasks",
  homeTasksFile:   "Home Tasks",
  savedLinksFile:  "Inbox To Sort/Saved Links.md",
  activeTag:       "active",
  greetingEmoji:   ["👋","🔬","📚","🧠","⚡"],
};

// ═══════════════════════════════════════════════════
//  HELPERS
// ═══════════════════════════════════════════════════
const C = dv.container;
C.style.cssText = "font-family:var(--font-interface);padding:0 0 48px;max-width:960px";

const div  = (parent, css) => { const e = parent.createEl("div"); if (css) e.style.cssText = css; return e; };
const span = (parent, text, css) => { const e = parent.createEl("span"); e.textContent = text; if (css) e.style.cssText = css; return e; };
const p    = (parent, text, css) => { const e = parent.createEl("p"); e.textContent = text; if (css) e.style.cssText = css; return e; };

const link = (parent, text, path, css) => {
  const a = parent.createEl("a", {cls:"internal-link"});
  a.textContent = text; a.href = path; a.dataset.href = path;
  a.style.cssText = `text-decoration:none;color:var(--text-accent);cursor:pointer;${css||""}`;
  a.addEventListener("click", (e) => { e.preventDefault(); app.workspace.openLinkText(path, "", "tab"); });
  return a;
};

const pill = (parent, text, bg, col) => {
  const e = parent.createEl("span"); e.textContent = text;
  e.style.cssText = `font-size:10px;font-weight:600;padding:2px 8px;border-radius:20px;background:${bg};color:${col};white-space:nowrap`;
  return e;
};

const sectionHead = (parent, label) => {
  const row = div(parent, "display:flex;align-items:center;gap:10px;margin:28px 0 14px");
  div(row, "width:3px;height:18px;background:#7C3AED;border-radius:2px;flex-shrink:0");
  const t = row.createEl("h2"); t.textContent = label;
  t.style.cssText = "margin:0;font-size:15px;font-weight:600;color:var(--text-normal);letter-spacing:0.01em";
  return row;
};

const card = (parent, css) => div(parent,
  `background:var(--background-primary);border:1px solid var(--background-modifier-border);border-radius:10px;padding:14px 16px;${css||""}`);

const progressBar = (parent, pct, color) => {
  const track = div(parent, "height:4px;background:var(--background-modifier-border);border-radius:3px;overflow:hidden;margin-top:8px");
  div(track, `height:100%;width:${Math.min(100,pct)}%;background:${color};border-radius:3px`);
};

const showModal = (title, options, onSelect) => {
  const overlay = document.body.createEl("div");
  overlay.style.cssText = "position:fixed;inset:0;background:rgba(0,0,0,0.55);z-index:9999;display:flex;align-items:center;justify-content:center";
  const box = overlay.createEl("div");
  box.style.cssText = "background:var(--background-primary);border:1px solid var(--background-modifier-border);border-radius:12px;padding:24px 28px;min-width:300px;box-shadow:0 20px 60px rgba(0,0,0,0.4)";
  const h = box.createEl("h3"); h.textContent = title;
  h.style.cssText = "margin:0 0 16px;font-size:15px;font-weight:700;color:var(--text-normal)";
  for (const opt of options) {
    const btn = box.createEl("button"); btn.textContent = opt.label;
    btn.style.cssText = "display:block;width:100%;text-align:left;background:var(--background-secondary);border:1px solid var(--background-modifier-border);border-radius:8px;padding:10px 14px;margin-bottom:8px;cursor:pointer;font-size:13px;font-weight:500;color:var(--text-normal);font-family:var(--font-interface)";
    btn.addEventListener("mouseenter", () => btn.style.background = "var(--background-modifier-hover)");
    btn.addEventListener("mouseleave", () => btn.style.background = "var(--background-secondary)");
    btn.addEventListener("click", () => { overlay.remove(); onSelect(opt.value); });
  }
  const cancel = box.createEl("button"); cancel.textContent = "Cancel";
  cancel.style.cssText = "display:block;width:100%;text-align:center;background:none;border:none;padding:8px;cursor:pointer;font-size:12px;color:var(--text-muted);font-family:var(--font-interface)";
  cancel.addEventListener("click", () => overlay.remove());
  overlay.addEventListener("click", (e) => { if (e.target === overlay) overlay.remove(); });
};

// ═══════════════════════════════════════════════════
//  DATA
// ═══════════════════════════════════════════════════
const now        = DateTime.now();
const hour       = now.hour;
const greeting   = hour < 12 ? "Good Morning" : hour < 17 ? "Good Afternoon" : "Good Evening";
const emoji      = CFG.greetingEmoji[Math.floor(Math.random() * CFG.greetingEmoji.length)];
const dateStr    = now.toFormat("cccc, d MMMM yyyy");

const allPages   = dv.pages();
const projects   = dv.pages(`"${CFG.projectsFolder}"`).sort(p => p.file.mtime,"desc");
const activeProj = projects.where(p => p.tags && (Array.isArray(p.tags) ? p.tags.includes(CFG.activeTag) : p.tags === CFG.activeTag));
const reading    = dv.pages(`"${CFG.readingFolder}"`).sort(p => p.file.mtime,"desc");
const movies     = dv.pages(`"${CFG.moviesFolder}"`).sort(p => p.file.mtime,"desc");
const inbox      = dv.pages(`"${CFG.inboxFolder}"`).sort(p => p.file.ctime,"desc");
const recent     = allPages.sort(p => p.file.mtime,"desc").limit(8);

const workPage  = dv.page(CFG.workTasksFile);
const homePage  = dv.page(CFG.homeTasksFile);
const workTasks = workPage ? workPage.file.tasks.where(t => !t.completed) : dv.array([]);
const homeTasks = homePage ? homePage.file.tasks.where(t => !t.completed) : dv.array([]);
const workDone  = workPage ? workPage.file.tasks.where(t =>  t.completed) : dv.array([]);
const homeDone  = homePage ? homePage.file.tasks.where(t =>  t.completed) : dv.array([]);
const allTasks   = allPages.file.tasks;
const openTasks  = allTasks.where(t => !t.completed);
const todayStr   = now.toFormat("yyyy-MM-dd");
const todayTasks = openTasks.filter(t => {
  if (!t.due) return false;
  try { return DateTime.fromJSDate(t.due).toFormat("yyyy-MM-dd") === todayStr; } catch(e) { return false; }
});

// ═══════════════════════════════════════════════════
//  ① HERO HEADER
// ═══════════════════════════════════════════════════
const hero = div(C, `background:linear-gradient(135deg,#1e1b4b 0%,#312e81 50%,#4c1d95 100%);border-radius:14px;padding:32px 36px;margin-bottom:4px;position:relative;overflow:hidden`);
div(hero, "position:absolute;top:-40px;right:-40px;width:180px;height:180px;background:rgba(167,139,250,0.15);border-radius:50%");
div(hero, "position:absolute;bottom:-30px;left:60px;width:120px;height:120px;background:rgba(139,92,246,0.12);border-radius:50%");
const heroInner = div(hero, "position:relative;z-index:1");
const heroTop   = div(heroInner, "display:flex;justify-content:space-between;align-items:flex-start;flex-wrap:wrap;gap:12px");
const heroLeft  = div(heroTop);
const greetEl = heroLeft.createEl("h1"); greetEl.textContent = `${emoji} ${greeting}, ${CFG.name}`;
greetEl.style.cssText = "margin:0 0 4px;font-size:26px;font-weight:700;color:#fff;line-height:1.2";
p(heroLeft, dateStr, "margin:0;font-size:13px;color:rgba(255,255,255,0.6)");
const chips = div(heroTop, "display:flex;gap:8px;flex-wrap:wrap;align-items:flex-start;margin-top:4px");
for (const ch of [
  { label: allPages.length + " notes",             bg:"rgba(139,92,246,0.35)", col:"#c4b5fd" },
  { label: openTasks.length + " open tasks",       bg:"rgba(16,185,129,0.25)", col:"#6ee7b7" },
  { label: activeProj.length + " active projects", bg:"rgba(59,130,246,0.25)", col:"#93c5fd" },
  { label: reading.length + " to read",            bg:"rgba(245,158,11,0.25)", col:"#fcd34d" },
  { label: inbox.length + " in inbox",             bg:"rgba(239,68,68,0.25)",  col:"#fca5a5" },
]) { const chip = div(chips, `background:${ch.bg};border-radius:20px;padding:5px 12px`); span(chip, ch.label, `font-size:12px;font-weight:600;color:${ch.col}`); }
const heroBottom = div(heroInner, "margin-top:20px;display:flex;align-items:center;gap:12px;flex-wrap:wrap");
const taskSummary = div(heroBottom, "font-size:12px;color:rgba(255,255,255,0.55)");
taskSummary.textContent = todayTasks.length > 0 ? `${todayTasks.length} task${todayTasks.length>1?"s":""} due today` : "";

// ═══════════════════════════════════════════════════
//  ② SEARCH BAR
// ═══════════════════════════════════════════════════
sectionHead(C, "🔍 Search Vault");
const searchCard = card(C);

const searchWrap = div(searchCard, "position:relative");
const searchInput = searchWrap.createEl("input");
searchInput.type = "text"; searchInput.placeholder = "Search notes, tasks, projects, books…";
searchInput.style.cssText = `width:100%;padding:10px 14px 10px 38px;background:var(--background-secondary);border:1px solid var(--background-modifier-border);border-radius:8px;color:var(--text-normal);font-size:13px;font-family:var(--font-interface);box-sizing:border-box;outline:none;line-height:1.5`;
const searchIcon = div(searchWrap, "position:absolute;left:12px;top:50%;transform:translateY(-50%);font-size:14px;pointer-events:none");
searchIcon.textContent = "🔍";

// Filter pills
const searchFilterBar = div(searchCard, "display:flex;gap:6px;flex-wrap:wrap;margin:10px 0 0");
const SEARCH_FILTERS = [
  { label:"All",      val:"all" },
  { label:"📝 Notes", val:"note" },
  { label:"🚀 Projects", val:"project" },
  { label:"📚 Books", val:"book" },
  { label:"🎬 Movies", val:"movie" },
  { label:"📥 Inbox", val:"inbox" },
];
let searchFilter = "all";
const searchFBtns = [];
for (const f of SEARCH_FILTERS) {
  const fb = searchFilterBar.createEl("button"); fb.textContent = f.label; fb._val = f.val;
  fb.style.cssText = `font-size:11px;font-weight:600;padding:3px 10px;border-radius:20px;cursor:pointer;font-family:var(--font-interface);border:1px solid var(--background-modifier-border);background:${f.val==="all"?"#7C3AED":"var(--background-secondary)"};color:${f.val==="all"?"#fff":"var(--text-muted)"}`;
  fb.addEventListener("click", () => {
    searchFilter = f.val;
    searchFBtns.forEach(b => { b.style.background = b._val===f.val?"#7C3AED":"var(--background-secondary)"; b.style.color = b._val===f.val?"#fff":"var(--text-muted)"; });
    doSearch(searchInput.value.trim());
  });
  searchFBtns.push(fb);
}

const searchResults = div(searchCard, "margin-top:12px");

// Build a searchable index once
const searchIndex = allPages.array().map(pg => {
  let type = "note";
  if (pg.file.path.startsWith(CFG.projectsFolder + "/")) type = "project";
  else if (pg.file.path.startsWith(CFG.readingFolder + "/")) type = "book";
  else if (pg.file.path.startsWith(CFG.moviesFolder + "/")) type = "movie";
  else if (pg.file.path.startsWith(CFG.inboxFolder + "/")) type = "inbox";
  return {
    name:   pg.file.name,
    path:   pg.file.path,
    folder: pg.file.folder,
    mtime:  pg.file.mtime,
    tags:   pg.tags || [],
    status: pg.status || "",
    author: pg.author || "",
    type,
  };
});

const typeIcon  = { note:"📝", project:"🚀", book:"📚", movie:"🎬", inbox:"📥" };
const typePill  = { note:["#F3F4F6","#374151"], project:["#EDE9FE","#5B21B6"], book:["#DBEAFE","#1E40AF"], movie:["#FEF3C7","#92400E"], inbox:["#FEE2E2","#991B1B"] };

const doSearch = (query) => {
  searchResults.empty();
  if (!query || query.length < 2) {
    span(searchResults, "Type at least 2 characters to search…", "font-size:12px;color:var(--text-faint)");
    return;
  }
  const q = query.toLowerCase();
  let hits = searchIndex.filter(item => {
    if (searchFilter !== "all" && item.type !== searchFilter) return false;
    return item.name.toLowerCase().includes(q)
      || item.folder.toLowerCase().includes(q)
      || String(item.status).toLowerCase().includes(q)
      || String(item.author).toLowerCase().includes(q)
      || (Array.isArray(item.tags) ? item.tags.join(" ").toLowerCase().includes(q) : false);
  });

  const total = hits.length;
  hits = hits.slice(0, 20);

  if (hits.length === 0) {
    const none = div(searchResults, "padding:12px 0;text-align:center");
    span(none, `No results for "${query}"`, "font-size:12px;color:var(--text-faint)");
    return;
  }

  const meta = div(searchResults, "display:flex;justify-content:space-between;align-items:center;margin-bottom:8px");
  span(meta, `${total} result${total!==1?"s":""}${total>20?" (showing 20)":""}`, "font-size:11px;color:var(--text-faint)");
  if (searchFilter !== "all") {
    const clear = meta.createEl("button"); clear.textContent = "Clear filter";
    clear.style.cssText = "background:none;border:none;font-size:11px;color:#7C3AED;cursor:pointer;font-family:var(--font-interface)";
    clear.addEventListener("click", () => { searchFilter="all"; searchFBtns.forEach(b=>{ b.style.background=b._val==="all"?"#7C3AED":"var(--background-secondary)"; b.style.color=b._val==="all"?"#fff":"var(--text-muted)"; }); doSearch(query); });
  }

  const grid = div(searchResults, "display:grid;grid-template-columns:repeat(auto-fill,minmax(260px,1fr));gap:8px");
  for (const item of hits) {
    const rc = card(grid, "display:flex;flex-direction:column;gap:4px;padding:10px 14px;cursor:pointer");
    rc.addEventListener("click", () => app.workspace.openLinkText(item.path, "", "tab"));
    rc.addEventListener("mouseenter", () => rc.style.background = "var(--background-secondary)");
    rc.addEventListener("mouseleave", () => rc.style.background = "var(--background-primary)");

    const top = div(rc, "display:flex;justify-content:space-between;align-items:flex-start;gap:6px");
    const nameEl = top.createEl("span");
    // Highlight matching part of name
    const lname = item.name.toLowerCase();
    const idx = lname.indexOf(q);
    if (idx >= 0) {
      nameEl.style.cssText = "font-size:13px;font-weight:600;color:var(--text-normal);flex:1;line-height:1.4";
      nameEl.appendChild(document.createTextNode(item.name.slice(0, idx)));
      const hl = nameEl.createEl("mark"); hl.textContent = item.name.slice(idx, idx+q.length);
      hl.style.cssText = "background:#7C3AED33;color:#7C3AED;border-radius:2px;padding:0 1px";
      nameEl.appendChild(document.createTextNode(item.name.slice(idx+q.length)));
    } else {
      nameEl.textContent = item.name;
      nameEl.style.cssText = "font-size:13px;font-weight:600;color:var(--text-normal);flex:1;line-height:1.4";
    }
    const tc = typePill[item.type] || typePill.note;
    pill(top, typeIcon[item.type]+" "+item.type, tc[0], tc[1]);

    const bot = div(rc, "display:flex;align-items:center;gap:6px;flex-wrap:wrap");
    span(bot, item.folder.split("/").pop()||"root", "font-size:10px;color:var(--text-faint)");
    span(bot, "·", "font-size:10px;color:var(--text-faint)");
    span(bot, item.mtime.toRelative(), "font-size:10px;color:var(--text-faint)");
    if (item.status) pill(bot, item.status, "#EDE9FE", "#5B21B6");
  }
};

// Debounce input
let searchTimer;
searchInput.addEventListener("input", () => {
  clearTimeout(searchTimer);
  searchTimer = setTimeout(() => doSearch(searchInput.value.trim()), 200);
});
searchInput.addEventListener("keydown", (e) => { if (e.key === "Escape") { searchInput.value=""; searchResults.empty(); } });

// Initial hint
span(searchResults, "Type to search across your entire vault…", "font-size:12px;color:var(--text-faint)");

// ═══════════════════════════════════════════════════
//  ③ QUICK CAPTURE
// ═══════════════════════════════════════════════════
sectionHead(C, "⚡ Quick Capture");
const captureCard = card(C);
p(captureCard, "Save notes, tasks, links and ideas directly into your vault.", "font-size:11px;color:var(--text-muted);margin:0 0 10px");
const captureArea = captureCard.createEl("textarea");
captureArea.placeholder = "Brain dump, idea, task, link…";
captureArea.style.cssText = `width:100%;min-height:70px;padding:10px 12px;background:var(--background-secondary);border:1px solid var(--background-modifier-border);border-radius:8px;color:var(--text-normal);font-size:13px;font-family:var(--font-interface);resize:vertical;box-sizing:border-box;outline:none;line-height:1.5`;
const captureBtns = div(captureCard, "display:flex;gap:8px;margin-top:10px;flex-wrap:wrap");
const showFeedback = (msg, isError) => { const fb = captureCard.createEl("p"); fb.textContent = msg; fb.style.cssText = `font-size:12px;color:${isError?"#DC2626":"#059669"};margin:6px 0 0`; setTimeout(() => fb.remove(), 3500); };
const makeCapBtn = (label, bg, hoverBg, handler) => {
  const b = captureBtns.createEl("button"); b.textContent = label;
  b.style.cssText = `background:${bg};color:#fff;border:none;border-radius:7px;padding:7px 16px;font-size:12px;font-weight:600;cursor:pointer;font-family:var(--font-interface)`;
  b.addEventListener("mouseenter", () => b.style.background = hoverBg);
  b.addEventListener("mouseleave", () => b.style.background = bg);
  b.addEventListener("click", handler); return b;
};
const appendToFile = async (filePath, line) => {
  let existing = ""; try { existing = await app.vault.adapter.read(filePath); } catch(e) {}
  await app.vault.adapter.write(filePath, existing + (existing.endsWith("\n") ? "" : "\n") + line + "\n");
};
const createInboxFile = async (prefix, content) => {
  const ts = now.toFormat("yyyy-MM-dd HH-mm-ss");
  const fileName = `${CFG.inboxFolder}/${prefix} ${ts}.md`;
  await app.vault.adapter.write(fileName, content); return fileName;
};
makeCapBtn("💬 Save note", "#7C3AED", "#6d28d9", async () => {
  const text = captureArea.value.trim(); if (!text) { captureArea.focus(); return; }
  const title = text.split("\n")[0].slice(0,40).replace(/[\\/:*?"<>|]/g,"");
  await app.vault.adapter.write(`${CFG.inboxFolder}/${title} ${now.toFormat("yyyy-MM-dd HH-mm")}.md`, `# ${title}\n\n${text}`);
  captureArea.value = ""; showFeedback(`✓ Saved: "${title}"`, false);
});
makeCapBtn("✅ Add task", "#059669", "#047857", async () => {
  const text = captureArea.value.trim(); if (!text) { captureArea.focus(); return; }
  showModal("Where should this task go?", [
    { label:"💼 Work Tasks", value:CFG.workTasksFile+".md" },
    { label:"🏠 Home Tasks", value:CFG.homeTasksFile+".md" },
  ], async (filePath) => {
    await appendToFile(filePath, `- [ ] ${text} [added:: ${now.toFormat("yyyy-MM-dd")}]`);
    captureArea.value = ""; showFeedback(`✓ Task added to ${filePath.includes("Work")?"Work":"Home"} Tasks`, false);
  });
});
makeCapBtn("🔗 Save link", "#2563EB", "#1d4ed8", async () => {
  const text = captureArea.value.trim(); if (!text) { captureArea.focus(); return; }
  let existing = ""; try { existing = await app.vault.adapter.read(CFG.savedLinksFile); } catch(e) { existing = "# Saved Links\n\n"; }
  await app.vault.adapter.write(CFG.savedLinksFile, existing + `- [${now.toFormat("yyyy-MM-dd HH:mm")}] ${text}\n`);
  captureArea.value = ""; showFeedback("✓ Link saved to Saved Links.md", false);
});
makeCapBtn("💡 Idea", "#D97706", "#b45309", async () => {
  const text = captureArea.value.trim(); if (!text) { captureArea.focus(); return; }
  const title = text.split("\n")[0].slice(0,40).replace(/[\\/:*?"<>|]/g,"");
  await createInboxFile("💡 Idea —", `# 💡 ${title}\n\n${text}`);
  captureArea.value = ""; showFeedback("✓ Idea saved as new note", false);
});
if (inbox.length > 0) {
  const unsortedRow = div(captureCard, "display:flex;align-items:center;gap:8px;margin-top:10px");
  span(unsortedRow, `📥 ${inbox.length} note${inbox.length>1?"s":""} waiting in Inbox To Sort`, "font-size:11px;color:var(--text-muted)");
  link(unsortedRow, "Open inbox →", CFG.inboxFolder, "font-size:11px;margin-left:4px");
}

// ═══════════════════════════════════════════════════
//  ④ TASKS
// ═══════════════════════════════════════════════════
const parseTaskSections = async (filePath) => {
  let raw = ""; try { raw = await app.vault.adapter.read(filePath + ".md"); } catch(e) { return []; }
  const lines = raw.split("\n"); const sections = []; let cur = null;
  for (const line of lines) {
    const hm = line.match(/^(#{1,4})\s+(.+)/);
    const tm = line.match(/^(\s*)- \[( |x)\] (.+)/);
    if (hm) {
      const txt = hm[2].replace(/\[([^\]]+)\]\([^)]+\)/g,"$1").replace(/\[\[([^\]|]+)(?:\|([^\]]+))?\]\]/g,(_,t,a)=>a||t).trim();
      cur = { heading:txt, level:hm[1].length, tasks:[] }; sections.push(cur);
    } else if (tm) {
      const indent=tm[1].length, completed=tm[2]==="x", text=tm[3];
      if (!cur) { cur={heading:null,level:0,tasks:[]}; sections.push(cur); }
      if (!completed) cur.tasks.push({ text, indent, filePath:filePath+".md" });
    }
  }
  return sections.filter(s => s.tasks.length > 0);
};

const makeTaskCard = async (parent, title, filePath, openList, doneList, accentCol) => {
  const tc = card(parent, "display:flex;flex-direction:column;max-height:480px;min-width:0");
  const header = div(tc, "display:flex;justify-content:space-between;align-items:center;margin-bottom:6px;flex-shrink:0");
  const titleEl = div(header, "display:flex;flex-direction:column;gap:2px");
  span(titleEl, title, "font-size:13px;font-weight:700;color:var(--text-normal)");
  span(titleEl, `${openList.length} open · ${doneList.length} done`, "font-size:10px;color:var(--text-muted)");
  link(header, "Open →", filePath, `font-size:11px;color:${accentCol};text-decoration:none`);
  const totalTC = openList.length+doneList.length;
  const pct = totalTC>0?Math.round((doneList.length/totalTC)*100):0;
  const pBar = div(tc,"height:3px;background:var(--background-modifier-border);border-radius:2px;overflow:hidden;margin-bottom:10px;flex-shrink:0");
  div(pBar,`height:100%;width:${pct}%;background:${accentCol};border-radius:2px`);
  const scrollEl = div(tc,"overflow-y:auto;flex:1;padding-right:4px");
  if (openList.length===0) { p(scrollEl,"All clear! 🎉","font-size:12px;color:var(--text-faint);margin:0"); return; }
  const sections = await parseTaskSections(filePath);
  const renderRow = (t, container) => {
    const row = div(container,"display:flex;align-items:flex-start;gap:8px;padding:4px 0;border-bottom:1px solid var(--background-modifier-border)");
    if (t.indent>0) row.style.paddingLeft=(t.indent*8)+"px";
    const cb = row.createEl("input"); cb.type="checkbox"; cb.checked=false;
    cb.style.cssText=`margin-top:3px;accent-color:${accentCol};cursor:pointer;flex-shrink:0`;
    cb.addEventListener("change", async()=>{
      if(cb.checked){ row.style.opacity="0.35"; try { const fc=await app.vault.adapter.read(t.filePath); const esc=t.text.replace(/[.*+?^${}()|[\]\\]/g,"\\$&"); await app.vault.adapter.write(t.filePath,fc.replace(new RegExp(`(- \\[) (\\] ${esc})`),"$1x$2")); } catch(e){} }
    });
    const tl=row.createEl("a",{cls:"internal-link"}); tl.href=t.filePath; tl.textContent=t.text.slice(0,55)+(t.text.length>55?"…":"");
    tl.style.cssText="font-size:12px;color:var(--text-normal);text-decoration:none;line-height:1.5;flex:1;cursor:pointer;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;min-width:0";
    tl.addEventListener("click",(e)=>{e.preventDefault();app.workspace.openLinkText(t.filePath,"","tab");});
  };
  if (sections.length>0) { for (const sec of sections) { if(sec.heading){const sh=div(scrollEl,"margin:10px 0 4px"); span(sh,sec.heading,`font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:0.05em;color:${accentCol}`);} for(const t of sec.tasks) renderRow(t,scrollEl); } }
  else { for(const t of openList){ renderRow({text:t.text,indent:0,filePath:t.path},scrollEl); } }
};

sectionHead(C, "Tasks");
const taskGrid = div(C,"display:grid;grid-template-columns:minmax(0,1fr) minmax(0,1fr);gap:10px");
await makeTaskCard(taskGrid,"💼 Work Tasks",CFG.workTasksFile,workTasks,workDone,"#2563EB");
await makeTaskCard(taskGrid,"🏠 Home Tasks",CFG.homeTasksFile,homeTasks,homeDone,"#059669");

// ═══════════════════════════════════════════════════
//  ⑤ ACTIVE PROJECTS  (with inline tasks)
// ═══════════════════════════════════════════════════
sectionHead(C, "Active Projects");
if (projects.length === 0) {
  p(C, `No notes found in "${CFG.projectsFolder}".`, "font-size:12px;color:var(--text-muted);font-style:italic");
} else {
  const statuses = [...new Set(projects.map(p => (p.status||"active").toLowerCase()))].sort();
  const projFilterBar = div(C, "display:flex;gap:6px;flex-wrap:wrap;margin-bottom:12px");
  let projFilter = "all";
  const projGrid = div(C, "display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));gap:10px");

  const renderProjects = async () => {
    projGrid.empty();
    const filtered = projFilter==="all" ? projects : projects.where(p => (p.status||"active").toLowerCase()===projFilter);
    if (filtered.length===0) { p(projGrid,`No projects with status "${projFilter}".`,"font-size:12px;color:var(--text-faint)"); return; }

    for (const proj of filtered.limit(9)) {
      const pc = card(projGrid, "display:flex;flex-direction:column;gap:6px");

      // Header row
      const ptop = div(pc, "display:flex;justify-content:space-between;align-items:flex-start;gap:8px");
      link(ptop, proj.file.name, proj.file.path, "font-size:13px;font-weight:600;line-height:1.4;flex:1");
      const status=(proj.status||"active").toLowerCase();
      const sc=status==="complete"||status==="done"?["#D1FAE5","#065F46"]:status==="paused"||status==="on hold"?["#FEF3C7","#92400E"]:["#EDE9FE","#5B21B6"];
      pill(ptop, status, sc[0], sc[1]);

      if (proj.description) p(pc, proj.description.slice(0,90)+(proj.description.length>90?"…":""), "font-size:11px;color:var(--text-muted);margin:0;line-height:1.5");

      // ── Inline tasks from the project file ──
      const projTasks = proj.file.tasks ? proj.file.tasks.where(t => !t.completed) : dv.array([]);
      const projDone  = proj.file.tasks ? proj.file.tasks.where(t =>  t.completed) : dv.array([]);
      const totalPT   = projTasks.length + projDone.length;

      if (totalPT > 0) {
        // Mini progress bar
        const ptPct = Math.round((projDone.length / totalPT) * 100);
        const ptBarWrap = div(pc, "margin-top:2px");
        const ptBarRow  = div(ptBarWrap, "display:flex;justify-content:space-between;align-items:center;margin-bottom:3px");
        span(ptBarRow, `${projTasks.length} open task${projTasks.length!==1?"s":""}`, "font-size:10px;color:var(--text-muted)");
        span(ptBarRow, ptPct+"%", "font-size:10px;font-weight:600;color:#7C3AED");
        const ptTrack = div(ptBarWrap, "height:3px;background:var(--background-modifier-border);border-radius:2px;overflow:hidden");
        div(ptTrack, `height:100%;width:${ptPct}%;background:#7C3AED;border-radius:2px`);

        // Show up to 3 open tasks
        if (projTasks.length > 0) {
          const ptList = div(pc, "margin-top:6px;display:flex;flex-direction:column;gap:0");
          for (const t of projTasks.limit(3)) {
            const trow = div(ptList, "display:flex;align-items:center;gap:6px;padding:3px 0;border-bottom:1px solid var(--background-modifier-border)");
            const cb = trow.createEl("input"); cb.type="checkbox"; cb.checked=false;
            cb.style.cssText="accent-color:#7C3AED;cursor:pointer;flex-shrink:0;margin:0";
            cb.addEventListener("change", async()=>{
              if(cb.checked){ trow.style.opacity="0.35"; try { const fc=await app.vault.adapter.read(t.path); const esc=t.text.replace(/[.*+?^${}()|[\]\\]/g,"\\$&"); await app.vault.adapter.write(t.path,fc.replace(new RegExp(`(- \\[) (\\] ${esc})`),"$1x$2")); } catch(e){} }
            });
            const tspan = trow.createEl("span"); tspan.textContent = t.text.slice(0,45)+(t.text.length>45?"…":"");
            tspan.style.cssText="font-size:11px;color:var(--text-muted);overflow:hidden;text-overflow:ellipsis;white-space:nowrap;flex:1";
          }
          if (projTasks.length > 3) {
            const more = div(pc,"margin-top:4px");
            const moreLink = link(more, `+${projTasks.length-3} more tasks…`, proj.file.path, "font-size:11px;color:#7C3AED");
          }
        }
      }

      // Footer
      const pBottom = div(pc,"display:flex;justify-content:space-between;align-items:center;margin-top:auto;padding-top:6px");
      span(pBottom,"Modified "+proj.file.mtime.toRelative(),"font-size:10px;color:var(--text-faint)");
      if (proj.progress!=null) { span(pBottom,proj.progress+"%","font-size:11px;font-weight:600;color:#7C3AED"); progressBar(pc,proj.progress,"#7C3AED"); }
    }
  };

  const projFBtns = [];
  const makeFilterPill = (label, val) => {
    const fp = projFilterBar.createEl("button"); fp.textContent=label; fp._val=val;
    fp.style.cssText=`font-size:11px;font-weight:600;padding:3px 12px;border-radius:20px;cursor:pointer;font-family:var(--font-interface);border:1px solid var(--background-modifier-border);background:${val==="all"?"#7C3AED":"var(--background-secondary)"};color:${val==="all"?"#fff":"var(--text-muted)"}`;
    fp.addEventListener("click",()=>{ projFilter=val; projFBtns.forEach(b=>{b.style.background=b._val===val?"#7C3AED":"var(--background-secondary)";b.style.color=b._val===val?"#fff":"var(--text-muted)";}); renderProjects(); });
    projFBtns.push(fp);
  };
  makeFilterPill("All","all");
  for (const s of statuses) makeFilterPill(s.charAt(0).toUpperCase()+s.slice(1), s);
  await renderProjects();
}

// ═══════════════════════════════════════════════════
//  ⑥ READING LIST
// ═══════════════════════════════════════════════════
sectionHead(C, "Reading List");
if (reading.length===0) { p(C,`No notes found in reading folder.`,"font-size:12px;color:var(--text-muted);font-style:italic"); }
else {
  const READ_TABS=[{label:"All",val:"all"},{label:"📖 Currently Reading",val:"currently reading"},{label:"📚 To Read",val:"to read"},{label:"✅ Completed",val:"completed"}];
  const statusMeta={"currently reading":["#DBEAFE","#1E40AF","📖"],"reading":["#DBEAFE","#1E40AF","📖"],"to read":["#EDE9FE","#5B21B6","📚"],"completed":["#D1FAE5","#065F46","✅"],"done":["#D1FAE5","#065F46","✅"],"paused":["#FEF3C7","#92400E","⏸"]};
  const normS=(raw)=>{const s=(raw||"to read").toLowerCase().trim();if(s==="reading")return"currently reading";if(s==="done"||s==="finished")return"completed";return s;};
  const readFilterBar=div(C,"display:flex;gap:6px;flex-wrap:wrap;margin-bottom:14px");
  let readFilter="all";
  const readGrid=div(C,"display:grid;grid-template-columns:repeat(auto-fill,minmax(160px,1fr));gap:12px");
  const renderReading=()=>{
    readGrid.empty();
    const filtered=readFilter==="all"?reading:reading.where(b=>normS(b.status)===readFilter);
    if(filtered.length===0){p(readGrid,"Nothing here yet.","font-size:12px;color:var(--text-faint);grid-column:1/-1");return;}
    for(const book of filtered.limit(4)){
      const st=normS(book.status),meta=statusMeta[st]||statusMeta["to read"],prog=book.progress!=null?Number(book.progress):0;
      const bc=card(readGrid,"display:flex;flex-direction:column;gap:0;padding:0;overflow:hidden;position:relative");
      if(book.cover){const imgWrap=div(bc,"position:relative;width:100%;padding-top:148%;overflow:hidden;background:var(--background-secondary)");const img=imgWrap.createEl("img");img.src=book.cover;img.style.cssText="position:absolute;top:0;left:0;width:100%;height:100%;object-fit:cover";img.onerror=()=>{img.style.display="none";};pill(div(imgWrap,"position:absolute;top:8px;right:8px"),book.status||"To Read",meta[0]+"cc",meta[1]);}
      else{const ph=div(bc,"background:var(--background-secondary);display:flex;align-items:center;justify-content:center;padding:28px 0;font-size:36px");ph.textContent=meta[2];pill(div(bc,"padding:6px 10px 0;display:flex;justify-content:flex-end"),book.status||"To Read",meta[0],meta[1]);}
      const body=div(bc,"padding:10px 12px 12px;display:flex;flex-direction:column;gap:4px;flex:1");
      link(body,book.file.name,book.file.path,"font-size:12px;font-weight:700;display:block;line-height:1.4");
      if(book.author)p(body,book.author,"font-size:10px;color:var(--text-muted);margin:0");
      const sliderRow=div(body,"margin-top:6px"),sliderTop=div(sliderRow,"display:flex;justify-content:space-between;align-items:center;margin-bottom:4px");
      span(sliderTop,"Progress","font-size:9px;color:var(--text-faint)");
      const pctLabel=span(sliderTop,prog+"%","font-size:9px;font-weight:700;color:#7C3AED");
      const slider=sliderRow.createEl("input");slider.type="range";slider.min=0;slider.max=100;slider.value=prog;
      slider.style.cssText="width:100%;accent-color:#7C3AED;cursor:pointer;height:3px";
      slider.addEventListener("input",()=>{pctLabel.textContent=slider.value+"%";});
      slider.addEventListener("change",async()=>{try{let c=await app.vault.adapter.read(book.file.path);if(/^progress:/m.test(c))c=c.replace(/^progress:.*$/m,`progress: ${slider.value}`);else if(/^---\n/.test(c))c=c.replace(/^---\n/,`---\nprogress: ${slider.value}\n`);else c=`---\nprogress: ${slider.value}\n---\n`+c;await app.vault.adapter.write(book.file.path,c);pctLabel.style.color="#059669";setTimeout(()=>{pctLabel.style.color="#7C3AED";},1500);}catch(e){}});
    }
  };
  const readFBtns=[];
  for(const tab of READ_TABS){const fp=readFilterBar.createEl("button");fp.textContent=tab.label;fp._val=tab.val;fp.style.cssText=`font-size:11px;font-weight:600;padding:4px 14px;border-radius:20px;cursor:pointer;font-family:var(--font-interface);border:1px solid var(--background-modifier-border);background:${tab.val==="all"?"#7C3AED":"var(--background-secondary)"};color:${tab.val==="all"?"#fff":"var(--text-muted)"}`;fp.addEventListener("click",()=>{readFilter=tab.val;readFBtns.forEach(b=>{b.style.background=b._val===tab.val?"#7C3AED":"var(--background-secondary)";b.style.color=b._val===tab.val?"#fff":"var(--text-muted)";});renderReading();});readFBtns.push(fp);}
  renderReading();
}

// ═══════════════════════════════════════════════════
//  ⑦ WATCH LIST
// ═══════════════════════════════════════════════════
sectionHead(C, "Watch List 🎬");
if (movies.length===0) { p(C,`No notes found in "${CFG.moviesFolder}".`,"font-size:12px;color:var(--text-muted);font-style:italic"); }
else {
  const WATCH_TABS=[{label:"All",val:"all"},{label:"🎬 To Watch",val:"to watch"},{label:"✅ Watched",val:"watched"}];
  const movieMeta={"to watch":["#EDE9FE","#5B21B6","🎬"],"watching":["#DBEAFE","#1E40AF","▶️"],"watched":["#D1FAE5","#065F46","✅"],"paused":["#FEF3C7","#92400E","⏸"]};
  const normM=(raw)=>{const s=(raw||"to watch").toLowerCase().trim();if(s==="done"||s==="seen"||s==="complete"||s==="completed")return"watched";return s;};
  const watchFilterBar=div(C,"display:flex;gap:6px;flex-wrap:wrap;margin-bottom:14px");
  let watchFilter="all";
  const watchGrid=div(C,"display:grid;grid-template-columns:repeat(auto-fill,minmax(160px,1fr));gap:12px");
  const renderMovies=()=>{
    watchGrid.empty();
    const filtered=watchFilter==="all"?movies:movies.where(m=>normM(m.status)===watchFilter);
    if(filtered.length===0){p(watchGrid,"Nothing here yet.","font-size:12px;color:var(--text-faint);grid-column:1/-1");return;}
    for(const movie of filtered.limit(4)){
      const st=normM(movie.status),meta=movieMeta[st]||movieMeta["to watch"];
      const mc=card(watchGrid,"display:flex;flex-direction:column;gap:0;padding:0;overflow:hidden;position:relative");
      if(movie.image){const imgWrap=div(mc,"position:relative;width:100%;padding-top:148%;overflow:hidden;background:var(--background-secondary)");const img=imgWrap.createEl("img");img.src=movie.image;img.style.cssText="position:absolute;top:0;left:0;width:100%;height:100%;object-fit:cover";img.onerror=()=>{img.style.display="none";};pill(div(imgWrap,"position:absolute;top:8px;right:8px"),movie.status||"To Watch",meta[0]+"cc",meta[1]);}
      else{const ph=div(mc,"background:var(--background-secondary);display:flex;align-items:center;justify-content:center;padding:28px 0;font-size:36px");ph.textContent=meta[2];pill(div(mc,"padding:6px 10px 0;display:flex;justify-content:flex-end"),movie.status||"To Watch",meta[0],meta[1]);}
      const body=div(mc,"padding:10px 12px 12px;display:flex;flex-direction:column;gap:4px;flex:1");
      link(body,movie.file.name,movie.file.path,"font-size:12px;font-weight:700;display:block;line-height:1.4");
      if(movie.genre)p(body,movie.genre,"font-size:10px;color:var(--text-muted);margin:0");
      if(movie.rating)span(body,"⭐".repeat(Math.min(5,Math.round(Number(movie.rating)))),"font-size:11px");
    }
  };
  const watchFBtns=[];
  for(const tab of WATCH_TABS){const fp=watchFilterBar.createEl("button");fp.textContent=tab.label;fp._val=tab.val;fp.style.cssText=`font-size:11px;font-weight:600;padding:4px 14px;border-radius:20px;cursor:pointer;font-family:var(--font-interface);border:1px solid var(--background-modifier-border);background:${tab.val==="all"?"#7C3AED":"var(--background-secondary)"};color:${tab.val==="all"?"#fff":"var(--text-muted)"}`;fp.addEventListener("click",()=>{watchFilter=tab.val;watchFBtns.forEach(b=>{b.style.background=b._val===tab.val?"#7C3AED":"var(--background-secondary)";b.style.color=b._val===tab.val?"#fff":"var(--text-muted)";});renderMovies();});watchFBtns.push(fp);}
  renderMovies();
}

// ═══════════════════════════════════════════════════
//  ⑧ INBOX TO SORT
// ═══════════════════════════════════════════════════
sectionHead(C, "📥 Inbox To Sort");
if (inbox.length===0) { p(C,"Inbox is empty — all sorted! 🎉","font-size:12px;color:var(--text-muted);font-style:italic"); }
else {
  const inboxGrid=div(C,"display:grid;grid-template-columns:repeat(auto-fill,minmax(220px,1fr));gap:8px");
  for(const note of inbox.limit(12)){const nc=card(inboxGrid,"display:flex;flex-direction:column;gap:4px;border-left:3px solid #EF4444");link(nc,note.file.name,note.file.path,"font-size:13px;font-weight:600;line-height:1.4");span(nc,"Created "+note.file.ctime.toRelative(),"font-size:10px;color:var(--text-faint)");}
  if(inbox.length>12)p(C,`+${inbox.length-12} more in inbox`,"font-size:11px;color:var(--text-faint);margin:6px 0 0");
}

// ═══════════════════════════════════════════════════
//  ⑨ RECENTLY MODIFIED
// ═══════════════════════════════════════════════════
sectionHead(C, "Recently Modified");
const recentGrid=div(C,"display:grid;grid-template-columns:repeat(auto-fill,minmax(260px,1fr));gap:10px");
for(const rn of recent){const rc=card(recentGrid,"display:flex;flex-direction:column;gap:4px");link(rc,rn.file.name,rn.file.path,"font-size:13px;font-weight:600;line-height:1.4");const rmeta=div(rc,"display:flex;align-items:center;gap:8px;margin-top:2px");span(rmeta,rn.file.mtime.toRelative(),"font-size:10px;color:var(--text-faint)");if(rn.file.folder)span(rmeta,rn.file.folder.split("/").pop(),"font-size:10px;color:#7C3AED;background:#EDE9FE;padding:1px 7px;border-radius:10px");}

// ═══════════════════════════════════════════════════
//  FOOTER
// ═══════════════════════════════════════════════════
const footer=div(C,"margin-top:36px;padding-top:16px;border-top:1px solid var(--background-modifier-border);display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:8px");
span(footer,`${allPages.length} notes  ·  ${openTasks.length} open tasks  ·  refreshed ${now.toFormat("HH:mm")}`,"font-size:11px;color:var(--text-faint)");
span(footer,"🔬 Research vault","font-size:11px;color:var(--text-faint)");
```