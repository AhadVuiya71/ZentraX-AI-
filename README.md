from flask import Flask, request, jsonify, send_file
import sqlite3, os, io, requests, bcrypt, jwt, datetime
from PIL import Image

app = Flask(__name__)

SECRET = os.environ.get("JWT_SECRET", "zentrax_secret")
HF_TOKEN = os.environ.get("HF_TOKEN")

UPLOAD_FOLDER = "uploads"
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

# ---------------- DATABASE ----------------
def get_db():
    return sqlite3.connect("database.db")

def init_db():
    conn = get_db()
    c = conn.cursor()

    c.execute("""CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE,
        password BLOB
    )""")

    c.execute("""CREATE TABLE IF NOT EXISTS memory (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT,
        message TEXT
    )""")

    conn.commit()
    conn.close()

init_db()

# ---------------- JWT ----------------
def generate_token(username):
    payload = {
        "user": username,
        "exp": datetime.datetime.utcnow() + datetime.timedelta(days=1)
    }
    return jwt.encode(payload, SECRET, algorithm="HS256")

def verify_token(token):
    try:
        return jwt.decode(token, SECRET, algorithms=["HS256"])
    except:
        return None

# ---------------- MEMORY ----------------
def save_memory(user, msg):
    conn = get_db()
    c = conn.cursor()
    c.execute("INSERT INTO memory (username,message) VALUES (?,?)",(user,msg))
    conn.commit()
    conn.close()

def load_memory(user):
    conn = get_db()
    c = conn.cursor()
    c.execute("SELECT message FROM memory WHERE username=? ORDER BY id DESC LIMIT 5",(user,))
    data = c.fetchall()
    conn.close()
    return " ".join([x[0] for x in data])

# ---------------- AUTH ----------------
@app.route('/api/register', methods=['POST'])
def register():
    data = request.json
    u = data.get("username")
    p = data.get("password")

    hashed = bcrypt.hashpw(p.encode(), bcrypt.gensalt())

    try:
        conn = get_db()
        c = conn.cursor()
        c.execute("INSERT INTO users (username,password) VALUES (?,?)",(u,hashed))
        conn.commit()
        conn.close()
        return {"status": "registered"}
    except:
        return {"error": "User exists"}

@app.route('/api/login', methods=['POST'])
def login():
    data = request.json
    u = data.get("username")
    p = data.get("password")

    conn = get_db()
    c = conn.cursor()
    c.execute("SELECT password FROM users WHERE username=?",(u,))
    user = c.fetchone()
    conn.close()

    if user and bcrypt.checkpw(p.encode(), user[0]):
        token = generate_token(u)
        return {"token": token}

    return {"error": "Invalid login"}

# ---------------- AI CHAT ----------------
@app.route('/api/chat', methods=['POST'])
def chat():
    token = request.headers.get("Authorization")
    user_data = verify_token(token)

    if not user_data:
        return {"error": "Unauthorized"}

    user = user_data['user']
    msg = request.json.get("message")

    context = load_memory(user)

    headers = {"Authorization": f"Bearer {HF_TOKEN}"}

    res = requests.post(
        "https://api-inference.huggingface.co/models/microsoft/DialoGPT-medium",
        headers=headers,
        json={"inputs": context + " " + msg}
    )

    if res.status_code != 200:
        return {"error": "AI busy"}

    reply = res.json()[0]['generated_text']

    save_memory(user, msg)
    save_memory(user, reply)

    return {"reply": reply}

# ---------------- AI IMAGE ----------------
@app.route('/api/image', methods=['POST'])
def image():
    token = request.headers.get("Authorization")
    user_data = verify_token(token)

    if not user_data:
        return {"error": "Unauthorized"}

    prompt = request.json.get("prompt")

    headers = {"Authorization": f"Bearer {HF_TOKEN}"}

    res = requests.post(
        "https://api-inference.huggingface.co/models/runwayml/stable-diffusion-v1-5",
        headers=headers,
        json={"inputs": prompt}
    )

    if res.status_code != 200:
        return {"error": "Image failed"}

    img = Image.open(io.BytesIO(res.content))
    path = os.path.join(UPLOAD_FOLDER, f"{user_data['user']}.png")
    img.save(path)

    return send_file(path, mimetype='image/png')

# ---------------- FILE UPLOAD ----------------
@app.route('/api/upload', methods=['POST'])
def upload():
    token = request.headers.get("Authorization")
    user_data = verify_token(token)

    if not user_data:
        return {"error": "Unauthorized"}

    file = request.files['file']
    filename = f"{user_data['user']}_{file.filename}"
    path = os.path.join(UPLOAD_FOLDER, filename)

    file.save(path)

    return {"status": "uploaded"}

# ---------------- ROOT ----------------
@app.route('/')
def home():
    return "ZentraX CORE RUNNING 🚀"

if __name__ == '__main__':
    app.run()
