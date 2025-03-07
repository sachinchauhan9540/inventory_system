from flask import Flask, request, jsonify, render_template_string
from flask_sqlalchemy import SQLAlchemy
import csv
import os
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///inventory.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

MAX_BOOKINGS = 2  # Maximum bookings per member

# Database Models
class Member(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(100), unique=True, nullable=False)

class Inventory(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    item_name = db.Column(db.String(100), nullable=False)
    total_count = db.Column(db.Integer, nullable=False)
    remaining_count = db.Column(db.Integer, nullable=False)

class Booking(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    member_id = db.Column(db.Integer, db.ForeignKey('member.id'), nullable=False)
    inventory_id = db.Column(db.Integer, db.ForeignKey('inventory.id'), nullable=False)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)

# Function to Populate Database from CSV
def populate_db_from_csv(filename, model):
    with open(filename, 'r', encoding='utf-8') as file:
        reader = csv.DictReader(file)
        for row in reader:
            try:
                if model == 'member':
                    full_name = f"{row['name']} {row['surname']}"
                    email = f"{row['name']}.{row['surname']}@example.com"
                    
                    if not Member.query.filter_by(email=email).first():
                        member = Member(name=full_name, email=email)
                        db.session.add(member)

                elif model == 'inventory':
                    inventory = Inventory.query.filter_by(item_name=row['title']).first()
                    count = int(row['remaining_count'])

                    if inventory:
                        inventory.total_count += count
                        inventory.remaining_count += count
                    else:
                        inventory = Inventory(
                            item_name=row['title'],
                            total_count=count,
                            remaining_count=count
                        )
                        db.session.add(inventory)

            except KeyError as e:
                print(f"Missing column in CSV: {e}")
                continue

        db.session.commit()

# Routes
@app.route('/')
def index():
    members = Member.query.all()
    inventory = Inventory.query.all()
    bookings = Booking.query.all()
    return render_template_string(HTML_TEMPLATE, members=members, inventory=inventory, bookings=bookings)

@app.route('/upload', methods=['POST'])
def upload_files():
    if 'file' not in request.files:
        return jsonify({'error': 'No file uploaded'}), 400

    file = request.files['file']
    filename = file.filename
    file_path = os.path.join(os.getcwd(), filename)
    file.save(file_path)

    if filename.endswith('members.csv'):
        populate_db_from_csv(file_path, 'member')
    elif filename.endswith('inventory.csv'):
        populate_db_from_csv(file_path, 'inventory')
    else:
        os.remove(file_path)
        return jsonify({'error': 'Invalid file type'}), 400

    os.remove(file_path)
    return jsonify({'message': 'File uploaded and processed successfully'})

@app.route('/book', methods=['POST'])
def book_item():
    data = request.json
    member_id = data.get('member_id')
    inventory_id = data.get('inventory_id')

    member = Member.query.get(member_id)
    if not member:
        return jsonify({'error': 'Member not found'}), 404

    item = Inventory.query.get(inventory_id)
    if not item or item.remaining_count <= 0:
        return jsonify({'error': 'Item not available'}), 400

    booking_count = Booking.query.filter_by(member_id=member_id).count()
    if booking_count >= MAX_BOOKINGS:
        return jsonify({'error': 'Booking limit reached'}), 400

    booking = Booking(member_id=member_id, inventory_id=inventory_id)
    db.session.add(booking)
    item.remaining_count -= 1
    db.session.commit()

    return jsonify({'message': 'Item booked successfully'})

@app.route('/cancel', methods=['POST'])
def cancel_booking():
    data = request.json
    booking_id = data.get('booking_id')

    booking = Booking.query.get(booking_id)
    if not booking:
        return jsonify({'error': 'Booking not found'}), 404

    item = Inventory.query.get(booking.inventory_id)
    if item:
        item.remaining_count += 1

    db.session.delete(booking)
    db.session.commit()

    return jsonify({'message': 'Booking canceled successfully'})

# HTML Template with Improved Messaging
HTML_TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
    <title>Inventory Booking System</title>
    <script>
        function showMessage(message, type) {
            let messageBox = document.getElementById("messageBox");
            messageBox.innerText = message;
            messageBox.style.backgroundColor = type === "error" ? "red" : "green";
            messageBox.style.color = "white";
            messageBox.style.padding = "10px";
            messageBox.style.display = "block";
            setTimeout(() => { messageBox.style.display = "none"; }, 3000);
        }

        async function uploadFile() {
            let fileInput = document.getElementById('fileInput');
            let formData = new FormData();
            formData.append('file', fileInput.files[0]);

            let response = await fetch('/upload', { method: 'POST', body: formData });
            let result = await response.json();
            showMessage(result.message || result.error, response.ok ? "success" : "error");
            if (response.ok) location.reload();
        }

        async function bookItem() {
            let memberId = document.getElementById('memberId').value;
            let inventoryId = document.getElementById('inventoryId').value;

            let response = await fetch('/book', { 
                method: 'POST', 
                body: JSON.stringify({ member_id: memberId, inventory_id: inventoryId }), 
                headers: { 'Content-Type': 'application/json' }
            });

            let result = await response.json();
            showMessage(result.message || result.error, response.ok ? "success" : "error");
            if (response.ok) location.reload();
        }

        async function cancelBooking(bookingId) {
            let response = await fetch('/cancel', { 
                method: 'POST', 
                body: JSON.stringify({ booking_id: bookingId }), 
                headers: { 'Content-Type': 'application/json' }
            });

            let result = await response.json();
            showMessage(result.message || result.error, response.ok ? "success" : "error");
            if (response.ok) location.reload();
        }
    </script>
</head>
<body>
    <h1>Inventory Booking System</h1>

    <div id="messageBox" style="display: none;"></div>

    <h2>Upload CSV</h2>
    <input type="file" id="fileInput">
    <button onclick="uploadFile()">Upload</button>

    <h2>Book an Item</h2>
    <label>Member ID: <input type="number" id="memberId"></label>
    <label>Item ID: <input type="number" id="inventoryId"></label>
    <button onclick="bookItem()">Book</button>

    <h2>Members</h2>
    <ul>
        {% for member in members %}
            <li>{{ member.id }} - {{ member.name }} ({{ member.email }})</li>
        {% endfor %}
    </ul>
    
    <h2>Inventory</h2>
    <ul>
        {% for item in inventory %}
            <li>{{ item.id }} - {{ item.item_name }} - Remaining: {{ item.remaining_count }}</li>
        {% endfor %}
    </ul>
    <h2>Bookings</h2>
    <ul>
        {% for booking in bookings %}
            <li>Booking ID: {{ booking.id }} - Member: {{ booking.member_id }} - Item: {{ booking.inventory_id }} - Time: {{ booking.timestamp }}
                <button onclick="cancelBooking({{ booking.id }})">Cancel</button>
            </li>
        {% endfor %}
    </ul>
</body>
</html>
"""

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
