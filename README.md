"""
Single-file Flask website (stable, safer version)
"""

from flask import Flask, g, request, redirect, url_for, render_template_string, abort, flash
import sqlite3
import os
from markupsafe import escape

# -------------------------------------------
# Environment setup + directory safety
# -------------------------------------------

BASE_DIR = os.path.abspath(os.path.dirname(__file__))
os.makedirs(BASE_DIR, exist_ok=True)  # ensure folder exists

DB_PATH = os.path.join(BASE_DIR, "site.db")

app = Flask(__name__)
app.config.update(
    DATABASE=DB_PATH,
    SECRET_KEY=os.environ.get("FLASK_SECRET_KEY", "dev-secret-key"),
)

# -------------------------------------------
# Database helpers
# -------------------------------------------

def get_db():
    """Return SQLite connection (one per request)."""
    db = getattr(g, "_database", None)
    if db is None:
        try:
            db = g._database = sqlite3.connect(app.config["DATABASE"])
            db.row_factory = sqlite3.Row
        except Exception as e:
            app.logger.error(f"DB connection error: {e}")
            raise
    return db

@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, "_database", None)
    if db is not None:
        db.close()

def init_db():
    """Create posts table if not exists."""
    try:
        db = sqlite3.connect(app.config["DATABASE"])
        cur = db.cursor()
        cur.execute(
            """
            CREATE TABLE IF NOT EXISTS posts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                body TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """
        )
        db.commit()
        db.close()
    except Exception as e:
        print(f"DB init error: {e}")

# Safe database init
init_db()

# -------------------------------------------
# Templates
# -------------------------------------------

BASE_HTML = """
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>{{ title or 'My Flask Site' }}</title>
  <style>
    body{font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;margin:0;padding:0;background:#f7f7f8}
    .container{max-width:900px;margin:32px auto;padding:20px;background:white;border-radius:10px;box-shadow:0 6px 20px rgba(0,0,0,0.06)}
    header{display:flex;justify-content:space-between;align-items:center}
    h1{margin:0}
    form{margin-top:16px}
    input[type=text], textarea{width:100%;padding:10px;border:1px solid #ddd;border-radius:6px;margin-top:6px}
    button{padding:10px 16px;border-radius:8px;border:0;background:#2563eb;color:white;cursor:pointer}
    .post{border-top:1px solid #eee;padding:14px 0}
    .meta{color:#666;font-size:0.9rem}
    a{color:#2563eb;text-decoration:none}
    .flash{background:#fff3cd;padding:10px;border-radius:6px;margin-bottom:8px}
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1><a href="{{ url_for('index') }}">My Flask Site</a></h1>
      <nav>
        <a href="{{ url_for('index') }}">Home</a>
      </nav>
    </header>

    {% with messages = get_flashed_messages() %}
      {% if messages %}
        {% for m in messages %}
          <div class="flash">{{ m }}</div>
        {% endfor %}
      {% endif %}
    {% endwith %}

    {% block content %}{% endblock %}

    <footer style="margin-top:24px;color:#888;font-size:0.9rem">
      Built with Flask -- stable, single-file version
    </footer>
  </div>
</body>
</html>
"""

INDEX_HTML = """
{% extends base %}
{% block content %}
  <section>
    <h2>Create a post</h2>
    <form method="post" action="{{ url_for('add_post') }}">
      <label>Title</label>
      <input name="title" type="text" maxlength="200" required>
      <label>Body</label>
      <textarea name="body" rows="6" required></textarea>
      <div style="margin-top:10px"><button type="submit">Add post</button></div>
    </form>
  </section>

  <section style="margin-top:20px">
    <h2>All posts</h2>
    {% if posts %}
      {% for p in posts %}
        <article class="post">
          <h3><a href="{{ url_for('view_post', post_id=p['id']) }}">{{ p['title'] }}</a></h3>
          <div class="meta">{{ p['created_at'] }}</div>
          <p>{{ p['body'][:200] }}{% if p['body']|length > 200 %}...{% endif %}</p>
        </article>
      {% endfor %}
    {% else %}
      <p>No posts yet. Add one above.</p>
    {% endif %}
  </section>
{% endblock %}
"""

POST_HTML = """
{% extends base %}
{% block content %}
  <article>
    <h2>{{ post['title'] }}</h2>
    <div class="meta">{{ post['created_at'] }}</div>
    <div style="margin-top:14px;white-space:pre-wrap">{{ post['body'] }}</div>
    <form method="post" action="{{ url_for('delete_post', post_id=post['id']) }}" style="margin-top:12px">
      <button type="submit" onclick="return confirm('Delete this post?')">Delete post</button>
      <a href="{{ url_for('index') }}" style="margin-left:12px">Back</a>
    </form>
  </article>
{% endblock %}
"""

# -------------------------------------------
# Routes (now wrapped in stability try/except)
# -------------------------------------------

@app.route("/", methods=["GET"])
def index():
    try:
        db = get_db()
        cur = db.execute(
            "SELECT id, title, body, created_at FROM posts ORDER BY created_at DESC LIMIT 100"
        )
        posts = cur.fetchall()
    except Exception as e:
        app.logger.error(f"Index DB error: {e}")
        flash("Database error.")
        posts = []
    return render_template_string(INDEX_HTML, base=BASE_HTML, title="Home", posts=posts)

@app.route("/add", methods=["POST"])
def add_post():
    title = request.form.get("title", "").strip()
    body = request.form.get("body", "").strip()

    if not title or not body:
        flash("Both title and body are required.")
        return redirect(url_for('index'))

    try:
        db = get_db()
        db.execute("INSERT INTO posts (title, body) VALUES (?, ?)", (title, body))
        db.commit()
        flash("Post added!")
    except Exception as e:
        app.logger.error(f"DB insert failed: {e}")
        flash("Could not add post.")
    return redirect(url_for('index'))

@app.route("/post/<int:post_id>", methods=["GET"])
def view_post(post_id):
    try:
        db = get_db()
        cur = db.execute(
            "SELECT id, title, body, created_at FROM posts WHERE id = ?", (post_id,)
        )
        post = cur.fetchone()
        if post is None:
            abort(404)
    except Exception as e:
        app.logger.error(f"DB read failed: {e}")
        abort(500)

    return render_template_string(POST_HTML, base=BASE_HTML, post=post, title=post['title'])

@app.route("/post/<int:post_id>/delete", methods=["POST"])
def delete_post(post_id):
    try:
        db = get_db()
        cur = db.execute("SELECT id FROM posts WHERE id = ?", (post_id,))
        if cur.fetchone() is None:
            abort(404)
        db.execute("DELETE FROM posts WHERE id = ?", (post_id,))
        db.commit()
        flash("Post deleted.")
    except Exception as e:
        app.logger.error(f"DB delete failed: {e}")
        flash("Could not delete post.")
    return redirect(url_for('index'))

@app.route('/ping')
def ping():
    return 'pong', 200

# -------------------------------------------
# Run app
# -------------------------------------------
if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False)