# KaggleCompetition
KaggleCompetition

This page records my experience for few different kaggle competition. It includes my approach and the top performance approach.

1. ICDM 2015: Drawbridge Cross-Device Connections. $10,000  Site: https://www.kaggle.com/c/icdm-2015-drawbridge-cross-device-connections

My approach:
1. Dump all the properties from the database to files, treat each device_id / cookie_id as a documents. For all categorical peoprty, treat them as a word. For all the property frequency, treat them as word frequency. So we will have following file structure:

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

For file properties/ip/device_id_1, it will have:

ip_123 ip_124 ip_124 ip_124

2. Since picking one cookie id from 1 million cookies will take too much time if we compare them one by one. I use solr as my filtering tool. I build index over the properties directory, and then use the query <all the ip from a device> to select potential cookie ids. A query will looks like:

  ip_123 ip_125 ip_127

Due to the internal implementation of solr and lucene, the default search already provide me tf-idf weighting and property ranking method. Just use the first cookie id from the search result, I get a accuracy of 42%.

I also tried to put property and basic info into the query, but they don't provide too much improvement.

3. Base on the first 500 cookie id filtered by solr, I'm building a training set and a testing set, which is a list of device, cookie pair. Then I fetch the td-idf weighting of all the ip,basic,property,property_category for each cookie id and device id from solr, use these as my training feature.

To make the feature more robust, I normalize all the feature vector by x = (x - x_min) / ( x_max - x_min), so that some big number won't skew the whole dataset.

4. Use the feature vector created in 3, build binary prediction model. Note that the class is highly skewed, 1:499. We will need to adjust the weight of each class to make sure the model does not predict 0 all the time. To do this, there is a class_weight parameter we can set in sklearn, in pylearn2, I duplicate the positive dataset to make them stay in similar size.

The prediction model improve the accuracy to 62%, which is the best score I got in this competition. The different model does not matter that much, the basic linear model and multilayer perceptron gives me similar performance.


