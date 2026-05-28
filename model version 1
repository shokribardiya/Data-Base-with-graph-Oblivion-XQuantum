"""
Oblivion XQDatabase – Professional Knowledge Graph Server (Upgraded)
====================================================================
- Cumulative graph grows with every imported file
- Stores sentences and presents article‑like summaries for each node
- Imports .txt, .docx, .py, .ipynb, .md and any plain‑text file
- Search returns fluent, article‑style excerpts from your documents
- Pure Python stdlib – no external dependencies
"""
import http.server, socketserver, json, os, re, io, zipfile
import xml.etree.ElementTree as ET, threading, sys, traceback
import urllib.parse, cgi, shutil, math, random, uuid, sqlite3
from collections import defaultdict, deque, Counter
from pathlib import Path
from datetime import datetime

# ─── config ──────────────────────────────────────────────────────
HOST, PORT = "127.0.0.1", 8080
DATA_DIR = Path("data")
UPLOAD_DIR = DATA_DIR / "files"
DB_PATH = DATA_DIR / "knowledge.db"
for d in [DATA_DIR, UPLOAD_DIR]:
    d.mkdir(parents=True, exist_ok=True)

STOPWORDS = frozenset({
    "a","an","the","and","or","but","if","because","as","what","which",
    "this","that","these","those","then","just","so","than","such","both",
    "through","about","for","is","of","while","during","to","from","in",
    "on","at","by","with","without","up","down","out","off","over","under",
    "again","further","once","here","there","when","where","why","how",
    "all","each","every","few","more","most","other","some","no","nor",
    "not","only","own","same","too","very","can","will","should","now",
    "also","after","before","between","above","below","i","me","my","we",
    "our","you","he","she","it","they","am","is","are","was","were","be",
    "been","being","have","has","had","doing","do","does","did","would",
    "could","should","may","might","must","shall","can","need","dare","used",
    "و","در","به","از","که","با","برای","تا","را","این","آن","است","هست",
    "بود","شد","شده","می","ها","های","هر","هم","نیز","اما","یا","اگر",
    "چون","چه","چرا","کجا","کی","کدام","همه","یک","دو","سه","نه","بله",
    "خیر","بعد","قبل","بالا","پایین","بیش","کم","بزرگ","کوچک","خوب",
    "بد","نو","کهنه","اول","آخر","تنها","چنین","چنان","همان","همین",
    "آنجا","اینجا","من","تو","او","ما","شما","آنها","ایشان","خود","خویش",
    "همدیگر","یکدیگر","هستند","باشند","بودند","شوند","گردند","کرد","کرده",
    "کن","کنید","گفت","گفته","گوی","گو","ز","بر","اند","ای","ام","ات",
    "اش","مان","تان","شان"
})

