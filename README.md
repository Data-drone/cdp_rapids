# RAPIDS.ai on Cloudera CDP

RAPIDS.ai is a set of Nvidia libraries designed to help Data Scientists and Engineers to leverage the Nvidia GPUs to accelerate all the steps in the Data Science process. For data exploration through to model development. Recently, with the inclusion of GPU processing support in Spark 3, RAPIDS has released a set of plugins designed to accelerate Spark DataFrame operations with the aid of GPU accelerators.  

This repo gives an introduction to setting up RAPID.ai on Cloudera's CDP Platform
Before starting make sure that you have access to the following requirements first

## Requirements

- Ansible if using setup script
- Cloudera CDP Private Cloud Base 7.1.3+
- Cloudera CDS 3
- Centos 7
  - Epel Repo already added and installed
  - kernel 3.10 (deployment script was just tested on this version)
    - to check run `uname -r`
- User should have full administrator and sudo rights in order to follow this guide

## Nvidia Drivers

Firstly in order to be able to support GPUs in the OS layer, we need to install the Nvidia Hardware drivers. The `ansible_cdp_pvc_base` folder contains an ansible playbook to install Nvidia Drivers on Centos 7. It has been tested successfully on AWS and should in theory work for any Centos 7 install. Specific instructions for rhel / centos 8 and ubuntu flavours are not included here. See Nvidia's official notes or the plenty of other articles on the web for that.

Later Nvidia drivers are available but for this example I have selected `440.33.01` drivers with CUDA 10.2. This is because as of writing most data science and ML libraries are still compiled and distributed for CUDA 10.2 with CUDA 11 support being in the minority. Recompiling is also a painstaking process. As an example, compiling a working TensorFlow 2.x on CUDA 11 takes about 40 mins on an AWS `g4dn.16xlarge` instance.

---
**Note** 
a lot of the tutorials for installing Nvidia drivers on the web assume that you are running a local desktop setup. Xorg / Gnome Desktop etc are not needed to get a server working. `nouveau` drivers may also not be active for VGAoutput in a server setup.

---

**Common Issues**


* Incorrect kernel headers or kernel dev packages
  - Be very careful with these. Sometimes the specific version for your linux distro may take a fair bit of searching to find. A default Kernel-devel / Kernel-headers install with `yum` or `apt-get` may also end up updating your kernel to a newer version too after which you will have to ensure that Linux is booting with the correct kernel

* Not explicitly setting your driver and CUDA versions.
  - This can result in a `yum update` or `apt-get update` automatically updating everything to the latest and greatest and breaking all the libraries in your code which have only been compiled for an older CUDA version

## Validating Nvidia Driver Install

Before proceeding to enable gpu on yarn check that your Nvidia Drivers are installed correctly:

run `nvidia-smi` it should list out the GPUs on your system. If nothing shows up then your drivers weren't properly installed. If this is showing nothing or erroring out, there are a few things to check.

![Nvidia-smi screen](img/nvidia-smi.png?raw=true)

run `dmesg | grep nvidia` to see if nvidia drivers are being started on boot. 

![dmseg screen](img/dmsg_nvidia.png?raw=true)

## Setting up RAPIDs on Spark on CDP

Now that we have the drivers installed, we can work on getting Spark 3 on Yarn with GPUs scheduling enabled. Note that CSD 3 should be installed onto your cluster first. We won't cover there here.
See: https://docs.cloudera.com/cdp-private-cloud-base/latest/cds-3/topics/spark-install-spark-3-parcel.html


## Enabling GPU in Yarn

With Spark Enabled, we need to enable GPUs on Yarn. For those who have skimmed the Apache docs for enabling GPUs on Yarn, these instructions will differ as Cloudera Manager will manage changes to configuration files like `resource-types.xml` for you. 

To be able to schedule GPUs, we need gpu isolation and for that we need to leverage linux cgroups. Cgroups are linux controls to help limit the hardware resources that different applications will have access to. This includes cpu, ram, gpu etc and is the only way to truly guarantee the amount of resources allocated to a running application.

In Cloudera Manager, enabling cgroups is a host based setting that can be set From the **Hosts** >> **Hosts Configuration** option in the left side toolbar. Search for cgroups to find the tickbox. 

![enable cgroups](img/cm_cgroups.png?raw=true)

Ticking the big tickbox will enable it for all nodes but it can also be set on a host by host basis through the **Add Host Overrides** option. For more details on the finer details of the cgroup settings please see: https://docs.cloudera.com/cdp-private-cloud-base/7.1.4/managing-clusters/topics/cm-linux-cgroups.html

![cgroups hostgroups](img/host_groups_override.png?raw=true)

With cgroups enabled, we now need to turn on **Cgroup Scheduling** under the Yarn service. Go to the Yarn service Cloudera Manager. Go to `Resource Management` then search for cgroup. Tick the setting `Use CGroups for Resource Management`  

![yarn cgroups](img/yarn_use_cgroups.png?raw=true)

Now we can enable GPU on Yarn through the `Enable GPU Usage` tickbox. You can find that under Yarn >> Configuration then the GPU Management category. 

![yarn gpu](img/yarn_gpu_usage.png?raw=true)

Before you do that, however, it is worth noting that it will by default enable GPUs on **ALL** the nodes which means all your Yarn Node Managers will have to have GPUs. If only some of your nodes have GPUs, read on!

## Yarn Role Groups

As discussed, `Enable GPU Usage` would enable it for the whole cluster. That means that all the Yarn nodes would have to have GPUs and for a large sized cluster that could get very expensive. There are two options if you have an expense limit on the company card.

