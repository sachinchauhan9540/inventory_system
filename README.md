# Inventory Booking System

## Description
This project is a Flask application that manages an inventory booking system. It allows members to book items from an inventory, upload data from CSV files, and manage bookings.

## Installation
1. Clone the repository:
   
   git clone <repository-url>
   cd <repository-directory>

2. Install the required packages:

   pip install -r requirements.txt


## Usage
1. Run the application:

   python app.py

2. Access the application in your web browser at `http://127.0.0.1:5000/`.

### Endpoints
- **GET /**: Displays members, inventory, and bookings.
- **POST /upload**: Uploads CSV files to populate the database.
- **POST /book**: Books an item for a member.
- **POST /cancel**: Cancels a booking.

## Database Setup
To populate the database, prepare CSV files for members and inventory:
- `members.csv`: Should contain columns for `name` and `surname`.
- `inventory.csv`: Should contain columns for `title` and `remaining_count`.

Upload these files using the upload functionality in the application.

## Contributing
Contributions are welcome! Please submit a pull request or open an issue for discussion.

## License
This project is licensed under the MIT License.