# ─── SQLite graph database (professional schema) ─────────────────
class GraphDB:
    def __init__(self, path):
        self.path = path
        self._init()

    def _init(self):
        with sqlite3.connect(self.path) as conn:
            conn.execute("PRAGMA foreign_keys = ON")
            conn.executescript("""
            CREATE TABLE IF NOT EXISTS graphs (
                id TEXT PRIMARY KEY,
                name TEXT UNIQUE,
                created REAL DEFAULT (julianday('now'))
            );
            CREATE TABLE IF NOT EXISTS nodes (
                id TEXT PRIMARY KEY,
                graph_id TEXT,
                label TEXT,
                x REAL,
                y REAL,
                size REAL DEFAULT 10,
                shape TEXT DEFAULT 'circle',
                color TEXT,
                community INT DEFAULT -1,
                betweenness REAL DEFAULT 0,
                source_file TEXT DEFAULT '',
                created_at TEXT DEFAULT (datetime('now')),
                FOREIGN KEY(graph_id) REFERENCES graphs(id) ON DELETE CASCADE
            );
            CREATE TABLE IF NOT EXISTS edges (
                id TEXT PRIMARY KEY,
                graph_id TEXT,
                source TEXT,
                target TEXT,
                weight REAL DEFAULT 1,
                source_file TEXT DEFAULT '',
                created_at TEXT DEFAULT (datetime('now')),
                FOREIGN KEY(graph_id) REFERENCES graphs(id) ON DELETE CASCADE,
                FOREIGN KEY(source) REFERENCES nodes(id) ON DELETE CASCADE,
                FOREIGN KEY(target) REFERENCES nodes(id) ON DELETE CASCADE
            );
            CREATE INDEX IF NOT EXISTS idx_nodes_graph ON nodes(graph_id);
            CREATE INDEX IF NOT EXISTS idx_edges_graph ON edges(graph_id);
            CREATE VIRTUAL TABLE IF NOT EXISTS nodes_fts USING fts5(
                label, content='nodes', content_rowid='rowid'
            );
            -- new tables for storing sentences and linking them to nodes
            CREATE TABLE IF NOT EXISTS sentences (
                id TEXT PRIMARY KEY,
                graph_id TEXT,
                filename TEXT,
                text TEXT,
                created_at TEXT DEFAULT (datetime('now')),
                FOREIGN KEY(graph_id) REFERENCES graphs(id) ON DELETE CASCADE
            );
            CREATE TABLE IF NOT EXISTS sentence_nodes (
                sentence_id TEXT,
                node_id TEXT,
                PRIMARY KEY (sentence_id, node_id),
                FOREIGN KEY(sentence_id) REFERENCES sentences(id) ON DELETE CASCADE,
                FOREIGN KEY(node_id) REFERENCES nodes(id) ON DELETE CASCADE
            );
            """)
            self._migrate(conn)
            conn.commit()

    def _migrate(self, conn):
        """Add columns that may be missing from older databases."""
        try:
            conn.execute("ALTER TABLE nodes ADD COLUMN source_file TEXT DEFAULT ''")
        except sqlite3.OperationalError:
            pass
        try:
            conn.execute("ALTER TABLE nodes ADD COLUMN created_at TEXT DEFAULT (datetime('now'))")
        except sqlite3.OperationalError:
            pass
        try:
            conn.execute("ALTER TABLE edges ADD COLUMN source_file TEXT DEFAULT ''")
        except sqlite3.OperationalError:
            pass
        try:
            conn.execute("ALTER TABLE edges ADD COLUMN created_at TEXT DEFAULT (datetime('now'))")
        except sqlite3.OperationalError:
            pass

    def create_graph(self, name):
        gid = str(uuid.uuid4())
        with sqlite3.connect(self.path) as conn:
            conn.execute("INSERT INTO graphs (id, name) VALUES(?,?)", (gid, name))
        return gid

    def get_or_create_master(self):
        with sqlite3.connect(self.path) as conn:
            row = conn.execute(
                "SELECT id FROM graphs WHERE name='Master Graph'"
            ).fetchone()
            if row:
                return row[0]
        return self.create_graph("Master Graph")

    def add_node(self, gid, label, **kw):
        nid = str(uuid.uuid4())
        with sqlite3.connect(self.path) as conn:
            conn.execute(
                """INSERT INTO nodes
                   (id, graph_id, label, x, y, size, shape, color,
                    community, betweenness, source_file)
                   VALUES(?,?,?,?,?,?,?,?,?,?,?)""",
                (nid, gid, label,
                 kw.get('x', random.uniform(100, 700)),
                 kw.get('y', random.uniform(100, 500)),
                 kw.get('size', 10),
                 kw.get('shape', 'circle'),
                 kw.get('color', '#%06x' % random.randint(0, 0xFFFFFF)),
                 kw.get('community', -1),
                 kw.get('betweenness', 0.0),
                 kw.get('source_file', ''))
            )
        return nid

    def add_edge(self, gid, src, dst, weight=1, source_file=''):
        eid = str(uuid.uuid4())
        with sqlite3.connect(self.path) as conn:
            conn.execute(
                "INSERT INTO edges (id,graph_id,source,target,weight,source_file) VALUES(?,?,?,?,?,?)",
                (eid, gid, src, dst, weight, source_file)
            )
        return eid

    def get_graph(self, gid):
        with sqlite3.connect(self.path) as conn:
            g = conn.execute(
                "SELECT id, name FROM graphs WHERE id=?", (gid,)
            ).fetchone()
            if not g:
                return None
            nodes = []
            for r in conn.execute(
                "SELECT id, label, x, y, size, shape, color, community, betweenness, source_file "
                "FROM nodes WHERE graph_id=?", (gid,)
            ):
                nodes.append({
                    "id": r[0], "label": r[1], "x": r[2], "y": r[3],
                    "size": r[4], "shape": r[5], "color": r[6],
                    "community": r[7], "betweenness": r[8],
                    "source_file": r[9]
                })
            edges = []
            for r in conn.execute(
                "SELECT id, source, target, weight, source_file FROM edges WHERE graph_id=?", (gid,)
            ):
                edges.append({
                    "id": r[0], "source": r[1], "target": r[2],
                    "weight": r[3], "source_file": r[4]
                })
            return {"graph": {"id": g[0], "name": g[1]}, "nodes": nodes, "edges": edges}

    def get_node_details(self, node_id):
        """Return node info + incident edges + article composed from sentences."""
        with sqlite3.connect(self.path) as conn:
            node = conn.execute(
                "SELECT id, label, x, y, size, shape, color, community, betweenness, source_file "
                "FROM nodes WHERE id=?", (node_id,)
            ).fetchone()
            if not node:
                return None
            node_data = {
                "id": node[0], "label": node[1], "x": node[2], "y": node[3],
                "size": node[4], "shape": node[5], "color": node[6],
                "community": node[7], "betweenness": node[8], "source_file": node[9]
            }
            edges = []
            for r in conn.execute(
                "SELECT id, source, target, weight, source_file "
                "FROM edges WHERE source=? OR target=?", (node_id, node_id)
            ):
                edges.append({
                    "id": r[0], "source": r[1], "target": r[2],
                    "weight": r[3], "source_file": r[4]
                })
            # collect sentences linked to this node
            sentences = []
            for r in conn.execute("""
                SELECT s.text, s.filename
                FROM sentences s
                JOIN sentence_nodes sn ON s.id = sn.sentence_id
                WHERE sn.node_id = ?
                ORDER BY s.filename, s.created_at
            """, (node_id,)):
                sentences.append({"text": r[0], "filename": r[1]})
            # Build an article‑like string from the collected sentences
            article = "\n\n".join(f"[{s['filename']}] {s['text']}" for s in sentences[:15])  # limit
            if not article:
                article = "No source sentences found for this term."
            return {"node": node_data, "edges": edges, "article": article}

    def search_nodes(self, gid, query):
        """FTS search with prefix wildcard for partial matches."""
        with sqlite3.connect(self.path) as conn:
            safe = query.strip()
            if not any(c in safe for c in '*"^'):
                safe = safe + '*'
            rows = conn.execute(
                """SELECT n.id, n.label, n.x, n.y, n.size, n.shape,
                          n.color, n.community, n.betweenness, n.source_file
                   FROM nodes n
                   JOIN nodes_fts f ON n.rowid = f.rowid
                   WHERE n.graph_id = ? AND f.label MATCH ?""",
                (gid, safe)
            ).fetchall()
            return [{
                "id": r[0], "label": r[1], "x": r[2], "y": r[3],
                "size": r[4], "shape": r[5], "color": r[6],
                "community": r[7], "betweenness": r[8],
                "source_file": r[9]
            } for r in rows]

    def update_node(self, nid, **kw):
        sets = ', '.join(f"{k}=?" for k in kw)
        with sqlite3.connect(self.path) as conn:
            conn.execute(f"UPDATE nodes SET {sets} WHERE id=?", (*kw.values(), nid))

    def node_exists(self, label, gid):
        with sqlite3.connect(self.path) as conn:
            return conn.execute(
                "SELECT id FROM nodes WHERE graph_id=? AND label=?", (gid, label)
            ).fetchone() is not None

    def get_node_id(self, label, gid):
        """Return node id by label, or None."""
        with sqlite3.connect(self.path) as conn:
            r = conn.execute(
                "SELECT id FROM nodes WHERE graph_id=? AND label=?", (gid, label)
            ).fetchone()
            return r[0] if r else None

    def store_sentence(self, gid, filename, text, node_ids):
        """Insert a sentence and link it to every node_id in the list."""
        sid = str(uuid.uuid4())
        with sqlite3.connect(self.path) as conn:
            conn.execute(
                "INSERT INTO sentences (id, graph_id, filename, text) VALUES(?,?,?,?)",
                (sid, gid, filename, text)
            )
            for nid in node_ids:
                conn.execute(
                    "INSERT OR IGNORE INTO sentence_nodes (sentence_id, node_id) VALUES(?,?)",
                    (sid, nid)
                )
        return sid

