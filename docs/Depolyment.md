ETL
===

The ETL is covered by two projects

* [TestLog-ETL](https://github.com/klahnakoski/TestLog-ETL) (using the `etl` branch) - is the workhorse
* [SpotManager](https://github.com/klahnakoski/SpotManager) (using the `beta` branch) - responsible for deploying the above


Production Deployment Steps
---------------------------

In the event we are adding new ETL pipelines, we must make sure the schema is properly established before the army of robots start filling it.  Otherwise we run the risk of creating multiple table instances, all with similar names.

With new ETL comes new tables, new S3 buckets and new Amazon Queues.

1. Stop SpotManager
2. Terminate all ETL instances
3. Create S3 buckets and queues
4. Use development run ETL and confirm use of buckets and queues, and confirm creation of new tables in production
5. Push code to `etl` branch
6. Pull SpotManager changes (on `beta` branch) to its instance
7. Start Spot manager; 
8. Confirm nothing blows up; which will take several minutes as SpotManager negotiates pricing and waits to setup instances 
9. Run the backfill, if desired
10. Increase SpotManager budget, if desired
 

ActiveData Architecture
=======================

Nginx
-----

Nginx is not required, but used in production for three reasons.

- Serve static files (located at `~/ActiveData/active_data/public`)
- Serve SSH (The Flask server can deliver its own SSH, probably slower, if required)
- Forward requests to Gunicorn (The Flask server does not pre-fork processes)

Configuration is `~/ActiveData/resources/config/nginx.conf`

Gunicorn
--------

Gunicorn is used to pre-fork the ActiveData service, so it can serve multiple requests at once, and quicker than Flask.  Since ActiveData has no concept of session, so coordinating instances is not required. 

Configuration is `~/ActiveData/resources/config/gunicorn.conf`


ActiveData Python Program
-------------------------

The ActiveData program is a stateless query translation service.  It was designed to be agnostic about schema changes and migrations.  Upgrading is simple.     

Production Deployment Steps
---------------------------

1. On the `frontend` machine run `git pull origin master`
2. Use supervisor to restart the Gunicorn service


Config and Logs 
---------------

Configuration is `~/ActiveData/resources/config/supervisord.conf`
Logs are `/logs/active_data.log`


ActiveData's ElasticSearch Cluster Cheat Sheet
==============================================

Elasticsearch is powerful magic.  Only the ES developers really know the plethora of ways this magic will turn against you.  There are only two paths forward:  You have already performed the operation before on a multi-terabyte index, and you know the safe incantation. Or, you have not done this before, and you will screw things up.  Feeling lucky?  Please ensure you got the next two days free to fix your mistake: Everything is backed up to S3, so the worst case is you must rebuild the index.

Fixing Cluster
--------------

ES still breaks, sometimes.  All problems encountered so far only require a bounce, but that bounce must be controlled. 
 
 1. Ensure all nodes are reliable - This is a lengthy process, disable the SPOT nodes, or `curl -XPUT -d '{"persistent" : {"cluster.routing.allocation.exclude.zone" : "spot"}}' http://localhost:9200/_cluster/settings`
 2. Disable shard movement `curl -X PUT -d "{\"persistent\": {\"cluster.routing.allocation.enable\": \"none\"}}"  http://localhost:9200/_cluster/settings`
 3. Bounce the nodes as you see fit
 4. Enable shard movement `curl -XPUT -d "{\"persistent\": {\"cluster.routing.allocation.enable\": \"all\"}}"  http://localhost:9200/_cluster/settings`
 5. Re-enable spot nodes: `curl -XPUT -d '{"persistent" : {"cluster.routing.allocation.exclude.zone" : ""}}' http://localhost:9200/_cluster/settings`

Changing Cluster Config
-----------------------

ES will take any opportunity to loose your data if you make any configuration changes.  To prevent this you must ensure no important shards are on the node you are changing:  Either it has no primaries, or all primaries are replicated to another node.  You can ensure this happens by telling ES to exclude the machine you want cleared of shards.  Due to enormous size of the ActiveData indexes, you will probably need to deploy more machines to handle the volume while you make a transition.  

    curl -XPUT -d '{"persistent" : {"cluster.routing.allocation.exclude._ip" : "172.31.0.39"}}' http://localhost:9200/_cluster/settings
 
be sure the transient settings for the same do not interfere with your plans: 

    curl -XPUT -d '{"transient": {"cluster.routing.allocation.exclude._ip" : ""}}' http://localhost:9200/_cluster/settings


 1. Ensure all nodes are reliable - This is a lengthy process, disable the SPOT nodes, or `curl -XPUT -d '{"persistent" : {"cluster.routing.allocation.exclude.zone" : "spot"}}' http://localhost:9200/_cluster/settings`
 2. Move shards off the node `curl -XPUT -d '{"persistent" : {"cluster.routing.allocation.exclude._ip" : "172.31.0.39"}}' http://localhost:9200/_cluster/settings`
 3. Wait while ES moves the shards of the node.  *Jan2016 - took 1 day to move 2 terabytes off a node.* 
 4. Disable shard movement `curl -X PUT -d "{\"persistent\": {\"cluster.routing.allocation.enable\": \"none\"}}"  http://localhost:9200/_cluster/settings`
 5. Stop node, perform changes, start node
 6. Enable shard movement `curl -XPUT -d "{\"persistent\": {\"cluster.routing.allocation.enable\": \"all\"}}"  http://localhost:9200/_cluster/settings`
 7. Allow allocation back to node: `curl -XPUT -d '{"persistent" : {"cluster.routing.allocation.exclude._ip" : ""}}' http://localhost:9200/_cluster/settings`
 8. Re-enable spot nodes: `curl -XPUT -d '{"persistent" : {"cluster.routing.allocation.exclude.zone" : ""}}' http://localhost:9200/_cluster/settings`

Upon which another node can be upgraded