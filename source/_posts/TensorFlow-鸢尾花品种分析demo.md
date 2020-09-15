---
layout: next
title: 'TensorFlow:鸢尾花品种分析demo'
date: 2020-09-15 16:52:20
tags: TensorFlow Python
---
## 运行效果

![](https://note.youdao.com/yws/public/resource/1f52c84118d87ca95bb96cf11951ce88/xmlnote/DEE9C090352A4BBF8281CA269958696D/8343)

```
root@433ad0b9b2e7:/usr/src/app# python train2.py 
Loss test: 1.3583741188049316
Step: 0, Initial Loss: 1.3583741188049316
Step: 1,         Loss: 1.2449184656143188
Epoch 000: Loss: 1.061, Accuracy: 46.667%
Epoch 050: Loss: 0.418, Accuracy: 82.500%
Epoch 100: Loss: 0.307, Accuracy: 94.167%
Epoch 150: Loss: 0.221, Accuracy: 97.500%
Epoch 200: Loss: 0.162, Accuracy: 96.667%
Test set accuracy: 96.667%
root@433ad0b9b2e7:/usr/src/app# 
root@433ad0b9b2e7:/usr/src/app# python train2_test.py 
Example 0 prediction: Iris setosa (98.3%)
Example 1 prediction: Iris versicolor (82.5%)
Example 2 prediction: Iris virginica (75.2%)
root@433ad0b9b2e7:/usr/src/app# 
```

## 运行环境
- Python:3.7.4
- TensorFlow:1.14.0
- Matplotlib
- 数据来源：
    -  训练  http://download.tensorflow.org/data/iris_training.csv
    -  测试  http://download.tensorflow.org/data/iris_test.csv
    -  检测  `自定义`

## 实现步骤
### 创建模型
```python
from __future__ import absolute_import, division, print_function

import os
import matplotlib.pyplot as plt

import tensorflow as tf
import tensorflow.contrib.eager as tfe
os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'
tf.compat.v1.enable_eager_execution()

print("TensorFlow version: {}".format(tf.version.VERSION))
print("Eager execution: {}".format(tf.executing_eagerly()))


'''
用于训练生成鸢尾花iris识别模型
@训练图表：./data/images/iris_*.png
@模型文件：./data/model/iris-model
@使用环境：
    Python: 3.7.4
    TensorFlow version: 1.14.0
    Eager execution: True
@作者：baqianxin@163.com
'''
# 创建model

model = tf.keras.Sequential([
    tf.keras.layers.Dense(10, activation=tf.nn.relu, input_shape=(4,)), 
    tf.keras.layers.Dense(10, activation=tf.nn.relu),
    tf.keras.layers.Dense(3)
])

```
### 加载数据
> 加载的远端CSV数据，使用tf.data.experimental.make_csv_dataset转化为Dataset

```python
train_dataset_url = "http://download.tensorflow.org/data/iris_training.csv"

train_dataset_fp = tf.keras.utils.get_file(fname=os.path.basename(train_dataset_url),
                                           origin=train_dataset_url)

# print("Local copy of the dataset file: {}".format(train_dataset_fp))

# column order in CSV file
column_names = ['sepal_length', 'sepal_width',
                'petal_length', 'petal_width', 'species']

feature_names = column_names[:-1]
label_name = column_names[-1]

# print("Features: {}".format(feature_names))
# print("Label: {}".format(label_name))

# 定义标签
class_names = ['Iris setosa', 'Iris versicolor', 'Iris virginica']

# 创建一个 tf.data.Dataset
batch_size = 32

train_dataset = tf.data.experimental.make_csv_dataset(
    train_dataset_fp,
    batch_size,
    column_names=column_names,
    label_name=label_name,
    num_epochs=1)
```

### 定义损失和梯度函数
```python
# 1.定义损失 和梯度函数

def loss(model, x, y):
    y_ = model(x)
    return tf.compat.v1.losses.sparse_softmax_cross_entropy(labels=y, logits=y_)

l = loss(model, features, labels)
print("Loss test: {}".format(l))

# 2.定义用于优化模型的梯度

def grad(model, inputs, targets):
    with tf.GradientTape() as tape:
        loss_value = loss(model, inputs, targets)
    return loss_value, tape.gradient(loss_value, model.trainable_variables)

```

### 设置优化器，计数器
```python
# 设置优化器，计数器
optimizer = tf.compat.v1.train.GradientDescentOptimizer(learning_rate=0.01)
```

### 自定义训练
```python
train_loss_results = []
train_accuracy_results = []

num_epochs = 201

for epoch in range(num_epochs):
    epoch_loss_avg = tfe.metrics.Mean()
    epoch_accuracy = tfe.metrics.Accuracy()

    # Training loop - using batches of 32
    for x, y in train_dataset:
        # Optimize the model
        loss_value, grads = grad(model, x, y)
        optimizer.apply_gradients(zip(grads, model.variables),
                                  global_step)

        # Track progress
        epoch_loss_avg(loss_value)  # add current batch loss
        # compare predicted label to actual label
        epoch_accuracy(tf.argmax(model(x), axis=1, output_type=tf.int32), y)

    # end epoch
    train_loss_results.append(epoch_loss_avg.result())
    train_accuracy_results.append(epoch_accuracy.result())

    if epoch % 50 == 0:
        print("Epoch {:03d}: Loss: {:.3f}, Accuracy: {:.3%}".format(epoch,
                                                                    epoch_loss_avg.result(),
                                                                    epoch_accuracy.result()))

```

### 记录
> 把训练过程中的损失与正确率都记录到`train_loss_results`,`train_accuracy_results`数组中了，在通过pyplot输出到可视化图表文件

```python
fig, axes = plt.subplots(2, sharex=True, figsize=(12, 8))
fig.suptitle('Training Metrics')

axes[0].set_ylabel("Loss", fontsize=14)
axes[0].plot(train_loss_results)

axes[1].set_ylabel("Accuracy", fontsize=14)
axes[1].set_xlabel("Epoch", fontsize=14)
axes[1].plot(train_accuracy_results)
plt.savefig("./data/images/iris_loss.png")
```

### 获取测试数据并检测

```python
# 获取检测数据
test_url = "http://download.tensorflow.org/data/iris_test.csv"

test_fp = tf.keras.utils.get_file(fname=os.path.basename(test_url),
                                  origin=test_url)

test_dataset = tf.data.experimental.make_csv_dataset(
    test_fp,
    batch_size,
    column_names=column_names,
    label_name='species',
    num_epochs=1,
    shuffle=False)
    
# 其他方式格式化数据
def pack_features_vector(features, labels):
    """Pack the features into a single array."""
    features = tf.stack(list(features.values()), axis=1)
    return features, labels
    
test_dataset = test_dataset.map(pack_features_vector)
test_accuracy = tfe.metrics.Accuracy()

for (x, y) in test_dataset:
  logits = model(x)
  prediction = tf.argmax(logits, axis=1, output_type=tf.int32)
  test_accuracy(prediction, y)

print("Test set accuracy: {:.3%}".format(test_accuracy.result()))
```

### compile和保存model
```python
# compile 用于设定训练过程
model.compile(optimizer=tf.keras.optimizers.Adam(),
                loss=tf.keras.losses.sparse_categorical_crossentropy,
                metrics=['accuracy'])
# 全量保存模型
model.save("./data/model/iris-model")
```

### 使用模型检测未知数据`train_test.py`
```python
from __future__ import absolute_import, division, print_function

import os
import tensorflow as tf
import tensorflow.contrib.eager as tfe
os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'
tf.compat.v1.enable_eager_execution()

# 加载训练好的模型
file_path = './data/model'
model = tf.keras.models.load_model(file_path + "/iris-model")
# model.summary()

# 定义标签
class_names = ['Iris setosa', 'Iris versicolor', 'Iris virginica']

predict_dataset = tf.convert_to_tensor([
    [5.1, 3.3, 1.7, 0.5, ],
    [5.9, 3.0, 4.2, 1.5, ],
    [6.9, 3.1, 5.4, 2.1]
])

predictions = model(predict_dataset)

for i, logits in enumerate(predictions):
    class_idx = tf.argmax(logits).numpy()
    p = tf.nn.softmax(logits)[class_idx]
    name = class_names[class_idx]
    print("Example {} prediction: {} ({:4.1f}%)".format(i, name, 100*p))

```

## 遇到问题&思路
不好意思没能力提出问题，很多参数都看不懂，只是模拟操作一下

- **大致就是生成数据集**：可能比较复杂需要格式化
- **定义标签**：一个常量数组
- **创建模型**：配置层数及一些输入输出参数
- **定义损失与梯度函数**：用来检测是否训练到位了
- **定义优化器**，计数器：同名意
- ***可以单个数据测试下这些参数***
- **自定义训练方法**：记录下训练的次数，每一次训练的结果（）
- **输出训练数据**：可视化很直白表示模型当前数据下是否训练的合理
- ***再使用准备好的测试数据在这个训练过的模型上测试***
- **然后就是保存**：有几种，配置，权重，全量保存
- ++**最后就是拿未知数据来检测保存的模型是否准确了y**++