db = GraphDB(DB_PATH)
master_gid = db.get_or_create_master()

# ─── text processing ──────────────────────────────────────────────
def tokenize(text):
    return [w.lower() for w in re.findall(r'\b\w+\b', text) if w.isalpha() and len(w) > 1]

def lemma(w):
    for sf in ['ترین','تر','ها','ان','ات','ی','ای','ing','ed','ly','ment','ness','tion','s','es','er','est']:
        if w.endswith(sf) and len(w) > len(sf) + 2:
            w = w[:-len(sf)]
            break
    return w

def extract_sentences(text):
    """Split text into sentences using punctuation."""
    # simple split by .!? followed by space or end of line
    return re.split(r'(?<=[.!?])\s+', text)

def build_graph_from_text(text, gid, source_file=''):
    tokens = tokenize(text)
    lemmas = [lemma(t) for t in tokens if t not in STOPWORDS]
    if not lemmas:
        return 0
    freq = Counter(lemmas)
    maxf = max(freq.values()) if freq else 1

    # adjacency
    adj = defaultdict(lambda: defaultdict(float))
    win = 4
    for i, w in enumerate(lemmas):
        for j in range(max(0, i - win), min(len(lemmas), i + win + 1)):
            if i != j:
                adj[w][lemmas[j]] += 1

    # nodes – create new ones or update existing ones
    node_ids = {}
    for w, f in freq.items():
        nid = db.get_node_id(w, gid)
        if nid:
            # Update existing node: append source file if not already listed
            with sqlite3.connect(DB_PATH) as conn:
                row = conn.execute(
                    "SELECT source_file, size FROM nodes WHERE id=?", (nid,)
                ).fetchone()
            old_sources = row[0] if row else ''
            if source_file and source_file not in old_sources:
                new_sources = (old_sources + ',' + source_file).strip(',')
            else:
                new_sources = old_sources
            # increase size by a small fraction (cumulative growth)
            old_size = row[1] if row else 10
            new_size = old_size + (5 + (f / maxf) * 10)  # add a bonus
            db.update_node(nid, source_file=new_sources, size=new_size)
            node_ids[w] = nid
        else:
            size = 5 + (f / maxf) * 25
            nid = db.add_node(gid, w, size=size, source_file=source_file)
            node_ids[w] = nid

    # edges
    for src, neigh in adj.items():
        if src not in node_ids or node_ids[src] is None:
            continue
        for dst, wgt in neigh.items():
            if src < dst and dst in node_ids and node_ids[dst] is not None:
                # check if edge already exists? for simplicity we add (idempotent with unique constraint? not enforced)
                db.add_edge(gid, node_ids[src], node_ids[dst], wgt, source_file=source_file)

    # Louvain
    comms = louvain(adj)
    for w, c in comms.items():
        if w in node_ids and node_ids[w]:
            db.update_node(node_ids[w], community=c)

    # betweenness
    bcs = betweenness(adj)
    for w, bc in bcs.items():
        if w in node_ids and node_ids[w]:
            db.update_node(node_ids[w], betweenness=bc)

    # ----- new: store sentences and link to nodes -----
    sentences = extract_sentences(text)
    for sent in sentences:
        sent = sent.strip()
        if len(sent) < 10:  # skip very short fragments
            continue
        # tokenize+lemmatize the sentence to find which nodes it mentions
        sent_tokens = tokenize(sent)
        sent_lemmas = {lemma(t) for t in sent_tokens if t not in STOPWORDS}
        linked_node_ids = []
        for lem in sent_lemmas:
            if lem in node_ids and node_ids[lem] is not None:
                linked_node_ids.append(node_ids[lem])
        if linked_node_ids:
            db.store_sentence(gid, source_file, sent, linked_node_ids)

    return len(node_ids)

