```php
<?php
// =============================
// 连接 Redis Cluster
// =============================
$seeds = [
    '192.168.226.100:7001',
    '192.168.226.100:7002',
    '192.168.226.101:7003',
    '192.168.226.101:7004',
    '192.168.226.102:7005',
    '192.168.226.102:7006'
];

try {
    $redis = new RedisCluster(
        NULL,          // 名称
        $seeds,        // 节点列表
        1.5,           // 连接超时
        1.5,           // 读写超时
        true,          // 持久连接
        'redhat'       // 集群密码
    );
} catch (Exception $e) {
    die("Redis Cluster connection failed: " . $e->getMessage());
}

// =============================
// 连接 MySQL 数据库（使用 mysqli）
// =============================
$mysqli = new mysqli('192.168.226.100', 'access01', '123456', 'gzc');
if ($mysqli->connect_errno) {
    die("MySQL connection failed: " . $mysqli->connect_error);
}

$query = "SELECT * FROM abc LIMIT 10";

// =============================
// 查询逻辑：先查 Redis，再查 MySQL
// =============================
$data = [];
$myserver = '';

for ($key = 1; $key < 6; $key++) {
    $value = $redis->get($key);
    if ($value === false) {
        // Redis中没有该key → 从MySQL取数据并写入Redis
        $result = $mysqli->query($query);
        if ($result) {
            while ($row = $result->fetch_assoc()) {
                $redis->set($row['id'], $row['name']);
                $data[$row['id']] = $row['name'];
            }
            $result->free();
        }
        $myserver = 'mysql';
        break;
    } else {
        // Redis中有该key → 直接使用缓存
        $data[$key] = $value;
        $myserver = 'redis';
    }
}

// =============================
// 输出结果
// =============================
echo "<b>Data Source:</b> <font color=blue>$myserver</font><br><br>";

foreach ($data as $id => $name) {
    echo "number is <b><font color=#FF0000>$id</font></b><br>";
    echo "name is <b><font color=#FF0000>$name</font></b><br><br>";
}

// =============================
// 关闭数据库连接
// =============================
$mysqli->close();
?>

```