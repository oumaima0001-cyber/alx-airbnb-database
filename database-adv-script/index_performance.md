-- =================================================================
-- ALX Airbnb Database - Index Creation for Performance Optimization
-- File: database_index.sql
-- =================================================================

-- =================================================================
-- STEP 1: BASELINE PERFORMANCE TEST (Before Indexes)
-- =================================================================
-- Run the following queries BEFORE creating indexes to capture baseline performance

-- Query 1: User lookup by email (common for login)
EXPLAIN ANALYZE SELECT user_id, first_name, last_name 
FROM User 
WHERE email = 'user@example.com';

-- Query 2: Bookings by user (frequent for user dashboard)
EXPLAIN ANALYZE SELECT b.*, p.name 
FROM Booking b 
JOIN Property p ON b.property_id = p.property_id 
WHERE b.user_id = 123;

-- Query 3: Bookings by date range (common for availability checks)
EXPLAIN ANALYZE SELECT booking_id, property_id, start_date, end_date 
FROM Booking 
WHERE start_date >= '2024-01-01' AND end_date <= '2024-12-31';

-- Query 4: Properties by location (frequent search criteria)
EXPLAIN ANALYZE SELECT property_id, name, pricepernight 
FROM Property 
WHERE location = 'New York' 
ORDER BY pricepernight;

-- Query 5: Property bookings count (analytics query)
EXPLAIN ANALYZE SELECT p.property_id, p.name, COUNT(b.booking_id) as booking_count
FROM Property p
LEFT JOIN Booking b ON p.property_id = b.property_id
GROUP BY p.property_id, p.name
ORDER BY booking_count DESC;

-- =================================================================
-- INDEX MAPPING TO PERFORMANCE QUERIES
-- =================================================================
-- Query 1 → idx_user_email
-- Query 2 → idx_booking_user_id, idx_property_property_id
-- Query 3 → idx_booking_start_date, idx_booking_end_date
-- Query 4 → idx_property_location, idx_property_price
-- Query 5 → idx_booking_property_id

-- =================================================================
-- CREATE INDEX COMMANDS
-- =================================================================

-- USER TABLE INDEXES
CREATE INDEX idx_user_email ON User(email);
CREATE INDEX idx_user_created_at ON User(created_at);
CREATE INDEX idx_user_role ON User(role);
CREATE INDEX idx_user_fullname ON User(first_name, last_name);

-- PROPERTY TABLE INDEXES
CREATE INDEX idx_property_location ON Property(location);
CREATE INDEX idx_property_price ON Property(pricepernight);
CREATE INDEX idx_property_host_id ON Property(host_id);
CREATE INDEX idx_property_created_at ON Property(created_at);
CREATE INDEX idx_property_location_price ON Property(location, pricepernight);
CREATE INDEX idx_property_location_created ON Property(location, created_at);

-- BOOKING TABLE INDEXES
CREATE INDEX idx_booking_user_id ON Booking(user_id);
CREATE INDEX idx_booking_property_id ON Booking(property_id);
CREATE INDEX idx_booking_start_date ON Booking(start_date);
CREATE INDEX idx_booking_end_date ON Booking(end_date);
CREATE INDEX idx_booking_status ON Booking(status);
CREATE INDEX idx_booking_created_at ON Booking(created_at);
CREATE INDEX idx_booking_date_range ON Booking(start_date, end_date);
CREATE INDEX idx_booking_property_start_date ON Booking(property_id, start_date);
CREATE INDEX idx_booking_user_created ON Booking(user_id, created_at);
CREATE INDEX idx_booking_property_status ON Booking(property_id, status);

-- REVIEW TABLE INDEXES
CREATE INDEX idx_review_property_id ON Review(property_id);
CREATE INDEX idx_review_user_id ON Review(user_id);
CREATE INDEX idx_review_rating ON Review(rating);
CREATE INDEX idx_review_created_at ON Review(created_at);
CREATE INDEX idx_review_property_rating ON Review(property_id, rating);

-- PAYMENT TABLE INDEXES
CREATE INDEX idx_payment_booking_id ON Payment(booking_id);
CREATE INDEX idx_payment_date ON Payment(payment_date);
CREATE INDEX idx_payment_method ON Payment(payment_method);
CREATE INDEX idx_payment_booking_date ON Payment(booking_id, payment_date);

