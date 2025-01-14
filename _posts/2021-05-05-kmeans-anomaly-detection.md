---
layout:     post
title:      K-Means based Anomalous Email Detection in PySpark
subtitle:   Anomaly detection for emails based on Minhash and K-Means, implemented by PySpark and Colab.
date:       2021-05-05
author:     Zekun
header-img: img/kmeans-anomaly-detection/bg-email.png
catalog: true
tags:
    - Anomaly Detection
    - K-Means
    - Project
    - Python
    - PySpark
---


K-Means is known as a common unsupervised learning clustering method. But in fact, K-Means algorithm can be applied to more scenarios. This time, I will use a K-Means-based approach to complete anomaly detection for text-based email content.

All the data manipulation and modeling processes involved in this approach will be fully implemented based on [PySpark-3.1.1](https://spark.apache.org/docs/latest/api/python/index.html).  Considering the demand when facing large amount of text data and the high time complexity of K-Means class algorithm, the Spark-based practice can effectively improve the processing power and running speed. To facilitate the reproduction and reduce all kinds of problems caused by the construction of Spark environment, all the contents can be done in the notebook in [Google Colab](https://colab.research.google.com/notebooks/intro.ipynb).

The data I used is [Insider Threat Test Dataset](https://resources.sei.cmu.edu/library/asset-view.cfm?assetid=508099) from Carnegie Mellon University, which the CERT Division provides, collects synthetic insider threat test datasets that provide both background and malicious actor synthetic data. It contains 1000 users, 17 months long. Please download the dataset from [CMU kilthub](https://kilthub.cmu.edu/articles/dataset/Insider_Threat_Test_Dataset/12841247/1) and unzip them. Then put the CSV files into the folder `./data/` under the same folder of the notebook.

For more background on this data, please see the paper, Bridging the Gap: [A Pragmatic Approach to Generating Insider Threat Data](https://ieeexplore.ieee.org/document/6565236).



## Method overview

1. Split text to the word list
2. Remove stop words
3. Generate word count vector
4. Reduce dimension by MinHash
5. Find the appropriate centroid for each obs by K-Means
6. Calculate the distance
7. Sort the distance

If you want to accelerate the manipulation process, you can skip all of the `dataframe.show()` part.

## Build environment

Since Colab does not have PySpark module installed, we need to install PySpark and configure the related environment first.


```python
!pip install pyspark 
!pip install -U -q PyDrive
!apt update
!apt install openjdk-8-jdk-headless -qq
import os
os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-8-openjdk-amd64"
```



```python
from google.colab import drive
drive.mount('/content/drive')
```


Please locate to the location where this notebook is saved.


```python
import os
cur_path = "/content/drive/MyDrive/Insider-Risk-in-PySpark/"
os.chdir(cur_path)
!pwd
```


Start Spark session and then we can check the configuration details.


```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName('proj').getOrCreate()
```


```python
spark.sparkContext.getConf().getAll()
```




    [('spark.sql.warehouse.dir',
      'file:/content/drive/MyDrive/insider-risk-in-spark/spark-warehouse'),
     ('spark.driver.port', '36571'),
     ('spark.rdd.compress', 'True'),
     ('spark.driver.host', '5a29464ac89d'),
     ('spark.app.id', 'local-1619919014564'),
     ('spark.serializer.objectStreamReset', '100'),
     ('spark.master', 'local[*]'),
     ('spark.submit.pyFiles', ''),
     ('spark.app.startTime', '1619919013556'),
     ('spark.executor.id', 'driver'),
     ('spark.submit.deployMode', 'client'),
     ('spark.ui.showConsoleProgress', 'true'),
     ('spark.app.name', 'proj')]



Import necessary modules. 


```python
import matplotlib.pyplot as plt
import numpy as np

from pyspark.ml.feature import Tokenizer, StopWordsRemover, CountVectorizer, StandardScaler, MinHashLSH, VectorAssembler
from pyspark.sql.functions import udf, col
from pyspark.sql.types import *
from pyspark.ml.functions import vector_to_array
from pyspark.ml.linalg import Vectors

from pyspark.ml.clustering import KMeans
from pyspark.ml.evaluation import ClusteringEvaluator
from pyspark.mllib.stat import KernelDensity
```



## Load data

The **email.csv** file size is about 1GB and contains 2.6 million emails. Since we are only working on the email content this time, we can just read the contents of that file.


```python
email = spark.read.csv( './data/email.csv',inferSchema=True,header=True)
```


```python
email.printSchema()
email.show(5)
```

    root
     |-- id: string (nullable = true)
     |-- date: string (nullable = true)
     |-- user: string (nullable = true)
     |-- pc: string (nullable = true)
     |-- to: string (nullable = true)
     |-- cc: string (nullable = true)
     |-- bcc: string (nullable = true)
     |-- from: string (nullable = true)
     |-- size: integer (nullable = true)
     |-- attachments: integer (nullable = true)
     |-- content: string (nullable = true)
    
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+
    |                  id|               date|   user|     pc|                  to|                  cc|                 bcc|                from| size|attachments|             content|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+
    |{R3I7-S4TX96FG-82...|01/02/2010 07:11:45|LAP0338|PC-5758|Dean.Flynn.Hines@...|Nathaniel.Hunter....|                null|Lynn.Adena.Pratt@...|25830|          0|middle f2 systems...|
    |{R0R9-E4GL59IK-29...|01/02/2010 07:12:16|MOH0273|PC-6699|Odonnell-Gage@bel...|                null|                null| MOH68@optonline.net|29942|          0|the breaking call...|
    |{G2B2-A8XY58CP-28...|01/02/2010 07:13:00|LAP0338|PC-5758|Penelope_Colon@ne...|                null|                null|Lynn_A_Pratt@eart...|28780|          0|slowly this uncin...|
    |{A3A9-F4TH89AA-83...|01/02/2010 07:13:17|LAP0338|PC-5758|Judith_Hayden@com...|                null|                null|Lynn_A_Pratt@eart...|21907|          0|400 other difficu...|
    |{E8B7-C8FZ88UF-29...|01/02/2010 07:13:28|MOH0273|PC-6699|Bond-Raymond@veri...|                null|Odonnell-Gage@bel...| MOH68@optonline.net|17319|          0|this kmh october ...|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+
    only showing top 5 rows
    



## Extract word-vector

First, we need to split the email content into lists according to words, and then remove common meaningless words, aka "Stop words". The list of stop words is provided by PySpark. Sometimes, we can also use [`n-grams`](https://web.stanford.edu/~jurafsky/slp3/3.pdf) to get a more representative list of phrases.


```python
tokenizer = Tokenizer(inputCol="content", outputCol="words")
wordsData = tokenizer.transform(email)

remover = StopWordsRemover(inputCol="words", outputCol="clean_words")
wordsData = remover.transform(wordsData)
```


```python
wordsData.show()
```

    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+
    |                  id|               date|   user|     pc|                  to|                  cc|                 bcc|                from| size|attachments|             content|               words|         clean_words|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+
    |{R3I7-S4TX96FG-82...|01/02/2010 07:11:45|LAP0338|PC-5758|Dean.Flynn.Hines@...|Nathaniel.Hunter....|                null|Lynn.Adena.Pratt@...|25830|          0|middle f2 systems...|[middle, f2, syst...|[middle, f2, syst...|
    |{R0R9-E4GL59IK-29...|01/02/2010 07:12:16|MOH0273|PC-6699|Odonnell-Gage@bel...|                null|                null| MOH68@optonline.net|29942|          0|the breaking call...|[the, breaking, c...|[breaking, called...|
    |{G2B2-A8XY58CP-28...|01/02/2010 07:13:00|LAP0338|PC-5758|Penelope_Colon@ne...|                null|                null|Lynn_A_Pratt@eart...|28780|          0|slowly this uncin...|[slowly, this, un...|[slowly, uncinus,...|
    |{A3A9-F4TH89AA-83...|01/02/2010 07:13:17|LAP0338|PC-5758|Judith_Hayden@com...|                null|                null|Lynn_A_Pratt@eart...|21907|          0|400 other difficu...|[400, other, diff...|[400, difficult, ...|
    |{E8B7-C8FZ88UF-29...|01/02/2010 07:13:28|MOH0273|PC-6699|Bond-Raymond@veri...|                null|Odonnell-Gage@bel...| MOH68@optonline.net|17319|          0|this kmh october ...|[this, kmh, octob...|[kmh, october, ho...|
    |{X8T7-A6BT54FP-72...|01/02/2010 07:36:03|HVB0037|PC-7979|Gaines-Joseph@msn...|Hollee_Becker@hot...|                null|Hollee_Becker@hot...|44345|          0|little equal k is...|[little, equal, k...|[little, equal, k...|
    |{H5J6-G2RS59KI-83...|01/02/2010 07:52:20|NWK0215|PC-8370|Heidi_Wilson@msn....|Noelani.W.Kennedy...|                null|Noelani.W.Kennedy...|35328|          0|stroke menacing 1...|[stroke, menacing...|[stroke, menacing...|
    |{D9T8-M1HJ89XP-63...|01/02/2010 07:54:12|LRR0148|PC-4275|Eve.Isadora.Mcken...|                null|                null|Libby.Rosalyn.Ric...|25255|          1|leading companys ...|[leading, company...|[leading, company...|
    |{V3L7-L2RB92RV-91...|01/02/2010 07:54:49|LRR0148|PC-4275|Cedric.Herrod.Gil...|                null|                null|Libby.Rosalyn.Ric...|33967|          0|reception website...|[reception, websi...|[reception, websi...|
    |{D5K9-P0IJ71WK-63...|01/02/2010 07:54:58|LRR0148|PC-4275|Gay.Ria.Cantu@dta...|Zenia.Freya.Macia...|                null|Libby.Rosalyn.Ric...|19456|          1|smaller weather r...|[smaller, weather...|[smaller, weather...|
    |{R0A5-U4YQ17EA-34...|01/02/2010 07:55:16|LRR0148|PC-4275|August_Holt@boein...|                null|                null|Libby.Rosalyn.Ric...|23687|          0|do potentially 2 ...|[do, potentially,...|[potentially, 2, ...|
    |{Y8Z6-X5HU72BM-73...|01/02/2010 07:55:51|NWK0215|PC-8370|Tasha_Sanchez@opt...|Ulric-Knapp@earth...|Noelani.W.Kennedy...|Noelani.W.Kennedy...|28960|          0|villepin five sha...|[villepin, five, ...|[villepin, five, ...|
    |{K3B8-S0RJ27BU-68...|01/02/2010 07:56:09|AJR0319|PC-4736|Ulric.Ferdinand.K...|Meghan.Brianna.Je...|Arthur.Jacob.Raym...|Arthur.Jacob.Raym...|23116|          1|he instill prehis...|[he, instill, pre...|[instill, prehist...|
    |{J7Y1-G7KD78BQ-41...|01/02/2010 07:56:49|LRR0148|PC-4275|Gay.Ria.Cantu@dta...|Sasha.Rina.Huffma...|                null|Libby.Rosalyn.Ric...|53349|          1|went could jackso...|[went, could, jac...|[went, jackson, a...|
    |{D7P4-Z0PP26KM-17...|01/02/2010 07:57:10|AJR0319|PC-4736|Meredith.Ainsley....|Arthur.Jacob.Raym...|                null|Arthur.Jacob.Raym...|26284|          0|connections progr...|[connections, pro...|[connections, pro...|
    |{P6J4-Y0XJ63II-57...|01/02/2010 07:58:04|AJR0319|PC-4736|Connor.Phelan.Gue...|Arthur.Jacob.Raym...|                null|Arthur.Jacob.Raym...|25317|          0|rested considered...|[rested, consider...|[rested, consider...|
    |{K7Y5-V5IP47OA-83...|01/02/2010 07:58:07|LRR0148|PC-4275|Bevis.Brady.Shepp...|Gay.Ria.Cantu@dta...|                null|Libby.Rosalyn.Ric...|16168|          2|additional funera...|[additional, fune...|[additional, fune...|
    |{R9V2-W5OA43XS-14...|01/02/2010 07:58:13|LRR0148|PC-4275|Thomas.Vladimir.S...|                null|                null|Libby.Rosalyn.Ric...|52290|          0|need there did co...|[need, there, did...|[need, comes, nam...|
    |{X4R4-F1BP75UA-02...|01/02/2010 07:58:15|LRR0148|PC-4275|Sasha.Rina.Huffma...|                null|                null|Libby.Rosalyn.Ric...|18333|          0|since data englis...|[since, data, eng...|[since, data, eng...|
    |{N4L7-S2MN81EJ-50...|01/02/2010 07:58:25|LRR0148|PC-4275|Nissim.Gil.French...|                null|                null|Libby.Rosalyn.Ric...|35956|          3|orphan treatment ...|[orphan, treatmen...|[orphan, treatmen...|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+
    only showing top 20 rows
    


The function `CountVectorizer` can convert a collection of text documents to vectors of token counts. It can produce sparse representations for the documents over the vocabulary.

We choose 1000 as the vocabulary dimension under consideration. Of course, if the device allows, we can choose a larger dimension to obtain stronger representation ability.


```python
cv = CountVectorizer(inputCol="clean_words", outputCol="features", vocabSize=1000, minDF=2.0)

model = cv.fit(wordsData)

wordsCV = model.transform(wordsData)
```

Since the MinHash algorithm used in the later steps cannot handle the all-0 vector, we need to remove it in this step.
Of course, if the content of an email generates an all-0 vector as a result, it means that the content of that email is also anomalous. Therefore, the emails removed in this step also need to be treated as anomalous emails.


```python
all0vector = Vectors.dense([0]*1000) 

# Filter the empty Sparse Vector
def no_empty_vector(value):
    if value != all0vector:
        return True
    else:
        return False

no_empty_vector_udf = udf(no_empty_vector, BooleanType())
wordsCV = wordsCV.filter(no_empty_vector_udf('features'))
```


```python
wordsCV.show()
```

    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+--------------------+
    |                  id|               date|   user|     pc|                  to|                  cc|                 bcc|                from| size|attachments|             content|               words|         clean_words|            features|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+--------------------+
    |{R3I7-S4TX96FG-82...|01/02/2010 07:11:45|LAP0338|PC-5758|Dean.Flynn.Hines@...|Nathaniel.Hunter....|                null|Lynn.Adena.Pratt@...|25830|          0|middle f2 systems...|[middle, f2, syst...|[middle, f2, syst...|(1000,[28,59,105,...|
    |{R0R9-E4GL59IK-29...|01/02/2010 07:12:16|MOH0273|PC-6699|Odonnell-Gage@bel...|                null|                null| MOH68@optonline.net|29942|          0|the breaking call...|[the, breaking, c...|[breaking, called...|(1000,[5,10,22,29...|
    |{G2B2-A8XY58CP-28...|01/02/2010 07:13:00|LAP0338|PC-5758|Penelope_Colon@ne...|                null|                null|Lynn_A_Pratt@eart...|28780|          0|slowly this uncin...|[slowly, this, un...|[slowly, uncinus,...|(1000,[5,16,63,14...|
    |{A3A9-F4TH89AA-83...|01/02/2010 07:13:17|LAP0338|PC-5758|Judith_Hayden@com...|                null|                null|Lynn_A_Pratt@eart...|21907|          0|400 other difficu...|[400, other, diff...|[400, difficult, ...|(1000,[5,36,38,80...|
    |{E8B7-C8FZ88UF-29...|01/02/2010 07:13:28|MOH0273|PC-6699|Bond-Raymond@veri...|                null|Odonnell-Gage@bel...| MOH68@optonline.net|17319|          0|this kmh october ...|[this, kmh, octob...|[kmh, october, ho...|(1000,[9,14,56,72...|
    |{X8T7-A6BT54FP-72...|01/02/2010 07:36:03|HVB0037|PC-7979|Gaines-Joseph@msn...|Hollee_Becker@hot...|                null|Hollee_Becker@hot...|44345|          0|little equal k is...|[little, equal, k...|[little, equal, k...|(1000,[28,40,54,5...|
    |{H5J6-G2RS59KI-83...|01/02/2010 07:52:20|NWK0215|PC-8370|Heidi_Wilson@msn....|Noelani.W.Kennedy...|                null|Noelani.W.Kennedy...|35328|          0|stroke menacing 1...|[stroke, menacing...|[stroke, menacing...|(1000,[12,38,71,7...|
    |{D9T8-M1HJ89XP-63...|01/02/2010 07:54:12|LRR0148|PC-4275|Eve.Isadora.Mcken...|                null|                null|Libby.Rosalyn.Ric...|25255|          1|leading companys ...|[leading, company...|[leading, company...|(1000,[9,10,24,11...|
    |{V3L7-L2RB92RV-91...|01/02/2010 07:54:49|LRR0148|PC-4275|Cedric.Herrod.Gil...|                null|                null|Libby.Rosalyn.Ric...|33967|          0|reception website...|[reception, websi...|[reception, websi...|(1000,[28,40,41,4...|
    |{D5K9-P0IJ71WK-63...|01/02/2010 07:54:58|LRR0148|PC-4275|Gay.Ria.Cantu@dta...|Zenia.Freya.Macia...|                null|Libby.Rosalyn.Ric...|19456|          1|smaller weather r...|[smaller, weather...|[smaller, weather...|(1000,[16,25,38,9...|
    |{R0A5-U4YQ17EA-34...|01/02/2010 07:55:16|LRR0148|PC-4275|August_Holt@boein...|                null|                null|Libby.Rosalyn.Ric...|23687|          0|do potentially 2 ...|[do, potentially,...|[potentially, 2, ...|(1000,[7,29,122,1...|
    |{Y8Z6-X5HU72BM-73...|01/02/2010 07:55:51|NWK0215|PC-8370|Tasha_Sanchez@opt...|Ulric-Knapp@earth...|Noelani.W.Kennedy...|Noelani.W.Kennedy...|28960|          0|villepin five sha...|[villepin, five, ...|[villepin, five, ...|(1000,[10,29,78,1...|
    |{K3B8-S0RJ27BU-68...|01/02/2010 07:56:09|AJR0319|PC-4736|Ulric.Ferdinand.K...|Meghan.Brianna.Je...|Arthur.Jacob.Raym...|Arthur.Jacob.Raym...|23116|          1|he instill prehis...|[he, instill, pre...|[instill, prehist...|(1000,[3,116,126,...|
    |{J7Y1-G7KD78BQ-41...|01/02/2010 07:56:49|LRR0148|PC-4275|Gay.Ria.Cantu@dta...|Sasha.Rina.Huffma...|                null|Libby.Rosalyn.Ric...|53349|          1|went could jackso...|[went, could, jac...|[went, jackson, a...|(1000,[33,43,45,4...|
    |{D7P4-Z0PP26KM-17...|01/02/2010 07:57:10|AJR0319|PC-4736|Meredith.Ainsley....|Arthur.Jacob.Raym...|                null|Arthur.Jacob.Raym...|26284|          0|connections progr...|[connections, pro...|[connections, pro...|(1000,[35,52,128,...|
    |{P6J4-Y0XJ63II-57...|01/02/2010 07:58:04|AJR0319|PC-4736|Connor.Phelan.Gue...|Arthur.Jacob.Raym...|                null|Arthur.Jacob.Raym...|25317|          0|rested considered...|[rested, consider...|[rested, consider...|(1000,[4,10,38,10...|
    |{K7Y5-V5IP47OA-83...|01/02/2010 07:58:07|LRR0148|PC-4275|Bevis.Brady.Shepp...|Gay.Ria.Cantu@dta...|                null|Libby.Rosalyn.Ric...|16168|          2|additional funera...|[additional, fune...|[additional, fune...|(1000,[23,65,74,1...|
    |{R9V2-W5OA43XS-14...|01/02/2010 07:58:13|LRR0148|PC-4275|Thomas.Vladimir.S...|                null|                null|Libby.Rosalyn.Ric...|52290|          0|need there did co...|[need, there, did...|[need, comes, nam...|(1000,[9,65,80,11...|
    |{X4R4-F1BP75UA-02...|01/02/2010 07:58:15|LRR0148|PC-4275|Sasha.Rina.Huffma...|                null|                null|Libby.Rosalyn.Ric...|18333|          0|since data englis...|[since, data, eng...|[since, data, eng...|(1000,[4,23,24,33...|
    |{N4L7-S2MN81EJ-50...|01/02/2010 07:58:25|LRR0148|PC-4275|Nissim.Gil.French...|                null|                null|Libby.Rosalyn.Ric...|35956|          3|orphan treatment ...|[orphan, treatmen...|[orphan, treatmen...|(1000,[47,77,96,1...|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+--------------------+
    only showing top 20 rows
    


## Dimension reduction by MinHash

In this step, we reduce the dimensionality of the features used by using the MinHash algorithm, while ensuring that the similarity between the email is maintained. Also, it converts the sparse features into dense features.

For more details about the MinHash algorithm, there is an good [explanation](https://www.cs.utah.edu/~jeffp/teaching/cs5955/L5-Minhash.pdf) from Utah University.

```python
mh = MinHashLSH(inputCol="features", outputCol="hashes", numHashTables=20)
model = mh.fit(wordsCV)
wordsHash = model.transform(wordsCV)
```


```python
wordsHash.show()
```

    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+--------------------+--------------------+
    |                  id|               date|   user|     pc|                  to|                  cc|                 bcc|                from| size|attachments|             content|               words|         clean_words|            features|              hashes|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+--------------------+--------------------+
    |{R3I7-S4TX96FG-82...|01/02/2010 07:11:45|LAP0338|PC-5758|Dean.Flynn.Hines@...|Nathaniel.Hunter....|                null|Lynn.Adena.Pratt@...|25830|          0|middle f2 systems...|[middle, f2, syst...|[middle, f2, syst...|(1000,[28,59,105,...|[[8.776332E7], [1...|
    |{R0R9-E4GL59IK-29...|01/02/2010 07:12:16|MOH0273|PC-6699|Odonnell-Gage@bel...|                null|                null| MOH68@optonline.net|29942|          0|the breaking call...|[the, breaking, c...|[breaking, called...|(1000,[5,10,22,29...|[[195285.0], [1.2...|
    |{G2B2-A8XY58CP-28...|01/02/2010 07:13:00|LAP0338|PC-5758|Penelope_Colon@ne...|                null|                null|Lynn_A_Pratt@eart...|28780|          0|slowly this uncin...|[slowly, this, un...|[slowly, uncinus,...|(1000,[5,16,63,14...|[[2.90624351E8], ...|
    |{A3A9-F4TH89AA-83...|01/02/2010 07:13:17|LAP0338|PC-5758|Judith_Hayden@com...|                null|                null|Lynn_A_Pratt@eart...|21907|          0|400 other difficu...|[400, other, diff...|[400, difficult, ...|(1000,[5,36,38,80...|[[8.2692039E7], [...|
    |{E8B7-C8FZ88UF-29...|01/02/2010 07:13:28|MOH0273|PC-6699|Bond-Raymond@veri...|                null|Odonnell-Gage@bel...| MOH68@optonline.net|17319|          0|this kmh october ...|[this, kmh, octob...|[kmh, october, ho...|(1000,[9,14,56,72...|[[2.62393241E8], ...|
    |{X8T7-A6BT54FP-72...|01/02/2010 07:36:03|HVB0037|PC-7979|Gaines-Joseph@msn...|Hollee_Becker@hot...|                null|Hollee_Becker@hot...|44345|          0|little equal k is...|[little, equal, k...|[little, equal, k...|(1000,[28,40,54,5...|[[3.66359397E8], ...|
    |{H5J6-G2RS59KI-83...|01/02/2010 07:52:20|NWK0215|PC-8370|Heidi_Wilson@msn....|Noelani.W.Kennedy...|                null|Noelani.W.Kennedy...|35328|          0|stroke menacing 1...|[stroke, menacing...|[stroke, menacing...|(1000,[12,38,71,7...|[[1.3212552E7], [...|
    |{D9T8-M1HJ89XP-63...|01/02/2010 07:54:12|LRR0148|PC-4275|Eve.Isadora.Mcken...|                null|                null|Libby.Rosalyn.Ric...|25255|          1|leading companys ...|[leading, company...|[leading, company...|(1000,[9,10,24,11...|[[1.88348622E8], ...|
    |{V3L7-L2RB92RV-91...|01/02/2010 07:54:49|LRR0148|PC-4275|Cedric.Herrod.Gil...|                null|                null|Libby.Rosalyn.Ric...|33967|          0|reception website...|[reception, websi...|[reception, websi...|(1000,[28,40,41,4...|[[4.6514943E7], [...|
    |{D5K9-P0IJ71WK-63...|01/02/2010 07:54:58|LRR0148|PC-4275|Gay.Ria.Cantu@dta...|Zenia.Freya.Macia...|                null|Libby.Rosalyn.Ric...|19456|          1|smaller weather r...|[smaller, weather...|[smaller, weather...|(1000,[16,25,38,9...|[[5266566.0], [2....|
    |{R0A5-U4YQ17EA-34...|01/02/2010 07:55:16|LRR0148|PC-4275|August_Holt@boein...|                null|                null|Libby.Rosalyn.Ric...|23687|          0|do potentially 2 ...|[do, potentially,...|[potentially, 2, ...|(1000,[7,29,122,1...|[[1.05851868E8], ...|
    |{Y8Z6-X5HU72BM-73...|01/02/2010 07:55:51|NWK0215|PC-8370|Tasha_Sanchez@opt...|Ulric-Knapp@earth...|Noelani.W.Kennedy...|Noelani.W.Kennedy...|28960|          0|villepin five sha...|[villepin, five, ...|[villepin, five, ...|(1000,[10,29,78,1...|[[1.3212552E7], [...|
    |{K3B8-S0RJ27BU-68...|01/02/2010 07:56:09|AJR0319|PC-4736|Ulric.Ferdinand.K...|Meghan.Brianna.Je...|Arthur.Jacob.Raym...|Arthur.Jacob.Raym...|23116|          1|he instill prehis...|[he, instill, pre...|[instill, prehist...|(1000,[3,116,126,...|[[1.3864811E8], [...|
    |{J7Y1-G7KD78BQ-41...|01/02/2010 07:56:49|LRR0148|PC-4275|Gay.Ria.Cantu@dta...|Sasha.Rina.Huffma...|                null|Libby.Rosalyn.Ric...|53349|          1|went could jackso...|[went, could, jac...|[went, jackson, a...|(1000,[33,43,45,4...|[[4.6514943E7], [...|
    |{D7P4-Z0PP26KM-17...|01/02/2010 07:57:10|AJR0319|PC-4736|Meredith.Ainsley....|Arthur.Jacob.Raym...|                null|Arthur.Jacob.Raym...|26284|          0|connections progr...|[connections, pro...|[connections, pro...|(1000,[35,52,128,...|[[2.8426395E7], [...|
    |{P6J4-Y0XJ63II-57...|01/02/2010 07:58:04|AJR0319|PC-4736|Connor.Phelan.Gue...|Arthur.Jacob.Raym...|                null|Arthur.Jacob.Raym...|25317|          0|rested considered...|[rested, consider...|[rested, consider...|(1000,[4,10,38,10...|[[9.5709306E7], [...|
    |{K7Y5-V5IP47OA-83...|01/02/2010 07:58:07|LRR0148|PC-4275|Bevis.Brady.Shepp...|Gay.Ria.Cantu@dta...|                null|Libby.Rosalyn.Ric...|16168|          2|additional funera...|[additional, fune...|[additional, fune...|(1000,[23,65,74,1...|[[1.30702124E8], ...|
    |{R9V2-W5OA43XS-14...|01/02/2010 07:58:13|LRR0148|PC-4275|Thomas.Vladimir.S...|                null|                null|Libby.Rosalyn.Ric...|52290|          0|need there did co...|[need, there, did...|[need, comes, nam...|(1000,[9,65,80,11...|[[5.953221E7], [3...|
    |{X4R4-F1BP75UA-02...|01/02/2010 07:58:15|LRR0148|PC-4275|Sasha.Rina.Huffma...|                null|                null|Libby.Rosalyn.Ric...|18333|          0|since data englis...|[since, data, eng...|[since, data, eng...|(1000,[4,23,24,33...|[[3.15474607E8], ...|
    |{N4L7-S2MN81EJ-50...|01/02/2010 07:58:25|LRR0148|PC-4275|Nissim.Gil.French...|                null|                null|Libby.Rosalyn.Ric...|35956|          3|orphan treatment ...|[orphan, treatmen...|[orphan, treatmen...|(1000,[47,77,96,1...|[[1.10923149E8], ...|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+--------------------+--------------------+-----+-----------+--------------------+--------------------+--------------------+--------------------+--------------------+
    only showing top 20 rows
    



```python
id_hash = wordsHash.select('id', 'hashes')
```

Since the features generated by the `MinHashLSH` function are a 20-dimensional list which is composed of 20 DenseVectors, we need to convert it to a flat 20-dimensional DenseVector.

$
[DenseVector, DenseVector, ..., DenseVector] \rightarrow DenseVector[...]
$

Therefore, we first split the list into 20 columns, then convert the DenseVector in each column to a pure value, and finally merge the 20 columns.


```python
sc = spark.sparkContext

numAttrs = 20
attrs = sc.parallelize(["hash_" + str(i) for i in range(numAttrs)]).zipWithIndex().collect()
for name, index in attrs:
    id_hash = id_hash.withColumn(name, id_hash['hashes'].getItem(index))
```


```python
id_hash.show()
```

    +--------------------+--------------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+
    |                  id|              hashes|        hash_0|        hash_1|        hash_2|        hash_3|        hash_4|        hash_5|        hash_6|        hash_7|        hash_8|        hash_9|       hash_10|       hash_11|       hash_12|       hash_13|       hash_14|       hash_15|       hash_16|       hash_17|       hash_18|       hash_19|
    +--------------------+--------------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+
    |{R3I7-S4TX96FG-82...|[[8.776332E7], [1...|  [8.776332E7]| [1.7940101E7]| [6.4377752E7]| [4.5060227E7]| [9.4146089E7]| [5.4730737E7]|   [2045183.0]| [3.5318959E7]| [9.7305009E7]|[1.60568104E8]|[1.26579267E8]| [4.1193117E7]| [5.1198766E7]| [4.7158998E7]|   [7507190.0]| [8.2002584E7]| [5.0971487E7]| [1.9229588E7]| [2.8138237E7]| [5.5599848E7]|
    |{R0R9-E4GL59IK-29...|[[195285.0], [1.2...|    [195285.0]|[1.26315806E8]|[2.26939755E8]| [4.3066557E7]| [2.9401395E7]|[1.49026164E8]|[4.65667429E8]|[1.36802781E8]|  [6.212708E7]|  [7.625038E7]|[1.48161613E8]| [5.1362335E7]|  [7.259244E7]|[2.81864322E8]| [3.2096901E8]|[1.63367351E8]| [2.3015655E7]| [7.5967366E7]|[1.23447019E8]|  [2.548758E7]|
    |{G2B2-A8XY58CP-28...|[[2.90624351E8], ...|[2.90624351E8]|[1.04640665E8]| [1.2043641E8]| [4.8549236E7]|[1.15042474E8]|[1.14372217E8]| [1.7074233E8]|  [3.854587E8]|[5.72021968E8]|  [4.425368E7]|[3.20208273E8]| [4.1193117E7]| [8.5952566E7]| [2.4460579E7]|[1.45858361E8]|[5.25808011E8]|[1.32294306E8]| [2.6179662E7]|[1.05112325E8]|[2.40326464E8]|
    |{A3A9-F4TH89AA-83...|[[8.2692039E7], [...| [8.2692039E7]|[3.39218494E8]|[2.10928785E8]|[1.98576277E8]| [3.1456934E7]| [3.9409618E7]|[2.73620356E8]| [2.1076498E7]|[1.17151187E8]|[1.40896553E8]| [2.9764764E7]| [9.0192079E7]|   [4627505.0]|[2.00334998E8]|   [9798842.0]|     [50832.0]|[1.48256912E8]|[1.71309613E8]|[1.52766716E8]| [2.8672437E7]|
    |{E8B7-C8FZ88UF-29...|[[2.62393241E8], ...|[2.62393241E8]| [3.9615242E7]|[1.05215857E8]| [1.1665476E7]|[1.56835244E8]|[1.24038631E8]|[2.29117145E8]|[2.73739681E8]|[2.59651373E8]|[1.80239655E8]|[1.57012455E8]| [1.2203462E7]|[3.78507397E8]|[2.73371579E8]|  [7.467758E7]| [1.7048611E7]|   [8562838.0]|[1.07621761E8]|[1.21234205E8]|[4.88462473E8]|
    |{X8T7-A6BT54FP-72...|[[3.66359397E8], ...|[3.66359397E8]| [1.7940101E7]|   [8319094.0]| [4.1571218E7]|[1.47414821E8]| [3.3919616E8]| [6.5667663E7]| [8.2053606E7]|[1.27968511E8]|[1.03921106E8]|  [5.134711E7]| [1.7288071E7]| [6.0774979E7]| [1.7357741E7]|  [9.706771E7]| [8.3227663E7]| [5.0971487E7]| [9.3342551E7]| [7.2868565E7]|[1.97474768E8]|
    |{H5J6-G2RS59KI-83...|[[1.3212552E7], [...| [1.3212552E7]|  [5.501013E7]|[1.05215857E8]| [1.4923182E8]| [8.0614588E7]| [6.4397151E7]| [2.5395109E7]| [5.3568684E7]| [8.1973258E7]| [3.1928531E7]| [6.4623373E7]|   [5929983.0]|[1.49900727E8]|[1.98945093E8]|[4.34638399E8]| [2.3122897E7]| [1.7536486E7]| [4.7408946E7]| [4.3548868E7]|  [1.680116E7]|
    |{D9T8-M1HJ89XP-63...|[[1.88348622E8], ...|[1.88348622E8]|[1.45559416E8]|   [5116900.0]| [2.9110521E7]|[2.62515436E8]| [7.0841427E7]| [4.5531386E7]| [5.3568684E7]| [3.5978005E7]|[1.08899853E8]|[2.44976116E8]| [6.3909293E7]| [1.0419805E7]|[2.07437836E8]| [1.0371755E7]|[1.63367351E8]|   [8562838.0]|[1.99488971E8]|[1.68888596E8]| [3.0989143E7]|
    |{V3L7-L2RB92RV-91...|[[4.6514943E7], [...| [4.6514943E7]| [1.7940101E7]| [4.8366782E7]| [9.6398354E7]| [4.4988435E7]|   [4755671.0]| [4.1300729E7]| [4.8080974E7]| [9.8809818E7]| [4.0580479E7]| [6.0197952E7]| [3.6108508E7]| [6.0774979E7]| [6.9857417E7]|  [2.989732E7]|  [8.807687E7]| [7.5907741E7]|   [1475341.0]| [1.8722329E8]| [3.5042151E7]|
    |{D5K9-P0IJ71WK-63...|[[5266566.0], [2....|   [5266566.0]| [2.1788823E7]|  [7.078214E7]|  [3.259953E7]| [3.1456934E7]|  [5.229817E7]| [1.3720146E7]| [6.7811145E7]|[2.48834049E8]| [8.0576354E7]| [3.4734944E7]| [3.6108508E7]|   [4627505.0]| [9.2555836E7]| [3.1043146E7]|[1.43332429E8]|[1.97654519E8]| [2.6179662E7]|[2.40014558E8]| [8.0210553E7]|
    |{R0A5-U4YQ17EA-34...|[[1.05851868E8], ...|[1.05851868E8]| [2.1788823E7]| [2.0337453E7]|[1.95087268E8]|   [1140126.0]|  [3.618748E7]| [4.1300729E7]|[1.42290491E8]|[1.11131951E8]|[2.16562329E8]| [6.9593553E7]| [9.5276688E7]|[1.69286014E8]| [2.4460579E7]| [3.0470233E7]|[4.14709908E8]|[2.03133688E8]| [6.2067218E7]|   [4666666.0]|[1.15824384E8]|
    |{Y8Z6-X5HU72BM-73...|[[1.3212552E7], [...| [1.3212552E7]|[2.48669208E8]|[1.89303844E8]|[2.79820146E8]| [8.6781205E7]| [2.0866361E7]|[3.17106559E8]| [2.1076498E7]| [5.2814565E7]|[1.08899853E8]|[2.49401537E8]|   [5929983.0]|  [8.417704E7]|[1.91842255E8]|[1.45858361E8]|     [50832.0]|[3.39333283E8]|[3.65469144E8]|[4.31343371E8]| [2.8672437E7]|
    |{K3B8-S0RJ27BU-68...|[[1.3864811E8], [...| [1.3864811E8]|[1.16186831E8]| [9.2407081E7]|[3.61064015E8]|[3.36680553E8]|[1.08717512E8]|[1.43161747E8]|[1.13805569E8]| [8.3478067E7]|[1.00900678E8]|[3.72768566E8]| [5.8824684E7]| [6.4791753E7]|[1.27298417E8]| [3.2761885E7]| [4.4263118E8]|[1.21810869E8]|  [6.863823E7]|[4.04236488E8]|   [4061732.0]|
    |{J7Y1-G7KD78BQ-41...|[[4.6514943E7], [...| [4.6514943E7]| [6.5139105E7]| [8.9204887E7]| [3.6088539E7]| [3.3512473E7]|[1.52248302E8]| [1.5483671E8]|   [1346327.0]| [9.8809818E7]| [3.9927706E7]|[3.72768566E8]| [7.2560512E7]| [7.4367966E7]|  [5.920316E7]| [3.2188972E7]|[1.39070207E8]| [4.7477008E7]|   [1475341.0]| [1.8722329E8]| [3.0989143E7]|
    |{D7P4-Z0PP26KM-17...|[[2.8426395E7], [...| [2.8426395E7]|[1.47990947E8]| [5.7973364E7]| [7.4965969E7]|   [3195665.0]| [3.8620047E7]|   [6275840.0]|   [6093814.0]|[1.35492556E8]|[1.56894903E8]| [4.6921689E7]|[1.69369566E8]| [4.3398079E7]|  [4.005616E7]|  [1.642381E8]| [4.0120676E7]|[1.71208476E8]| [1.2279514E7]| [3.4338428E8]| [1.3616303E7]|
    |{P6J4-Y0XJ63II-57...|[[9.5709306E7], [...| [9.5709306E7]| [5.1161408E7]| [4.8366782E7]|[1.56209838E8]| [4.7043974E7]|   [4755671.0]| [8.8000581E7]| [2.5400951E8]|   [3809694.0]| [2.3929356E7]| [6.9048794E7]| [7.0182772E7]| [5.8999453E7]|  [5.920316E7]| [5.7443667E7]|[1.63367351E8]| [7.0428572E7]|[1.14950897E8]|  [3.912324E7]| [1.0076825E8]|
    |{K7Y5-V5IP47OA-83...|[[1.30702124E8], ...|[1.30702124E8]|[1.31581719E8]|   [1914706.0]|[1.01382702E8]|[7.65567938E8]|[1.20816493E8]|[3.12875902E8]| [1.3353574E8]| [3.5978005E7]|[1.75913681E8]| [2.5884102E7]| [2.7457289E7]| [2.8029566E7]|[2.19481998E8]| [3.5053537E7]|[1.57293065E8]|[1.07832953E8]| [3.0033761E7]| [1.7864483E7]| [1.5933009E7]|
    |{R9V2-W5OA43XS-14...|[[5.953221E7], [3...|  [5.953221E7]| [3.3334989E7]| [2.0337453E7]|   [2693788.0]| [6.1773742E7]| [3.2965342E7]| [8.0556275E7]|   [1346327.0]|  [7.896364E7]|[1.60568104E8]| [6.9593553E7]|[1.02739037E8]| [1.0419805E7]|[1.42893998E8]| [3.1616059E7]|   [7937182.0]| [6.3439614E7]|  [6.863823E7]| [1.7864483E7]|[1.42751795E8]|
    |{X4R4-F1BP75UA-02...|[[3.15474607E8], ...|[3.15474607E8]|    [113682.0]| [4.8366782E7]|  [7.347063E7]|[2.76046937E8]|   [4755671.0]| [9.7478903E7]|[1.47037978E8]|[1.78478191E8]|   [7931006.0]|[1.04452162E8]| [7.0182772E7]| [1.3051544E8]|[3.18768417E8]| [2.0901836E8]| [1.8036513E8]|[2.78977338E8]|[1.92538897E8]|  [3.912324E7]|[1.15824384E8]|
    |{N4L7-S2MN81EJ-50...|[[1.10923149E8], ...|[1.10923149E8]|[2.09167648E8]| [3.9550617E7]| [8.0448648E7]| [7.3525113E8]|[1.52248302E8]| [6.0419998E7]| [7.2558632E7]| [1.4627018E7]| [7.2577179E7]|[6.67637496E8]|[2.17179658E8]| [4.9190379E7]|   [7475093.0]|[1.87774056E8]|[1.37258143E8]| [1.2057317E7]| [8.9867514E7]| [4.4260117E7]| [3.0989143E7]|
    +--------------------+--------------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+--------------+
    only showing top 20 rows
    



```python
udf_getNumber = udf(lambda x: int(x[0]), LongType())

for col_num in range(20):
    id_hash = id_hash.withColumn('hash_'+str(col_num), udf_getNumber('hash_'+str(col_num)))
```


```python
id_hash.show()
```

    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+
    |                  id|              hashes|   hash_0|   hash_1|   hash_2|   hash_3|   hash_4|   hash_5|   hash_6|   hash_7|   hash_8|   hash_9|  hash_10|  hash_11|  hash_12|  hash_13|  hash_14|  hash_15|  hash_16|  hash_17|  hash_18|  hash_19|
    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+
    |{R3I7-S4TX96FG-82...|[[8.776332E7], [1...| 87763320| 17940101| 64377752| 45060227| 94146089| 54730737|  2045183| 35318959| 97305009|160568104|126579267| 41193117| 51198766| 47158998|  7507190| 82002584| 50971487| 19229588| 28138237| 55599848|
    |{R0R9-E4GL59IK-29...|[[195285.0], [1.2...|   195285|126315806|226939755| 43066557| 29401395|149026164|465667429|136802781| 62127080| 76250380|148161613| 51362335| 72592440|281864322|320969010|163367351| 23015655| 75967366|123447019| 25487580|
    |{G2B2-A8XY58CP-28...|[[2.90624351E8], ...|290624351|104640665|120436410| 48549236|115042474|114372217|170742330|385458700|572021968| 44253680|320208273| 41193117| 85952566| 24460579|145858361|525808011|132294306| 26179662|105112325|240326464|
    |{A3A9-F4TH89AA-83...|[[8.2692039E7], [...| 82692039|339218494|210928785|198576277| 31456934| 39409618|273620356| 21076498|117151187|140896553| 29764764| 90192079|  4627505|200334998|  9798842|    50832|148256912|171309613|152766716| 28672437|
    |{E8B7-C8FZ88UF-29...|[[2.62393241E8], ...|262393241| 39615242|105215857| 11665476|156835244|124038631|229117145|273739681|259651373|180239655|157012455| 12203462|378507397|273371579| 74677580| 17048611|  8562838|107621761|121234205|488462473|
    |{X8T7-A6BT54FP-72...|[[3.66359397E8], ...|366359397| 17940101|  8319094| 41571218|147414821|339196160| 65667663| 82053606|127968511|103921106| 51347110| 17288071| 60774979| 17357741| 97067710| 83227663| 50971487| 93342551| 72868565|197474768|
    |{H5J6-G2RS59KI-83...|[[1.3212552E7], [...| 13212552| 55010130|105215857|149231820| 80614588| 64397151| 25395109| 53568684| 81973258| 31928531| 64623373|  5929983|149900727|198945093|434638399| 23122897| 17536486| 47408946| 43548868| 16801160|
    |{D9T8-M1HJ89XP-63...|[[1.88348622E8], ...|188348622|145559416|  5116900| 29110521|262515436| 70841427| 45531386| 53568684| 35978005|108899853|244976116| 63909293| 10419805|207437836| 10371755|163367351|  8562838|199488971|168888596| 30989143|
    |{V3L7-L2RB92RV-91...|[[4.6514943E7], [...| 46514943| 17940101| 48366782| 96398354| 44988435|  4755671| 41300729| 48080974| 98809818| 40580479| 60197952| 36108508| 60774979| 69857417| 29897320| 88076870| 75907741|  1475341|187223290| 35042151|
    |{D5K9-P0IJ71WK-63...|[[5266566.0], [2....|  5266566| 21788823| 70782140| 32599530| 31456934| 52298170| 13720146| 67811145|248834049| 80576354| 34734944| 36108508|  4627505| 92555836| 31043146|143332429|197654519| 26179662|240014558| 80210553|
    |{R0A5-U4YQ17EA-34...|[[1.05851868E8], ...|105851868| 21788823| 20337453|195087268|  1140126| 36187480| 41300729|142290491|111131951|216562329| 69593553| 95276688|169286014| 24460579| 30470233|414709908|203133688| 62067218|  4666666|115824384|
    |{Y8Z6-X5HU72BM-73...|[[1.3212552E7], [...| 13212552|248669208|189303844|279820146| 86781205| 20866361|317106559| 21076498| 52814565|108899853|249401537|  5929983| 84177040|191842255|145858361|    50832|339333283|365469144|431343371| 28672437|
    |{K3B8-S0RJ27BU-68...|[[1.3864811E8], [...|138648110|116186831| 92407081|361064015|336680553|108717512|143161747|113805569| 83478067|100900678|372768566| 58824684| 64791753|127298417| 32761885|442631180|121810869| 68638230|404236488|  4061732|
    |{J7Y1-G7KD78BQ-41...|[[4.6514943E7], [...| 46514943| 65139105| 89204887| 36088539| 33512473|152248302|154836710|  1346327| 98809818| 39927706|372768566| 72560512| 74367966| 59203160| 32188972|139070207| 47477008|  1475341|187223290| 30989143|
    |{D7P4-Z0PP26KM-17...|[[2.8426395E7], [...| 28426395|147990947| 57973364| 74965969|  3195665| 38620047|  6275840|  6093814|135492556|156894903| 46921689|169369566| 43398079| 40056160|164238100| 40120676|171208476| 12279514|343384280| 13616303|
    |{P6J4-Y0XJ63II-57...|[[9.5709306E7], [...| 95709306| 51161408| 48366782|156209838| 47043974|  4755671| 88000581|254009510|  3809694| 23929356| 69048794| 70182772| 58999453| 59203160| 57443667|163367351| 70428572|114950897| 39123240|100768250|
    |{K7Y5-V5IP47OA-83...|[[1.30702124E8], ...|130702124|131581719|  1914706|101382702|765567938|120816493|312875902|133535740| 35978005|175913681| 25884102| 27457289| 28029566|219481998| 35053537|157293065|107832953| 30033761| 17864483| 15933009|
    |{R9V2-W5OA43XS-14...|[[5.953221E7], [3...| 59532210| 33334989| 20337453|  2693788| 61773742| 32965342| 80556275|  1346327| 78963640|160568104| 69593553|102739037| 10419805|142893998| 31616059|  7937182| 63439614| 68638230| 17864483|142751795|
    |{X4R4-F1BP75UA-02...|[[3.15474607E8], ...|315474607|   113682| 48366782| 73470630|276046937|  4755671| 97478903|147037978|178478191|  7931006|104452162| 70182772|130515440|318768417|209018360|180365130|278977338|192538897| 39123240|115824384|
    |{N4L7-S2MN81EJ-50...|[[1.10923149E8], ...|110923149|209167648| 39550617| 80448648|735251130|152248302| 60419998| 72558632| 14627018| 72577179|667637496|217179658| 49190379|  7475093|187774056|137258143| 12057317| 89867514| 44260117| 30989143|
    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+
    only showing top 20 rows
    



```python
hash_cols = ['hash_'+str(i) for i in range(20)]

assembler = VectorAssembler(inputCols=hash_cols, outputCol="features")
id_hash = assembler.transform(id_hash)
```


```python
id_hash.show()
```

    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+--------------------+
    |                  id|              hashes|   hash_0|   hash_1|   hash_2|   hash_3|   hash_4|   hash_5|   hash_6|   hash_7|   hash_8|   hash_9|  hash_10|  hash_11|  hash_12|  hash_13|  hash_14|  hash_15|  hash_16|  hash_17|  hash_18|  hash_19|            features|
    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+--------------------+
    |{R3I7-S4TX96FG-82...|[[8.776332E7], [1...| 87763320| 17940101| 64377752| 45060227| 94146089| 54730737|  2045183| 35318959| 97305009|160568104|126579267| 41193117| 51198766| 47158998|  7507190| 82002584| 50971487| 19229588| 28138237| 55599848|[8.776332E7,1.794...|
    |{R0R9-E4GL59IK-29...|[[195285.0], [1.2...|   195285|126315806|226939755| 43066557| 29401395|149026164|465667429|136802781| 62127080| 76250380|148161613| 51362335| 72592440|281864322|320969010|163367351| 23015655| 75967366|123447019| 25487580|[195285.0,1.26315...|
    |{G2B2-A8XY58CP-28...|[[2.90624351E8], ...|290624351|104640665|120436410| 48549236|115042474|114372217|170742330|385458700|572021968| 44253680|320208273| 41193117| 85952566| 24460579|145858361|525808011|132294306| 26179662|105112325|240326464|[2.90624351E8,1.0...|
    |{A3A9-F4TH89AA-83...|[[8.2692039E7], [...| 82692039|339218494|210928785|198576277| 31456934| 39409618|273620356| 21076498|117151187|140896553| 29764764| 90192079|  4627505|200334998|  9798842|    50832|148256912|171309613|152766716| 28672437|[8.2692039E7,3.39...|
    |{E8B7-C8FZ88UF-29...|[[2.62393241E8], ...|262393241| 39615242|105215857| 11665476|156835244|124038631|229117145|273739681|259651373|180239655|157012455| 12203462|378507397|273371579| 74677580| 17048611|  8562838|107621761|121234205|488462473|[2.62393241E8,3.9...|
    |{X8T7-A6BT54FP-72...|[[3.66359397E8], ...|366359397| 17940101|  8319094| 41571218|147414821|339196160| 65667663| 82053606|127968511|103921106| 51347110| 17288071| 60774979| 17357741| 97067710| 83227663| 50971487| 93342551| 72868565|197474768|[3.66359397E8,1.7...|
    |{H5J6-G2RS59KI-83...|[[1.3212552E7], [...| 13212552| 55010130|105215857|149231820| 80614588| 64397151| 25395109| 53568684| 81973258| 31928531| 64623373|  5929983|149900727|198945093|434638399| 23122897| 17536486| 47408946| 43548868| 16801160|[1.3212552E7,5.50...|
    |{D9T8-M1HJ89XP-63...|[[1.88348622E8], ...|188348622|145559416|  5116900| 29110521|262515436| 70841427| 45531386| 53568684| 35978005|108899853|244976116| 63909293| 10419805|207437836| 10371755|163367351|  8562838|199488971|168888596| 30989143|[1.88348622E8,1.4...|
    |{V3L7-L2RB92RV-91...|[[4.6514943E7], [...| 46514943| 17940101| 48366782| 96398354| 44988435|  4755671| 41300729| 48080974| 98809818| 40580479| 60197952| 36108508| 60774979| 69857417| 29897320| 88076870| 75907741|  1475341|187223290| 35042151|[4.6514943E7,1.79...|
    |{D5K9-P0IJ71WK-63...|[[5266566.0], [2....|  5266566| 21788823| 70782140| 32599530| 31456934| 52298170| 13720146| 67811145|248834049| 80576354| 34734944| 36108508|  4627505| 92555836| 31043146|143332429|197654519| 26179662|240014558| 80210553|[5266566.0,2.1788...|
    |{R0A5-U4YQ17EA-34...|[[1.05851868E8], ...|105851868| 21788823| 20337453|195087268|  1140126| 36187480| 41300729|142290491|111131951|216562329| 69593553| 95276688|169286014| 24460579| 30470233|414709908|203133688| 62067218|  4666666|115824384|[1.05851868E8,2.1...|
    |{Y8Z6-X5HU72BM-73...|[[1.3212552E7], [...| 13212552|248669208|189303844|279820146| 86781205| 20866361|317106559| 21076498| 52814565|108899853|249401537|  5929983| 84177040|191842255|145858361|    50832|339333283|365469144|431343371| 28672437|[1.3212552E7,2.48...|
    |{K3B8-S0RJ27BU-68...|[[1.3864811E8], [...|138648110|116186831| 92407081|361064015|336680553|108717512|143161747|113805569| 83478067|100900678|372768566| 58824684| 64791753|127298417| 32761885|442631180|121810869| 68638230|404236488|  4061732|[1.3864811E8,1.16...|
    |{J7Y1-G7KD78BQ-41...|[[4.6514943E7], [...| 46514943| 65139105| 89204887| 36088539| 33512473|152248302|154836710|  1346327| 98809818| 39927706|372768566| 72560512| 74367966| 59203160| 32188972|139070207| 47477008|  1475341|187223290| 30989143|[4.6514943E7,6.51...|
    |{D7P4-Z0PP26KM-17...|[[2.8426395E7], [...| 28426395|147990947| 57973364| 74965969|  3195665| 38620047|  6275840|  6093814|135492556|156894903| 46921689|169369566| 43398079| 40056160|164238100| 40120676|171208476| 12279514|343384280| 13616303|[2.8426395E7,1.47...|
    |{P6J4-Y0XJ63II-57...|[[9.5709306E7], [...| 95709306| 51161408| 48366782|156209838| 47043974|  4755671| 88000581|254009510|  3809694| 23929356| 69048794| 70182772| 58999453| 59203160| 57443667|163367351| 70428572|114950897| 39123240|100768250|[9.5709306E7,5.11...|
    |{K7Y5-V5IP47OA-83...|[[1.30702124E8], ...|130702124|131581719|  1914706|101382702|765567938|120816493|312875902|133535740| 35978005|175913681| 25884102| 27457289| 28029566|219481998| 35053537|157293065|107832953| 30033761| 17864483| 15933009|[1.30702124E8,1.3...|
    |{R9V2-W5OA43XS-14...|[[5.953221E7], [3...| 59532210| 33334989| 20337453|  2693788| 61773742| 32965342| 80556275|  1346327| 78963640|160568104| 69593553|102739037| 10419805|142893998| 31616059|  7937182| 63439614| 68638230| 17864483|142751795|[5.953221E7,3.333...|
    |{X4R4-F1BP75UA-02...|[[3.15474607E8], ...|315474607|   113682| 48366782| 73470630|276046937|  4755671| 97478903|147037978|178478191|  7931006|104452162| 70182772|130515440|318768417|209018360|180365130|278977338|192538897| 39123240|115824384|[3.15474607E8,113...|
    |{N4L7-S2MN81EJ-50...|[[1.10923149E8], ...|110923149|209167648| 39550617| 80448648|735251130|152248302| 60419998| 72558632| 14627018| 72577179|667637496|217179658| 49190379|  7475093|187774056|137258143| 12057317| 89867514| 44260117| 30989143|[1.10923149E8,2.0...|
    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+--------------------+
    only showing top 20 rows
    

Now the column `features` formed as the DenseVector we want.

## Rescale data 

At the same time, we can find that the values in the column `features` we obtained are very large, which is not conducive to subsequent steps such as model training. Therefore, we need to use Scaler to scale them down to a suitable size.


```python
scaler = StandardScaler(inputCol="features", outputCol="scaledFeatures",
                        withStd=True, withMean=False)

# Compute summary statistics by fitting the StandardScaler

scalerModel = scaler.fit(id_hash)

# Normalize each feature to have unit standard deviation.

id_hash_scaled = scalerModel.transform(id_hash)
id_hash_scaled.show()
```

    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+--------------------+--------------------+
    |                  id|              hashes|   hash_0|   hash_1|   hash_2|   hash_3|   hash_4|   hash_5|   hash_6|   hash_7|   hash_8|   hash_9|  hash_10|  hash_11|  hash_12|  hash_13|  hash_14|  hash_15|  hash_16|  hash_17|  hash_18|  hash_19|            features|      scaledFeatures|
    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+--------------------+--------------------+
    |{R3I7-S4TX96FG-82...|[[8.776332E7], [1...| 87763320| 17940101| 64377752| 45060227| 94146089| 54730737|  2045183| 35318959| 97305009|160568104|126579267| 41193117| 51198766| 47158998|  7507190| 82002584| 50971487| 19229588| 28138237| 55599848|[8.776332E7,1.794...|[0.31075800760838...|
    |{R0R9-E4GL59IK-29...|[[195285.0], [1.2...|   195285|126315806|226939755| 43066557| 29401395|149026164|465667429|136802781| 62127080| 76250380|148161613| 51362335| 72592440|281864322|320969010|163367351| 23015655| 75967366|123447019| 25487580|[195285.0,1.26315...|[6.91477686985905...|
    |{G2B2-A8XY58CP-28...|[[2.90624351E8], ...|290624351|104640665|120436410| 48549236|115042474|114372217|170742330|385458700|572021968| 44253680|320208273| 41193117| 85952566| 24460579|145858361|525808011|132294306| 26179662|105112325|240326464|[2.90624351E8,1.0...|[1.02906139238169...|
    |{A3A9-F4TH89AA-83...|[[8.2692039E7], [...| 82692039|339218494|210928785|198576277| 31456934| 39409618|273620356| 21076498|117151187|140896553| 29764764| 90192079|  4627505|200334998|  9798842|    50832|148256912|171309613|152766716| 28672437|[8.2692039E7,3.39...|[0.29280128970411...|
    |{E8B7-C8FZ88UF-29...|[[2.62393241E8], ...|262393241| 39615242|105215857| 11665476|156835244|124038631|229117145|273739681|259651373|180239655|157012455| 12203462|378507397|273371579| 74677580| 17048611|  8562838|107621761|121234205|488462473|[2.62393241E8,3.9...|[0.92909886252100...|
    |{X8T7-A6BT54FP-72...|[[3.66359397E8], ...|366359397| 17940101|  8319094| 41571218|147414821|339196160| 65667663| 82053606|127968511|103921106| 51347110| 17288071| 60774979| 17357741| 97067710| 83227663| 50971487| 93342551| 72868565|197474768|[3.66359397E8,1.7...|[1.2972289138598,...|
    |{H5J6-G2RS59KI-83...|[[1.3212552E7], [...| 13212552| 55010130|105215857|149231820| 80614588| 64397151| 25395109| 53568684| 81973258| 31928531| 64623373|  5929983|149900727|198945093|434638399| 23122897| 17536486| 47408946| 43548868| 16801160|[1.3212552E7,5.50...|[0.04678385383486...|
    |{D9T8-M1HJ89XP-63...|[[1.88348622E8], ...|188348622|145559416|  5116900| 29110521|262515436| 70841427| 45531386| 53568684| 35978005|108899853|244976116| 63909293| 10419805|207437836| 10371755|163367351|  8562838|199488971|168888596| 30989143|[1.88348622E8,1.4...|[0.66691691367766...|
    |{V3L7-L2RB92RV-91...|[[4.6514943E7], [...| 46514943| 17940101| 48366782| 96398354| 44988435|  4755671| 41300729| 48080974| 98809818| 40580479| 60197952| 36108508| 60774979| 69857417| 29897320| 88076870| 75907741|  1475341|187223290| 35042151|[4.6514943E7,1.79...|[0.16470310159982...|
    |{D5K9-P0IJ71WK-63...|[[5266566.0], [2....|  5266566| 21788823| 70782140| 32599530| 31456934| 52298170| 13720146| 67811145|248834049| 80576354| 34734944| 36108508|  4627505| 92555836| 31043146|143332429|197654519| 26179662|240014558| 80210553|[5266566.0,2.1788...|[0.01864819559125...|
    |{R0A5-U4YQ17EA-34...|[[1.05851868E8], ...|105851868| 21788823| 20337453|195087268|  1140126| 36187480| 41300729|142290491|111131951|216562329| 69593553| 95276688|169286014| 24460579| 30470233|414709908|203133688| 62067218|  4666666|115824384|[1.05851868E8,2.1...|[0.37480710166053...|
    |{Y8Z6-X5HU72BM-73...|[[1.3212552E7], [...| 13212552|248669208|189303844|279820146| 86781205| 20866361|317106559| 21076498| 52814565|108899853|249401537|  5929983| 84177040|191842255|145858361|    50832|339333283|365469144|431343371| 28672437|[1.3212552E7,2.48...|[0.04678385383486...|
    |{K3B8-S0RJ27BU-68...|[[1.3864811E8], [...|138648110|116186831| 92407081|361064015|336680553|108717512|143161747|113805569| 83478067|100900678|372768566| 58824684| 64791753|127298417| 32761885|442631180|121810869| 68638230|404236488|  4061732|[1.3864811E8,1.16...|[0.49093414449531...|
    |{J7Y1-G7KD78BQ-41...|[[4.6514943E7], [...| 46514943| 65139105| 89204887| 36088539| 33512473|152248302|154836710|  1346327| 98809818| 39927706|372768566| 72560512| 74367966| 59203160| 32188972|139070207| 47477008|  1475341|187223290| 30989143|[4.6514943E7,6.51...|[0.16470310159982...|
    |{D7P4-Z0PP26KM-17...|[[2.8426395E7], [...| 28426395|147990947| 57973364| 74965969|  3195665| 38620047|  6275840|  6093814|135492556|156894903| 46921689|169369566| 43398079| 40056160|164238100| 40120676|171208476| 12279514|343384280| 13616303|[2.8426395E7,1.47...|[0.10065400754767...|
    |{P6J4-Y0XJ63II-57...|[[9.5709306E7], [...| 95709306| 51161408| 48366782|156209838| 47043974|  4755671| 88000581|254009510|  3809694| 23929356| 69048794| 70182772| 58999453| 59203160| 57443667|163367351| 70428572|114950897| 39123240|100768250|[9.5709306E7,5.11...|[0.33889366585199...|
    |{K7Y5-V5IP47OA-83...|[[1.30702124E8], ...|130702124|131581719|  1914706|101382702|765567938|120816493|312875902|133535740| 35978005|175913681| 25884102| 27457289| 28029566|219481998| 35053537|157293065|107832953| 30033761| 17864483| 15933009|[1.30702124E8,1.3...|[0.46279848625170...|
    |{R9V2-W5OA43XS-14...|[[5.953221E7], [3...| 59532210| 33334989| 20337453|  2693788| 61773742| 32965342| 80556275|  1346327| 78963640|160568104| 69593553|102739037| 10419805|142893998| 31616059|  7937182| 63439614| 68638230| 17864483|142751795|[5.953221E7,3.333...|[0.21079547774769...|
    |{X4R4-F1BP75UA-02...|[[3.15474607E8], ...|315474607|   113682| 48366782| 73470630|276046937|  4755671| 97478903|147037978|178478191|  7931006|104452162| 70182772|130515440|318768417|209018360|180365130|278977338|192538897| 39123240|115824384|[3.15474607E8,113...|[1.11705277697287...|
    |{N4L7-S2MN81EJ-50...|[[1.10923149E8], ...|110923149|209167648| 39550617| 80448648|735251130|152248302| 60419998| 72558632| 14627018| 72577179|667637496|217179658| 49190379|  7475093|187774056|137258143| 12057317| 89867514| 44260117| 30989143|[1.10923149E8,2.0...|[0.39276381956480...|
    +--------------------+--------------------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------+--------------------+--------------------+
    only showing top 20 rows
    

This step may cost several minutes.

```python
id_hash_scaled = id_hash_scaled.select('id','scaledFeatures')
id_hash_scaled.show()
```

    +--------------------+--------------------+
    |                  id|      scaledFeatures|
    +--------------------+--------------------+
    |{R3I7-S4TX96FG-82...|[0.31075800760838...|
    |{R0R9-E4GL59IK-29...|[6.91477686985905...|
    |{G2B2-A8XY58CP-28...|[1.02906139238169...|
    |{A3A9-F4TH89AA-83...|[0.29280128970411...|
    |{E8B7-C8FZ88UF-29...|[0.92909886252100...|
    |{X8T7-A6BT54FP-72...|[1.2972289138598,...|
    |{H5J6-G2RS59KI-83...|[0.04678385383486...|
    |{D9T8-M1HJ89XP-63...|[0.66691691367766...|
    |{V3L7-L2RB92RV-91...|[0.16470310159982...|
    |{D5K9-P0IJ71WK-63...|[0.01864819559125...|
    |{R0A5-U4YQ17EA-34...|[0.37480710166053...|
    |{Y8Z6-X5HU72BM-73...|[0.04678385383486...|
    |{K3B8-S0RJ27BU-68...|[0.49093414449531...|
    |{J7Y1-G7KD78BQ-41...|[0.16470310159982...|
    |{D7P4-Z0PP26KM-17...|[0.10065400754767...|
    |{P6J4-Y0XJ63II-57...|[0.33889366585199...|
    |{K7Y5-V5IP47OA-83...|[0.46279848625170...|
    |{R9V2-W5OA43XS-14...|[0.21079547774769...|
    |{X4R4-F1BP75UA-02...|[1.11705277697287...|
    |{N4L7-S2MN81EJ-50...|[0.39276381956480...|
    +--------------------+--------------------+
    only showing top 20 rows
    


## Save & Retrieve data

The amount of data was simply too large to be handled by Colab during the modeling process and caused a disconnection of its Java back-end server. Therefore, we only extract a portion of the data for demonstration. Now I only randomly sampled 0.1% of the original data. 


```python
id_hash_sub = id_hash_scaled.sample(withReplacement=False, fraction=0.001, seed=42)

id_hash_sub_split = id_hash_sub.withColumn("scaledHash", vector_to_array("scaledFeatures")).select(['id'] + [col("scaledHash")[i] for i in range(20)])
```


```python
id_hash_sub_split.show()
```

    +--------------------+--------------------+-------------------+--------------------+-------------------+--------------------+--------------------+-------------------+-------------------+--------------------+--------------------+-------------------+--------------------+--------------------+--------------------+-------------------+--------------------+-------------------+--------------------+--------------------+--------------------+
    |                  id|       scaledHash[0]|      scaledHash[1]|       scaledHash[2]|      scaledHash[3]|       scaledHash[4]|       scaledHash[5]|      scaledHash[6]|      scaledHash[7]|       scaledHash[8]|       scaledHash[9]|     scaledHash[10]|      scaledHash[11]|      scaledHash[12]|      scaledHash[13]|     scaledHash[14]|      scaledHash[15]|     scaledHash[16]|      scaledHash[17]|      scaledHash[18]|      scaledHash[19]|
    +--------------------+--------------------+-------------------+--------------------+-------------------+--------------------+--------------------+-------------------+-------------------+--------------------+--------------------+-------------------+--------------------+--------------------+--------------------+-------------------+--------------------+-------------------+--------------------+--------------------+--------------------+
    |{K2S4-F1DY33JO-15...| 0.29280128970411595| 0.3091033263397524|   2.476281640149856|0.12861855363131602|  0.2518061180151361|  0.5796193263919759| 3.0776898705378564|  2.177209668935742|  0.6748656539993456|  0.2110169547910796|  0.673424066914295|   5.619513628670911|   1.177556880743654|   2.331892220940217| 0.6603510093508081|  1.7811379014403648|  2.232125442478518|  1.0194858268210645|  1.2534815642644868|0.036371409483113765|
    |{D1I9-G5HX22BQ-41...|0.024633768226013954| 0.4878114669475884|  2.2940673130343248|  8.753337779706603|   2.143880850303592| 0.16856546777515793| 2.3061401268552855|  1.506277442688671|   2.813599251203017|   1.716103268825347|0.28752009454781946|   4.639510658817771|  3.3263900582637285|  1.1344919733439336|  3.161960857414897| 0.14624060997894936| 3.2281803139849057|  2.1853046676413554|  0.7301844611439626|  2.3284683037114586|
    |{I9E9-Q3OT41AO-31...|   0.368821529025777| 2.0278705582652594|   1.740669600417486| 1.2875654000968493|  1.3308415278261754| 0.02247731958342665| 0.8790143676268818|0.44944703745559733| 0.30799466065031955|  0.5543026192616594| 1.9935958416189692|  0.8092609117203796|  0.3960105278308719| 0.39284644224423737| 0.2514173549141416| 0.24072594895414837|  0.786553328360483| 0.10594780803748093| 0.12258914456450333| 0.32261815583704795|
    |{S7Z6-V6GC84AJ-34...|   6.375029393281213|  5.300359716339008|  0.4256292299089153| 3.0779641787054346|   2.376968625846961|   5.457482574221892|0.04448809445873216| 1.9842605794427406|  6.0733492705399925|    6.01806834603609|  4.028590989786734|  0.9145188180748663|   4.414433387405006|  2.0352340085003386|  5.976271936486997|   5.947038568675222|  2.543187373343792|   5.714168897515404|  0.6037371409555599|    4.00748258766664|
    |{T5S5-V5TC63CJ-15...|  0.3388936658519919|0.36601829572652156|  0.9009350880476696|0.09011486551094054|  0.2257600876666489|  1.0819782776286258| 0.8518278896657661|0.05733830764840028|  0.5022728966838492|  0.3048177593351857| 1.4817236690544147| 0.08242913749621385|  0.5356322755638796| 0.42035878392087134| 0.8874882439177393|0.028886254236132933|0.26545000731622687| 0.23942811693198968| 0.03829093110556815|  0.5041896108235284|
    |{O0Q0-G0QO68BH-68...|  1.4031770163552462| 0.5959418033380846|   4.129720046730127| 0.9063834315029069|  0.2518061180151361|  0.8855818575822001| 1.9196773763784638| 0.2502873971414017|  0.7998503797803153|0.002787595423446...| 0.3120126942966307| 0.17771347359345907|  1.8612002930473002|0.002883806937687...| 0.3503168850896909|  2.1533017452257455| 1.2360705709721302|   0.525164762290429|   1.055056672798426|  0.6387669533362803|
    |{T6J3-O6HK21DK-63...|   1.129023922242387|  2.894085591643298|0.015564634988835327| 0.7446695450906622|  0.5109399239069924|  0.6526634004878414| 0.9234079927146928| 0.4892789655184364|  0.7305568695277551|  0.1728741031832374| 0.9293601531965723|   1.400914068818334|  0.6190698898116233|  0.1512129131576268| 0.2540844616610355| 0.21207942007111574| 0.6688154793457498|   0.201876061793146|  0.7723335678734302| 0.12395463952213706|
    |{W9R3-C0TQ20IV-04...| 0.19283875984342688|0.06551698389761866|  1.3890915374458344|0.14401949431502212|0.058488935703786944| 0.11825719592046345| 1.3098707431915144|0.05112775682720604|0.003616030191778...|  0.1141035012959747| 0.2185364656196012|  0.5361425755769211| 0.27085410645925634| 0.45863852343427475|0.13918229100412294|  0.6759603626210096|0.22329839113730143|  0.9839818875907766|   0.321014036030564|  0.9955048684008074|
    |{Q0P2-K8RK31CY-66...| 0.22875219565196894|0.14405802056248707|  0.5753876432353795| 0.4982491485069237|  0.1763976199812685|  0.6847106538185695| 1.8838873248540005| 0.7681024619583103|  0.3976916129891054|  0.3079304082861845| 1.1243119823228696|  0.5560897160914042|  0.1729511658501205|  1.7828330815769586|0.04294986757546754|  0.6759603626210096| 1.1437670298078815| 0.12472383560690277|0.014298407346348845|  0.5447787283888231|
    |{N1I1-L9QL88UM-07...|   1.303214486494557| 1.0535251884967243| 0.11968710762628161| 0.9217843721866129|  0.6946652317649812|0.008691084776664991| 2.9789232895278137|  3.433199714483011|     1.8094843708568|  0.3223328606636074| 0.6694243810281758|    3.32102572749565|   1.667071583789333| 0.17872525483426077| 0.6656852228445959|  2.3937879688267936| 1.1310498758826393|   1.899568022282916|  1.8370843570847268|   1.655581591147699|
    |{P1U7-G4IM71WT-84...| 0.19283875984342688|   0.44455933239139| 0.14571772578564318|0.19792412311910368| 0.30116858570051647|  1.4702963340042725| 2.0728169133107377| 1.9045967233170622|  0.6680645066372703|  0.3635883612224484| 0.4744725519018785|  0.6513740521886495|   1.177556880743654|   2.359404562616851|0.13918229100412294| 0.10904832876981561|0.10156038771936927|  1.5554551783076553|   0.321014036030564| 0.16454375708743177|
    |{X6E3-Z7LK32OI-96...| 0.07072614437388991|0.10080588600628863|  0.9073603836773748| 1.0834982585988577|0.058488935703786944| 0.15030444925119152| 0.4567615656255967| 0.2564979479625959|0.030820619640079327|  0.2491598063989218|0.34450466581768024|  0.6087186692686476| 0.34150356630591244|  1.0136752088006284|0.24608314142035387| 0.00878547767920146|  2.022084052299536|0.007971438373260023|  0.2490364647529061|  0.6387669533362803|
    |{T9L8-P4FF61BO-55...|   1.245150965077167| 0.6528567727248538|  0.7251460565618438| 0.1709718087170629|  1.8422468581681217| 0.15030444925119152|  2.736996502419918| 0.1307916129528843| 0.37728817090287964|  0.4811295649969738| 0.2835204086617003|  2.3709434684142336|  0.5500976019252717|  0.3653341005676034| 1.0906215177626255|  2.1962715385502944|0.10156038771936927|   0.963157744112799|   0.321014036030564|  0.6387669533362803|
    |{G7Q1-O9SX68CM-70...|  0.2048099051129409|0.26585119178355393|  1.3890915374458344| 2.4503618135277536|  0.4355314258731248|  0.5796193263919759| 1.1123375811714298|0.29632987602543504|  0.5022728966838492|  0.4605018147175533| 1.1528042679577999|  0.5560897160914042|  0.6480005425344075| 0.13446796931776228| 0.2514173549141416| 0.12337159321133193| 0.9798774102110238|  0.5231166463818732| 0.03829093110556815|0.036371409483113765|
    |{V4Y4-D2OI92EQ-25...|    0.59688224699076|0.05185414906704792| 0.11968710762628161|0.09011486551094054|  1.4322960562085303|  1.2921609324818133| 0.8160378381413028|0.29632987602543504| 0.40449276035118065|   0.519272416604816|  4.682430106093844|  0.1023762780106969|  0.6190698898116233|  2.0902586918536064| 0.8688184966894822| 0.18343289118808312|0.42933962691308447|  2.8547543280224548|  1.2294890405052674|  1.4804151310696525|
    |{K8U6-M2IB80FI-83...| 0.49691971713007094|  1.648828209771488|  1.5844858914613165|0.14401949431502212|  0.9469367562150711| 0.09999617739649704| 0.6456911540823338| 0.6087747497069538| 0.14220305069689843|  0.5543026192616594| 0.3525040375899186|0.029800184318970472| 0.10397887796376892|  1.6177590315171548| 0.8954895641584208|  0.2035336677450146| 1.5638498101658456| 0.46883667958216346|  1.2956306709939545|   2.503634763789505|
    |{B5X0-S1HT16TF-47...|   6.375029393281213|  5.300359716339008|  0.4256292299089153| 3.0779641787054346|   2.376968625846961|   5.457482574221892|0.04448809445873216| 1.9842605794427406|  6.0733492705399925|    6.01806834603609|  4.028590989786734|  0.9145188180748663|   4.414433387405006|  2.0352340085003386|  5.976271936486997|   5.947038568675222|  2.543187373343792|   5.714168897515404|  0.6037371409555599|    4.00748258766664|
    |{W3F3-H9CT13FV-87...|   0.560968811182218| 0.1087691184538171|  0.7772072928805669|0.15557086803335682| 0.10098912194740091|   0.451792196724211| 1.2468942137059356| 0.8017238391999552|0.017218324915928816|  0.7862723778597114|0.15755220846362125|  0.6940294351086513|0.062260070839897076|  0.9036258420940925|0.03494854733478592|  0.9307698506635739| 1.2193532626436892|  1.3260466156574815|  0.7120278781737144| 0.14745169575900136|
    |{V1G2-M6NB39HJ-20...|   0.368821529025777|  2.801881720147859|  0.9529963243663927|  1.071946884880523|  1.2979332160359218| 0.04073833810739306|0.08027814598319546|  2.091335261988869| 0.15580534542104893|0.002787595423446...| 1.2827721540419983|  1.9072564600762847|  1.8612002930473002|  1.0244426066373977| 2.0180567837990835|  1.7668146369988484| 1.9632151277921694|  0.5231166463818732| 0.03829093110556815|   1.463323069741222|
    |{I0L5-D2RB95MK-64...| 0.22276662301721195|  0.587978570890556|  0.4452345524385717| 1.4107755983887185|  0.1763976199812685| 0.31465361596688923|0.32220493309109105| 0.2564979479625959| 0.48058741796581594|  0.8213025805165548| 0.9293601531965723|  0.3455739033824307|  0.5645629282866638|0.013651204774456999|0.24608314142035387| 0.16056387442046569| 0.3116017778983513| 0.25820414450141155|   0.321014036030564| 0.12395463952213706|
    +--------------------+--------------------+-------------------+--------------------+-------------------+--------------------+--------------------+-------------------+-------------------+--------------------+--------------------+-------------------+--------------------+--------------------+--------------------+-------------------+--------------------+-------------------+--------------------+--------------------+--------------------+
    only showing top 20 rows
    


Also, the data was stored and then retrieved for manipulation to speed up the subsequent modeling process.


```python
id_hash_sub_split.write.csv('./data/id_hash_sub_split.csv', header = True, mode = 'error')
```



Now, we need to retrieve the previously saved data and perform the modeling operation. If the runtime was ever interrupted after the previous step, you need to run the Build environment section at the beginning of the notebook.


```python
id_hash_sub_split = spark.read.csv( './data/id_hash_sub_split.csv',inferSchema=True,header=True)
```


```python
id_hash_sub_split.show()
```

    +--------------------+--------------------+--------------------+-------------------+-------------------+-------------------+-------------------+--------------------+-------------------+--------------------+--------------------+-------------------+-------------------+-------------------+--------------------+--------------------+--------------------+--------------------+-------------------+--------------------+-------------------+
    |                  id|       scaledHash[0]|       scaledHash[1]|      scaledHash[2]|      scaledHash[3]|      scaledHash[4]|      scaledHash[5]|       scaledHash[6]|      scaledHash[7]|       scaledHash[8]|       scaledHash[9]|     scaledHash[10]|     scaledHash[11]|     scaledHash[12]|      scaledHash[13]|      scaledHash[14]|      scaledHash[15]|      scaledHash[16]|     scaledHash[17]|      scaledHash[18]|     scaledHash[19]|
    +--------------------+--------------------+--------------------+-------------------+-------------------+-------------------+-------------------+--------------------+-------------------+--------------------+--------------------+-------------------+-------------------+-------------------+--------------------+--------------------+--------------------+--------------------+-------------------+--------------------+-------------------+
    |{R0A8-D0KA23YU-26...| 0.05276942646961887|  1.3756525676037266| 2.6324653491060253| 2.0268265898480644| 0.7536195739037219| 0.5796193263919759| 0.25062483004216446|  3.001259056612975|   1.117831304963007|  0.6718438230361848| 0.9293601531965723| 0.5034607629141609|  1.652606257427941|  0.5028958089507732|  0.9730522403588192|  0.4440198913460626|  1.3999601905689878|0.10389969212892508|  0.7963260916326496|  1.527409243543381|
    |{C4W6-P6ZE51MH-58...|  0.4909341444953139|  1.2618226288301881| 0.7251460565618438| 0.5521537773110052|0.41907726997799805| 0.2736567952017516|  2.5838569654876444|  0.722059983074277|   0.847458411314842|  0.5749303695410799| 0.7344083240702749| 0.1876870438507006| 0.5500976019252717|  0.5411755484641766|  0.7752531800077206|   0.509858701438229| 0.17714662055517702|  2.126928469024534|  0.5136029867076537| 0.9485107559270788|
    |{Q7V6-N6NE83EZ-03...|    1.00511910184267|   0.853190980610789| 0.0415952531481969| 0.1825231824353976| 0.6946652317649812| 1.0637172591046593|  1.8394936997661895|0.29632987602543504|  0.1762087875072747|   0.709986674644027|0.32001206606886906| 0.3029185204624289|  0.466659987677528|  1.0686998921538964| 0.06161961480372464|0.028886254236132933|  1.4715462690015966|0.06839575289863725| 0.14074572753475162|0.02996641457467989|
    |{R1W9-Z1SB61VN-25...|  0.4909341444953139|6.387820633209871E-4|0.14571772578564318| 1.7265015644602446| 0.6192567337311136| 0.6847106538185695|  0.8790143676268818|  1.432824137384187|  0.6191744384709361|  0.2316447050705001| 1.3802480686051468|0.29294495020518735| 0.3960105278308719|  1.6452713731937887|   0.368986632317948|  0.6616370981794932|  0.9210084857036572|0.33330825477909887|  0.2970215122713447|0.07696052704840847|
    |{T8V6-A0YZ92MM-43...| 0.06474057173913289|    0.44455933239139| 0.3411120798011254|0.09011486551094054|  1.766838360134254|0.09552139367929229| 0.08027814598319546| 1.1203792637026682|  0.7998503797803153|  0.6893589243646066| 0.3120126942966307| 0.1023762780106969| 1.9173844265325641|  0.7828090775507872|  0.6656852228445959|0.037432006562234076|  0.9671602562857817|0.31453222720967705|  2.2760832467463157| 0.1175496446137032|
    |{J5S7-D3EN76YN-47...|   6.375029393281213|   5.300359716339008| 0.4256292299089153| 3.0779641787054346|  2.376968625846961|  5.457482574221892| 0.04448809445873216| 1.9842605794427406|  6.0733492705399925|    6.01806834603609|  4.028590989786734| 0.9145188180748663|  4.414433387405006|  2.0352340085003386|   5.976271936486997|   5.947038568675222|   2.543187373343792|  5.714168897515404|  0.6037371409555599|   4.00748258766664|
    |{V6X5-A6JS37ZC-78...| 0.12280409315652288| 0.10080588600628863| 0.9269657062070312|0.07471392482723446| 1.0717077219343192|  1.488557352528239|  1.2740806916670513|0.29011932520424083|  1.0349354999862965|  1.1264453933792924|0.18604449409855162| 1.1676900137038417| 1.1503033999811743|  1.9586745294735317| 0.14184939775101682|  0.9164465862220575| 0.23601554506254357| 0.6190449001375382|  0.6458862476850274|0.12395463952213706|
    |{K9Q3-C6MB20EN-05...| 0.05276942646961887|  0.1656840878405863| 0.2694455209527459|0.09011486551094054|0.13389743373765453|0.02247731958342665|  0.5111345215478281| 1.1079581620602796|  0.3011935132882443| 0.07907329863913132|0.38499600911096815| 0.1023762780106969| 1.8339468122848206|0.013651204774456999|  0.7725860732608267|  0.2980190067202136|   0.786553328360483| 0.4480125361041858| 0.08044003783503574| 0.0940525883768389|
    |{E2N0-S3EQ10XZ-59...| 0.05276942646961887|    0.44455933239139| 0.8032379110399285| 0.7831732332110376| 1.4981126797890374| 0.9038428761061664|  0.4481579920622491|0.21666601989975678| 0.07290954044433837| 0.11721615024697352|  1.798644012492672|0.23034242677070246| 1.0106816522481665|  0.3103094172143355| 0.15251782473859232| 0.12337159321133193|  0.9082913317784151| 1.0986861689158636|0.014298407346348845| 0.5853678459541178|
    |{X2V0-G5QC95QV-97...|   0.374807101660534|  1.1400294576091212| 0.5233264069166564| 0.3480866358130137| 0.8125739160424628|0.18682648629912435|  1.4444273757260202| 1.6257732268771883|0.059307245720187855|   0.519272416604816| 0.4419805803808289| 0.3029185204624289|0.20188181857290474| 0.18949265267103022| 0.24341603467345999| 0.17488713886198198| 0.08884323379412713|0.31453222720967705|  1.0790491965576454|0.14745169575900136|
    |{W8Y0-Q6VB44AY-48...|    0.24670891355624| 0.22259905722735548| 0.0612005756778533|0.09011486551094054| 0.2093059317715221| 1.0957645124353874|  1.1123375811714298|0.28390877438304657|  0.4329793864312892|   0.900700932683238|0.12106055105645251| 0.1023762780106969|  0.814875771029895|  1.3486131607539102| 0.34764977834279703|  1.3087112065643698|   0.072125925465686| 0.6002688725681163| 0.40531224948949923|  0.615269897099416|
    |{Z3O3-C5WT32II-33...|   0.748922725634082|  0.9829473842793844| 0.4452345524385717|  1.672596935656163|0.36012292783925726| 0.3832229063455501| 0.32220493309109105| 0.4096151093927582| 0.21829770831153375|  0.4223589631097111|0.12106055105645251|0.25028956728518553| 1.2754598213527897|  0.4478711255975053|  1.1921881546850688| 0.16056387442046569|  0.6226637087636253| 0.0663476369900814| 0.20688735802343852| 0.2991210996001837|
    |{J0V1-Q3VT17LQ-96...| 0.08269728964340393|  1.6408649773239594|  1.005057560685116| 0.6484116612008337|  2.586739964053437|0.09552139367929229|  0.4567615656255967|  5.626524381074837|  0.4397805337933644|  0.6893589243646066| 3.5452111028571096| 1.3156033029783303|0.21466997297399235|   0.661992313007482|  0.3583182053303725|  2.6257284401017404|   2.471601294911183| 1.2488943894712383| 0.12258914456450333| 0.9250136996902145|
    |{I3C6-O0BG43IS-00...|   0.801000674416715|   0.674482840002953| 0.7772072928805669|  2.434960872844048|  1.599567208171392| 0.5293110545372813|  0.8518278896657661| 0.2502873971414017| 0.02401947227800407|  0.4223589631097111|  0.380996323224849|0.08242913749621385|  2.113190307750836| 0.42035878392087134|  0.8874882439177393| 0.12337159321133193|5.398470330772673E-4| 0.5231166463818732| 0.03829093110556815|  1.661986586056133|
    |{Z1D7-Y9ON91MY-18...| 0.19283875984342688|  0.6881456748335237| 0.3995986117495537| 1.2490617119764738| 0.8619363837278432| 1.8675639578143288|   1.318474316754862| 1.5461093707515101|   1.511906887760334|0.002787595423446...| 1.8961199270558207| 0.4934871926569193| 1.8612002930473002|  0.9861628671239944| 0.13918229100412294|  0.6845061149471107|  0.7738361744352408| 1.5366791507382336|   0.315178095241593| 0.6387669533362803|
    |{E8R7-M4NQ76XB-96...|  0.3388936658519919|  1.0751512557748235|   1.56488056893166| 0.0323606697414876|  2.645694306192178|0.16856546777515793|  2.2431635973697066| 1.3531602812585088|0.030820619640079327|  0.6718438230361848|0.06007629390047258| 2.1704012259625016|0.20188181857290474|  1.1344919733439336|0.045616974322361406| 0.14624060997894936| 0.38718801073415904| 0.7357972973711809|  0.2490364647529061|  1.433421018595924|
    |{M2B5-U6QV73PC-27...| 0.11681852052176586|  0.5526896687818861|0.17174834394500474| 0.5675547179947114| 0.6946652317649812| 0.8445850368170625|0.017301616497616433| 0.2564979479625959|  1.0417366473483716|  0.5749303695410799|0.40948860885977933| 0.7566319585431363| 0.2563887800978642|  1.1620043150205677| 0.24608314142035387|  0.2035336677450146| 0.05940877154044385| 1.1174621964852853|    2.99196930054393| 0.9485107559270788|
    |{X1H9-X8SF98ZC-71...|     1.2972289138598|    0.44455933239139| 0.5689623476056743| 0.9602880603069884|0.23535196212000928| 0.3832229063455501|  0.6099011025578706| 0.2564979479625959|  0.5022728966838492|  1.2202461979233985| 0.3769966373387298| 0.7466583882858947|0.45219466131613584|  3.2218668582598524| 0.24608314142035387|0.028886254236132933| 0.19386392888361814| 0.0663476369900814| 0.03829093110556815| 0.2991210996001837|
    |{K3G8-R5SI91WD-94...| 0.21678105038245493|  0.3091033263397524| 0.6534794977134643|0.19792412311910368| 0.7700737297988488|  0.543097289344043|  0.9777809486369242|  2.177209668935742|   0.522676338770075|  0.2110169547910796| 1.1203122964367505|0.13505809067345723|0.07504822524098467|  1.5244546086504835|  0.4598848422528157|   0.080401799886783|  1.1437670298078815|0.10389969212892508|  0.3573272019710606|0.07696052704840847|
    |{J9N9-W9ZA59WS-07...|0.024633768226013954|   0.939695249723186| 0.2954761391121074| 0.1709718087170629|0.04203477980866014|0.22334852334705715|  0.9777809486369242| 0.6025641988857596|  0.6123732911088609|  0.2316447050705001|0.28752009454781946| 0.5361425755769211| 2.6844654530839542|  0.3103094172143355| 0.13918229100412294|  0.6329905692964606|  0.0426914632120027|0.10389969212892508|  1.7949352503552594| 1.0531860472945325|
    +--------------------+--------------------+--------------------+-------------------+-------------------+-------------------+-------------------+--------------------+-------------------+--------------------+--------------------+-------------------+-------------------+-------------------+--------------------+--------------------+--------------------+--------------------+-------------------+--------------------+-------------------+
    only showing top 20 rows
    



```python
hash_cols = ['scaledHash['+str(i)+']' for i in range(20)]

assembler = VectorAssembler(inputCols=hash_cols, outputCol="scaledFeatures")
id_hash_sub = assembler.transform(id_hash_sub_split).select('id','scaledFeatures')
```


```python
id_hash_sub.show()
```

    +--------------------+--------------------+
    |                  id|      scaledFeatures|
    +--------------------+--------------------+
    |{R0A8-D0KA23YU-26...|[0.05276942646961...|
    |{C4W6-P6ZE51MH-58...|[0.49093414449531...|
    |{Q7V6-N6NE83EZ-03...|[1.00511910184267...|
    |{R1W9-Z1SB61VN-25...|[0.49093414449531...|
    |{T8V6-A0YZ92MM-43...|[0.06474057173913...|
    |{J5S7-D3EN76YN-47...|[6.37502939328121...|
    |{V6X5-A6JS37ZC-78...|[0.12280409315652...|
    |{K9Q3-C6MB20EN-05...|[0.05276942646961...|
    |{E2N0-S3EQ10XZ-59...|[0.05276942646961...|
    |{X2V0-G5QC95QV-97...|[0.37480710166053...|
    |{W8Y0-Q6VB44AY-48...|[0.24670891355624...|
    |{Z3O3-C5WT32II-33...|[0.74892272563408...|
    |{J0V1-Q3VT17LQ-96...|[0.08269728964340...|
    |{I3C6-O0BG43IS-00...|[0.80100067441671...|
    |{Z1D7-Y9ON91MY-18...|[0.19283875984342...|
    |{E8R7-M4NQ76XB-96...|[0.33889366585199...|
    |{M2B5-U6QV73PC-27...|[0.11681852052176...|
    |{X1H9-X8SF98ZC-71...|[1.2972289138598,...|
    |{K3G8-R5SI91WD-94...|[0.21678105038245...|
    |{J9N9-W9ZA59WS-07...|[0.02463376822601...|
    +--------------------+--------------------+
    only showing top 20 rows
    


As you can see, the data we have now is totally same with the one before **Save & Retrieve data** step. Therefore, if your device(assigned Colab resource) is powerful enough, you can skip this step.



## K-Means Modeling

Now we need to train the K-Means model. And determine the most suitable number of clustering categories.


```python
errors = []
results = []

for k in range(2,10):
    kmeansmodel = KMeans().setK(k).setMaxIter(10).setFeaturesCol('scaledFeatures').setPredictionCol('prediction').fit(id_hash_sub)

    print("With K={}".format(k))
    
    kmeans_results = kmeansmodel.transform(id_hash_sub)
    results.append(kmeans_results)
    
    # Evaluate clustering by computing Silhouette score
    evaluator = ClusteringEvaluator()
    evaluator.setFeaturesCol('scaledFeatures').setPredictionCol("prediction")

    silhouette = evaluator.evaluate(kmeans_results)
    errors.append(silhouette)
    print("Silhouette with squared euclidean distance = " + str(silhouette))
    
    print('--'*30)
```

    With K=2
    Silhouette with squared euclidean distance = 0.9054606158887029
    ------------------------------------------------------------
    With K=3
    Silhouette with squared euclidean distance = 0.378622885181141
    ------------------------------------------------------------
    With K=4
    Silhouette with squared euclidean distance = 0.27556189292394295
    ------------------------------------------------------------
    With K=5
    Silhouette with squared euclidean distance = 0.17562579597023814
    ------------------------------------------------------------
    With K=6
    Silhouette with squared euclidean distance = 0.22579471704625662
    ------------------------------------------------------------
    With K=7
    Silhouette with squared euclidean distance = 0.18564839132066216
    ------------------------------------------------------------
    With K=8
    Silhouette with squared euclidean distance = 0.16016978372894128
    ------------------------------------------------------------
    With K=9
    Silhouette with squared euclidean distance = 0.11530445939779198
    ------------------------------------------------------------
    



```python
plt.figure()
k_number = range(2,10)
plt.plot(k_number,errors)
plt.xlabel('Number of K')
plt.ylabel('Silhouette')
plt.title('K - Silhouette')
plt.show()
```


    
![png](https://github.com/waittim/waittim.github.io/raw/master/img/kmeans-anomaly-detection/output_56_0.png)
    


Based on the variation of Silhouette with squared euclidean distance with k in the above figure, according to the elbow principle, we can consider 5 as the most appropriate number of categories that can bring the maximum classification gain with as few categories as possible.


```python
k = 5

kmeansmodel = KMeans().setK(k).setMaxIter(10).setFeaturesCol('scaledFeatures').setPredictionCol('prediction').fit(id_hash_sub)

kmeans_results = kmeansmodel.transform(id_hash_sub)

clusterCenters = kmeansmodel.clusterCenters()
```


```python
kmeans_results.show()
```

    +--------------------+--------------------+----------+
    |                  id|      scaledFeatures|prediction|
    +--------------------+--------------------+----------+
    |{R0A8-D0KA23YU-26...|[0.05276942646961...|         0|
    |{C4W6-P6ZE51MH-58...|[0.49093414449531...|         3|
    |{Q7V6-N6NE83EZ-03...|[1.00511910184267...|         3|
    |{R1W9-Z1SB61VN-25...|[0.49093414449531...|         3|
    |{T8V6-A0YZ92MM-43...|[0.06474057173913...|         3|
    |{J5S7-D3EN76YN-47...|[6.37502939328121...|         1|
    |{V6X5-A6JS37ZC-78...|[0.12280409315652...|         3|
    |{K9Q3-C6MB20EN-05...|[0.05276942646961...|         3|
    |{E2N0-S3EQ10XZ-59...|[0.05276942646961...|         3|
    |{X2V0-G5QC95QV-97...|[0.37480710166053...|         3|
    |{W8Y0-Q6VB44AY-48...|[0.24670891355624...|         3|
    |{Z3O3-C5WT32II-33...|[0.74892272563408...|         3|
    |{J0V1-Q3VT17LQ-96...|[0.08269728964340...|         4|
    |{I3C6-O0BG43IS-00...|[0.80100067441671...|         0|
    |{Z1D7-Y9ON91MY-18...|[0.19283875984342...|         0|
    |{E8R7-M4NQ76XB-96...|[0.33889366585199...|         0|
    |{M2B5-U6QV73PC-27...|[0.11681852052176...|         3|
    |{X1H9-X8SF98ZC-71...|[1.2972289138598,...|         3|
    |{K3G8-R5SI91WD-94...|[0.21678105038245...|         3|
    |{J9N9-W9ZA59WS-07...|[0.02463376822601...|         3|
    +--------------------+--------------------+----------+
    only showing top 20 rows
    


Based on the clustering results obtained from the final model, we can calculate the distance between each data point and its corresponding clustering center.


```python
df_list = []
for row in kmeans_results.collect():
    id = row['id']
    distance = np.linalg.norm(row['scaledFeatures'] - clusterCenters[row['prediction']])
    item = (id, row['scaledFeatures'],row['prediction'], str(distance))
    df_list.append(item)

rdd = sc.parallelize(df_list)
results = spark.createDataFrame(rdd,['id', 'scaledFeatures','prediction', 'distance'])
```


```python
results = results.withColumn('distance', col('distance').cast(DoubleType()))
results = results.orderBy('distance', ascending=False)
results.show()
```

    +--------------------+--------------------+----------+------------------+
    |                  id|      scaledFeatures|prediction|          distance|
    +--------------------+--------------------+----------+------------------+
    |{D4D6-O3CF39OC-21...|[4.76243985455224...|         4|14.589378928119531|
    |{I0N3-A1QK61ZK-75...|[0.78902952914720...|         2|14.079435575560426|
    |{E0Y8-O4HG80PC-27...|[1.84134173438094...|         2|12.950850280350657|
    |{F6X4-S4PK22AK-22...|[0.04678385383486...|         2|11.292647843883817|
    |{E7R0-U5JS84PD-21...|[6.37502939328121...|         1|11.188457733982629|
    |{K0W8-S1FN43PK-21...|[1.05898925555548...|         2|10.659549142628387|
    |{A7O1-Z0XQ34YC-09...|[2.33756997382402...|         2|10.403478656841392|
    |{P8K2-Q4DT05VF-43...|[2.48961045246734...|         2| 9.751568445106406|
    |{N8G3-T8ZQ91IA-15...|[2.95171746103207...|         2| 9.697390803040353|
    |{K1R6-N7JJ70ZO-34...|[0.85307862319934...|         2|  9.66275103075823|
    |{D1I9-G5HX22BQ-41...|[0.02463376822601...|         2| 9.400571104136892|
    |{Q8D1-I2CB87LP-11...|[1.75335034978976...|         2| 9.315246083895909|
    |{A3D9-T0QN45UG-36...|[0.26885899916508...|         2|  8.90252992949646|
    |{K5V2-Q0UP20CL-70...|[0.80698624705147...|         2| 8.835153657957248|
    |{Q4C5-H9UB97OL-06...|[1.35109906757261...|         4|    8.667696292388|
    |{M7U9-M3KZ03UJ-30...|[0.04678385383486...|         4|  8.34628083353364|
    |{X2F5-V0KB19QO-49...|[0.81895739232098...|         2| 8.305671532451226|
    |{V6U4-Z0RL57LU-20...|[0.44484176834743...|         4| 8.159222661283914|
    |{N8D3-U7OT51OU-46...|[0.24072334092148...|         4| 8.139282456385539|
    |{Z2H0-J1RJ27TB-54...|[0.57293995645173...|         0| 7.938517702943826|
    +--------------------+--------------------+----------+------------------+
    only showing top 20 rows
    

The few data points farthest from their corresponding clustering centers are the anomalous emails we are looking for.

In addition, we can also get a better visual of the distance distribution by drawing the image of the KDE probability distribution for each distance. According to this, we can determine the appropriate distance threshold as the criterion for classifying anomalies.


```python
distance = results.select('distance')
kd = KernelDensity()
kd.setSample(distance.rdd.map(lambda x: x[0]))

all_distance = list(np.arange(0,20,0.1))
prob_all_distance = kd.estimate(all_distance)

prob_max = max(prob_all_distance)
prob_min = min(prob_all_distance)

plt.plot(all_distance,prob_all_distance)
plt.xlim(0, 20)
plt.title("KDE Curve")
plt.show()
```


    
![png](https://github.com/waittim/waittim.github.io/raw/master/img/kmeans-anomaly-detection/output_65_0.png)
    




## Example of anomalous email

Based on the above steps, we obtain the list of emails sorted by anomaly degree.

For the obtained list of abnormal emails, we can take out the content of that email and review it.


```python
targetId = results.take(1)[0]['id']
targetId
```




    '{D4D6-O3CF39OC-2139MWTY}'




```python
targetEmail = email.where(col('id') == targetId)
targetEmail.show()
```

    +--------------------+-------------------+-------+-------+--------------------+--------------------+------------------+--------------------+-----+-----------+--------------------+
    |                  id|               date|   user|     pc|                  to|                  cc|               bcc|                from| size|attachments|             content|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+------------------+--------------------+-----+-----------+--------------------+
    |{D4D6-O3CF39OC-21...|10/14/2010 14:07:39|NIH0969|PC-3978|FRR127@earthlink....|Lilah.R.Wagner@be...|SBD478@verizon.net|Naida.I.Hughes@ju...|44147|          1|eruption unknown ...|
    +--------------------+-------------------+-------+-------+--------------------+--------------------+------------------+--------------------+-----+-----------+--------------------+
    



```python
targetEmail.collect()[0]['content']
```




    'eruption unknown 1956 advocated 1860s between common proclaimed blizzards rich 25 lifted blocks used clinton mistake sharp visible continues say'



We can see that the first abnormal email in the list is an email containing non-sense content. This shows that our algorithm is really effective in finding anomalous emails in the huge volume of emails.


So **why do we need to cluster before calculating the distance to the centroid?** In other words, why can't we just use the full email-generated vector as one cluster to find outliers?

Because emails often have multiple types, such as official notifications, work schedules, personal matters, and so on. If all emails are treated as one class, then in the high-dimensional Euclidean space formed by the features, an anomaly cannot be successfully distinguished if it does not belong to any class but is in between multiple classes.

![clusters](https://github.com/waittim/waittim.github.io/raw/master/img/kmeans-anomaly-detection/cluster.png)


**P.S.** In addition to the classical K-Means algorithm used in the previous section, the Bisecting KMeans algorithm described below can also be used as an alternative when we have high requirements on the running time of the algorithm.


```python
from pyspark.ml.clustering import BisectingKMeans
k = 2
bkm = BisectingKMeans().setK(k).setMaxIter(1).setFeaturesCol('scaledFeatures').setPredictionCol('prediction')
model = bkm.fit(id_hash_sub)

results = model.transform(id_hash_sub)

results.show()
```

    +--------------------+--------------------+----------+
    |                  id|      scaledFeatures|prediction|
    +--------------------+--------------------+----------+
    |{R0A8-D0KA23YU-26...|[0.33982462203890...|         0|
    |{C4W6-P6ZE51MH-58...|[1.02642189827338...|         1|
    |{Q7V6-N6NE83EZ-03...|[0.73497742184762...|         0|
    |{R1W9-Z1SB61VN-25...|[0.23160965393239...|         0|
    |{T8V6-A0YZ92MM-43...|[0.10576771195358...|         0|
    |{J5S7-D3EN76YN-47...|[2.31353510123168...|         1|
    |{V6X5-A6JS37ZC-78...|[0.26030343710261...|         0|
    |{K9Q3-C6MB20EN-05...|[0.52305413035815...|         0|
    |{E2N0-S3EQ10XZ-59...|[0.18528889688987...|         0|
    |{X2V0-G5QC95QV-97...|[0.48329353789000...|         0|
    |{W8Y0-Q6VB44AY-48...|[0.00861955314499...|         0|
    |{Z3O3-C5WT32II-33...|[0.38614537908142...|         0|
    |{J0V1-Q3VT17LQ-96...|[2.15899937608265...|         0|
    |{I3C6-O0BG43IS-00...|[0.07707392878336...|         0|
    |{Z1D7-Y9ON91MY-18...|[1.30679956540122...|         0|
    |{E8R7-M4NQ76XB-96...|[1.76384657627393...|         1|
    |{M2B5-U6QV73PC-27...|[0.13446149512380...|         0|
    |{X1H9-X8SF98ZC-71...|[0.26030343710261...|         0|
    |{K3G8-R5SI91WD-94...|[0.26030343710261...|         0|
    |{J9N9-W9ZA59WS-07...|[0.32875781274097...|         0|
    +--------------------+--------------------+----------+
    only showing top 20 rows
    