def louvain(adj, max_iter=50):
    nodes = list(adj.keys())
    if not nodes:
        return {}
    comm = {n: i for i, n in enumerate(nodes)}
    m = sum(sum(nb.values()) for nb in adj.values()) / 2
    if m == 0:
        return comm
    for _ in range(max_iter):
        changed = False
        random.shuffle(nodes)
        for n in nodes:
            cur = comm[n]
            nb_comms = {comm[nb] for nb in adj[n] if nb in comm}
            comm_w = defaultdict(float)
            for nn, cc in comm.items():
                for nb, w in adj[nn].items():
                    if comm.get(nb) == cc:
                        comm_w[cc] += w
            best_gain, best = 0, cur
            for c in nb_comms:
                k_i = sum(adj[n].values())
                k_i_in = sum(w for nb, w in adj[n].items() if comm.get(nb) == c)
                sum_tot = comm_w.get(c, 0)
                gain = (k_i_in / (2 * m)) - (sum_tot * k_i / (2 * m) ** 2)
                if gain > best_gain:
                    best_gain, best = gain, c
            if best != cur:
                comm[n] = best
                changed = True
        if not changed:
            break
    mapping = {o: i for i, o in enumerate(sorted(set(comm.values())))}
    return {n: mapping[c] for n, c in comm.items()}

def betweenness(adj):
    nodes = list(adj.keys())
    C = {n: 0.0 for n in nodes}
    for s in nodes:
        S, P = [], {n: [] for n in nodes}
        sigma = {n: 0 for n in nodes}
        sigma[s] = 1
        d = {n: -1 for n in nodes}
        d[s] = 0
        Q = deque([s])
        while Q:
            v = Q.popleft()
            S.append(v)
            for w in adj[v]:
                if d[w] < 0:
                    Q.append(w)
                    d[w] = d[v] + 1
                if d[w] == d[v] + 1:
                    sigma[w] += sigma[v]
                    P[w].append(v)
        delta = {n: 0.0 for n in nodes}
        while S:
            w = S.pop()
            for v in P[w]:
                delta[v] += (sigma[v] / sigma[w]) * (1 + delta[w])
            if w != s:
                C[w] += delta[w]
    n = len(nodes)
    if n > 2:
        for nn in C:
            C[nn] /= ((n - 1) * (n - 2))
    return C

def extract_docx(path):
    try:
        with zipfile.ZipFile(path) as z:
            xml = z.read('word/document.xml')
            root = ET.fromstring(xml)
            ns = 'http://schemas.openxmlformats.org/wordprocessingml/2006/main'
            paragraphs = []
            for p in root.iter(f'{{{ns}}}p'):
                texts = [t.text or '' for t in p.iter(f'{{{ns}}}t')]
                paragraphs.append(''.join(texts))
            return '\n'.join(paragraphs)
    except Exception:
        return ""

def extract_ipynb_text(path):
    """Extract all text (code + markdown) from a Jupyter notebook."""
    try:
        with open(path, 'r', encoding='utf-8') as f:
            nb = json.load(f)
        texts = []
        for cell in nb.get('cells', []):
            if cell.get('cell_type') in ('code', 'markdown'):
                texts.append(''.join(cell.get('source', [])))
        return '\n'.join(texts)
    except Exception:
        return ""

