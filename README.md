# ElasticsearchForPHP
基于Laravel框架，实现Elasticsearch搜索引擎的使用
首先我们现在本地搭建一个laravel项目，在你的专门存放代码的目录下通过composer安装一个laravel新项目包。
```
composer create-project laravel/laravel elasticsearch_demo
```
安装完成后进入项目根目录，依次安装好elasticsearch所需的第三方扩展包
```
composer require  elasticsearch/elasticsearch //elasticsearch核心包

composer require tamayo/laravel-scout-elastic

composer require laravel/scout "^8.x-dev" --dev

php artisan vendor:publish
```
##### 安装顺序不能弄错，否则会报错

在运行上面最后一条指令后，会出现这个界面：
![微信截图_20211209153900.png][1]


  [1]: https://www.wanone.cn/usr/uploads/2021/12/1888066600.png

我们选择 9 就行啦！

然后，我们找到config目录下的scout.php文件，修改驱动为elasticsearch
```
'driver' => env('SCOUT_DRIVER', 'elasticsearch'),
```
并在下方添加驱动：
```
    'elasticsearch' => [
        'hosts' => [
            env('ELASTICSEARCH_HOSTS','http://127.0.0.1:9200')
        ],
        'index' => env('ELASTICSEARCH_INDEX','laravel_es_test'),
    ]
```
在.env文件中配置上你的elasticsearch服务器地址及索引名称
```
ELASTICSEARCH_HOSTS= 你的服务器地址:9200
ELASTICSEARCH_INDEX= 你创建的索引名称
```

当索引创建完毕后，我们就可以通过laravel的command进行索引的创建和初始化。

```
php artisan make:command ESOpenCommand
```
在Console/Kernel.php中挂载上生成的类
```
protected $commands = [
\App\Console\Commands\ESOpenCommand::class
];
```
##### 如果laravel版本在8.0以上可以跳过这个步骤。
来到我们创建的ESOpenCommand.php中，写入以下代码：

```
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'es:open';
 /**
     * Execute the console command.
     *
     * @return array
     */
    public function handle(): array
    {
        $host = config('scout.elasticsearch.hosts');
        $index = config('scout.elasticsearch.index');
        $client = ClientBuilder::create()->setHosts($host)->build();

        if ($client->indices()->exists(['index' => $index])) {
            $this->warn("Index {$index} exists, deleting...");
            $client->indices()->delete(['index' => $index]);
        }

        $this->info("Creating index: {$index}");

        return $client->indices()->create([
            'index' => $index,
            'body' => [
                'settings' => [
                    'number_of_shards' => 1,
                    'number_of_replicas' => 0
                ],
                'mappings' => [
                    '_source' => [
                        'enabled' => true
                    ],
                    'properties' => [
                        'mapping' => [ // 字段的处理方式
                            'type' => 'keyword', // 字段类型限定为 string
                            'fields' => [
                                'raw' => [
                                    'type' => 'keyword',
                                    'ignore_above' => 256, // 字段是索引时忽略长度超过定义值的字段。
                                ]
                            ],
                        ],
                    ],
                ]
            ]
        ]);

    }
```
随后在控制台运行这个文件指令
```
php artisan es:open
```
这样我们索引也就弄好了！

接着，我们需要创建一个model对象来将我们数据填入elasticsearch索引中去。
```
php artian make:model Goods
```
编辑Goods文件如下：
```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Goods extends Model
{
    use HasFactory,Searchable;

    protected $table = 'goods';

    /**
     * 可以注入的字段
     * @var string[]
     */
    protected $fillable = [
        'name','detail'
    ];

    /**
     * 定义索引的类型
     * @return string
     */
    public function searchableAs(): string
    {
        return 'doc';
    }

    /**
     * 定义需要做搜索的字段
     * @return array
     */
    public function toSearchableArray(): array
    {
        return [
            'name' => $this->name,
            'detail' => $this->detail
        ];
    }
}

```
##### 请根据你自己的数据表情况而自行定义，我的仅供参考
导入我们数据
```
php artisan scout:import "App\Models\goods"
```
接下来，我们创建一个控制器类，来测试我们的elasticsearch是否配置成功
创建控制器的步骤我就不做赘述，验证方法如下:
```
    public function index(Request $request,string $name): JsonResponse
    {
        $res = Goods::search($name)->get();
        return new JsonResponse($res);
    }
```
浏览器访问该控制器 http://127.0.0.1:8000/goods/search/包包
返回结果如下：
```
[
  {
    "id": 1,
    "store_id": 2,
    "name": "2019新款新加坡限定情书包小 ｃｋ链条小包包潮时尚单肩斜挎包!!!!!!1",
    "price": "8520.00",
    "original_price": "9980.00",
    "detail": "<p>&nbsp;<img src=\"http://img.xinglico.com/xl-mall/uploads/image/store_2/201909/f6c587d3300788d9da53ebef1652a3b8edd9db24.jpeg\" title=\"\" alt=\"\"/></p>"
  },
  {
    "id": 3,
    "store_id": 2,
    "name": "COACH 蔻驰女士Messenger牛皮单肩斜挎信封包女包包",
    "price": "7998.00",
    "original_price": "8000.00",
    "detail": "<p><br/></p><p><img src=\"http://img.xinglico.com/xl-mall/uploads/image/store_2/201909/5b5399fd338bb3e6432b499075291223d45bbc28.jpeg\" title=\"\" alt=\"\"/>1</p>"
  },
  {
    "id": 4,
    "store_id": 2,
    "name": "COACH_蔲驰包包女PVC大号单肩手提托特tote女包 58292",
    "price": "1220.00",
    "original_price": "1450.00",
    "detail": "<p>&nbsp;&nbsp;&nbsp;&nbsp; <br/></p><p><img src=\"http://img.xinglico.com/xl-mall/uploads/image/store_2/201909/356933f72b5ce4ce6f7506396e3df3140d80502e.jpeg\" title=\"\" alt=\"\"/></p>"
  },
  {
    "id": 5,
    "store_id": 2,
    "name": "Fion_菲安妮2019新款小众轻奢女包包 链条小方包 蜜蜂单肩斜挎包",
    "price": "879.00",
    "original_price": "1200.00",
    "detail": "<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <br/></p><p>&nbsp;<img src=\"http://img.xinglico.com/xl-mall/uploads/image/store_2/201909/d42d1d0f36aa404094a1b7aca090f2d86f25d8c6.jpeg\" title=\"\" alt=\"\"/></p>"
  },
  {
    "id": 13,
    "store_id": 2,
    "name": "JELLYTOYBOY潮牌包包女单肩包女包",
    "price": "1152.00",
    "original_price": "1500.00",
    "detail": "<p><img src=\"http://img.xinglico.com/xl-mall/uploads/image/store_2/201909/5487e2423ce051f2a9fef5d3f84796b41de1ae6c.jpg\"/></p><p><img src=\"http://img.xinglico.com/xl-mall/uploads/image/store_2/201909/710ffa7df086d72bb1c9fe066734008460620679.jpg\"/></p><p><img src=\"http://img.xinglico.com/xl-mall/uploads/image/store_2/201909/a5d3a32400f71a2426c976693396e1c257a7754c.jpg\"/></p><p><br/></p>"
  }
]
```
至此，我们的elasticsearch配置完成啦！！！！属于我们自己的搜索引擎成功搭建！
