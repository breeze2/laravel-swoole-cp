# laravel-swoole-cp
swoole connection pool for laravel

[php-cp](https://github.com/swoole/php-cp)的Laravel驱动

**注意**
1. 仅在Laravel5.4~5.5上测试通过，其他版本可能不适用；
2. PHP7环境请使用[breeze2/php-cp](https://github.com/breeze2/php-cp)，目前[swoole/php-cp](https://github.com/swoole/php-cp)v1.5.0版本对PHP7存在问题。

主要是三个文件：
1. laravel/framework/src/Illuminate/Database/MySqlSwooleProxyConnection.php
2. laravel/framework/src/Illuminate/Database/Connectors/MySqlSwooleProxyConnector.php
3. laravel/framework/src/Illuminate/Database/Connectors/ConnectionFactory.php

只要将这三个文件放到Laravel项目的vendor文件夹下便可。

`MySqlSwooleProxyConnection.php`和`MySqlSwooleProxyConnector.php`是模仿Laravel框架自有的`MySqlConnection.php`和`MySqlConnector.php`新增编写的；而`ConnectionFactory.php`则是在Laravel框架原有的基础上修改的，主要是调用`MySqlSwooleProxyConnection`和`MySqlSwooleProxyConnector`这两个新增类。

`ConnectionFactory.php`修改的地方：

```php
<?php
namespace Illuminate\Database\Connectors;

use Illuminate\Database\MySqlSwooleProxyConnection;
...
class ConnectionFactory
{
    ...
    public function createConnector(array $config)
    {
        if (! isset($config['driver'])) {
            throw new InvalidArgumentException('A driver must be specified.');
        }

        if ($this->container->bound($key = "db.connector.{$config['driver']}")) {
            return $this->container->make($key);
        }

        switch ($config['driver']) {
            case 'mysql-cp':
                return new MySqlSwooleProxyConnector;
            case 'mysql':
                return new MySqlConnector;
            case 'pgsql':
                return new PostgresConnector;
            case 'sqlite':
                return new SQLiteConnector;
            case 'sqlsrv':
                return new SqlServerConnector;
        }

        throw new InvalidArgumentException("Unsupported driver [{$config['driver']}]");
    }

    protected function createConnection($driver, $connection, $database, $prefix = '', array $config = [])
    {
        if ($resolver = Connection::getResolver($driver)) {
            return $resolver($connection, $database, $prefix, $config);
        }

        switch ($driver) {
            case 'mysql-cp':
                return new MySqlSwooleProxyConnection($connection, $database, $prefix, $config);
            case 'mysql':
                return new MySqlConnection($connection, $database, $prefix, $config);
            case 'pgsql':
                return new PostgresConnection($connection, $database, $prefix, $config);
            case 'sqlite':
                return new SQLiteConnection($connection, $database, $prefix, $config);
            case 'sqlsrv':
                return new SqlServerConnection($connection, $database, $prefix, $config);
        }

        throw new InvalidArgumentException("Unsupported driver [$driver]");
    }
}
```

注意，这里将php-cp的数据库连接驱动命名为`mysql-cp`，所以在Laravel项目的数据库配置`config/database.php`里应该这样配置php-cp的连接驱动：

```php
<?php
// config/database.php

return [
    'default' => env('DB_CONNECTION', 'mysql-cp'),

    'connections' => [

        'mysql-cp' => [
            'driver' => 'mysql-cp',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', '3306'),
            'database' => env('DB_DATABASE', 'forge'),
            'username' => env('DB_USERNAME', 'forge'),
            'password' => env('DB_PASSWORD', ''),
            'unix_socket' => env('DB_SOCKET', ''),
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
            'prefix' => '',
            'strict' => true,
            'engine' => null,
        ],
        ...
    ],
]

```


