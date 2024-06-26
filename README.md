#################### BACKEND FLASK PYTHON ####################

from flask import Flask, render_template, redirect, url_for, request, flash
from flask_socketio import SocketIO, send
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, login_required, logout_user, current_user
from datetime import datetime, timezone
from flask_socketio import emit
import pytz

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

    def __repr__(self):
        return f'<Message {self.username}>'
    
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


@socketio.on('message')
def handle_message(data):
    panama_tz = pytz.timezone('America/Panama')
    timestamp = datetime.now(panama_tz)
    message = Message(username=data['username'], message=data['message'], timestamp=timestamp)
    db.session.add(message)
    db.session.commit()
    data['timestamp'] = message.timestamp.isoformat()
    socketio.emit('message', data)
    
    
@app.route('/clear')
@login_required
def clear_chat():
    Message.query.delete()
    db.session.commit()
    flash('Chat cleared!')
    return redirect(url_for('chat'))

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

#################### FRONTEND HTML & JS ####################
<!DOCTYPE html>
<html>
<head>
    <title>Web Chat</title>
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black">
    <link rel="icon" href="{{ url_for('static', filename='imagenes/icono_05.png') }}" type="image/png">
    <link rel="stylesheet" type="text/css" href="{{ url_for('static', filename='css/chat.css') }}">
</head>
<body>
    <div class="topnav">
        <ul>
           <li><img class="img-fluid icon" src="../static/imagenes/icono_05.png"></li>
           <li><p class="header">Welcome, {{ username }}!</p></li>
        </ul>
    </div>       
    <div class="row">
        <div class="col-3"></div>

        <div class="col-6">
            
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
                    <p class="message">
                        <strong>{{ message.username }}:</strong>
                        <span style="white-space: pre-wrap;">{{ message.message }}</span>
                        <span class="timestamp" data-utc="{{ message.timestamp}}">{{ message.timestamp}}</span>
                    </p>
                {% endfor %}
            </div>
            <form id="messageForm">
                <div class="input-container">
                    <textarea id="message" placeholder="Enter message" rows="1"></textarea>
                    <button class="send" type="submit">Send</button>
                </div>    
                <div class="button-container">
                    <button class="clear" type="button" onclick="location.href='{{ url_for('clear_chat') }}'">Clear Chat</button>
                    <button class="logout" type="button" onclick="location.href='{{ url_for('logout') }}'">Logout</button>
                </div>
            </form>
        </div>
        <div class="col-3"></div>
    </div>
    
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="//cdnjs.cloudflare.com/ajax/libs/socket.io/4.0.0/socket.io.js"></script>
    <script>
        $(document).ready(function() {
            var socket = io();
        
            // Función para ajustar la altura del textarea
            function adjustTextareaHeight(textarea) {
                textarea.style.height = '35px';
                textarea.style.height = (textarea.scrollHeight) + 'px';
            }
        
            $('#message').on('input', function() {
                adjustTextareaHeight(this);
            });
        
            $('#messageForm').submit(function(e) {
                e.preventDefault();
                var message = $('#message').val();
                if (message) {
                    socket.emit('message', {username: '{{ username }}', message: message});
                    $('#message').val('');
                    $('#message').css('height', '35px'); // Resetear la altura después de enviar el mensaje
                }
            });
        
            socket.on('message', function(msg) {
                var messageElement = $('<p>');
                var utcDate = new Date(msg.timestamp);
                var localDate = new Date(utcDate.toLocaleString("en-US", {timeZone: "America/Panama"}));  // Convertir a la hora local de Panamá
                // Use <pre-warp> to preserve spaces and newlines
                var messageContent = $('<span>').text(msg.message).css('white-space', 'pre-wrap');
                messageElement.html('<strong>' + msg.username + ':</strong> ').append(messageContent).append(' <span class="timestamp">' + localDate.toLocaleString() + '</span>');
                $('#chat').append(messageElement);

                // Desplazar automáticamente hacia el final del contenedor del chat
                $('#chat').scrollTop($('#chat')[0].scrollHeight);
            });
        
            // Convertir la hora UTC a la hora local al cargar la página
            var timestamps = document.querySelectorAll('.timestamp');
            timestamps.forEach(function(timestamp) {
                var utcDate = new Date(timestamp.getAttribute('data-utc'));
                var localDate = new Date(utcDate.toLocaleString("en-US", {timeZone: "America/Panama"}));  // Convertir a la hora local de Panamá
                timestamp.textContent = localDate.toLocaleString();
            });

            // Desplazar automáticamente hacia el final del contenedor del chat al cargar la página
            $('#chat').scrollTop($('#chat')[0].scrollHeight);
        });                        
    </script>