def read_text_file(path):
    """Attempt to read file as UTF-8 text; if binary, return empty."""
    try:
        return Path(path).read_text(encoding='utf-8')
    except Exception:
        return ""

# ─── HTTP request handler ────────────────────────────────────────
class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        p = urllib.parse.urlparse(self.path)
        q = urllib.parse.parse_qs(p.query)
        if p.path == '/' or p.path == '/dashboard':
            self._dash()
        elif p.path == '/api/master':
            self._json(db.get_graph(master_gid))
        elif p.path == '/api/search':
            self._search(q)
        elif p.path == '/api/node':
            self._node_detail(q)
        elif p.path == '/api/files':
            self._json([f.name for f in UPLOAD_DIR.iterdir() if f.is_file()])
        elif p.path == '/api/file':
            self._file(q)
        else:
            self.send_error(404)

    def do_POST(self):
        p = urllib.parse.urlparse(self.path)
        if p.path == '/api/import':
            self._import()
        elif p.path == '/api/run':
            self._run()
        elif p.path == '/api/notebook':
            self._notebook()
        else:
            self.send_error(404)

    def _dash(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html; charset=utf-8')
        self.end_headers()
        self.wfile.write(DASHBOARD.encode())

    def _json(self, data, code=200):
        self.send_response(code)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps(data, ensure_ascii=False).encode())

    def _search(self, q):
        term = q.get('q', [''])[0]
        gid = q.get('gid', [master_gid])[0]
        if not term:
            self._json([])
            return
        self._json(db.search_nodes(gid, term))

    def _node_detail(self, q):
        nid = q.get('id', [None])[0]
        if not nid:
            self._json({'error': 'missing id'}, 400)
            return
        details = db.get_node_details(nid)
        if not details:
            self._json({'error': 'node not found'}, 404)
            return
        self._json(details)

    def _file(self, q):
        name = q.get('name', [None])[0]
        if not name:
            self._json({'error': 'no name'}, 400)
            return
        path = UPLOAD_DIR / name
        if not path.exists():
            self._json({'error': 'not found'}, 404)
            return
        try:
            self._json({'content': path.read_text(encoding='utf-8')})
        except Exception:
            self._json({'error': 'binary'}, 400)

    def _import(self):
        ct = self.headers.get('Content-Type', '')
        if 'multipart/form-data' not in ct:
            self._json({'error': 'multipart required'}, 400)
            return
        form = cgi.FieldStorage(
            fp=self.rfile,
            headers=self.headers,
            environ={'REQUEST_METHOD': 'POST', 'CONTENT_TYPE': ct}
        )
        item = form['file']
        if not item.file:
            self._json({'error': 'no file'}, 400)
            return
        fname = Path(item.filename).name
        dest = UPLOAD_DIR / fname
        with open(dest, 'wb') as f:
            shutil.copyfileobj(item.file, f)

        # Determine file type and extract text
        ext = fname.lower().rsplit('.', 1)[-1] if '.' in fname else ''
        text = ""
        if ext in ('txt', 'py', 'md', 'csv', 'rst', 'tex', 'log'):
            text = read_text_file(dest)
        elif ext == 'docx':
            text = extract_docx(dest)
        elif ext == 'ipynb':
            text = extract_ipynb_text(dest)
        else:
            # try plain text for unknown extensions
            text = read_text_file(dest)

        if text.strip():
            added = build_graph_from_text(text, master_gid, source_file=fname)
        else:
            added = 0
        self._json({'status': 'ok', 'filename': fname, 'new_nodes': added})

    def _run(self):
        length = int(self.headers.get('Content-Length', 0))
        data = json.loads(self.rfile.read(length))
        old = sys.stdout
        sys.stdout = io.StringIO()
        out = err = None
        try:
            exec(data['code'], {'__builtins__': __builtins__})
            out = sys.stdout.getvalue()
        except Exception as e:
            out = traceback.format_exc()
            err = str(e)
        finally:
            sys.stdout = old
        self._json({'output': out, 'error': err})

    def _notebook(self):
        length = int(self.headers.get('Content-Length', 0))
        nb = json.loads(self.rfile.read(length))
        code = '\n'.join(
            ''.join(c.get('source', []))
            for c in nb.get('cells', [])
            if c.get('cell_type') == 'code'
        )
        old = sys.stdout
        sys.stdout = io.StringIO()
        out = err = None
        try:
            exec(code, {'__builtins__': __builtins__})
            out = sys.stdout.getvalue()
        except Exception as e:
            out = traceback.format_exc()
            err = str(e)
        finally:
            sys.stdout = old
        self._json({'output': out, 'error': err})

