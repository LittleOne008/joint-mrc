# 数据集格式

每个数据集为一个独立的目录，具有相同的格式，包括train.json、valid.json、test.json三个文件，数据格式为json，每行一个样本，包含一段文本和对应的一个或多个问题，结构如下：

```
{
    "qas":[
        {
            "question": "张艺谋电影《英雄》中秦始皇是哪位内地男演员扮演的?",
            "answer": "陈道明",
            "answer_start": 20,
            "answer_end": None,
            "q_id": "12345",
        },
        ...
    ],
    "context":"在表演上，张丰毅表示：不能简单地和先前陈道明在《英雄》里扮演的秦始皇相比。"
}
```

其中"answer_end"和"q_id"为选填字段，"answer_end"可以没有或者为None，但"answer_start"为必填字段；对于训练集或者单文档问题"q_id"可以没有，但是对于多文档测试集需要使用"q_id"关联文档段落与排序。如果"answer"字段为None，则表示本段落中无答案，即负样本。