# Setting up MordecAI on a Windows Machine using Docker Toolbox
### Overview of how MordecAI is working
MordecAI reads in your text and then uses spaCy's named entity recognition to extract placenames from the text. Now, it may not be possible every time that just from your placename you are able to resolve the correct coordinates. There can be ambiguous names/common names, etc. To try and solve this MordecAI uses a Gazetteer. For simplicity, just consider a Gazetteer as a massive data source where place names are linked to their significance. For instance New York could be linked to USA (is a city), lat and lon coordinates and Statue of Liberty (contains monument). Yeah, how it uses this gazetteer is a bit complicated but we don't need to get into that

### Installation Steps
I suggest creating a virtual environment for python and then installing mordecai

`python -m venv virtenv`

`pip install mordecai`

Note: At this step, you may need to ensure you have Microsoft Visual C++ Build tools and also be cautious about the versions of tensorflow and keras at this point. Otherwise you should be good to go!

Since, we need spaCy, let us download the NLP model 

`python -m spacy download en_core_web_lg`

Now, the next step is to get our Gazetteer ready for which we use Elasticsearch. Note that at this point, there is no hard and fast rule to setup Elasticsearch using Docker you may refer to Elasticsearch docs to search for other ways to install the gazetteer.
https://www.elastic.co/guide/en/elasticsearch/reference/master/install-elasticsearch.html

### Let's look at the Docker Method 

First, ensure that you have Docker running.

https://docs.docker.com/docker-for-windows/release-notes/

Now, if you have Windows 10 64-bit: Pro, Enterprise, or Education consider yourself a bit lucky as Docker will work right off the bat for you! Unfortunately I don't have one of these and so the only option for users like me is to use Docker Toolbox which creates a VM and then runs docker.

Once docker is up and running, you again have two options. The first is easier: directly pull the elasticsearch image, load your index and run the container. If this doesn't work try a workaround suggested by creating a dockerfile (yml). You may look up the issues section in the mordecai repo for this

### Use the ElasticSearch Docker Image

Pull elasticsearch using 

`docker pull elasticsearch:5.5.2`

Get the geonames_index from here 

https://s3.amazonaws.com/ahalterman-geo/geonames_index.tar.gz 

Note: You may do it manually or use curl.

Extract the download into a folder called geonames_index. Note the full path for this folder as you'll need it 

To run the container, use the following command

`docker run -d -p 9200:9200 -v $(pwd)/geonames_index/:/usr/share/elasticsearch/data elasticsearch:5.5.2`

Important point (for users using Docker Toolbox) : By default, your VMs localhost is not accessible from outside the VM network. Your PC is outside the VM's network. Thus, if you want to access localhost:9200 you'll actually need to GET {VM's IP (something like 192.168.99.100)}:9200 . You might be able to fix this by playing around with the Port Fowarding settings on your VM Box Manager but I'm too lazy

For my enterpise/pro users this might be it! But for others and those of you who encounter issues here check your installation

### Checking a correct installation
`docker ps -a`

This list should show your elasticsearch container as running. If not, run your container in interactive mode (-it instead of -d in the docker run command)

Next, let's check if your geonames index is loaded correctly. Open the following in your browser

http://{VM's IP (something like 192.168.99.100)}:9200/_cat/indices?v 

The output should be something like the following:
```
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size

yellow open   geonames eWVK3y2ETaufWKEFZmeK2Q   1   1   11741135            0      2.8gb          2.8gb
```

If you face issues, check the following pointers:

1. Case-sensitivity issues of your path may come which can be resolved by using "$PWD" instead of $(pwd). However, I strongly suggest using the full path
2. Your Docker VM and Windows env will share certain directories which can create permission issues. You can check this any time using Oracle VM Virtual Box Manager in the Shared Folders option under Settings. Keep your geonames_index within one of these shared folders
3. You will not be able to access files in your VM directly from Windows (or might be very painful if there exists a way). If you want to check something, SSH into your machine using docker-machine ssh [machine name]
4. You may be facing memory issues in your VM. Solve that by creating a new VM with higher memory limit

    `docker-machine create -d virtualbox --virtualbox-memory 4096 default`

    `docker-machine env default`

    `eval $(docker-machine env default)`

    `docker-machine ssh default`

    `sudo sysctl --w vm.max_map_count=262144`

    default is the name of your VM

5. If like me, you ended up using the hardcoded VM IP, then just ensure that you make the corresponding changes to your mordecai library also. I think you'll need to make a change in the steup_es function in utilities.py in mordecai to change the default connection host setting from localhost to your VM's IP. 
