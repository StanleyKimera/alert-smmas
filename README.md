# alert-smmas
#alert entity for the SMMAS project
<?php
require_once 'config/database.php';
require_once 'includes/auth.php';
checkAuth();

$conn = getConnection();
$message = '';

// Clear old resolved alerts
$conn->exec("DELETE FROM Alert WHERE alert_status = 'Resolved' AND date_resolved < DATE_SUB(NOW(), INTERVAL 1 DAY)");

// Check for out of stock alerts (Critical)
$out_of_stock = $conn->query("
    SELECT m.medicine_id, m.medicine_name, m.reorder_level, COALESCE(SUM(b.qty_remaining), 0) as total_stock
    FROM Medicine m
    LEFT JOIN Batch b ON m.medicine_id = b.medicine_id AND b.batch_status = 'Active'
    GROUP BY m.medicine_id
    HAVING total_stock = 0
");

while($medicine = $out_of_stock->fetch()) {
    $check = $conn->prepare("SELECT alert_id FROM Alert WHERE medicine_id = ? AND alert_type = 'Stockout' AND alert_status = 'Active'");
    $check->execute([$medicine['medicine_id']]);
    if($check->rowCount() == 0) {
        $alert_id = 'ALT-' . date('Ymd') . '-' . rand(100, 999);
        $message_text = "CRITICAL: {$medicine['medicine_name']} is OUT OF STOCK!";
        $conn->prepare("INSERT INTO Alert (alert_id, medicine_id, alert_type, severity_level, date_generated, alert_message) VALUES (?, ?, 'Stockout', 'Critical', NOW(), ?)")->execute([$alert_id, $medicine['medicine_id'], $message_text]);
    }
}

// Check for low stock alerts (Warning)
$low_stock = $conn->query("
    SELECT m.medicine_id, m.medicine_name, m.reorder_level, COALESCE(SUM(b.qty_remaining), 0) as total_stock
    FROM Medicine m
    LEFT JOIN Batch b ON m.medicine_id = b.medicine_id AND b.batch_status = 'Active'
    GROUP BY m.medicine_id
    HAVING total_stock <= m.reorder_level AND total_stock > 0
");

while($medicine = $low_stock->fetch()) {
    $check = $conn->prepare("SELECT alert_id FROM Alert WHERE medicine_id = ? AND alert_type = 'Low Stock' AND alert_status = 'Active'");
    $check->execute([$medicine['medicine_id']]);
    if($check->rowCount() == 0) {
        $alert_id = 'ALT-' . date('Ymd') . '-' . rand(100, 999);
        $message_text = "Warning: {$medicine['medicine_name']} has only {$medicine['total_stock']} units left. Reorder level is {$medicine['reorder_level']}.";
        $conn->prepare("INSERT INTO Alert (alert_id, medicine_id, alert_type, severity_level, date_generated, alert_message) VALUES (?, ?, 'Low Stock', 'Warning', NOW(), ?)")->execute([$alert_id, $medicine['medicine_id'], $message_text]);
    }
}

// Check for expiry alerts
$expiring = $conn->query("
    SELECT b.batch_id, b.expiry_date, m.medicine_name, m.medicine_id, DATEDIFF(b.expiry_date, CURDATE()) as days_left
    FROM Batch b
    JOIN Medicine m ON b.medicine_id = m.medicine_id
    WHERE b.expiry_date BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)
    AND b.batch_status = 'Active'
");

while($batch = $expiring->fetch()) {
    $check = $conn->prepare("SELECT alert_id FROM Alert WHERE batch_id = ? AND alert_status = 'Active'");
    $check->execute([$batch['batch_id']]);
    if($check->rowCount() == 0) {
        $alert_id = 'ALT-' . date('Ymd') . '-' . rand(100, 999);
        $severity = ($batch['days_left'] <= 7) ? 'Critical' : 'Warning';
        $message_text = ($batch['days_left'] <= 7 ? "CRITICAL: " : "Warning: ") . "Batch of {$batch['medicine_name']} expires in {$batch['days_left']} days on " . date('d M Y', strtotime($batch['expiry_date']));
        $conn->prepare("INSERT INTO Alert (alert_id, medicine_id, batch_id, alert_type, severity_level, date_generated, alert_message) VALUES (?, ?, ?, 'Expiry Warning', ?, NOW(), ?)")->execute([$alert_id, $batch['medicine_id'], $batch['batch_id'], $severity, $message_text]);
    }
}

// Handle alert actions
if(isset($_POST['acknowledge_alert'])) {
    $alert_id = $_POST['alert_id'];
    $conn->prepare("UPDATE Alert SET alert_status = 'Acknowledged' WHERE alert_id = ?")->execute([$alert_id]);
    $message = '<div class="success" style="background:#d4edda; color:#155724; padding:10px; border-radius:5px; margin-bottom:15px;">✅ Alert acknowledged</div>';
}

if(isset($_POST['resolve_alert']) && $_SESSION['role'] == 'Admin') {
    $alert_id = $_POST['alert_id'];
    $conn->prepare("UPDATE Alert SET alert_status = 'Resolved', date_resolved = NOW() WHERE alert_id = ?")->execute([$alert_id]);
    $message = '<div class="success" style="background:#d4edda; color:#155724; padding:10px; border-radius:5px; margin-bottom:15px;">✅ Alert resolved</div>';
}

// Get counts (ONLY active alerts)
$critical_count = $conn->query("SELECT COUNT(*) as count FROM Alert WHERE severity_level = 'Critical' AND alert_status = 'Active'")->fetch()['count'];
$warning_count = $conn->query("SELECT COUNT(*) as count FROM Alert WHERE severity_level = 'Warning' AND alert_status = 'Active'")->fetch()['count'];

// Get critical alerts (ONLY active)
$critical_alerts = $conn->query("
    SELECT a.*, m.medicine_name, b.batch_number 
    FROM Alert a
    JOIN Medicine m ON a.medicine_id = m.medicine_id
    LEFT JOIN Batch b ON a.batch_id = b.batch_id
    WHERE a.severity_level = 'Critical' AND a.alert_status = 'Active'
    ORDER BY a.date_generated DESC
")->fetchAll();

// Get warning alerts (ONLY active)
$warning_alerts = $conn->query("
    SELECT a.*, m.medicine_name, b.batch_number 
    FROM Alert a
    JOIN Medicine m ON a.medicine_id = m.medicine_id
    LEFT JOIN Batch b ON a.batch_id = b.batch_id
    WHERE a.severity_level = 'Warning' AND a.alert_status = 'Active'
    ORDER BY a.date_generated DESC
")->fetchAll();
?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>Alert Dashboard - SMMAS</title>
<link rel="stylesheet" href="css/style.css" />
<style>
body { background: #f5f5f5; }
.alert-stats {
    display: flex;
    gap: 20px;
    margin: 20px 0 30px;
}
.stat-box {
    flex: 1;
    background: white;
    padding: 25px;
    border-radius: 10px;
    text-align: center;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}
.stat-box.critical { border-bottom: 4px solid #e74c3c; }
.stat-box.warning { border-bottom: 4px solid #f39c12; }
.stat-number {
    font-size: 42px;
    font-weight: bold;
}
.stat-box.critical .stat-number { color: #e74c3c; }
.stat-box.warning .stat-number { color: #f39c12; }
.stat-label {
    font-size: 14px;
    color: #666;
    margin-top: 5px;
}
.alert-list {
    margin-top: 20px;
}
.alert-item {
    background: white;
    border-radius: 8px;
    padding: 15px 20px;
    margin-bottom: 15px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}
.alert-item.critical {
    border-left: 4px solid #e74c3c;
    background: #fff5f5;
}
.alert-item.warning {
    border-left: 4px solid #f39c12;
    background: #fffbf0;
}
.alert-title {
    font-weight: bold;
    font-size: 15px;
    margin-bottom: 5px;
}
.alert-title.critical { color: #c0392b; }
.alert-title.warning { color: #e67e22; }
.alert-message {
    color: #555;
    margin: 8px 0;
    font-size: 13px;
}
.alert-date {
    font-size: 11px;
    color: #999;
    margin-bottom: 10px;
}
.empty-state {
    text-align: center;
    padding: 50px;
    background: white;
    border-radius: 10px;
    color: #999;
}
.empty-icon {
    font-size: 48px;
    margin-bottom: 15px;
}
.btn-sm {
    padding: 5px 12px;
    border: none;
    border-radius: 3px;
    cursor: pointer;
    font-size: 12px;
}
.btn-acknowledge { background: #3498db; color: white; }
.btn-resolve { background: #2ecc71; color: white; margin-left: 5px; }
.refresh-btn {
    background: #0A4F6E;
    color: white;
    padding: 8px 20px;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    margin-bottom: 20px;
}
h2 {
    color: #2c3e50;
    margin: 25px 0 15px;
    font-size: 20px;
}
hr {
    margin: 20px 0;
    border: none;
    border-top: 1px solid #ddd;
}
</style>
</head>
<body>
<?php include 'includes/navbar.php'; ?>

<div class="container">
    <h1>🔔 Alert Dashboard</h1>
    <p style="color: #000000; margin-bottom: 15px;">Real-time Medicine Expiry and Stock Alerts</p>
    
    <button onclick="location.reload()" class="refresh-btn">🔄 Refresh Alerts</button>
    
    <?php echo $message; ?>
    
    <!-- Statistics -->
    <div class="alert-stats">
        <div class="stat-box critical">
            <div class="stat-number"><?php echo $critical_count; ?></div>
            <div class="stat-label">🔥 Critical Alerts</div>
            <small style="color:#999;">Requires immediate action</small>
        </div>
        <div class="stat-box warning">
            <div class="stat-number"><?php echo $warning_count; ?></div>
            <div class="stat-label">⚠️ Warning Alerts</div>
            <small style="color:#999;">Needs attention</small>
        </div>
    </div>
    
    <!-- Critical Alerts Section -->
    <h2>🔥 Critical Alerts</h2>
    <div class="alert-list">
        <?php if(count($critical_alerts) > 0): ?>
            <?php foreach($critical_alerts as $alert): ?>
            <div class="alert-item critical">
                <div class="alert-title critical">🚨 <?php echo $alert['alert_type']; ?> - <?php echo $alert['medicine_name']; ?></div>
                <div class="alert-message"><?php echo $alert['alert_message']; ?></div>
                <div class="alert-date">📅 <?php echo date('d M Y h:i A', strtotime($alert['date_generated'])); ?></div>
                <form method="post">
                    <input type="hidden" name="alert_id" value="<?php echo $alert['alert_id']; ?>">
                    <button type="submit" name="acknowledge_alert" class="btn-sm btn-acknowledge">👁️ Acknowledge</button>
                    <?php if($_SESSION['role'] == 'Admin'): ?>
                    <button type="submit" name="resolve_alert" class="btn-sm btn-resolve">✅ Resolve</button>
                    <?php endif; ?>
                </form>
            </div>
            <?php endforeach; ?>
        <?php else: ?>
            <div class="empty-state">
                <div class="empty-icon">✅</div>
                <div><strong>No Critical Alerts</strong></div>
                <small>All medicines are in stock and no batches are expiring soon.</small>
            </div>
        <?php endif; ?>
    </div>
    
    <hr />
    
    <!-- Warning Alerts Section -->
    <h2>⚠️ Warning Alerts</h2>
    <div class="alert-list">
        <?php if(count($warning_alerts) > 0): ?>
            <?php foreach($warning_alerts as $alert): ?>
            <div class="alert-item warning">
                <div class="alert-title warning">⚠️ <?php echo $alert['alert_type']; ?> - <?php echo $alert['medicine_name']; ?></div>
                <div class="alert-message"><?php echo $alert['alert_message']; ?></div>
                <div class="alert-date">📅 <?php echo date('d M Y h:i A', strtotime($alert['date_generated'])); ?></div>
                <form method="post">
                    <input type="hidden" name="alert_id" value="<?php echo $alert['alert_id']; ?>">
                    <button type="submit" name="acknowledge_alert" class="btn-sm btn-acknowledge">👁️ Acknowledge</button>
                </form>
            </div>
            <?php endforeach; ?>
        <?php else: ?>
            <div class="empty-state">
                <div class="empty-icon">✅</div>
                <div><strong>No Warning Alerts</strong></div>
                <small>All stock levels are adequate and no batches are approaching expiry.</small>
            </div>
        <?php endif; ?>
    </div>
    
    <!-- Info Box -->
    <div style="margin-top: 30px; padding: 15px; background: #e7f3ff; border-radius: 8px;">
        <strong>ℹ️ Alert Information:</strong><br />
        <span style="color:#e74c3c;">🔴 Critical Alerts</span> - Medicine OUT OF STOCK or EXPIRING within 7 days<br />
        <span style="color:#f39c12;">🟡 Warning Alerts</span> - LOW stock (below reorder level) or EXPIRING within 30 days<br />
        <span style="color:#3498db;">👁️ Acknowledge</span> - Mark alert as SEEN<br />
        <span style="color:#2ecc71;">✅ Resolve</span> - Mark alert as RESOLVED (Admin only)
    </div>
</div>

<script>
// Auto-refresh every 120 seconds
setTimeout(function() {
    location.reload();
}, 120000);
</script>
</body>
</html>