</body>
</html>

#################### CSS ####################

* {
    box-sizing: border-box;
}
  

body {
    background-color: rgba(3, 3, 3, 0.35);
    overflow-x: hidden;
}

.topnav ul {
    display: flex;
    align-items: center;
    list-style-type: none;
    padding: 0;
    margin: 0;
}

.topnav ul li {
    margin-right: 10px; /* Adjust as needed for spacing between items */
}

.topnav .icon {
    width: 30px; /* Adjust as needed */
    height: auto; /* Maintain aspect ratio */
    margin: 5px;
}

.topnav .header {
    margin: 0;
    font-size: 18px; /* Adjust as needed */
}


ul {
    list-style-type: none;
    margin: 0;
    padding: 0;
    overflow: hidden;
    background-color: rgba(3, 3, 3, 0.35);
  }
  
  li {
    float: left;
  }
  
  li a {
    display: block;
    padding: 8px;
    background-color: #dddddd;
  }

 .header {
    font-size: 14px;
    font-weight: bold;
    color: rgba(255, 255, 255, 1);
 }
.col-3 {
    width: 0%;
}

.col-6 {
    width: 100%;
    
}

 /* Estilo básico para el chat */
#chat {
    width: 100%;
    background-color: rgba(191, 250, 241, 1);
    border-radius: 5px;
    padding: 10px;
    margin-bottom: 10px;
    height: 400px;
    max-height: 400px;
    overflow-y: auto;
}

#chat p {
    margin: 5px 0;
    background-color: rgba(210, 210, 210,0.5);
    border-radius: 15px;
    padding: 10px;
}

.timestamp {
    font-size: 0.8em;
    font-weight: bold;
    color: rgb(110, 110, 110);
    margin-left: 10px;
}

.message {
    font-size: 14px;
    font-family: Helvetica, sans-serif;
}

#messageForm {
    display: flex;
    flex-direction: column;
}
#messageForm input, #messageForm button {
    margin: 5px 0;
}

.row {
    display: flex;
    align-items: center; /* Alinea los elementos verticalmente en el centro */
}

.input-container {
    display: flex;
    gap: 10px; /* Ajusta el valor según lo necesites */
    align-items: center;
}

#message-input {
    flex-grow: 1;
}

#send-button {
    flex-shrink: 0;
}

textarea {
    width: 85%;
    height: 35px;
    border-radius: 5px;
    border: 1px solid rgba(250,250,250,1);
    overflow-y: none;
    resize: none;
    flex-grow: 1; /* Permite que el textarea crezca para ocupar el espacio disponible */
    padding: 9px;
}

.send {
    text-shadow: 2px 2px 5px rgb(90, 90, 90);
    width: auto;
    height: 35px;
    border: 3px solid gray;
    border-radius: 10px;
    margin-left: 10px; /* Espacio entre el textarea y el botón */
    background-color: rgb(250, 246, 5);
    color:black;
    font-weight: bold;
    
}

.send:hover {
    background-color: rgba(0, 250, 0, 1);
    color: rgba(10, 10, 10, 1);

}

.clear {
    width: auto;
    height: 35px;
    border-radius: 5px;
    font-weight: bold;
    background-color: rgb(250, 246, 5);
    
}

.clear:hover {
    background-color: rgba(255, 0, 0, 1);
    color: rgba(255, 255, 255, 1);
}

.logout {
    width: auto;
    height: 35px;
    border-radius: 5px;
    font-weight: bold;
    background-color: rgb(250, 246, 5);
    
}

.logout:hover {
    background-color: rgba(255, 0, 0, 1);
    color: rgba(255, 255, 255, 1);
}

.button-container {
    margin-top: 20px; /* Espacio superior para separar los botones del resto del contenido */
}

#clear-button,
#logout-button {
    margin-right: 10px; /* Espacio derecho entre los botones */
}

#logout-button {
    margin-left: 10px; /* Espacio izquierdo entre los botones */
}

