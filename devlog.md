## Objective
To create a traffic prediction prototype for the bay area. The prototype should allow for the visualization of bay area traffic at a specific point in the future. For example, one should be able to answer the question: "What is the expected congestion in Petaluma next Wednesday at 3pm?"

## Data
There are two main datasets powering the prediction capabilities of the prototype:
1. The timed and freeflow speed profiles for the Bay Area as generated by [speedracer](https://github.com/mapbox/speedracer/).
2. The road network for the bay area as a set of GeoJSON features.


## Methodology

### Understanding the data
The first step in solving this puzzle was understanding what the data looked like. Not being very familiar with map and traffic data, it did take a little bit of getting used to new ways of inspecting the files. As it turns out, computers get pretty sad when you ask them to open a number of very large text files!
Once I had tamed the files a little, it didn't take long to get an idea of how the data could be used to solve the task.

Here is a (truncated) sample line from the road network file:
```
{
    'geometry': {
        'coordinates': [[-123.639842, 38.915233] ... [-123.612864, 38.884164]], 'type': 'LineString'},
    'properties': {
        'highway': 'secondary',
        'id': 'r62527306!1',
        'name': 'Tenmile Cutoff Road',
        'oneway': 1,
        'refs': ['90177798', '1157191848', '1157191848', '1157191879', '1157191879', '1157191796', '1157191796', '90177799']
    },
    'type': 'Feature'
 }
```
 If you are curious, you can find more information on what the different pieces mean [here](https://github.com/mapbox/graph-normalizer). For now though, the relevant parts are `id` and `refs`.
* `id` allows you to independently identify a road segment in the network.
* `refs` contains a list of the node ids making up that particular road.
You can check out @moganherlocker's visualization of said refs in [this beautiful map](https://api.mapbox.com/styles/v1/morganherlocker/citdyuxos00442hog0xn57xmu.html?fresh=true&title=true&access_token=pk.eyJ1IjoibW9yZ2FuaGVybG9ja2VyIiwiYSI6Ii1zLU4xOWMifQ.FubD68OEerk74AYCLduMZQ#11.99/37.7638/-122.44).

Similarly, here are a few sample lines from the traffic data files:
```
90177798,1157191848,9
1157191848,1157191879,9
1157191879,1157191796,9
1157191796,90177799,9
90177799,1157191778,9
1157191778,1157191821,9
```
Each line in the traffic data file represents the speed between those two nodes at a particular point in time. So! If we can map the pairs of nodes in the speed profiles with the network `refs`... cha-ching!

### Plan A
Cool, even though me and my computer had a few fights to get here, I now had a pretty clear idea of what needed to be done.
1. Find the node ids for each of the road network Features.
2. Find the corresponding speed for the set of nodes making up the road.
3. Store this data for every time bucket.

Following the steps above sounded pretty straight forward, so I went a head and created a number of Python scripts to build the appropriate dictionaries to hold the mapped data. Of course again the size of the data came to hunt me down, and quickly dealing with the data in memory and temporary files became incredibly cumbersome. Time for a new plan!


### Plan B
If you zoom in a little bit in @morganherlocker's map, you may notice a particular road has many nodes (pretty much one for every intersection along said road). What if we could shrink the data down a bit before assigning each road speeds? Furthermore, what if instead of storing data for every 5 minutes, we aggregated and stored it in 1 hour chunks? That would certainly reduce the data to deal with, so I came up with a new plan.
1. Assuming the speed of a road segment is the same at every node, find a first pair of node ids for each road segment. For example, in the Feature above, we would use `90177798,1157191848` as the pair of nodes for the segment.
2. Reduce the speed profile files to only include the pairs of nodes relevant to the road network.
3. Aggregate the speed profile data per hour.
4. Annotate the road segments with the appropriate hourly speeds.


### Master Plan
Although Plan B was definitely an improvement, I still needed to solve a few issues.
1. Dealing with all the data in memory was still cumbersome, even after it had been reduced (There are A LOT of road segments in the Bay Area). I decided to set up Redis and store the aggregated hourly data for a pair of node ids. The data in Redis would be stored as `MED:<node_id_a>,<node_id_b>:D:H` where each line contained the computed speed for day `D` and hour `H`. Storing the data in this format would allow me to query for particular speed values as needed.

2. Simply storing the speeds for a road was not going to tell me whether or not the road was congested. For example, 30mph on a highway probably means a lot of traffic and 30mph on a country road probably means no traffic at all. Solving this was a matter of using the freeflow data for a pair of nodes. If the speed of a segment was slower than the free flow for said segment, then I knew the segment was congested. The freeflow data also allowed me to fill in any gaps in the time profiles. If there was a segment for which I had no hourly data, I could safely assume at least freeflow speeds. I did find a few segments for which there was no freeflow data. For such segments there was not enough information to predict traffic. Freeflow data was also to be stored in Redis under a similar format which was easy to query `FF:<node_id_a>,<node_id_b>:D:H`.

With the above solved, it was also possible to split the process of predicting traffic in two parts.

#### Part 1: Reducing the data
1. Extract a pair of node ids for each road segment.
2. Reduce the speed data profiles to include only relevant node pairs.
3. Reduce the freeflow data profile to include only the relevant node pairs.
4. Use the 5 minute reduced time profiles to compute hourly speeds for each node pair. Store in Redis.
5. Store the reduced freeflow data in Redis.

#### Part 2: Predicting speeds
1. For each road segments, compute the predicted congestion at each hour of the week. The congestion was computed by fetching the road segment speed and freeflow data.
2. Annotate the computed predictions as properties in the road network Features.

```
If we fetch the data for Wednesday at 10 am we can see there was no traffic for that particular segment.

redis> GET MED:90177798,1157191848:3:10
35.0
redis> GET FF:90177798,1157191848:3:10
30.0
```


### Vizualizing the data
The outcome of the steps above was a set of GeoJSON Features with speed data annotated in it. Speed prediction information was stored as `sp<N>` where N is the hour of the week in question. For example, data for Sunday at 12:00am would be stored as `sp0`, and data for Wednesday at 10:00AM would be stored as `sp81` (3 (wednesday) * 24 (hrs in a day) + 9 (hour in question 0 indexed)).
Once the GeoJSON data was ready, I used the magic of [tippecanoe](https://github.com/mapbox/tippecanoe) to turn the data into a tileset.
I also used the magic of @morganherlocker to help me create an interface where the tileset could be visualized for a selected point of time.

Here is what our first go with a small sample set looked like.
<img src="https://github.com/angelanavarro/samples/blob/master/traffic_take_one.gif?raw=true" width="500" align="center">

Sweet! Not bad! Looking around I could spot some pretty expected results like afternoon traffic on the Bay Bridge.
![img](https://github.com/angelanavarro/samples/blob/master/bay_bridge.gif?raw=true)


As well as some interesting ones, like traffic near Land's End on Tuesdays at 1am?!
![img](https://github.com/angelanavarro/samples/blob/master/lands_end.jpg?raw=true)


## Next steps
Asside from addressing all the cut corners, there are a number of issues with this prediction experiment. First, it used only a small set of the available data. A better job at mapping the speed profile with the road network is definitely in order.
It is also easy to think of other variables that may affect traffic patterns
* Holidays, people don't commute on Thankgsgiving Thursday.
* Weather, for example you can expect some road congestion when there are storms in the forecast.
* Time of the year, what if you could predict an influx of visitors to your town?
* Sporting events, known road closures

Anyway! The possibilities are plenty. Fun times!
Big shoutout to Mapbox, Morgan and the telemetry team for a fun mini sprint!

