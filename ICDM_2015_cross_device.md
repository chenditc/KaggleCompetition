###1. ICDM 2015: Drawbridge Cross-Device Connections. $10,000  
####Site: https://www.kaggle.com/c/icdm-2015-drawbridge-cross-device-connections

#####My approach:
######Step 1.
Dump all the properties from the database to files, treat each device_id / cookie_id as a documents. For all categorical peoprty, treat them as a word. For all the property frequency, treat them as word frequency. So we will have following file structure:
```
 - proprties
   - ip
     - device_id_1
     - device_id_2
   - basic
     - device_id_1
     - device_id_2
   - property
     - device_id_1
     - device_id_2
   - property_category
     - device_id_1
     - device_id_2
  ```
For file properties/ip/device_id_1, it will have:
```
ip_123 ip_124 ip_124 ip_124
```
######Step 2.
Since picking one cookie id from 1 million cookies will take too much time if we compare them one by one. I use solr as my filtering tool. I build index over the properties directory, and then use the query <all the ip from a device> to select potential cookie ids. A query will looks like:
```
ip_123 ip_125 ip_127 ip_190 
```
Due to the internal implementation of solr and lucene, the default search already provide me tf-idf weighting and property ranking method. Just use the first cookie id from the search result, I get a accuracy of **42%**.

I also tried to put property and basic info into the query, but they don't provide too much improvement.

######Step 3.
Base on the first 500 cookie id filtered by solr, I'm building a training set and a testing set, which is a list of device, cookie pair. Then I fetch the td-idf weighting of all the ip,basic,property,property_category for each cookie id and device id from solr, use these as my training feature.

To make the feature more robust, I normalize all the feature vector by x = (x - x_min) / ( x_max - x_min), so that some big number won't skew the whole dataset.

######Step 4.
Use the feature vector created in 3, build binary prediction model. Note that the class is highly skewed, 1:499. We will need to adjust the weight of each class to make sure the model does not predict 0 all the time. To do this, there is a class_weight parameter we can set in sklearn, in pylearn2, I duplicate the positive dataset to make them stay in similar size.

The prediction model improve the accuracy to **62%**, which is the best score I got in this competition. The different model does not matter that much, the basic linear model and multilayer perceptron gives me similar performance.

#####First place apparoach:
- We generated a training set from the data as a collection triples of (device_id, cookie_id, ip) for each instance where a device and cookie appeared on the same ip.
- To this collection of triples, we join all the basic information about the device, cookie, and ip address.
- In addition, we generate a few hundred features based on the interaction between device, cookie, and ip. The most influential of these were ranks of each cookie partitioned by device and ordered by a basic attribute, say idxip_anonymous_c3 or ip_anonymous_c2 for instance.
- Cookie properties and any categorical features were one-hot encoded and also joined to the training set. We winnowed the list down substantially through cross-validation and were left with a few hundred in our final model.
- To this we added out-of-sample predictions of a few xgboost models built on cookie_all_basic to predict whether a cookie was matched to any device in the train set.
- To reduce the size of the training set we downsampled the cellular ip addresses quite heavily and also dropped any -1 drawbridge handle cookies.
- On this reduced dataset we built a learning-to-rank model which was a modified version of xgboost's "rank:pairwise" partitioning by device.
- For each device, we took the drawbridge handle of the highest scoring cookie and submitted all the cookie_ids for that drawbridge handle.

original site: https://www.kaggle.com/c/icdm-2015-drawbridge-cross-device-connections/forums/t/16122/1st-place-solution-summary

#####Paper: <to be collected> 

#####Gotchas:
1. When building feature vectors, check if the sample are equivalent between each other. In this competition, each sample is a "relation". And what are trying to predict is the "relative" significance compare to other sample. Thus, a normalization among all interested sample will help. Otherwise, the feature vector does not capture the "rank" information. The normalization is not the best way to do it, since we are interested in the most significant one, the top2 might be really close to each other. A simple ranking number might be better (not tested).
2. Solr is good at generic search, but does not have traing feature, the xgboost's "rank:pairwise" seems to be useful in this case.
