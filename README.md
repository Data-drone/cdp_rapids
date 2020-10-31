# RAPIDS.ai on Cloudera CDP

This repo gives an introduction to setting up Nvidia RAPID.ai on Cloudera's CDP Platform
Before starting make sure that you have access to the following requirements first

## Requirements

- Ansible if using setup script
- Cloudera CDP Private Cloud Base 7.1+
- Cloudera CDS 3.x
- Centos 7
  - Epel Repo already added and installed
  - kernel 3.10
    - to check run `uname -r`

## Nvidia Drivers

The `ansible_cdp_pvc_base` folder contains an example Nvidia Driver install ansible playbook for Centos 7.
It has been tested successfully on AWS and should in theory work for any Centos 7 install.

Later Nvidia drivers are available but this example is pinned to `440.33.01` drivers with CUDA 10.2. This is because as of writing most data science and ML libraries are still compiled and distributed for CUDA 10.2 with CUDA 11 support being in the minority. Recompiling can also take a while. For example compiling a working TensorFlow 2.x on CUDA 11 with most bells and whistles took about 40 mins on an AWS `g4dn.16xlarge` instance.

Specific instructions for rhel / centos 8 and ubuntu flavours are not included here. See Nvidia's official notes or the plenty of other articles on the web for that. Note a lot of the tutorials on the web assume that you are running a local desktop setup. Xorg / Gnome Desktop etc are not needed to get a server working. `nouveau` drivers may also not be active for VGA output on a server.

Note that CSD 3 should be installed onto your cluster first
See: https://docs.cloudera.com/cdp-private-cloud-base/latest/cds-3/topics/spark-install-spark-3-parcel.html

I will include a quick list of things that can go wrong. These should generalise to other linux flavours as well.

* Incorrect kernel headers or kernel dev packages
  - Be very careful with these. Sometimes the specific version for your linux distro may take a fair bit of searching to find. A default Kernel-devel / Kernel-headers install with `yum` or `apt-get` may also end up updating your kernel to a newer version too after which you will have to ensure that Linux is booting with the correct kernel

* Not setting your driver and CUDA versions.
  - This can result in a `yum update` or `apt-get update` automatically updating everything to the latest and greatest and breaking all the libraries in your code which have only been compiled for an older CUDA version

## Validating Nvidia Drivers

Before proceeding to enable gpu on yarn check that your Nvidia Drivers are installed correctly:

run `nvidia-smi` it should list out the GPUs on your system. If nothing shows up then your drivers weren't properly installed. If this is showing nothing or erroring out, there are a few things to check.

run `dmesg | grep nvidia` to see if nvidia drivers are being started on boot. 

## Enabling GPU in Yarn

There are a few steps that need to be taken in order to enable GPUs on Yarn. These differ from the default instructions from Apache Docs as Cloudera Manager helps to automate some of these steps and will automatically change some of the configuration flags. 

In order to achieve gpu isolation, we need to leverage linux cgroups. Cgroups are linux controls to limit help limit what hardware resources different applications have access to. This includes cpu, ram, gpu etc and is the only way to truly guarantee the amount of resources allocated to a running application. Other resource allocation methods will make a best effort to ensure that applications do not use too many resources but cannot guarantee this. So in order to make sure that we can properly allocate GPUs to Yarn containers we need to turn this on.

In Cloudera Manager, this is a host based setting that can be set From the **Hosts** >> **Hosts Configuration** option in the left side toolbar. Search for cgroups to find the tickbox. Ticking the big tickbox will enable it for all nodes but it can also be set on a host by host basis through the **Add Host Overrides** option. For more details on the finer details of the cgroup settings please see: https://docs.cloudera.com/cdp-private-cloud-base/7.1.4/managing-clusters/topics/cm-linux-cgroups.html

With cgroups enabled, it we now need to turn on Cgroup scheduling under the Yarn service. Go to the Yarn service Cloudera Manager. Go to `Resource Management` then search for cgroup. Tick the setting `Use CGroups for Resource Management`  

Now we can enable GPU on Yarn through the `Enable GPU Usage` tickbox


## Yarn Role Groups

By default, `Enable GPU Usage` would enable it for the whole cluster. That means that all the Yarn nodes would have to have GPUs for a large sized cluster that could be very expensive. There are two options that we can take. We can either add a small compute cluster with the GPU nodes on the side but then users would have to connect to two different clusters depending on if they want to use GPUs or not. The other option is to create **Role Groups** in order to separately configure the GPU and non GPU nodes but keep them in the same cluster.

First lets have a look at Role Groups. Role groups are configured on the service level. So go to **Yarn** >> **Instances** the Role Groups button is just above the table with the list of hosts. Click **Create a role group**. In my case, I entered the Group Name **GPU Nodes** of the Role Type NodeManager and I set the Copy From field to the **NodeManager Default Group**  

