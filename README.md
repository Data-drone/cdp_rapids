# RAPIDS.ai on Cloudera CDP

This repo gives an introduction to setting up Nvidia RAPID.ai on Cloudera's CDP Platform
Before starting make sure that you have access to the following requirements first

## Requirements

- Ansible
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

- Enable Yarn GPU tickbox under services Yarn
- Enable Cgroup scheduling under ?Yarn?
- Enable Cgroups under host settings
  - Need to examine this a bit more and understand how we tune cgroups at least superficially

## Yarn Role Groups


- need to look at role groups: https://docs.cloudera.com/documentation/enterprise/5-6-x/topics/cm_mc_role_groups.html
  - Otherwise all nodes need to have gpu cause there will be findgpu script issues otherwise

- or can use compute clusters with a separate gpu compute group

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

## Some GPU considerations in no particular order

- Which are GPU ops and which arent
- rapids memory pinning model and association with Spark Executors

## Future to look at

- TonY?
- XGB on Spark 3 on Yarn with rapids
- SKLearn on Spark 3 on Yarn with rapids