
# An Amphours Postmortem

I helped build the backend for Amphours, the first battery benchmark tailored towards real world use. 

A few days ago my partner [Ming](https://github.com/ming08108?tab=overview&from=2016-08-01&to=2016-08-31&utf8=%E2%9C%93) finally decided to not renew the .xyz domain (Send all hate mail to him). 


But before I say goodbye, I'd like to showcase the data we collected over the course of ~1 year before we all switched to iOS. 




# Grabbing the Data


Amphours was built with 2 collections in MongoDB - one for unique devices, and one for statistics read-outs. 

First, lets grab some data from both MongoDB instances. 


`$ mongoexport -h secret_DB.db_host.com:231337 -d amphours -c statistics -u username -p pass -o statistics.json --jsonArray`

=> 63 records

`$ mongoexport -h secret_DB.db_host.com:231337 -d amphours -c devices -u username -p pass -o devices.json --jsonArray`

=> 54881 records

Let's load it up. 


```python
import json 
from pprint import pprint

with open('devices.json') as devices_raw: 
    devices = json.load(devices_raw)

with open('statistics.json') as statistics_raw: 
    statistics = json.load(statistics_raw)
    
# Print out the last few
print "Devices"
pprint(devices[:3]) 

print "Statistics"
pprint(statistics[:3])

```

    Devices
    [{u'_id': {u'$oid': u'57649f238b09bce65c43c7bb'},
      u'count': 38,
      u'friendlyname': u'Asus Nexus 7 (2013)',
      u'name': u'asus Nexus 7',
      u'rank': 1,
      u'sot': {u'$numberLong': u'41976548'},
      u'standby': {u'$numberLong': u'154386092'},
      u'uniquedevices': 2},
     {u'_id': {u'$oid': u'57649f238b09bce65c43c7bc'},
      u'count': 75,
      u'friendlyname': u'Motorola Moto X 2013 (Europe)',
      u'name': u'motorola XT1052',
      u'rank': 2,
      u'sot': {u'$numberLong': u'36933090'},
      u'standby': {u'$numberLong': u'63887738'},
      u'uniquedevices': 2},
     {u'_id': {u'$oid': u'57649f238b09bce65c43c7bd'},
      u'count': 661,
      u'friendlyname': u'Motorola Moto X Style (Pure Edition)',
      u'name': u'motorola XT1575',
      u'rank': 3,
      u'sot': {u'$numberLong': u'32610548'},
      u'standby': {u'$numberLong': u'39241215'},
      u'uniquedevices': 10}]
    Statistics
    [{u'_id': {u'$oid': u'55ebccdfb94be457b7f38c2c'},
      u'date': {u'$numberLong': u'1441516699'},
      u'device': u'LGE LG-H811',
      u'friendlyname': u'LGE LG-H811',
      u'sot': {u'$numberLong': u'284368'},
      u'total': {u'$numberLong': u'3718542'},
      u'used': 3,
      u'uuid': u'bc39765e-005d-4ecc-beb0-8eeb46e5edd5',
      u'version': 22},
     {u'_id': {u'$oid': u'55ec6235b94be457b7f38c2d'},
      u'date': {u'$numberLong': u'1441554929'},
      u'device': u'LGE LG-H811',
      u'friendlyname': u'LGE LG-H811',
      u'sot': {u'$numberLong': u'1832653'},
      u'total': {u'$numberLong': u'38208845'},
      u'used': 37,
      u'uuid': u'bc39765e-005d-4ecc-beb0-8eeb46e5edd5',
      u'version': 22},
     {u'_id': {u'$oid': u'55ec77c7b94be457b7f38c2e'},
      u'date': {u'$numberLong': u'1441560451'},
      u'device': u'LGE LG-H811',
      u'friendlyname': u'LGE LG-H811',
      u'sot': {u'$numberLong': u'197945'},
      u'total': {u'$numberLong': u'1963997'},
      u'used': 4,
      u'uuid': u'bc39765e-005d-4ecc-beb0-8eeb46e5edd5',
      u'version': 22}]
    

# Devices

From the start, our strategy for Amphours was to start small on several different Android phone specific subreddits.  





```python
print len(devices)
```

    63
    

We got 63 different types! Yowza. Didn't expect that. 

(Though in fairness, most major manufacturers come up as different units because of region locking and carriers)

Here's the distribution of users with these devices in an inappropriately large bar graph:


```python
%matplotlib inline

from pylab import *

# Get # of uniques per device 

sorted_devices = sorted(devices, key=lambda device: device['uniquedevices'])

names = [device['friendlyname'] for device in sorted_devices]
uniques = [device['uniquedevices'] for device in sorted_devices]

figure(figsize=(len(names), 100))
pos = arange(len(names))+  0.5    # the bar centers on the y axis

barh(pos, uniques, align='center', height=1)
yticks(pos, tuple(names))
xlabel('Number of Devices')
title('Devices by Type')
grid(True)


show()
```


![png](output_7_0.png)



Our efforts on /r/OnePlus were far and away the most successful, though more with the PlusOne than the PlusTwo. 


The most surprising result is the Droid Turbo - we don't even know anybody with a Droid Turbo. 


# Statistics 

Next lets dive into the statistics. 

Who has the most battery life?

First, lets find out how much "fluff" data we have. We can do that by removing all of the 0-used elements, which yield completely absurd predictions.  


```python
zero_values = filter(lambda stat: stat['used'] == 0, statistics)

print "Number of stats", len(statistics)
print "Number of zeros", len(zero_values)
print "Number of non-zeroes", len(statistics) - len(zero_values)
```

    Number of stats 54881
    Number of zeros 41175
    Number of non-zeroes 13706
    

Lets filter these further to get where all have at least 20% battery used, which is the threshold used in the ranking process. 



```python
THRESHOLD = 20

thresholded_values = filter(lambda stat: stat['used'] > THRESHOLD, statistics)

print "Tresholded values", len(thresholded_values)
```

    Tresholded values 7211
    

Now we have 7211 statistics to work with. 


```python
import datetime

# Lets find the total first 
sorted_statistics_total = sorted(thresholded_values, key=lambda stat: int(stat['total']['$numberLong']))
sorted_statistics_sot = sorted(thresholded_values, key=lambda stat: int(stat['sot']['$numberLong']))

total_time = int(sorted_statistics_total[-1]['total']['$numberLong'])
sot_time = int(sorted_statistics_sot[-1]['sot']['$numberLong'])

print str(datetime.timedelta(milliseconds=total_time))
print str(datetime.timedelta(milliseconds=sot_time))
```

    40 days, 0:30:03.049000
    40 days, 0:11:22.334000
    

There are some clear outliers here - the highest value lasts *16718 days*. 

However, if we slide a few spaces down, we get more reasonable high bar. 






```python
device_150 = sorted_statistics_total[-150]

total_time = int(device_150['total']['$numberLong'])
sot_time = int(device_150['sot']['$numberLong'])

print device_150['friendlyname']
print str(datetime.timedelta(milliseconds=total_time))
print str(datetime.timedelta(milliseconds=sot_time))
```

    HTC One (M8)
    1 day, 2:50:17.299000
    1:55:00.848000
    

Yeah, that's better and aligns with my own use of the N5.  

Now lets apply some outlier detection to this set of data. 



```python
%matplotlib inline

import numpy as np
import matplotlib.pyplot as plt
import pandas as pd 
import matplotlib.font_manager
from sklearn import svm

xx, yy = np.meshgrid(np.linspace(-5, 5, 500), np.linspace(-5, 5, 500))

THRESHOLD = 20
thresholded_values = filter(lambda stat: stat['used'] > THRESHOLD, statistics)

# Shuffle the thresholded data 
np.random.shuffle(thresholded_values)

# Get training data 
X = [[
        int(stat['total']['$numberLong']), 
        int(stat['sot']['$numberLong']), 
        int(stat['used'])
     ] for stat in thresholded_values]
Y = [stat['uuid'] for stat in thresholded_values]


# fit the model
clf = svm.OneClassSVM(kernel="linear")
clf.fit(X)
scores = clf.decision_function(X)


results = []

idx = 0
outliers = 0 

for score in scores:
    if score < 0: 
        outliers += 1
        
        outlier = thresholded_values[idx]
        total_time = int(outlier['total']['$numberLong'])
        sot_time = int(outlier['sot']['$numberLong'])
        
        total = str(datetime.timedelta(milliseconds=total_time))
        sot = str(datetime.timedelta(milliseconds=sot_time))
        results.append([outlier['uuid'], sot, total,  outlier['used']]); 
    idx += 1

result_df = pd.DataFrame(results)
result_df.columns = ['UUID', 'SOT', 'Total', 'Used']

print "Outliers: %d,\t %f of dataset" % (outliers, 1.0 * outliers/len(thresholded_values))

result_df.head()
```

    Outliers: 3605,	 0.499931 of dataset
    




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>UUID</th>
      <th>SOT</th>
      <th>Total</th>
      <th>Used</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>20e6a383-5193-4740-8e7a-d63d6536f31c</td>
      <td>1:47:07.515000</td>
      <td>7:09:00.779000</td>
      <td>50</td>
    </tr>
    <tr>
      <th>1</th>
      <td>234958b7-ad05-4f31-b288-4c4621708d29</td>
      <td>0:59:43.030000</td>
      <td>5:13:24.243000</td>
      <td>36</td>
    </tr>
    <tr>
      <th>2</th>
      <td>954a3516-d524-457d-9997-6bf60dd88f47</td>
      <td>1:37:44.197000</td>
      <td>8:23:02.389000</td>
      <td>69</td>
    </tr>
    <tr>
      <th>3</th>
      <td>ea50222a-5c3c-4dc8-bd4a-8e53d85e3da7</td>
      <td>0:37:18.002000</td>
      <td>4:50:35.411000</td>
      <td>28</td>
    </tr>
    <tr>
      <th>4</th>
      <td>c1fea4fe-b82e-4a65-92e7-4205a152cb70</td>
      <td>1:42:29.282000</td>
      <td>7:13:45.010000</td>
      <td>49</td>
    </tr>
  </tbody>
</table>
</div>




```python

```
