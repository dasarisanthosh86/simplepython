from flask import Flask, jsonify, request, render_template, abort
from flask_cors import CORS
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from email_validator import validate_email, EmailNotValidError
import logging
import threading

app = Flask(__name__)
app.config['DEBUG'] = False
CORS(app, resources={r"/api/*": {"origins": ["http://localhost:5000"], "methods": ["GET", "POST", "DELETE"]}})

# Sample data
class UserData:
    def __init__(self):
        self.users = [
            {"id": 1, "name": "John Doe", "email": "john@example.com"},
            {"id": 2, "name": "Jane Smith", "email": "jane@example.com"}
        ]
        self.lock = threading.Lock()

    def get_users(self):
        with self.lock:
            return self.users.copy()

    def add_user(self, new_user):
        with self.lock:
            self.users.append(new_user)

    def delete_user(self, user_id):
        with self.lock:
            self.users = [user for user in self.users if user['id'] != user_id]

user_data = UserData()

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/', methods=['GET'])
@limiter.limit("10 per minute")
def home():
    return render_template('index.html')

@app.route('/api/users', methods=['GET'])
@limiter.limit("10 per minute")
def get_users():
    users = user_data.get_users()
    masked_users = [{"id": user['id'], "name": user['name'], "email": user['email'].split('@')[0] + '@***'} for user in users]
    return jsonify(masked_users)

@app.route('/api/users', methods=['POST'])
@limiter.limit("5 per minute")
def add_user():
    data = request.get_json()
    if not data or 'name' not in data or 'email' not in data:
        abort(400)
    try:
        validate_email(data['email'])
    except EmailNotValidError:
        abort(400)
    new_user = {
        "id": len(user_data.get_users()) + 1,
        "name": data['name'],
        "email": data['email']
    }
    user_data.add_user(new_user)
    return jsonify(new_user), 201

@app.route('/api/users/<int:user_id>', methods=['DELETE'])
@limiter.limit("5 per minute")
def delete_user(user_id):
    users = user_data.get_users()
    user_to_delete = next((user for user in users if user['id'] == user_id), None)
    if user_to_delete is None:
        abort(404)
    user_data.delete_user(user_id)
    return jsonify({"message": "User deleted"}), 200

@app.errorhandler(404)
def not_found(error):
    return jsonify({"message": "Not found"}), 404

@app.errorhandler(400)
def bad_request(error):
    return jsonify({"message": "Bad request"}), 400

@app.errorhandler(Exception)
def handle_exception(error):
    logging.error(error)
    return jsonify({"message": "Internal server error"}), 500

@app.after_request
def after_request(response):
    response.headers.add('Access-Control-Allow-Headers', 'Content-Type,Authorization')
    response.headers.add('Access-Control-Allow-Methods', 'GET,PUT,POST,DELETE,OPTIONS')
    return response

if __name__ == '__main__':
    logging.basicConfig(level=logging.ERROR)
    app.run(port=5000)