We can add a small dedicated compute cluster with all GPU nodes on the side. But then users would have to connect to two different clusters depending on if they want to use GPUs or not. Now that can in itself be a good way to filter out users who **really really** need GPUs and can be trusted with them versus users that just want to try GPUs cause it sounds cool but that is a different matter.

Anyway if you wish to pursue that option then follow the instructions here: https://docs.cloudera.com/cdp-private-cloud-base/7.1.4/managing-clusters/topics/cm-add-compute-cluster.html. Once your dedicated compute cluster is up and running you will have to follow the previous steps to enable CGroups, Cgroup Scheduling and then turn on `Enable GPU Usage` in Yarn.

The other option, to stick with one cluster whilst adding a GPU or two is to create **Role Groups**. These allow for different Yarn configs on a host by host basis. So some can have `Enable GPU Usage` turned on whilst others don't.

**Role groups** are configured on the service level. To setup a GPU role group for Yarn, in Cloudera Manager navigate to **Yarn** >> **Instances**. You will see the Role Groups button just above the table listing all the hosts.

<TODO add in screenshot for hostgroups>

Click **Create a role group**. In my case, I entered the Group Name **GPU Nodes** of the Role Type NodeManager and I set the `Copy From field` to **NodeManager Default Group**  

With our role group created, it is time to assign the GPU hosts to the **GPU Nodes** group. With our role group created, We will be able to `Enable GPU Usage` just for the GPU Nodes. For more details on role groups, see: https://docs.cloudera.com/cdp-private-cloud-base/7.1.4/configuring-clusters/topics/cm-role-groups.html. 

<TODO screen shot for role groups>


## Adding Rapids Libraries

In order to be able to leverage Nvidia RAPIDS, yarn and the spark executors have to be able to access the spark RAPIDS libraries.
The required jars are here: https://nvidia.github.io/spark-rapids/docs/get-started/getting-started-on-prem.html
In my sample code, I have created a /opt/rapids folder on all the yarn nodes. Spark also requires a discover GPU resources script too. See: https://spark.apache.org/docs/3.0.1/running-on-yarn.html#resource-allocation-and-configuration-overview I have also included the `getGpusResources.sh` script under `ansible_cdp_pvc_base/files/getGpusResources.sh` in this repo

## Launching spark-shell

We now finally have everything configured, now we can launch gpu enabled Spark-Shell.
Run the following on an edge node. Note that the commands below assume that
`/opt/rapids/cudf-0.15-cuda10-2.jar`, `/opt/rapids/rapids-4-spark_2.12-0.2.0.jar` and `/opt/rapids/getGpusResources.sh` exist. You will need them on each GPU node as well. Also you will need different `.jar` files depending on the CUDA version that you have installed. See the official spark-rapids docs to get the jars: `https://nvidia.github.io/spark-rapids/`

On a kerberised cluster, you will need to `kinit` as a valid kerberos user first for the following to work.


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

- The GPU on G4 instance I had has 16GB VRAM so I upped executor memory to match so that there is some data buffering capability in the executor. Since GPUs are typically RAM constrained it is important to try and minimise the amount of time that you are bottlenecking waiting for data. There is no science around how much to up the executor memory. It just depends on your application and how much IO you have going to and from the GPU. Also non GPU friendly ops will be completed by the executor on CPU so ensure that the executors are beefy enough to handle those. Same with executor cores, I upped it to a recommended level for pulling and pushing data from HDFS.
- `spark.rapids.sql.explain=ALL` helps to highlight which parts of a spark operation can and can't be done on GPU. Example of operations which aren't current currently be done on GPU include: Regex splitting of strings, Datetime logic, some statistical operations. Overall the less non supported operations that you have better the performance.
- `spark.rapids.shims-provider-override=com.nvidia.spark.rapids.shims.spark301.SparkShimServiceProvider` The reality of the Spark ecosystem today is that not all Spark is equal. Each commercial distribution can be slightly different and as such the `shims` layer provides a kind of mapping to make sure that rapids performs seamlessly. Currently, RAPIDs doesn't have a cloudera shim out of the box but Cloudera's distribution of Spark 3 closely mirrors the open source. So we need explicitly specify this mapping of the opensource Spark 3.0.1 Shim to the Cloudera CDS parcel for now. And for the observant, yes, this is version specific and as Spark 3 matures we will need to ensure the current Shim layer is loaded.
-  `spark.rapids.memory.pinnedPool.size` - in order to optimise for GPU performance, Rapids allows for "pinning" memory specific to assist in transfering data to the GPU. GPU that is "pinned" for this purpose won't be available for other operations.

The other flags are pretty self explanatory. It is worth noting that spark rapids sql needs to be explicitly enabled. See the official RAPIDs tuning guide for more details: https://nvidia.github.io/spark-rapids/docs/tuning-guide.html

## Conclusion

Spark RAPIDs is a very promising technology that can help to greatly accelerate big data operations. Good Data Analysis like other intellectual tasks requires getting into the "flow" or so call it "into the zone". Interruptions, like waiting for 10 minutes for your query to run, can be very disruptive to this. RAPIDs is a good acceleration package in other to help reduce these wait times.

In it's current iteration, however, it does still have quite a few limitations. There is no native support for datetime and more complicated statistical operations. GPUs outside of extremely high end models are typically highly RAM bound so users need to be aware of and manage memory well. Managing memory on GPUs, however, is more an art than a science and requires good understanding of GPU technology and access to specialist tools. Support for data formats is also not as extensive as with native Spark. There is a lot of promise however and it will be good to see where this technology goes. Happy Coding!


## Future Topics to look at

- Linkedin Tensorflow on Yarn
- XGB on Spark 3 on Yarn with RAPIDs
- SKLearn on Spark 3 on Yarn with RAPIDs
- Rapids on Dask