# ─── Frontend dashboard (HTML/JS) with article modal ─────────────
DASHBOARD = r"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Oblivion XQDatabase</title>
<style>
:root{
  --bg:#1e1e1e; --pan:#252526; --bdr:#333; --txt:#d4d4d4;
  --ac:#0e639c; --ac2:#007acc; --modal-bg:#1e1e1e;
}
*{box-sizing:border-box;margin:0;padding:0}
body{display:flex;height:100vh;background:var(--bg);color:var(--txt);font-family:'Segoe UI',sans-serif}
#sidebar{width:240px;background:var(--pan);border-right:1px solid var(--bdr);display:flex;flex-direction:column}
#brand{font-size:10px;color:#888;padding:6px 10px;border-bottom:1px solid var(--bdr);text-align:center}
#sidebar h3{padding:10px;font-size:13px;color:#888;border-bottom:1px solid var(--bdr)}
#files{flex:1;overflow-y:auto;padding:4px}
.file{padding:6px 10px;cursor:pointer;font-size:13px;border-radius:2px}
.file:hover{background:#37373d}
#import-btn{margin:8px;background:var(--ac);color:#fff;border:none;padding:8px;border-radius:4px;cursor:pointer;font-weight:bold}
#main{flex:1;display:flex;flex-direction:column;min-width:0}
#toolbar{background:var(--pan);padding:6px 10px;display:flex;gap:6px;border-bottom:1px solid var(--bdr);align-items:center}
#toolbar button{background:var(--ac);color:#fff;border:none;padding:5px 12px;border-radius:3px;cursor:pointer;font-size:12px}
#stats{margin-left:auto;font-size:12px;color:#ccc}
#panels{flex:1;display:flex;min-height:0}
#editor-panel{width:45%;display:flex;flex-direction:column;border-right:1px solid var(--bdr)}
#editor{flex:1;background:#1e1e1e;color:#dcdcaa;border:none;padding:10px;font-family:Consolas,monospace;resize:none;font-size:13px}
#output{height:160px;background:#111;color:#4ec9b0;border-top:1px solid var(--bdr);padding:8px;overflow:auto;font-size:11px;white-space:pre-wrap}
#graph-panel{flex:1;position:relative;background:#2d2d2d}
#search-wrap{position:absolute;top:8px;left:8px;z-index:5;width:200px}
#search{width:100%;padding:6px;background:#111;border:1px solid #555;color:#fff;border-radius:4px;font-size:12px}
#search-results{position:absolute;top:100%;left:0;width:100%;background:#252526;border:1px solid #555;max-height:150px;overflow-y:auto;display:none;z-index:10}
.search-item{padding:6px;cursor:pointer;font-size:12px;border-bottom:1px solid #333}
.search-item:hover{background:#094771}
canvas{display:block}
/* modal */
.modal{display:none;position:fixed;z-index:100;left:0;top:0;width:100%;height:100%;background:rgba(0,0,0,0.7)}
.modal-content{background:var(--modal-bg);margin:5% auto;padding:20px;border:1px solid var(--bdr);width:600px;max-height:80vh;overflow-y:auto;border-radius:6px;color:#ddd}
.close{color:#aaa;float:right;font-size:24px;font-weight:bold;cursor:pointer}
.close:hover{color:#fff}
.modal-table{width:100%;border-collapse:collapse;font-size:13px;margin-top:12px}
.modal-table td,.modal-table th{padding:6px;border-bottom:1px solid #333;text-align:left}
.modal-table th{color:#888;width:140px}
.article-box{background:#2a2a2a;border:1px solid #444;padding:10px;margin-top:16px;max-height:300px;overflow-y:auto;white-space:pre-wrap;font-size:13px;line-height:1.5}
</style>
</head>
<body>
<div id="sidebar">
  <div id="brand">Oblivion XQDatabase</div>
  <h3>Files</h3>
  <div id="files"></div>
  <button id="import-btn">Import File</button>
  <input type="file" id="file-input" hidden multiple>
</div>
<div id="main">
  <div id="toolbar">
    <button id="run-btn">Run Python</button>
    <button id="nb-btn">Run Notebook</button>
    <button id="graph-refresh">Refresh Graph</button>
    <span id="stats"></span>
  </div>
  <div id="panels">
    <div id="editor-panel">
      <textarea id="editor" placeholder="Write or load Python code..."></textarea>
      <div id="output">[Output]</div>
    </div>
    <div id="graph-panel">
      <div id="search-wrap">
        <input id="search" placeholder="Search nodes...">
        <div id="search-results"></div>
      </div>
      <canvas id="c"></canvas>
    </div>
  </div>
</div>
<!-- Node detail modal with article -->
<div id="node-modal" class="modal">
  <div class="modal-content">
    <span class="close" onclick="closeModal()">&times;</span>
    <h3 id="modal-title">Node Details</h3>
    <table class="modal-table" id="modal-info"></table>
    <h4 style="margin-top:16px">Connected Edges</h4>
    <table class="modal-table" id="modal-edges"></table>
    <h4 style="margin-top:16px">Source Article</h4>
    <div class="article-box" id="modal-article"></div>
  </div>
</div>
<script>
let graphData = null, currentFile = null;
const canvas = document.getElementById('c'), ctx = canvas.getContext('2d');
let panX = 0, panY = 0, zoom = 1;
let dragNode = null, isPan = false, lastMouse = {};
let clickCandidate = null;
const CLICK_THRESH = 3;

function resize(){
  canvas.width = canvas.parentElement.clientWidth;
  canvas.height = canvas.parentElement.clientHeight;
  draw();
}
window.onresize = resize; resize();

function loadFiles(){
  fetch('/api/files').then(r=>r.json()).then(fs=>{
    document.getElementById('files').innerHTML = fs.map(f=>
      `<div class="file" onclick="loadFile('${f}')">${f}</div>`
    ).join('');
  });
}
loadFiles();

function loadFile(name){
  fetch('/api/file?name='+encodeURIComponent(name)).then(r=>r.json()).then(d=>{
    if(d.error) alert(d.error);
    else{
      document.getElementById('editor').value = d.content;
      currentFile = name;
    }
  });
}

document.getElementById('import-btn').onclick = ()=> document.getElementById('file-input').click();
document.getElementById('file-input').onchange = function(){
  for(let f of this.files){
    let fd = new FormData(); fd.append('file', f);
    fetch('/api/import', {method:'POST', body:fd})
      .then(r=>r.json()).then(d=>{ loadFiles(); loadMaster(); });
  }
};

function loadMaster(){
  fetch('/api/master').then(r=>r.json()).then(d=>{
    graphData = d;
    draw();
    document.getElementById('stats').textContent =
      `Nodes: ${d.nodes.length}  Edges: ${d.edges.length}`;
  });
}
loadMaster();
document.getElementById('graph-refresh').onclick = loadMaster;

const communityColors = [
  '#ff6b6b','#4ecdc4','#ffe66d','#a06cd5','#6abf69','#ff8c42','#45b7d1','#f9ca24',
  '#e056a0','#7f8c8d','#2ecc71','#9b59b6','#f39c12','#1abc9c','#e74c3c','#3498db',
  '#c0392b','#16a085','#d35400','#8e44ad'
];

function draw(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  if(!graphData) return;
  ctx.save();
  ctx.translate(panX, panY);
  ctx.scale(zoom, zoom);

  (graphData.edges||[]).forEach(e=>{
    let src = graphData.nodes.find(n=>n.id===e.source);
    let tgt = graphData.nodes.find(n=>n.id===e.target);
    if(!src||!tgt) return;
    ctx.beginPath();
    ctx.moveTo(src.x, src.y);
    ctx.lineTo(tgt.x, tgt.y);
    ctx.strokeStyle = 'rgba(255,255,255,0.2)';
    ctx.lineWidth = Math.min(e.weight*0.8, 3);
    ctx.stroke();
  });

  const searchTerm = document.getElementById('search').value.trim().toLowerCase();
  (graphData.nodes||[]).forEach(n=>{
    ctx.beginPath();
    let s = n.size||10, x = n.x||100, y = n.y||100;
    if(n.shape==='square'){
      ctx.rect(x-s/2, y-s/2, s, s);
    } else if(n.shape==='triangle'){
      ctx.moveTo(x, y-s/2);
      ctx.lineTo(x+s/2, y+s/2);
      ctx.lineTo(x-s/2, y+s/2);
      ctx.closePath();
    } else {
      ctx.arc(x, y, s/2, 0, Math.PI*2);
    }
    let color = n.color;
    if(n.community>=0){
      color = communityColors[n.community % communityColors.length];
    }
    ctx.fillStyle = color;
    ctx.fill();
    if(searchTerm && n.label.toLowerCase().includes(searchTerm)){
      ctx.strokeStyle = '#fff';
      ctx.lineWidth = 3;
      ctx.stroke();
    }
    ctx.fillStyle = '#fff';
    ctx.font = `${Math.max(8, s/2.5)}px sans-serif`;
    ctx.textAlign = 'center';
    ctx.fillText(n.label, x, y - s/2 - 4);
  });

  ctx.restore();
}

canvas.onmousedown = e => {
  if(!graphData) return;
  const rect = canvas.getBoundingClientRect();
  const mx = (e.clientX - rect.left - panX) / zoom;
  const my = (e.clientY - rect.top - panY) / zoom;
  const node = graphData.nodes.find(n=>{
    const dx = n.x - mx, dy = n.y - my;
    return Math.sqrt(dx*dx+dy*dy) < (n.size||10);
  });
  if(node){
    clickCandidate = { node, startX: e.clientX, startY: e.clientY };
    dragNode = null;
    isPan = false;
  } else {
    isPan = true;
    lastMouse = { x: e.clientX, y: e.clientY };
  }
};

canvas.onmousemove = e => {
  if(dragNode){
    const rect = canvas.getBoundingClientRect();
    dragNode.x = (e.clientX - rect.left - panX) / zoom;
    dragNode.y = (e.clientY - rect.top - panY) / zoom;
    draw();
  } else if(isPan){
    panX += e.clientX - lastMouse.x;
    panY += e.clientY - lastMouse.y;
    lastMouse = { x: e.clientX, y: e.clientY };
    draw();
  } else if(clickCandidate){
    const dx = e.clientX - clickCandidate.startX;
    const dy = e.clientY - clickCandidate.startY;
    if(Math.sqrt(dx*dx+dy*dy) > CLICK_THRESH){
      dragNode = clickCandidate.node;
      clickCandidate = null;
    }
  }
};

canvas.onmouseup = e => {
  if(clickCandidate){
    openNodeDetail(clickCandidate.node);
    clickCandidate = null;
  }
  dragNode = null;
  isPan = false;
};

canvas.onwheel = e => {
  e.preventDefault();
  zoom *= e.deltaY < 0 ? 1.1 : 0.9;
  draw();
};

const searchInput = document.getElementById('search');
const searchResults = document.getElementById('search-results');
let searchTimeout;

searchInput.addEventListener('input', () => {
  draw();
  clearTimeout(searchTimeout);
  searchTimeout = setTimeout(doSearch, 200);
});

function doSearch(){
  const term = searchInput.value.trim();
  if(!term || !graphData){
    searchResults.style.display = 'none';
    return;
  }
  fetch('/api/search?q='+encodeURIComponent(term))
    .then(r=>r.json())
    .then(nodes=>{
      if(!nodes.length){
        searchResults.style.display = 'none';
        return;
      }
      searchResults.innerHTML = nodes.map(n=>
        `<div class="search-item" data-id="${n.id}">${n.label} (comm:${n.community})</div>`
      ).join('');
      searchResults.style.display = 'block';
      document.querySelectorAll('.search-item').forEach(el=>{
        el.onclick = ()=>{
          const nid = el.dataset.id;
          const node = graphData.nodes.find(n=>n.id===nid);
          if(node) openNodeDetail(node);
          searchResults.style.display = 'none';
        };
      });
    });
}

document.addEventListener('click', e => {
  if(!e.target.closest('#search-wrap')){
    searchResults.style.display = 'none';
  }
});

function openNodeDetail(node){
  document.getElementById('modal-title').textContent = 'Node: ' + node.label;
  const infoTable = document.getElementById('modal-info');
  const edgesTable = document.getElementById('modal-edges');
  const articleDiv = document.getElementById('modal-article');
  infoTable.innerHTML = `
    <tr><th>Label</th><td>${node.label}</td></tr>
    <tr><th>Community</th><td>${node.community>=0?node.community:'N/A'}</td></tr>
    <tr><th>Betweenness</th><td>${node.betweenness.toFixed(4)}</td></tr>
    <tr><th>Size</th><td>${node.size.toFixed(1)}</td></tr>
    <tr><th>Source File</th><td>${node.source_file||'N/A'}</td></tr>
  `;
  // fetch detailed info including article from backend
  fetch('/api/node?id=' + encodeURIComponent(node.id))
    .then(r=>r.json())
    .then(detail=>{
      const connected = detail.edges || [];
      edgesTable.innerHTML = connected.length ?
        connected.map(e=>{
          const otherId = e.source===node.id ? e.target : e.source;
          const otherNode = graphData.nodes.find(n=>n.id===otherId);
          const otherLabel = otherNode ? otherNode.label : otherId;
          return `<tr><td>${otherLabel}</td><td>weight: ${e.weight.toFixed(1)}</td><td>${e.source_file||''}</td></tr>`;
        }).join('') :
        '<tr><td colspan="3">No edges</td></tr>';
      articleDiv.textContent = detail.article || 'No article information.';
    })
    .catch(()=>{
      edgesTable.innerHTML = '<tr><td colspan="3">Error loading edges.</td></tr>';
      articleDiv.textContent = 'Error loading article.';
    });
  document.getElementById('node-modal').style.display = 'block';
}

function closeModal(){
  document.getElementById('node-modal').style.display = 'none';
}
window.onclick = function(event){
  if(event.target === document.getElementById('node-modal')) closeModal();
};

document.getElementById('run-btn').onclick = ()=>{
  fetch('/api/run',{
    method:'POST',
    headers:{'Content-Type':'application/json'},
    body: JSON.stringify({code: document.getElementById('editor').value})
  })
  .then(r=>r.json())
  .then(d=> document.getElementById('output').textContent = d.output || d.error);
};

document.getElementById('nb-btn').onclick = ()=>{
  let nb;
  try{ nb = JSON.parse(document.getElementById('editor').value); }
  catch(e){ alert('Invalid JSON notebook'); return; }
  fetch('/api/notebook',{
    method:'POST',
    headers:{'Content-Type':'application/json'},
    body: JSON.stringify(nb)
  })
  .then(r=>r.json())
  .then(d=> document.getElementById('output').textContent = d.output || d.error);
};
</script>
</body>
</html>"""

# ─── server start ─────────────────────────────────────────────────
if __name__ == '__main__':
    print(f"Oblivion XQDatabase running on http://{HOST}:{PORT}")
    server = socketserver.ThreadingTCPServer((HOST, PORT), Handler)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        server.shutdown()
