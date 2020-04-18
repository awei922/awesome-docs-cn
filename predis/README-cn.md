# Predis中文说明

> GitHub：https://github.com/nrk/predis

灵活且功能齐全的[Redis](http://redis.io/)客户端，适用于PHP> = 5.3和HHVM> = 2.3.0。

Predis默认不需要任何额外的C扩展，但它可以选择与[phpiredis](https://github.com/nrk/phpiredis)配对，以降低[Redis RESP协议](http://redis.io/topics/protocol)的序列化和解析的开销。对于客户端的**试验性的**异步实现，您可以参考[PredisAsync](https://github.com/nrk/predis-async)。

有关此项目的更多详细信息，请参阅[常见问题解答](https://wpfaq.cn/archives/FAQ.md)。

主要特点
----

*   使用配置可以支持不同版本的Redis（从**2.0**到**3.2**）。
*   支持使用客户端分片和可插拔密钥空间分发器进行集群。
*   支持[redis-cluster](http://redis.io/topics/cluster-tutorial)（Redis> = 3.0）。
*   支持主从复制设置和[redis-sentinel](http://redis.io/topics/sentinel)。
*   使用可自定义前缀策略对键进行透明的键前缀。
*   可在单个节点和集群上使用命令管道操作（仅限客户端分片）。
*   Redis事务的抽象（Redis> = 2.0）和CAS操作（Redis> = 2.2）。
*   Lua脚本的抽象（Redis> = 2.6）可以在`EVALSHA`和`EVAL`之间自动切换。
*   在`SCAN`，`SSCAN`，`ZSCAN`和`HSCAN`（Redis的> = 2.8）的基础上抽象PHP迭代器。
*   客户端在第一个命令时惰性地建立连接后，可以保持连接。
*   可以通过TCP / IP（也是TLS / SSL加密）或UNIX域套接字建立连接。
*   支持[Webdis](http://webd.is/)（需要`ext-curl`和`ext-phpiredis`）。
*   支持自定义连接类，以提供不同的网络或协议后端。
*   灵活的系统，用于定义自定义命令和配置文件来覆盖默认命令。

安装
--

可以在[Packagist](http://packagist.org/packages/predis/predis)上找到此库，然后使用[Composer](http://packagist.org/about-composer)更轻松地管理项目依赖项，也可以在我们[自己的PEAR通道](http://pear.nrk.io/)上使用PEAR进行更传统的安装。总之，每个版本的压缩归档都可以在[GitHub](https://github.com/nrk/predis/releases)上获取。

```
composer require predis/predis
```

使用
--

### 加载库

Predis可以依靠PHP的自动加载功能按需加载并符合[PSR-4标准](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md)。当通过Composer管理依赖项时会自动处理自动加载，当项目中缺少自动加载功能或脚本时可以利用自己的自动加载器：

```
// Prepend a base path if Predis is not available in your "include_path".
require 'Predis/Autoloader.php';

PredisAutoloader::register(); 
```

也可以通过启动脚本`bin/create-phar`直接从存储库创建[phar](http://www.php.net/manual/en/intro.phar.php)存档。生成的phar已经包含一个定义自己的自动加载器的存根，因此您只需要`require()`它就可以开始使用该库。

### 连接到Redis

如果不传递任何连接参数创建一个客户端实例，Predis假设`127.0.0.1`和`6379`为默认的主机和端口。该`connect()`操作的默认超时为5秒：

```
$client = new PredisClient();
$client->set('foo', 'bar');
$value = $client->get('foo'); 
```

连接参数可以以URI字符串或命名数组的形式提供。后者是提供参数的首选方法，但是当从非结构化或部分结构化的源读取参数时，URI字符串可能很有用：

```
// Parameters passed using a named array:
$client = new PredisClient([
    'scheme' => 'tcp',
    'host'   => '10.0.0.1',
    'port'   => 6379,
]);

// Same set of parameters, passed using an URI string:
$client = new PredisClient('tcp://10.0.0.1:6379'); 
```

也可以使用UNIX域套接字连接到Redis的本地实例，在这种情况下，参数必须使用该`unix`方案并指定套接字文件的路径：

```
$client = new PredisClient(['scheme' => 'unix', 'path' => '/path/to/redis.sock']);
$client = new PredisClient('unix:/path/to/redis.sock'); 
```

客户端可以利用TLS/SSL加密连接到安全的远程Redis实例，而无需配置像stunnel这样的SSL代理。当连接到在各种云托管提供商上运行的节点时，这可能很有用。可以使用该`tls`方案和通过参数传递的合适[选项](http://php.net/manual/context.ssl.php)数组来启用加密`ssl`：

```
// Named array of connection parameters:
$client = new PredisClient([
  'scheme' => 'tls',
  'ssl'    => ['cafile' => 'private.pem', 'verify_peer' => true],
]);

// Same set of parameters, but using an URI string:
$client = new PredisClient('tls://127.0.0.1?ssl[cafile]=private.pem&ssl[verify_peer]=1'); 
```

还支持连接方案[`redis`](http://www.iana.org/assignments/uri-schemes/prov/redis)（别名`tcp`）和[`rediss`](http://www.iana.org/assignments/uri-schemes/prov/rediss)（别名`tls`），区别在于包含这些方案的URI字符串是按照各自的IANA临时注册文档中描述的规则进行解析的。

支持的连接参数的实际列表可能因每个连接后端而异，因此建议您参考其特定文档或实现以获取详细信息。

提供连接参数数组时，Predis将使用客户端分片自动在群集模式下工作。在为每个节点提供配置时，可以混合使用命名数组和URI字符串：

```
$client = new PredisClient([
    'tcp://10.0.0.1?alias=first-node',
    ['host' => '10.0.0.2', 'alias' => 'second-node'],
]); 
```

有关更多详细信息，请参阅本文档的[聚合连接](https://wpfaq.cn/archives/20034.html#aggregate-connections)部分。

与Redis的连接是惰性的，这意味着客户端仅在需要时才连接到服务器。虽然建议让客户端在底层自动处理，但有时候仍然需要控制何时打开或关闭连接：这可以通过调用`$client->connect()`和轻松实现`$client->disconnect()`。请注意，这些方法对聚合连接的影响可能因具体实现而异。

### 客户端配置

客户端的许多方面和行为可以通过将特定客户端选项传递给`PredisClient::__construct()`的第二个参数来配置：

```
$client = new PredisClient($parameters, ['profile' => '2.8', 'prefix' => 'sample:']); 
```

使用一个迷你类似DI的容器来管理选项，只有在需要时才可以懒惰地初始化它们的值。Predis中默认支持的客户端选项是：

*   `profile`：指定用于匹配特定版本Redis的配置文件。
*   `prefix`：前缀字符串，自动应用于命令中找到的键。
*   `exceptions`：客户端是否应该在Redis错误时抛出或返回响应。
*   `connections`：连接后端列表或连接工厂实例。
*   `cluster`：指定簇后端（`predis`，`redis`或可调用对象）。
*   `replication`：指定一个复制的后端（`TRUE`，`sentinel`或可调用对象）。
*   `aggregate`：覆盖`cluster`并`replication`提供自定义连接聚合器。
*   `parameters`：聚合连接的默认连接参数列表。

用户还可以提供带有值或可调用对象（用于延迟初始化）的自定义选项，这些对象存储在选项容器中供以后通过库使用。

### 聚合连接

聚合连接是Predis实现集群和复制的基础，它们用于将多个连接分组到单个Redis节点，并隐藏特定的逻辑，以根据上下文适当地处理它们。在创建新客户端实例时，聚合连接通常需要一个连接参数数组。

#### 集群

默认情况下，当没有设置特定的客户端选项只是将连接参数数组传递给客户端的构造函数时，PrdIs使用传统的客户端分割方法来配置自己在集群模式下工作，创建独立节点的集群，并在其中分配密钥空间。 这种方法需要某种形式的外部健康监测节点，需要手动操作来重新平衡密钥空间，并通过添加或删除节点改变其配置：

```
$parameters = ['tcp://10.0.0.1', 'tcp://10.0.0.2', 'tcp://10.0.0.3'];

$client = new PredisClient($parameters); 
```

随着Redis 3.0的出现，以[redis-cluster](http://redis.io/topics/cluster-tutorial)的形式引入了一种新的监督和协调类型的集群。这种方法使用不同的算法来分配密钥空间，Redis节点通过通过gossip协议进行通信来协调自身，以处理健康状态，重新平衡，节点发现和请求重定向。为了连接到redis-cluster管理的集群，客户端需要一个节点列表（不一定完整，因为它会在必要时自动发现新节点），并设置客户端`cluster`选项为`redis`：

```
$parameters = ['tcp://10.0.0.1', 'tcp://10.0.0.2', 'tcp://10.0.0.3'];
$options    = ['cluster' => 'redis'];

$client = new PredisClient($parameters, $options); 
```

#### 主从复制

客户端可以配置在单个主/从设备设置中运行，以可靠地提供更好的服务。使用复制时，Predis会识别只读命令并将它们发送到随机从服务器，以便提供某种负载平衡，并在检测到执行任何操作的命令时立即切换到主服务器，比如该命令将修改键空间或键的值。当从设备发生故障时，不会引发连接错误，客户机试图在配置中提供的从服务器中返回到另一个从服务器。

在复制模式下需要客户端在基本配置将一个Redis服务器标识为主服务器（这可以通过连接参数设置`alias`为`master`）和一个或多个从服务器：

```
$parameters = ['tcp://10.0.0.1?alias=master', 'tcp://10.0.0.2', 'tcp://10.0.0.3'];
$options    = ['replication' => true];

$client = new PredisClient($parameters, $options); 
```

上述配置有一个静态的服务器列表，完全依赖于客户端的逻辑，但在一个更强大的HA环境可以依靠于[`redis-sentinel`](http://redis.io/topics/sentinel)，而sentinel服务器充当服务发现的客户端。客户端使用redis-sentinel所需的最小配置是指向一堆Sentinel实例的连接参数列表，`replication`选项设置为`sentinel`，并且`service`选项设置为服务名称：

```
$sentinels = ['tcp://10.0.0.1', 'tcp://10.0.0.2', 'tcp://10.0.0.3'];
$options   = ['replication' => 'sentinel', 'service' => 'mymaster'];

$client = new PredisClient($sentinels, $options); 
```

如果主节点和从节点配置为需要客户端进行身份验证，则必须通过全局`parameters`客户端选项提供密码。此选项还可用于指定其他数据库索引。客户端选项数组将如下所示：

```
$options = [
    'replication' => 'sentinel',
    'service' => 'mymaster',
    'parameters' => [
        'password' => $secretpassword,
        'database' => 10,
    ],
]; 
```

虽然Predis能够区分执行写入和只读操作的命令，`EVAL`并且`EVALSHA`表示客户端切换到主节点的一种极端情况，因为它无法判断何时可以安全地在从属服务器上执行Lua脚本。虽然这确实是默认行为，但是当某些Lua脚本不执行写操作时，可以提供一个提示，告诉客户端在从服务器继续执行:

```
$parameters = ['tcp://10.0.0.1?alias=master', 'tcp://10.0.0.2', 'tcp://10.0.0.3'];
$options    = ['replication' => function () {
    // Set scripts that won't trigger a switch from a slave to the master node.
    $strategy = new PredisReplicationReplicationStrategy();
    $strategy->setScriptReadOnly($LUA_SCRIPT);

    return new PredisConnectionAggregateMasterSlaveReplication($strategy);
}];

$client = new PredisClient($parameters, $options);
$client->eval($LUA_SCRIPT, 0);             // Sticks to slave using `eval`...
$client->evalsha(sha1($LUA_SCRIPT), 0);    // ... and `evalsha`, too. 
```

该[`examples`](https://wpfaq.cn/archives/examples/)目录包含一些脚本，用于演示如何配置客户端以及如何在基本和复杂方案中利用主从复制。

### 命令管道

当许多命令需要发送到服务器时，通过减少网络往返时间带来的延迟，管道操作可以帮助提高性能。管道操作也适用于聚合连接。客户端可以在可调用块内执行管道，或者通过其流畅的接口返回能够链接命令的管道实例：

```
// Executes a pipeline inside the given callable block:
$responses = $client->pipeline(function ($pipe) {
    for ($i = 0; $i < 1000; $i++) {
        $pipe->set("key:$i", str_pad($i, 4, '0', 0));
        $pipe->get("key:$i");
    }
});

// Returns a pipeline that can be chained thanks to its fluent interface:
$responses = $client->pipeline()->set('foo', 'bar')->get('foo')->execute(); 
```

### 交易

客户端提供了用于基于Redis的事务的抽象`MULTI`和`EXEC`具有相似的接口命令管道：

```
// Executes a transaction inside the given callable block:
$responses = $client->transaction(function ($tx) {
    $tx->set('foo', 'bar');
    $tx->get('foo');
});

// Returns a transaction that can be chained thanks to its fluent interface:
$responses = $client->transaction()->set('foo', 'bar')->get('foo')->execute(); 
```

这个抽象可以执行检查和设置操作，这要归功于`WATCH`和`UNWATCH`并且在触摸`WATCH` ed键时Redis提供中止的事务的自动重试。有关使用CAS的事务示例，您可以看到[以下示例](https://wpfaq.cn/archives/examples/transaction_using_cas.php)。

### 添加新命令

虽然我们尝试更新Predis以及时了解Redis中可用的所有命令，但您可能更喜欢使用旧版本的库或提供不同的方法来过滤参数或解析特定命令的响应。为实现这一目标，Predis提供了实现新命令类的能力，以定义或覆盖客户端使用的默认服务器配置文件中的命令：

```
// Define a new command by extending PredisCommandCommand:
class BrandNewRedisCommand extends PredisCommandCommand
{
    public function getId()
    {
        return 'NEWCMD';
    }
}

// Inject your command in the current profile:
$client = new PredisClient();
$client->getProfile()->defineCommand('newcmd', 'BrandNewRedisCommand');

$response = $client->newcmd(); 
```

还有一种方法可以在不过滤参数或解析响应的情况下发送原始命令。用户必须按照[Redis文档中为命令](http://redis.io/commands)定义的签名，以数组形式提供该命令的参数列表：

```
$response = $client->executeRaw(['SET', 'foo', 'bar']); 
```

### 脚本命令

虽然可以利用[Lua脚本](http://redis.io/commands/eval)在Redis的2.6+直接使用[`EVAL`](http://redis.io/commands/eval)和[`EVALSHA`](http://redis.io/commands/evalsha)，Predis为给他们提供了化繁为简的更高层次的抽象脚本命令。脚本命令可以在客户端使用的服务器配置文件中注册，并且可以像普通的Redis命令一样进行访问，但是它们定义了传输到服务器以进行远程执行的Lua脚本。在内部，它们默认使用[`EVALSHA`](http://redis.io/commands/evalsha)并通过其SHA1哈希标识脚本以节省带宽，但在需要时用[`EVAL`](http://redis.io/commands/eval)作后备：

```
// Define a new script command by extending PredisCommandScriptCommand:
class ListPushRandomValue extends PredisCommandScriptCommand
{
    public function getKeysCount()
    {
        return 1;
    }

    public function getScript()
    {
        return <<<LUA
math.randomseed(ARGV[1])
local rnd = tostring(math.random())
redis.call('lpush', KEYS[1], rnd)
return rnd
LUA;
    }
}

// Inject the script command in the current profile:
$client = new PredisClient();
$client->getProfile()->defineCommand('lpushrand', 'ListPushRandomValue');

$response = $client->lpushrand('random_values', $seed = mt_rand()); 
```

### 可定制的连接后端

Predis可以使用不同的连接后端连接到Redis。其中两个利用第三方扩展，如[phpiredis，](https://github.com/nrk/phpiredis)导致主要的性能提升，特别是在处理大型多重响应时。一个基于PHP流，另一个基于提供的套接字资源`ext-socket`。两者都支持TCP / IP和UNIX域套接字：

```
$client = new PredisClient('tcp://127.0.0.1', [
    'connections' => [
        'tcp'  => 'PredisConnectionPhpiredisStreamConnection',  // PHP stream resources
        'unix' => 'PredisConnectionPhpiredisSocketConnection',  // ext-socket resources
    ],
]); 
```

开发人员可以创建自己的连接类来支持全新的网络后端，扩展现有的类或提供完全不同的实现。连接类必须继承`PredisConnectionNodeConnectionInterface`接口或继承`PredisConnectionAbstractConnection`：

```
class MyConnectionClass implements PredisConnectionNodeConnectionInterface
{
    // Implementation goes here...
}

// Use MyConnectionClass to handle connections for the `tcp` scheme:
$client = new PredisClient('tcp://127.0.0.1', [
    'connections' => ['tcp' => 'MyConnectionClass'],
]); 
```

想更深入了解有关如何创建新连接后端，您可以参考`PredisConnection`命名空间中可用的标准连接类的实际案例。