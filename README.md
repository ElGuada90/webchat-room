# webchat-room
transmisión de mensajes en tiempo real a cualquier tipo de dispositivo.
from flask import Flask, render_template, redirect, url_for, request, flash
from flask_socketio import SocketIO, send
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, login_required, logout_user, current_user
from datetime import datetime

app = Flask(__name__)
app.config['SECRET_KEY'] = 'mysecret'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///chat.db'
db = SQLAlchemy(app)
socketio = SocketIO(app)
login_manager = LoginManager()
login_manager.init_app(app)
login_manager.login_view = 'login'

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(150), unique=True, nullable=False)
    password = db.Column(db.String(150), nullable=False)

class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(150), nullable=False)
    message = db.Column(db.String(500), nullable=False)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        user = User.query.filter_by(username=username, password=password).first()
        if user:
            login_user(user)
            return redirect(url_for('chat'))
    return render_template('login.html')


@app.route('/')
@login_required
def chat():
    messages = Message.query.order_by(Message.timestamp).all()
    return render_template('chat.html', username=current_user.username, messages=messages)

@app.route('/clear')
@login_required
def clear_chat():
    Message.query.delete()
    db.session.commit()
    flash('Chat cleared!')
    return redirect(url_for('chat'))

@socketio.on('message')
def handle_message(msg):
    if current_user.is_authenticated:
        message = Message(username=current_user.username, message=msg)
        db.session.add(message)
        db.session.commit()
        send(f"{current_user.username}: {msg}", broadcast=True)

        
@app.route('/logout')
@login_required
def logout():
    logout_user()
    flash('You have been logged out.')
    return redirect(url_for('login'))


if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    socketio.run(app, port=4000, debug=True)

ESTE ARCHIVO VA INDEPENDIENTE
create_user.py
from webchat import app, db, User

with app.app_context():
    db.create_all()

    user1 = User(username='jguadamuz', password='037679')
    user2 = User(username='user2', password='password2')

    db.session.add(user1)
    db.session.add(user2)
    db.session.commit()

Templates
chat.html
<!DOCTYPE html>
<html>
<head>
    <title>Chat</title>
</head>
<body>
    <h1>Welcome, {{ username }}!</h1>
    <div>
        {% with messages = get_flashed_messages() %}
            {% if messages %}
                <ul>
                    {% for message in messages %}
                        <li>{{ message }}</li>
                    {% endfor %}
                </ul>
            {% endif %}
        {% endwith %}
    </div>
    <div id="chat">
        {% for message in messages %}
            <p><strong>{{ message.username }}:</strong> {{ message.message }}</p>
        {% endfor %}
    </div>
    <form id="messageForm">
        <input type="text" id="message" placeholder="Enter message">
        <button type="submit">Send</button>
        <button type="button"><a href="{{ url_for('clear_chat') }}"></a>Clear Chat</button>
        <button type="button"><a href="{{ url_for('logout') }}"></a>Logout</button>
    </form>
    
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="//cdnjs.cloudflare.com/ajax/libs/socket.io/4.0.0/socket.io.js"></script>
    <script>
        $(document).ready(function() {
            var socket = io();
            $('#messageForm').submit(function(e) {
                e.preventDefault();
                socket.send($('#message').val());
                $('#message').val('');
            });
            socket.on('message', function(msg) {
                $('#chat').append($('<p>').text(msg));
            });
        });
    </script>
</body>
</html>

login.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login</title>
</head>
<body>
    <form method="POST">
        <input type="text" name="username" placeholder="Username" required>
        <input type="password" name="password" placeholder="Password" required>
        <button type="submit">Login</button>
    </form>
</body>
</html>

