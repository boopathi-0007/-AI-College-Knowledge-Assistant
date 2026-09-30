# -AI-College-Knowledge-Assistant
1. db/schema.js
// db/schema.js
// Creates all tables if they don't already exist. Kept deliberately
// simple: one table per knowledge category, plus faqs, college_info
// (generic key/value facts) and admins (for the admin dashboard login).
const db = require("../config/db");
function initSchema() {
 db.exec(`
 CREATE TABLE IF NOT EXISTS departments (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 name TEXT NOT NULL,
 hod TEXT,
 description TEXT,
 location TEXT
 );
 CREATE TABLE IF NOT EXISTS faculty (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 name TEXT NOT NULL,
 designation TEXT,
 department_id INTEGER,
 email TEXT,
 specialization TEXT,
 FOREIGN KEY (department_id) REFERENCES departments (id) ON DELETE SET NULL
 );
 CREATE TABLE IF NOT EXISTS courses (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 name TEXT NOT NULL,
 department_id INTEGER,
 duration TEXT,
 description TEXT,
 FOREIGN KEY (department_id) REFERENCES departments (id) ON DELETE SET NULL
 );
 CREATE TABLE IF NOT EXISTS facilities (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 name TEXT NOT NULL,
 description TEXT,
 location TEXT,
 timings TEXT
 );
 CREATE TABLE IF NOT EXISTS rules (
 id INTEGER PRIMARY KEY AUTOINCREMENT,
 title TEXT NOT NULL,
 description TEXT
 );
 -- ... remaining tables (events, faqs, college_info, admins, chat_logs) omitted for 
brevity ...
24 `);
}
module.exports = initSchema;
2. utils/matcher.js
// utils/matcher.js
//
// A simple, explainable "AI" matcher: no external API calls, no API key.
// It builds a searchable list of knowledge items from every table in the
// database, then scores each item against the student's question using
// keyword overlap. This keeps the assistant fast, free, fully offline,
// and easy to explain in a project demo/viva.
const db = require("../config/db");
// A few common synonym groups so slightly different wording still matches
// (e.g. "HOD" vs "head of department", "timings" vs "hours").
const SYNONYMS = {
 hod: ["hod", "head", "headofdepartment"],
 timings: ["timings", "timing", "hours", "time", "schedule"],
 location: ["location", "located", "where", "address", "place"],
 facility: ["facility", "facilities", "amenities", "amenity"],
 faculty: ["faculty", "professor", "professors", "staff", "teacher", "teachers", 
"lecturer"],
 department: ["department", "departments", "dept", "branch"],
 course: ["course", "courses", "program", "programs", "programme", "degree"],
 rule: ["rule", "rules", "regulation", "regulations", "policy", "policies"],
 event: ["event", "events", "fest", "festival", "function"],
 admission: ["admission", "admissions", "apply", "applying", "enrol", "enroll", "join", 
"joining"],
 fee: ["fee", "fees", "payment", "cost"],
 contact: ["contact", "phone", "email", "reach"],
};
const STOPWORDS = new Set([
 "a", "an", "the", "is", "are", "was", "were", "of", "in", "on", "at", "to", "for",
 "what", "who", "when", "where", "how", "which", "does", "do", "can", "i", "we",
 "you", "please", "tell", "me", "about", "and", "or", "our", "college", "there",
 "be", "it", "this", "that", "by", "with", "from", "us", "my",
]);
function normalize(text) {
 return (text || "")
 .toLowerCase()
 .replace(/[^a-z0-9\s]/g, " ")
 .split(/\s+/)
 .filter(Boolean);
}
function expandTokens(tokens) {
 const expanded = new Set();
 for (const token of tokens) {
 expanded.add(token);
 for (const [canonical, group] of Object.entries(SYNONYMS)) {
 if (group.includes(token)) {
 expanded.add(canonical);
 group.forEach((g) => expanded.add(g));
 }
 }
 }
 return expanded;
}
25 function tokenize(text) {
 return normalize(text).filter((t) => !STOPWORDS.has(t));
}
// ... buildCorpus() omitted for brevity ...
// Scores one knowledge item against the (already expanded) question tokens.
// Title/keyword matches are weighted higher than plain body-text matches.
function scoreItem(item, queryTokens) {
 const titleTokens = expandTokens(tokenize(item.title));
 const keywordTokens = expandTokens(tokenize(item.keywords));
 const bodyTokens = expandTokens(tokenize(item.text));
 let score = 0;
 for (const token of queryTokens) {
 if (titleTokens.has(token)) score += 3;
 if (keywordTokens.has(token)) score += 2;
 if (bodyTokens.has(token)) score += 1;
 }
 return score;
}
const MIN_SCORE = 3; // below this, we treat the question as "unanswerable"
// ... list-intent helpers (detectListIntent, listAllAnswer) omitted for brevity ...
function findAnswer(question) {
 const rawTokens = tokenize(question);
 if (rawTokens.length === 0) {
 return {
 answer:
 "I didn't quite catch a question there. Could you rephrase it? For example: \"What
departments are available?\"",
 matched: false,
 category: null,
 score: 0,
 };
 }
 const listCategory = detectListIntent(rawTokens);
 if (listCategory) {
 const listAnswer = listAllAnswer(listCategory);
 if (listAnswer) {
 return { answer: listAnswer, matched: true, category: listCategory, score: 99 };
 }
 }
 const queryTokens = expandTokens(rawTokens);
 const corpus = buildCorpus();
 let best = null;
 for (const item of corpus) {
 const score = scoreItem(item, queryTokens);
 if (!best || score > best.score) {
 best = { ...item, score };
 }
 }
 if (!best || best.score < MIN_SCORE) {
 return {
 answer:
 "Sorry, that information is currently unavailable in the college knowledge base. 
Please try rephrasing your question, or check with the admin office.",
 matched: false,
26 category: best ? best.category : null,
 score: best ? best.score : 0,
 };
 }
 return {
 answer: best.text,
 matched: true,
 category: best.category,
 title: best.title,
 score: best.score,
 };
}
// Returns the top N matches (used by the /search endpoint, which shows
// several relevant results instead of a single best answer).
function searchKnowledge(query, limit = 10) {
 const rawTokens = tokenize(query);
 if (rawTokens.length === 0) return [];
 const queryTokens = expandTokens(rawTokens);
 const corpus = buildCorpus();
 return corpus
 .map((item) => ({ ...item, score: scoreItem(item, queryTokens) }))
 .filter((item) => item.score > 0)
 .sort((a, b) => b.score - a.score)
 .slice(0, limit);
}
module.exports = { findAnswer, searchKnowledge, buildCorpus };
