/*
AI File Transfer - MVP (Node.js + Express)

Files contained in this single-file MVP:
- server: Express server handling chunked uploads, intelligent compression, simple CDN routing, adaptive learning (SQLite), link generation, email sending stub.
- client: minimal HTML form for uploads (embedded below as string served at root)

How to run:
1. Create a project folder and save this file as server.js
2. Install dependencies:
   npm init -y
   npm install express multer uuid sharp sqlite3 nodemailer body-parser cors
   (ffmpeg is required for video compression — install on your OS separately and ensure `ffmpeg` is in PATH)
3. Run: node server.js
4. Open http://localhost:4000 in browser to try upload.

ENV variables (or set defaults in code):
- SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS
- BASE_URL (e.g. http://localhost:4000)

Notes & limitations (MVP):
- CDN optimization is simulated (choose from mirror list based on heuristic). Replace with real CDN or route logic.
- Video compression uses ffmpeg via child_process — ensure ffmpeg installed.
- Production requires secure link tokens, authentication, rate limiting, virus scanning, scalable storage (S3), and proper queueing (RabbitMQ/Kafka) for large files.

--------------------------------------------------------------------------------
*/

// ---------- server code ----------
const express = require('express');
const multer = require('multer');
const fs = require('fs');
const path = require('path');
const { v4: uuidv4 } = require('uuid');
const sharp = require('sharp');
const child_process = require('child_process');
const sqlite3 = require('sqlite3').verbose();
const nodemailer = require('nodemailer');
const bodyParser = require('body-parser');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

const PORT = process.env.PORT || 4000;
const UPLOAD_DIR = path.join(__dirname, 'uploads');
if (!fs.existsSync(UPLOAD_DIR)) fs.mkdirSync(UPLOAD_DIR);
const CHUNKS_DIR = path.join(__dirname, 'chunks');
if (!fs.existsSync(CHUNKS_DIR)) fs.mkdirSync(CHUNKS_DIR);

// simple SQLite DB for stats and file metadata
const DBPATH = path.join(__dirname, 'mvp.sqlite');
const db = new sqlite3.Database(DBPATH);

db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS files (
    id TEXT PRIMARY KEY,
    filename TEXT,
    mimetype TEXT,
    size INTEGER,
    path TEXT,
    created_at TEXT,
    download_count INTEGER DEFAULT 0,
    link TEXT
  )`);

  db.run(`CREATE TABLE IF NOT EXISTS stats (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    feature TEXT,
    value TEXT,
    created_at TEXT
  )`);

  db.run(`CREATE TABLE IF NOT EXISTS user_adapt (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    key TEXT UNIQUE,
    value TEXT
  )`);
});

// Simple transporter (configure envs for real emails)
const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST || 'smtp.example.com',
  port: process.env.SMTP_PORT ? Number(process.env.SMTP_PORT) : 587,
  secure: false,
  auth: {
    user: process.env.SMTP_USER || 'user@example.com',
    pass: process.env.SMTP_PASS || 'pass'
  }
});

// Simulated CDN mirrors
const MIRRORS = [
  { name: 'mirror-1', url: 'https://mirror1.example.com', avgLatencyMs: 50 },
  { name: 'mirror-2', url: 'https://mirror2.example.com', avgLatencyMs: 120 },
  { name: 'mirror-3', url: 'https://mirror3.example.com', avgLatencyMs: 30 }
];

// Utility: choose best mirror (in real product probe in real-time)
function chooseBestMirror() {
  // naive pick by min avgLatencyMs; later can use live telemetry
  MIRRORS.sort((a, b) => a.avgLatencyMs - b.avgLatencyMs);
  return MIRRORS[0];
}

// Utility: adjust compression quality based on past stats (adaptive learning stub)
function getAdaptiveQuality(key = 'default_quality') {
  return new Promise((resolve, reject) => {
    db.get('SELECT value FROM user_adapt WHERE key = ?', [key], (err, row) => {
      if (err) return resolve(80); // default
      if (!row) return resolve(80);
      const v = Number(row.value);
      if (isNaN(v)) return resolve(80);
      resolve(v);
    });
  });
}

function setAdaptiveQuality(key, value) {
  db.run('INSERT INTO user_adapt(key, value) VALUES(?,?) ON CONFLICT(key) DO UPDATE SET value=excluded.value', [key, String(value)]);
}

// Multer storage for completed single uploads
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, UPLOAD_DIR),
  filename: (req, file, cb) => cb(null, uuidv4() + path.extname(file.originalname))
});
const upload = multer({ storage });

// Root -> simple client HTML
app.get('/', (req, res) => {
  res.type('html').send(`<!doctype html>
