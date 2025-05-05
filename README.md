```
DROP DATABASE IF EXISTS tourgrid_demo;
CREATE DATABASE tourgrid_demo;
USE tourgrid_demo;

-- USERS Table
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(100) NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    role ENUM('admin', 'user') DEFAULT 'user',
    status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
    profile_image VARCHAR(255) DEFAULT 'default-user.jpg',
    company_name VARCHAR(100),
    company_address VARCHAR(255),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- TOURS Table
CREATE TABLE tours (
    tour_id INT PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    duration INT NOT NULL,
    max_people INT NOT NULL,
    inclusions TEXT,
    exclusions TEXT,
    image VARCHAR(255),
    is_featured TINYINT(1) DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1,
    admin_id INT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- TOUR_SCHEDULES Table
CREATE TABLE tour_schedules (
    schedule_id INT PRIMARY KEY,
    tour_id INT NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    available_seats INT NOT NULL,
    guide_name VARCHAR(100),
    price_adjustment DECIMAL(10,2) DEFAULT 0.00,
    status ENUM('upcoming', 'ongoing', 'completed', 'cancelled') DEFAULT 'upcoming',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- BOOKINGS Table
CREATE TABLE bookings (
    booking_id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    tour_id INT NOT NULL,
    schedule_id INT NOT NULL,
    booking_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    num_people INT NOT NULL,
    base_amount DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) DEFAULT 0.00,
    discount_amount DECIMAL(10,2) DEFAULT 0.00,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'confirmed', 'cancelled', 'completed') DEFAULT 'pending',
    cancellation_reason TEXT,
    special_requests TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- PAYMENTS Table
CREATE TABLE payments (
    payment_id INT AUTO_INCREMENT PRIMARY KEY,
    booking_id INT NOT NULL,
    user_id INT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    payment_status ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    transaction_id VARCHAR(100),
    payment_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert base sample data
INSERT INTO users (user_id, username, password, email, full_name)
VALUES (12, 'john_doe', 'hashedpwd', 'john@example.com', 'John Doe');

INSERT INTO tours (tour_id, title, description, price, duration, max_people)
VALUES (5, 'Ooty Adventure', 'Explore the scenic beauty of Ooty', 2500.00, 7, 20);

INSERT INTO tour_schedules (schedule_id, tour_id, start_date, end_date, available_seats)
VALUES (101, 5, '2025-06-01', '2025-06-07', 10);
```


```
-- Member 1: Transaction success demo
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;

-- Step 1: Deduct 2 seats
UPDATE tour_schedules
SET available_seats = available_seats - 2
WHERE schedule_id = 101 AND available_seats >= 2;

-- Step 2: Insert into bookings
INSERT INTO bookings (user_id, tour_id, schedule_id, num_people, base_amount, total_amount, status)
VALUES (12, 5, 101, 2, 5000.00, 5900.00, 'confirmed');

-- Step 3: Insert payment
INSERT INTO payments (booking_id, user_id, amount, payment_method, payment_status)
VALUES (LAST_INSERT_ID(), 12, 5900.00, 'UPI', 'completed');

-- Final step: Commit all
COMMIT;
```

##### “I successfully booked a tour and paid within a single transaction. If any step failed, nothing would be saved. Since all steps passed, COMMIT finalizes all actions.”

```
-- Member 2: Failure recovery demo
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;

-- Step 1: Deduct 3 seats
UPDATE tour_schedules
SET available_seats = available_seats - 3
WHERE schedule_id = 101 AND available_seats >= 3;

-- Step 2: Simulate failure with invalid booking_id (NULL)
INSERT INTO payments (booking_id, user_id, amount, payment_method, payment_status)
VALUES (NULL, 12, 7500.00, 'Credit Card', 'completed');

-- Step 3: ROLLBACK auto-triggers on error or we can add it manually
ROLLBACK;
```
##### “I intentionally caused an error by inserting a NULL booking_id. Since this failed, MySQL automatically rolled back earlier changes like seat deductions to maintain consistency.”

```
-- Member 3: Manual recovery after failure
-- Restore seats after previous failed transaction
UPDATE tour_schedules
SET available_seats = available_seats + 3
WHERE schedule_id = 101;
```
##### “In case the system crashed or a failure happened during booking, as an admin I manually restored the correct seat count to ensure the tour capacity is accurate again.”
