### 数据预处理
- 数据库：建立一个用户Document，可镶嵌的collections存入mongodb
- 利用Excel的筛选功能确定年龄、性别、学历等部分信息的属性先提取两千条数据进行预处理测试
- 目标用户表：ID唯一，遍历求所有用户停留时间平均值，有相同ID的属性列保留去除偏离平均值最小的数据项；同时记录各类用户数量，为了避免意外数据，非此三种的数据删除；并将停留时间分箱。
- 目标用户身份标签：遍历表项，根据不同属性的分类标准对年龄、性别、学历、专业属性进行量化
- 目标用户行为标签：遍历列表将一个用户的数据分为标签数据和指数数据两类，将行为标签归类进行二值量化，对指数数据分箱（500，1000，2000，5000、10000为分界线），建立统一文档记录。
- 电商行为：利用正则表达式匹配和分词手段提取品牌关键词列表，遍历表项根据描述信息匹配关键词，从而手机类型实现数据项的精确品牌分类，对于分类后列表中出现次数少于200条的品牌统一归入“小众品牌”；将除手机外的电商数据进行类别合并，减少类别数量；同样利用关键词列表中筛选出的电商网站名称，二次遍历列表提取用户的电商网站偏好，所有数据提取后统一编入数据库。
- 视频行为：首先将一级分类与二级分类合并（一级分类_二级分类)，由于包含大量未分类数据，因此首先遍历列表将带标签数据与未分类数据分离，之后分别对未分类数据在带标签数据集中进行相似度匹配，即将二者的视频内容字符串进行匹配，相同字符个数/较短字符串字符个数作为二者相似度，相似度超过0.5则记为二者内容相关，将带标签数据标签赋予未分类数据项；分类完成后，对出现次数少于500次的分类类别进行合并，之后再次遍历，统计各用户对于各类视频的浏览时长和，存入数据库。
- 触媒行为：舍弃搜索子类名称，保留浏览网页名称，按照网页名称进行分类统计，对于同一个用户，不同搜索子类名称记录一个，并记录搜索次数。
```json
 "User": {
        "id": "用户id",
        "stay_time": "网站平均停留时间/s",
        "next_step" : "search/buy/browse",
        "basic_labels": {
            "age": "年龄（'18-24': 0, '25-34': 1, '35-44': 2, '45-54': 3）",
            "gender": "性别（'男': 0, '女': 1）",
            "profession": "major_x",
            "edu": "学历（'高中及以下': 0, '大学专科': 1, '大学本科': 2, '硕士及以上': 3, '其他': -1）"
        },
        "behavior_labels":{
            "xx指数": "数值",
            "偏好标签": "url，列表形式"
        },
        "website_labels": {
            "website_x" : {
            "name" : "website_x",
            "class" : "网页类型",
            "url" : "网页地址",
            "name_count" : "访问频数"
            }
        },
        "video_labels": {
            "video_x": "视频类型累计观看时长"
        },
        "shopping_labels":{
            "phone": {
                "shopping_x": "手机品牌访问时间累计"
            },
            "type": {
                "shopping_x":"手机外网购产品种类浏览时间"
            },
            "website": {
                "shopping_x": "电商网站访问时间累计"
            },
        }
} 
```
## 算法要点（4题）
### （1）数据展开
将非结构数据转换为数值二维列表，以供决策树进行训练，展开后数据标签列表如下：
### 标签列表：
- f0 购买行为 next_step
- f1 用户标识 id
- f2 年龄 age
- f3 专业 major
- f4 性别 gender
- f5 学历 edu
- f6 平均停留时间 stay_time
- f7-13 用户偏好指数 index_n
- f14-39 用户网页偏好 top5_n
- f40-60 用户网页访问次数 website_n
- f60-111 用户电商购物偏好 shopping_n
- f111-177 用户视频浏览偏好 video_n
### 算法特点
- 缺失值处理：通过学习，对每个节点记录使损失函数最优的缺失值流向来对预测和训练数据中的缺失值进行特征分割，因此可以省去2、3问中避免缺失值而使用的数组包含特征判别条件，使模型更加简单。
- 样本平衡：题目提供样本数据中，负样本：正样本=32，xgboost算法为解决样本不平衡问题提供了增大采样间隔和提高正负样本比例的方法，效果显著
- 交叉验证和格网式参数优化，便于找到最佳的模型参数，并提高在没有测试数据情况下模型的稳定性，减少过拟合的出现。
经过多次交叉验证训练，得到参数如下：
- subsample=0.9
- colsample_bytree=0.7
- gamma=0
- reg_alpha=225
- reg_lambda=1
- max_depth=3
- min_child_weight=3
- n_estimators=406
- scale_pos_weight=9
- learning_rate=0.01
- missing=-1
### 2、3问CART决策树关键点
- 递归优化函数使用信息增量，即子节点熵加权均值与父节点熵之差作为衡量判别指标的优化标准。
- 判别指标分为三类：(1)数值比较(2)字符判断(3)数组包含（减少数据缺失）
- 缺失值处理方式：尝试将缺失值分割到左右子节点，选择信息增量大的子节点，之后在节点上记录缺失值走向，在预测时缺失值遵循该走向。
- 使用最小信息增量法对生成的决策树进行剪枝，合并多余节点，避免出现过拟合