<html>
<head><meta charset="utf-8"><title>AI File Transfer MVP</title></head>
<body>
<h3>AI File Transfer - MVP</h3>
<form id="f" enctype="multipart/form-data">
  <input type="file" name="file" id="file" /> <br/><br/>
  <button type="button" onclick="upload()">Upload</button>
</form>
<div id="out"></div>
<script>
async function upload(){
  const file = document.getElementById('file').files[0];
  if(!file) return alert('pick a file');
  // simple chunking demo: split into 1MB chunks
  const chunkSize = 1024*1024;
  const total = Math.ceil(file.size / chunkSize);
  const uploadId = Date.now() + '-' + Math.random().toString(36).slice(2,8);
  for(let i=0;i<total;i++){
    const start = i*chunkSize;
    const end = Math.min(start+chunkSize, file.size);
    const blob = file.slice(start,end);
    const form = new FormData();
    form.append('chunk', blob);
    form.append('uploadId', uploadId);
    form.append('index', i);
    form.append('total', total);
    form.append('filename', file.name);
    const r = await fetch('/upload-chunk', { method: 'POST', body: form });
    const j = await r.json();
    document.getElementById('out').innerText = `Uploaded chunk ${i+1}/${total}`;
  }
  const assemble = await fetch('/assemble', { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({ uploadId, filename:file.name }) });
  const result = await assemble.json();
  document.getElementById('out').innerHTML = 'Done. Download link: <a href="'+result.downloadUrl+'" target="_blank">'+result.downloadUrl+'</a>';
}
</script>
</body>
</html>
`);
});

// Endpoint: receive a chunk
const chunkUpload = multer({ dest: CHUNKS_DIR });
app.post('/upload-chunk', chunkUpload.single('chunk'), async (req, res) => {
  const { uploadId, index, total, filename } = req.body;
  if (!uploadId || typeof index === 'undefined') return res.status(400).json({ error: 'missing params' });
  const chunkPath = path.join(CHUNKS_DIR, `${uploadId}-${index}`);
  await fs.promises.rename(req.file.path, chunkPath);
  return res.json({ ok: true });
});

// Assemble chunks
app.post('/assemble', async (req, res) => {
  const { uploadId, filename } = req.body;
  const chunkFiles = (await fs.promises.readdir(CHUNKS_DIR)).filter(f => f.startsWith(uploadId + '-'));
  chunkFiles.sort((a,b)=>{
    const ai = Number(a.split('-').pop());
    const bi = Number(b.split('-').pop());
    return ai - bi;
  });
  const outName = uuidv4() + '-' + filename.replace(/[^a-zA-Z0-9.\-]/g,'_');
  const outPath = path.join(UPLOAD_DIR, outName);
  const ws = fs.createWriteStream(outPath);
  for (const f of chunkFiles) {
    const data = await fs.promises.readFile(path.join(CHUNKS_DIR, f));
    ws.write(data);
    await fs.promises.unlink(path.join(CHUNKS_DIR, f));
  }
  ws.end();
  ws.on('finish', async () => {
    const stat = await fs.promises.stat(outPath);
    // run intelligent compression
    const compressedPath = await intelligentCompress(outPath, filename);
    // choose mirror (simulated)
    const mirror = chooseBestMirror();
    // store metadata + link
    const id = uuidv4();
    const link = `${process.env.BASE_URL || 'http://localhost:' + PORT}/download/${id}`;
    db.run('INSERT INTO files(id,filename,mimetype,size,path,created_at,link) VALUES(?,?,?,?,?,?,?)', [id, filename, getMime(filename), stat.size, compressedPath, new Date().toISOString(), link]);

    // send email (example) - in real: recipient supplied
    // transporter.sendMail({from:'no-reply@example.com', to:'recipient@example.com', subject:'File ready', text: `Download: ${link}`});

    res.json({ ok: true, downloadUrl: link, mirror });
  });
});

// Download endpoint
app.get('/download/:id', (req, res) => {
  const id = req.params.id;
  db.get('SELECT * FROM files WHERE id = ?', [id], (err, row) => {
    if (err || !row) return res.status(404).send('Not found');
    const filePath = row.path;
    // increment download_count
    db.run('UPDATE files SET download_count = download_count + 1 WHERE id = ?', [id]);
    res.download(filePath, row.filename);
  });
});

// Simple stats endpoint
app.get('/stats', (req, res) => {
  db.all('SELECT id, filename, size, created_at, download_count FROM files ORDER BY created_at DESC LIMIT 20', [], (err, rows) => {
    if (err) return res.status(500).json({ error: 'db' });
    res.json(rows);
  });
});

// Helpers
function getMime(name){
  const ext = path.extname(name).toLowerCase();
  if(['.jpg','.jpeg','.png','.webp','.gif'].includes(ext)) return 'image';
  if(['.mp4','.mov','.mkv','.webm'].includes(ext)) return 'video';
  return 'document';
}

async function intelligentCompress(filePath, originalName){
  const type = getMime(originalName);
  const outPath = filePath + '.cmp';
  if (type === 'image'){
    const quality = await getAdaptiveQuality('image_quality');
    await sharp(filePath)
      .jpeg({ quality: quality })
      .toFile(outPath);
    // update adapt (naive learning: if compressed small enough, reduce quality next time)
    const origSize = (await fs.promises.stat(filePath)).size;
    const newSize = (await fs.promises.stat(outPath)).size;
    adaptQuality('image_quality', quality, origSize, newSize);
    await fs.promises.unlink(filePath);
    return outPath;
  } else if (type === 'video'){
    // require ffmpeg installed on host
    const quality = await getAdaptiveQuality('video_crf'); // CRF for ffmpeg
    const crf = Math.max(18, Math.min(32, Math.round(quality || 23)));
    await new Promise((resolve, reject) => {
      const cmd = `ffmpeg -y -i "${filePath}" -vcodec libx264 -crf ${crf} "${outPath}"`;
      child_process.exec(cmd, (err, stdout, stderr) => {
        if (err) return resolve();
        resolve();
      });
    });
    try{ await fs.promises.unlink(filePath); }catch(e){}
    return outPath;
  } else {
    // document: gzip as MVP
    await new Promise((resolve, reject) => {
      const rs = fs.createReadStream(filePath);
      const zlib = require('zlib');
      const ws = fs.createWriteStream(outPath);
      rs.pipe(zlib.createGzip()).pipe(ws).on('finish', resolve);
    });
    await fs.promises.unlink(filePath);
    return outPath;
  }
}

async function adaptQuality(key, prevQuality, origSize, newSize) {
  // naive: if compression ratio < 0.9, decrease quality a bit (more compression), else increase quality slightly
  const ratio = newSize / origSize;
  let next = Number(prevQuality) || 80;
  if (ratio > 0.95) next = Math.max(30, next - 5); // not compressing enough -> reduce quality
  else if (ratio < 0.6) next = Math.min(95, next + 5); // very small -> can improve quality
  setAdaptiveQuality(key, next);
  db.run('INSERT INTO stats(feature,value,created_at) VALUES(?,?,?)', [key, `${prevQuality}->${next} ratio=${ratio.toFixed(2)}`, new Date().toISOString()]);
}

app.listen(PORT, () => console.log('AI File Transfer MVP running on', PORT));

// EOF
