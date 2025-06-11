# appleciter
1st version of appleciter chatter# AppleCiter Project Code - Consolidated for Deployment
# AppleCiter Project Code - Consolidated for Deployment
# Contains app.py, requirements.txt, and frontend files (index.html, styles.css, script.js)
# Designed for Python/Flask backend, arXiv API, and Wix embedding via iframe
# Deploy on Render with build command: pip install -r requirements.txt
# Start command: gunicorn app:app
# Updated to fix missing requirements.txt and enforce explicit refusal for inappropriate queries

# --- app.py ---
from flask import Flask, request, jsonify, render_template
import requests
import re

app = Flask(__name__)

# Enhanced keyword-based safety filter for school-safe content
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

# --- requirements.txt ---
Flask==3.0.3
requests==2.32.3
gunicorn==23.0.0

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
body {
    font-family: Arial, sans-serif;
    background-color: #f4f4f9;
    margin: 0;
    padding: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
}

.container {
    background: white;
    padding: 20px;
    border-radius: 10px;
    box-shadow: 0 0 15px rgba(0, 0, 0, 0.1);
    width: 90%;
    max-width: 600px;
}

h1 {
    text-align: center;
    color: #333;
}

.chat-box {
    border: 1px solid #ccc;
    padding: 15px;
    height: 350px;
    overflow-y: auto;
    margin-bottom: 15px;
    background: #fafafa;
    border-radius: 5px;
}

.input-area {
    display: flex;
    gap: 10px;
}

input {
    flex: 1;
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 16px;
}

button {
    padding: 10px 20px;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 16px;
}

button:hover {
    background: #0056b3;
}

.message {
    margin: 8px 0;
    padding: 8px;
    border-radius: 5px;
}

.user-message {
    background: #e3f2fd;
    color: #007bff;
    text-align: right;
}

.bot-message {
    background: #f5f5f5;
    color: #333;
}

.error-message {
    background: #ffebee;
    color: #d32f2f;
}

# --- static/script.js ---
function sendQuery() {
    const userInput = document.getElementById('userInput').value.trim();
    if (!userInput) return;

    const chatHistory = document.getElementById('chatHistory');
    const userMessage = document.createElement('div');
    userMessage.className = 'message user-message';
    userMessage.textContent = `You: ${userInput}`;
    chatHistory.appendChild(userMessage);

    fetch('/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query: userInput })
    })
    .then(response => response.json())
    .then(data => {
        const botMessage = document.createElement('div');
        botMessage.className = 'message bot-message';
        
        if (data.error) {
            botMessage.className = 'message error-message';
            botMessage.textContent = `AppleCiter: ${data.error}`;
        } else {
            let responseText = 'AppleCiter: Found these papers for you:\n';
            data.results.forEach(result => {
                responseText += `<p><strong>${result.title}</strong><br>`;
                responseText += `${result.summary}<br>`;
                responseText += `Authors: ${result.authors}<br>`;
                responseText += `<a href="${result.link}" target="_blank">Read more</a></p>`;
            });
            botMessage.innerHTML = responseText;
        }
        chatHistory.appendChild(botMessage);
        chatHistory.scrollTop = chatHistory.scrollHeight;
    })
    .catch(error => {
        const errorMessage = document.createElement('div');
        errorMessage.className = 'message error-message';
        errorMessage.textContent = `AppleCiter: Failed to connect to server. Please try again.`;
        chatHistory.appendChild(errorMessage);
        chatHistory.scrollTop = chatHistory.scrollHeight;
    });

    document.getElementById('userInput').value = '';
}
