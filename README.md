# Flask.py
from flask import Flask, render_template_string

app = Flask(__name__)

# --- Templates --- #
HOME_HTML = """
<!DOCTYPE html>
<html lang='en'>
<head>
    <meta charset='UTF-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1.0'>
    <title>My Best Website</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; background: #f4f4f4; }
        header { background: #4a4aff; color: white; padding: 20px; text-align: center; }
        section { padding: 20px; }
        .card { background: white; padding: 20px; margin: 20px auto; border-radius: 10px; max-width: 600px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
        footer { background: #222; color: white; padding: 10px; text-align: center; margin-top: 40px; }
        a.button { display: inline-block; padding: 10px 20px; background: #4a4aff; color: white; text-decoration: none; border-radius: 8px; }
    </style>
</head>
<body>
    <header>
        <h1>Welcome to My Website</h1>
        <p>Fast • Clean • Built with Flask</p>
    </header>

    <section>
        <div class="card">
            <h2>Hello!</h2>
            <p>This website is built using your Flask base project. You can add more pages, forms, APIs, and features easily.</p>
            <a href='/about' class='button'>Go to About Page</a>
        </div>
    </section>

    <footer>
        &copy; 2025 My Best Website
    </footer>
</body>
</html>
"""

ABOUT_HTML = """
<!DOCTYPE html>
<html lang='en'>
<head>
    <meta charset='UTF-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1.0'>
    <title>About</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; background: #f4f4f4; }
        header { background: #4a4aff; color: white; padding: 20px; text-align: center; }
        .card { background: white; padding: 20px; margin: 20px auto; border-radius: 10px; max-width: 600px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
        a.button { display: inline-block; padding: 10px 20px; background: #4a4aff; color: white; text-decoration: none; border-radius: 8px; }
    </style>
</head>
<body>
    <header>
        <h1>About This Website</h1>
    </header>

    <div class="card">
        <p>This site is generated using Flask and your uploaded base files. You can extend it with more pages and API routes.</p>
        <a href='/' class='button'>Back to Home</a>
    </div>
</body>
</html>
"""

# --- Routes --- #
@app.route('/')
def home():
    return render_template_string(HOME_HTML)

@app.route('/about')
def about():
    return render_template_string(ABOUT_HTML)


# Run the app
if __name__ == '__main__':
    app.run(debug=True)
    
