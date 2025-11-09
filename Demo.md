# Demo Queries for NOVAIMS Gym Management System
-- ===== DEMO: Trigger #1 (check-ins -> visit_count++) =====

START TRANSACTION;
SELECT member_id, CONCAT(first_name,' ',last_name) AS name, visit_count
FROM members WHERE member_id=2;
INSERT INTO check_ins (member_id, check_in_time, location)
VALUES (2, NOW(), 'Front Desk');
SELECT member_id, CONCAT(first_name,' ',last_name) AS name, visit_count
FROM members WHERE member_id=2;
ROLLBACK;

-- ===== DEMO: Trigger #2 (order_items on Merch -> stock-- + log) =====
START TRANSACTION;
SELECT sku, stock_qty FROM products WHERE sku IN ('M-TEE','M-BOT');

INSERT INTO orders (member_id, order_date, status, payment_method)
VALUES (1, NOW(), 'paid', 'card');
SET @oid := LAST_INSERT_ID();

INSERT INTO order_items (order_id, product_id, description, qty, unit_price)
SELECT @oid, product_id, 'Demo T-Shirt', 1, unit_price
FROM products WHERE sku='M-TEE';

-- recompute order totals for the demo order
UPDATE orders o
JOIN (SELECT order_id, SUM(line_total) AS subtotal FROM order_items WHERE order_id=@oid) s
  ON s.order_id = o.order_id
SET o.subtotal = s.subtotal, o.tax_amount = ROUND(s.subtotal*0.23, 2);

-- show stock + log entry for this order
SELECT sku, stock_qty FROM products WHERE sku IN ('M-TEE','M-BOT');
SELECT action, table_name, record_id, details
FROM log_events
WHERE JSON_EXTRACT(details,'$.order_id') = @oid;

ROLLBACK;

-- ===== Invoice views (from seed data) =====
SELECT * FROM invoice_head_v ORDER BY order_id LIMIT 5;
SELECT * FROM invoice_lines_v WHERE order_id = 1;

-- ===== A couple KPIs =====
SELECT m.member_id, CONCAT(m.first_name,' ',m.last_name) AS member, SUM(o.total_amount) AS total_revenue
FROM orders o JOIN members m ON m.member_id=o.member_id
GROUP BY m.member_id, member ORDER BY total_revenue DESC LIMIT 5;

SELECT DATE_FORMAT(order_date, '%Y-%m') AS ym, SUM(total_amount) AS revenue
FROM orders GROUP BY ym ORDER BY ym;

SQL"