-- SPECIALIZED INDEXES
CREATE INDEX idx_booking_availability ON Booking(property_id, start_date, end_date);
CREATE INDEX idx_user_activity ON User(created_at, role);
CREATE INDEX idx_property_performance ON Property(location, pricepernight, created_at);
CREATE INDEX idx_booking_analytics ON Booking(created_at, status, total_price);

-- =================================================================
-- STEP 2: POST-INDEX PERFORMANCE TEST (After Indexes)
-- =================================================================
-- Re-run the same queries AFTER creating indexes to compare improvements

-- Query 1: User lookup by email
EXPLAIN ANALYZE SELECT user_id, first_name, last_name 
FROM User 
WHERE email = 'user@example.com';

-- Query 2: Bookings by user
EXPLAIN ANALYZE SELECT b.*, p.name 
FROM Booking b 
JOIN Property p ON b.property_id = p.property_id 
WHERE b.user_id = 123;

-- Query 3: Bookings by date range
EXPLAIN ANALYZE SELECT booking_id, property_id, start_date, end_date 
FROM Booking 
WHERE start_date >= '2024-01-01' AND end_date <= '2024-12-31';

-- Query 4: Properties by location
EXPLAIN ANALYZE SELECT property_id, name, pricepernight 
FROM Property 
WHERE location = 'New York' 
ORDER BY pricepernight;

-- Query 5: Property bookings count
EXPLAIN ANALYZE SELECT p.property_id, p.name, COUNT(b.booking_id) as booking_count
FROM Property p
LEFT JOIN Booking b ON p.property_id = b.property_id
GROUP BY p.property_id, p.name
ORDER BY booking_count DESC;

-- Additional test cases
EXPLAIN ANALYZE SELECT property_id, name, pricepernight 
FROM Property 
WHERE location = 'Paris' AND pricepernight BETWEEN 100 AND 300
ORDER BY pricepernight;

EXPLAIN ANALYZE SELECT booking_id, user_id, start_date, end_date 
FROM Booking 
WHERE property_id = 456 
AND start_date >= '2024-06-01' 
AND end_date <= '2024-08-31';

EXPLAIN ANALYZE SELECT b.booking_id, b.start_date, b.end_date, p.name 
FROM Booking b 
JOIN Property p ON b.property_id = p.property_id 
WHERE b.user_id = 789 
ORDER BY b.created_at DESC;

EXPLAIN ANALYZE SELECT property_id 
FROM Booking 
WHERE property_id = 123 
AND (
    (start_date <= '2024-07-01' AND end_date >= '2024-07-01') OR
    (start_date <= '2024-07-07' AND end_date >= '2024-07-07') OR
    (start_date >= '2024-07-01' AND end_date <= '2024-07-07')
);

-- =================================================================
-- MONITORING AND MAINTENANCE
-- =================================================================

ANALYZE TABLE User;
ANALYZE TABLE Property;
ANALYZE TABLE Booking;
ANALYZE TABLE Review;
ANALYZE TABLE Payment;

-- Check index usage
SELECT 
    TABLE_NAME,
    INDEX_NAME,
    NON_UNIQUE,
    COLUMN_NAME,
    CARDINALITY,
    INDEX_TYPE
FROM 
    INFORMATION_SCHEMA.STATISTICS 
WHERE 
    TABLE_SCHEMA = DATABASE()
    AND TABLE_NAME IN ('User', 'Property', 'Booking', 'Review', 'Payment')
ORDER BY 
    TABLE_NAME, INDEX_NAME;

-- Check index size
SELECT 
    TABLE_NAME,
    ROUND(((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024), 2) AS 'Total Size (MB)',
    ROUND((INDEX_LENGTH / 1024 / 1024), 2) AS 'Index Size (MB)'
FROM 
    INFORMATION_SCHEMA.TABLES 
WHERE 
    TABLE_SCHEMA = DATABASE()
    AND TABLE_NAME IN ('User', 'Property', 'Booking', 'Review', 'Payment')
ORDER BY 
    INDEX_LENGTH DESC;

-- =================================================================
-- END OF FILE
-- =================================================================
