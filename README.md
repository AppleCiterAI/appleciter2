# appleciter
# AppleCiter Project Code - Consolidated for Deployment
# Contains app.py, requirements.txt, and frontend files (index.html, styles.css, script.js)
# Designed for Python3-pip/Flask backend, arXiv API, and Wix embedding via iframe
# Deploy on Render with build command: pip install -r requirements.txt
# Start command: gunicorn app:app

# --- app.py ---
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

app = Flask(__name__)

 # --- requirements.txt ---
Flask==3.0.3
requests==2.32.3
gunicorn==23.0.0

# Safety features
def is_safe_query(query):
    unsafe_keywords = ['violence', 'sex', 'suicide', 'murder', 'kill', 'explicit', 'adult', 'hate', 'illegal', 'drugs', 'weapons', 'pornography']
    return not any(keyword in query.lower() for keyword in unsafe_keywords)

# Clean arXiv response to extract relevant data
def clean_arxiv_response(entries):
    results = []
    for entry in entries:
        title = entry.get('title', 'No title').replace('\n', ' ')
        summary = entry.get('summary', 'No summary').replace('\n', ' ')
        authors = [author.get('name', 'Unknown') for author in entry.get('author', [])]
        link = entry.get('link', [{'@href': '#'}])[0]['@href']
        results.append({
            'title': title,
            'summary': summary[:200] + '...',
            'authors': ', '.join(authors[:3]),
            'link': link
        })
    return results

# Route for the main page
@app.route('/')
def index():
    return render_template('index.html')

# Route for handling chatbot queries
@app.route('/query', methods=['POST'])
def query():
    data = request.get_json()
    user_query = data.get('query', '').strip()

    if not user_query:
        return jsonify({'error': 'Empty query provided. Please enter a valid research topic.'}), 400

    if not is_safe_query(user_query):
        return jsonify({'error': 'This query contains inappropriate content and cannot be processed. Please try a different topic.'}), 400

    try:
        # Query arXiv API
        arxiv_url = f'http://export.arxiv.org/api/query?search_query={user_query}&max_results=5'
        response = requests.get(arxiv_url, timeout=10)
        response.raise_for_status()

        # Parse XML response
        from xml.etree import ElementTree
        root = ElementTree.fromstring(response.content)
        ns = {'atom': 'http://www.w3.org/2005/Atom'}
        entries = root.findall('atom:entry', ns)
        
        results = clean_arxiv_response(entries)
        if not results:
            return jsonify({'error': 'No results found for your query. Try a different topic.'}), 404

        return jsonify({'results': results})

    except requests.exceptions.RequestException as e:
        return jsonify({'error': f'Failed to fetch results: {str(e)}. Please try again later.'}), 500
    except Exception as e:
        return jsonify({'error': f'Unexpected error: {str(e)}. Contact support if this persists.'}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)

# --- templates/index.html ---
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AppleCiter - Academic Search Chatbot</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='styles.css') }}">
</head>
<body>
    <div class="container">
        <h1>AppleCiter</h1>
        <p>Search for academic papers safely!</p>
        <div class="chat-box" id="chatHistory"></div>
        <div class="input-area">
            <input type="text" id="userInput" placeholder="Ask about a research topic...">
            <button onclick="sendQuery()">Send</button>
        </div>
    </div>
    <script src="{{ url_for('static', filename='script.js') }}"></script>
</body>
</html>

# --- static/styles.css ---


     
