Run an Experiment on FrameworkController
========================================

NNI supports running experiment using `FrameworkController <https://github.com/Microsoft/frameworkcontroller>`__\ , called frameworkcontroller mode. FrameworkController is built to orchestrate all kinds of applications on Kubernetes, you don't need to install Kubeflow for specific deep learning framework like tf-operator or pytorch-operator. Now you can use FrameworkController as the training service to run NNI experiment.

Prerequisite for on-premises Kubernetes Service
-----------------------------------------------


#. A **Kubernetes** cluster using Kubernetes 1.8 or later. Follow this `guideline <https://kubernetes.io/docs/setup/>`__ to set up Kubernetes
#. Prepare a **kubeconfig** file, which will be used by NNI to interact with your Kubernetes API server. By default, NNI manager will use $(HOME)/.kube/config as kubeconfig file's path. You can also specify other kubeconfig files by setting the**KUBECONFIG** environment variable. Refer this `guideline <https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig>`__ to learn more about kubeconfig.
#. If your NNI trial job needs GPU resource, you should follow this `guideline <https://github.com/NVIDIA/k8s-device-plugin>`__ to configure **Nvidia device plugin for Kubernetes**.
#. Prepare a **NFS server** and export a general purpose mount (we recommend to map your NFS server path in ``root_squash option``\ , otherwise permission issue may raise when NNI copies files to NFS. Refer this `page <https://linux.die.net/man/5/exports>`__ to learn what root_squash option is), or **Azure File Storage**.
#. Install **NFS client** on the machine where you install NNI and run nnictl to create experiment. Run this command to install NFSv4 client:

.. code-block:: bash

    apt-get install nfs-common

7. Install **NNI**\ , follow the install guide `here <../Tutorial/QuickStart.rst>`__.

Prerequisite for Azure Kubernetes Service
-----------------------------------------


#. NNI support Kubeflow based on Azure Kubernetes Service, follow the `guideline <https://azure.microsoft.com/en-us/services/kubernetes-service/>`__ to set up Azure Kubernetes Service.
#. Install `Azure CLI <https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest>`__ and **kubectl**.  Use ``az login`` to set azure account, and connect kubectl client to AKS, refer this `guideline <https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#connect-to-the-cluster>`__.
#. Follow the `guideline <https://docs.microsoft.com/en-us/azure/storage/common/storage-quickstart-create-account?tabs=portal>`__ to create azure file storage account. If you use Azure Kubernetes Service, NNI need Azure Storage Service to store code files and the output files.
#. To access Azure storage service, NNI need the access key of the storage account, and NNI uses `Azure Key Vault <https://azure.microsoft.com/en-us/services/key-vault/>`__ Service to protect your private key. Set up Azure Key Vault Service, add a secret to Key Vault to store the access key of Azure storage account. Follow this `guideline <https://docs.microsoft.com/en-us/azure/key-vault/quick-create-cli>`__ to store the access key.


Prerequisite for PVC storage mode
-----------------------------------------
In order to use persistent volume claims instead of NFS or Azure storage, related storage must
be created manually, in the namespace your trials will run later. This restriction is due to the
fact, that persistent volume claims are hard to recycle and thus can quickly mess with a cluster's
storage management. Persistent volume claims can be created by e.g. using kubectl. Please refer
to the official Kubernetes documentation for `further information <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims>`__.


Setup FrameworkController
-------------------------

Follow the `guideline <https://github.com/Microsoft/frameworkcontroller/tree/master/example/run>`__ to set up FrameworkController in the Kubernetes cluster, NNI supports FrameworkController by the stateful set mode. If your cluster enforces authorization, you need to create a service account with granted permission for FrameworkController, and then pass the name of the FrameworkController service account to the NNI Experiment Config. `refer <https://github.com/Microsoft/frameworkcontroller/tree/master/example/run#run-by-kubernetes-statefulset>`__.  
If the k8s cluster enforces Authorization, you also need to create a ServiceAccount with granted permission for FrameworkController, `refer <https://github.com/microsoft/frameworkcontroller/tree/master/example/run#prerequisite>`__.  

Design
------

Please refer the design of `Kubeflow training service <KubeflowMode.rst>`__\ , FrameworkController training service pipeline is similar.

Example
-------

The FrameworkController config file format is:

.. code-block:: yaml

   authorName: default
   experimentName: example_mnist
   trialConcurrency: 1
   maxExecDuration: 10h
   maxTrialNum: 100
   #choice: local, remote, pai, kubeflow, frameworkcontroller
   trainingServicePlatform: frameworkcontroller
   searchSpacePath: ~/nni/examples/trials/mnist-tfv1/search_space.json
   #choice: true, false
   useAnnotation: false
   tuner:
     #choice: TPE, Random, Anneal, Evolution
     builtinTunerName: TPE
     classArgs:
       #choice: maximize, minimize
       optimize_mode: maximize
   assessor:
     builtinAssessorName: Medianstop
     classArgs:
       optimize_mode: maximize
   trial:
     codeDir: ~/nni/examples/trials/mnist-tfv1
     taskRoles:
       - name: worker
         taskNum: 1
         command: python3 mnist.py
         gpuNum: 1
         cpuNum: 1
         memoryMB: 8192
         image: msranni/nni:latest
         frameworkAttemptCompletionPolicy:
           minFailedTaskCount: 1
           minSucceededTaskCount: 1
   frameworkcontrollerConfig:
     storage: nfs
     nfs:
       server: {your_nfs_server}
       path: {your_nfs_server_exported_path}

If you use Azure Kubernetes Service, you should  set ``frameworkcontrollerConfig`` in your config YAML file as follows:

.. code-block:: yaml

   frameworkcontrollerConfig:
     storage: azureStorage
     serviceAccountName: {your_frameworkcontroller_service_account_name}
     keyVault:
       vaultName: {your_vault_name}
       name: {your_secert_name}
     azureStorage:
       accountName: {your_storage_account_name}
       azureShare: {your_azure_share_name}

If you set `ServiceAccount <https://github.com/microsoft/frameworkcontroller/tree/master/example/run#prerequisite>`__ in your k8s, please set ``serviceAccountName`` in your config file: 
For example:

.. code-block:: yaml

   frameworkcontrollerConfig:
     serviceAccountName: frameworkcontroller

Note: You should explicitly set ``trainingServicePlatform: frameworkcontroller`` in NNI config YAML file if you want to start experiment in frameworkcontrollerConfig mode.

The trial's config format for NNI frameworkcontroller mode is a simple version of FrameworkController's official config, you could refer the `Tensorflow example of FrameworkController <https://github.com/microsoft/frameworkcontroller/blob/master/example/framework/scenario/tensorflow/ps/cpu/tensorflowdistributedtrainingwithcpu.yaml>`__ for deep understanding.

Trial configuration in frameworkcontroller mode have the following configuration keys:


* taskRoles: you could set multiple task roles in config file, and each task role is a basic unit to process in Kubernetes cluster.

  * name: the name of task role specified, like "worker", "ps", "master".
  * taskNum: the replica number of the task role.
  * command: the users' command to be used in the container.
  * gpuNum: the number of gpu device used in container.
  * cpuNum: the number of cpu device used in container.
  * memoryMB: the memory limitaion to be specified in container.
  * image: the docker image used to create pod and run the program.
  * frameworkAttemptCompletionPolicy: the policy to run framework, please refer the `user-manual <https://github.com/Microsoft/frameworkcontroller/blob/master/doc/user-manual.md#frameworkattemptcompletionpolicy>`__ to get the specific information. Users could use the policy to control the pod, for example, if ps does not stop, only worker stops, The completion policy could helps stop ps.

NNI also offers the possibility to include a customized frameworkcontroller template similar
to the aforementioned tensorflow example. A valid configuration the may look like:

.. code-block:: yaml

    experimentName: example_mnist_pytorch
    trialConcurrency: 1
    maxExecDuration: 1h
    maxTrialNum: 2
    logLevel: trace
    trainingServicePlatform: frameworkcontroller
    searchSpacePath: search_space.json
    tuner:
      builtinTunerName: TPE
      classArgs:
        optimize_mode: maximize
    assessor:
      builtinAssessorName: Medianstop
      classArgs:
        optimize_mode: maximize
    trial:
      codeDir: .
    frameworkcontrollerConfig:
      configPath: fc_template.yml
      storage: pvc
      namespace: twin-pipelines
      pvc:
        path: /mnt/data

Note that in this example a persistent volume claim has been used, that must be created manually in the specified namespace beforehand. Stick to the mnist-pytorch example (:githublink: `<examples/trials/mnist-pytorch>`__) for a more detailed config (:githublink: `<examples/trials/mnist-pytorch/config_frameworkcontroller_custom.yml>`__) and frameworkcontroller template (:githublink: `<examples/trials/fc_template.yml>`__).

How to run example
------------------

After you prepare a config file, you could run your experiment by nnictl. The way to start an experiment on FrameworkController is similar to Kubeflow, please refer the `document <KubeflowMode.rst>`__ for more information.

version check
-------------

NNI support version check feature in since version 0.6, `refer <PaiMode.rst>`__