Now assign the GPU hosts to the **GPU Nodes** We can now click `Enable GPU Usage` just for the GPU Nodes. For more details on role groups, see: https://docs.cloudera.com/cdp-private-cloud-base/7.1.4/configuring-clusters/topics/cm-role-groups.html. 

Another option is to have a dedicated GPU subcluster. To see how to do this, follow the instructions here: https://docs.cloudera.com/cdp-private-cloud-base/7.1.4/managing-clusters/topics/cm-add-compute-cluster.html then once your compute cluster is up and running you can follow the previous steps to enable CGroups and turn on `Enable GPU Usage`.

## Adding Rapids Libraries

In order to be able to leverage Nvidia RAPIDS, yarn and the spark executors have to be able to access the spark RAPIDS libraries.
The required jars are here: https://nvidia.github.io/spark-rapids/docs/get-started/getting-started-on-prem.html
In my sample code, I have created a /opt/rapids folder on all the yarn nodes  

## Example launching spark-shell

Running on an edge node:

```{bash}

export SPARK_RAPIDS_DIR=/opt/rapids
export SPARK_CUDF_JAR=${SPARK_RAPIDS_DIR}/cudf-0.15-cuda10-2.jar
export SPARK_RAPIDS_PLUGIN_JAR=${SPARK_RAPIDS_DIR}/rapids-4-spark_2.12-0.2.0.jar

spark3-shell \
  --master yarn \
  --deploy-mode client \
  --driver-cores 6 \
  --driver-memory 15G \
  --executor-cores 8 \
  --conf spark.executor.memory=15G \
  --conf spark.rapids.sql.concurrentGpuTasks=4 \
  --conf spark.executor.resource.gpu.amount=1 \
  --conf spark.rapids.sql.enabled=true \
  --conf spark.rapids.sql.explain=ALL \
  --conf spark.rapids.memory.pinnedPool.size=2G \
  --conf spark.kryo.registrator=com.nvidia.spark.rapids.GpuKryoRegistrator \
  --conf spark.plugins=com.nvidia.spark.SQLPlugin \
  --conf spark.rapids.shims-provider-override=com.nvidia.spark.rapids.shims.spark301.SparkShimServiceProvider \
  --conf spark.executor.resource.gpu.discoveryScript=${SPARK_RAPIDS_DIR}/getGpusResources.sh \
  --jars  ${SPARK_CUDF_JAR},${SPARK_RAPIDS_PLUGIN_JAR}

```

Breaking down the Spark Submit command:

- The GPU on G4 instance I had has 16GB VRAM so I upped executor memory to match so that there is some buffering capability in the executor. Since GPUs are typically RAM constrained it is important to try and minimise the amount of time that you are bottlenecking waiting for data. There is no science around how much to up the executor memory. It just depends on your application and how IO you have going to and from the GPU. Also non GPU friendly ops will be completed by the executor on CPU. Same with executor cores, I upped it to a recommended level for pulling and pushing data from HDFS.
- `spark.rapids.sql.explain=ALL` helps to highlight which parts of a spark operation can and can't be done on GPU. Example of operations which aren't current currently be done on GPU include: Regex splitting of strings, Datetime logic, some statistical operations. Overall the non supported operations that you have better the performance
- `spark.rapids.shims-provider-override=com.nvidia.spark.rapids.shims.spark301.SparkShimServiceProvider` - The reality of today is that not all Spark is equal. Each commercial distribution can be slightly different as such the `shims` layer provides a kind of mapping to make sure that rapids performs seamlessly. Currently, RAPIDs doesn't have a cloudera shim out of the box but Cloudera's distribution of Spark 3 closely mirrors the open source. So we need explicitly specify this mapping of Opensource to the Cloudera CDS parcel for now.
-  `spark.rapids.memory.pinnedPool.size` - in order to optimise for GPU performance, Rapids allows for "pinning" memory specific to assist in transfering data to the GPU. GPU that is "pinned" for this purpose won't be available for other operations.

The other flags are pretty self explanatory. It is worth noting that spark rapids sql needs to be explicitly enabled. See the official RAPIDs tuning guide for more details: https://nvidia.github.io/spark-rapids/docs/tuning-guide.html

## Some thoughts

Spark RAPIDs is a very promising technology that can help to greatly accelerate big data operations. Good Data Analysis like other intellectual tasks requires getting into the "flow", getting "into the zone". Interruptions can be very disruptive to this. Due to the volumes of data involved these days, even simple queries and operations can take minutes to run which interrupts "flow". RAPIDs is a good acceleration package in other to help reduce this.

It does, still currently have quite a few limitations however, there is no native support for datetime and some more complicated statistical operations. GPUs outside of extremely high end models are typically highly RAM bound. The data input format support is also not as extensive as with Spark on it's own. There is a lot of promise however and it will be good to see where this technology goes. 

## Future Topics to look at

- TonY?
- XGB on Spark 3 on Yarn with rapids
- SKLearn on Spark 3 on Yarn with rapids