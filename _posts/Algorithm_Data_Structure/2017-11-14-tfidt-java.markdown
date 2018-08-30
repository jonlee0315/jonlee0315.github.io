---
layout:     post
title:      "Java实现TFIDF算法"
subtitle:   ""
date:       2017-11-14
author:     "Jon Lee"
header-img: "img/in-post/2017-11-14-tfidf-java/bg.jpg"
catalog: true
categories : 算法与数据结构
tags:
    - 算法
    - Java
---

### 算法介绍

最近要做领域概念的提取，TFIDF作为一个很经典的算法可以作为其中的一步处理。

关于TFIDF算法的介绍可以参考这篇博客 http://www.ruanyifeng.com/blog/2013/03/tf-idf.html。

计算公式比较简单，如下：

![](/img/in-post/2017-11-14-tfidf-java/1.png)

### 预处理

由于需要处理的候选词大约后3w+，并且语料文档数有1w+，直接挨个文本遍历的话很耗时，每个词处理时间都要一分钟以上。

为了缩短时间，首先进行分词，一个词输出为一行方便统计，分词工具选择的是HanLp。

然后，将一个领域的文档合并到一个文件中，并用“$$$”标识符分割，方便记录文档数。

![](/img/in-post/2017-11-14-tfidf-java/2.png)

下面是选择的领域语料（PATH目录下）：

![](/img/in-post/2017-11-14-tfidf-java/3.png)

### 代码实现

    public class TfIdf {

        static final String PATH = "E:\\corpus"; // 语料库路径

        public static void main(String[] args) throws Exception {

            String test = "离退休人员"; // 要计算的候选词

            computeTFIDF(PATH, test);

        }

        static void computeTFIDF(String path, String word) throws Exception {

            File fileDir = new File(path);
            File[] files = fileDir.listFiles();

            // 每个领域出现候选词的文档数
            Map<String, Integer> containsKeyMap = new HashMap<>();
            // 每个领域的总文档数
            Map<String, Integer> totalDocMap = new HashMap<>();
            // TF = 候选词出现次数/总词数
            Map<String, Double> tfMap = new HashMap<>();

            // scan files
            for (File f : files) {

                // 候选词词频
                double termFrequency = 0;
                // 文本总词数
                double totalTerm = 0;
                // 包含候选词的文档数
                int containsKeyDoc = 0;
                // 词频文档计数
                int totalCount = 0;
                int fileCount = 0;
                // 标记文件中是否出现候选词
                boolean flag = false;

                FileReader fr = new FileReader(f);
                BufferedReader br = new BufferedReader(fr);
                String s = "";

                // 计算词频和总词数
                while ((s = br.readLine()) != null) {
                    if (s.equals(word)) {
                        termFrequency++;
                        flag = true;
                    }

                    // 文件标识符
                    if (s.equals("$$$")) {
                        if (flag) {
                            containsKeyDoc++;
                        }
                        fileCount++;
                        flag = false;
                    }
                    totalCount++;
                }

                // 减去文件标识符的数量得到总词数
                totalTerm += totalCount - fileCount;
                br.close();
                // key都为领域的名字
                containsKeyMap.put(f.getName(), containsKeyDoc);
                totalDocMap.put(f.getName(), fileCount);
                tfMap.put(f.getName(), (double) termFrequency / totalTerm);

                System.out.println("----------" + f.getName() + "----------");
                System.out.println("该领域文档数：" + fileCount);
                System.out.println("候选词出现词数：" + termFrequency);
                System.out.println("总词数：" + totalTerm);
                System.out.println("出现候选词文档总数：" + containsKeyDoc);
                System.out.println();
            }

            //计算TF*IDF
            for (File f : files) {

                // 其他领域包含候选词文档数
                int otherContainsKeyDoc = 0;
                // 其他领域文档总数
                int otherTotalDoc = 0;

                double idf = 0;
                double tfidf = 0;
                System.out.println("~~~~~" + f.getName() + "~~~~~");

                Set<Map.Entry<String, Integer>> containsKeyset = containsKeyMap.entrySet();
                Set<Map.Entry<String, Integer>> totalDocset = totalDocMap.entrySet();
                Set<Map.Entry<String, Double>> tfSet = tfMap.entrySet();

                // 计算其他领域包含候选词文档数
                for (Map.Entry<String, Integer> entry : containsKeyset) {
                    if (!entry.getKey().equals(f.getName())) {
                        otherContainsKeyDoc += entry.getValue();
                    }
                }

                // 计算其他领域文档总数
                for (Map.Entry<String, Integer> entry : totalDocset) {
                    if (!entry.getKey().equals(f.getName())) {
                        otherTotalDoc += entry.getValue();
                    }
                }

                // 计算idf
                idf = log((float) otherTotalDoc / (otherContainsKeyDoc + 1), 2);

                // 计算tf*idf并输出
                for (Map.Entry<String, Double> entry : tfSet) {
                    if (entry.getKey().equals(f.getName())) {
                        tfidf = (double) entry.getValue() * idf;
                        System.out.println("tfidf:" + tfidf);
                    }
                }
            }
        }

        static float log(float value, float base) {
            return (float) (Math.log(value) / Math.log(base));
        }
    }

### 运行结果

测试词为“离退休人员”，中间结果如下：

![](/img/in-post/2017-11-14-tfidf-java/4.png)

最终结果：

![](/img/in-post/2017-11-14-tfidf-java/5.png)

### 结论

可以看到“离退休人员”在养老保险和社保领域，tfidf值比较高，可以作为判断是否为领域概念的一个依据。

当然TF-IDF算法虽然很经典，但还是有许多不足，不能单独依赖其结果做出判断。

很多论文提出了改进方法，本文只是实现了最基本的算法。
