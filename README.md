
# Vertical Pod Autoscaling for Pods Launched by Airflow’s KubernetesPodOperator

Author: Ray Berger (GitHub: [RayBB](https://github.com/RayBB), Twitter: [@RayScript](https://twitter.com/rayscript))

Date: December 9, 2020

## Goal

Have the VPA (Vertical Pod Autoscaler) assign resource requests based on previous instances of a task when an Airflow task is deployed to Kubernetes.

## Problem

-   The VPA cannot look at the history of pods based on a [label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) anymore.
	-   If we could VPA version 0.4.0 the problem would be solved since you can assign labels when launching from Airflow. We cannot use the old version because it doesn’t support recent versions of Kubernetes.
-   The pods [must](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/MIGRATE.md#L11-L14) be specified via a [CrossVersionObjectReference](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#crossversionobjectreference-v1-autoscaling) (aka targetRef) such as a Deployment or custom controller.
-   The targetRef only needs to provide a [ScaleStatus](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#scalestatus-v1-autoscaling).
	-   A ScaleStatus has two fields:
		-   replicas (integer) - number of observed instances of the scaled object.
		-   selector (string) - a label selector (such as the one set by Airflow)
-   To my knowledge, there is no native Kubernetes controller that allows you to add instances of pods to it, as would be needed when pods are deployed from the KubernetesPodOperator.
-   As such, we cannot use the VPA for Airflow tasks.
    

## Airflow

Airflow is a workflow orchestration tool. It allows you to create DAGs ([directed acyclic graphs](http://v)) which describe tasks (units of work) and the relationships between them. Airflow runs DAGs on any cron schedule.

An example DAG may have these tasks:
1.  Query the database for all user activity over the last 24 hours and write it to a CSV.
2.  Filter the CSV for bad records, format it, and write to a new CSV.
3.  Run the data from the previous CSV through a model to predict a song each user may like and write that data to a CSV.
4.  Send an email to each user with the song that was predicted for them.

For each task in the DAG above Airflow would create a pod with the label corresponding to each task’s name, and any label specified It creates the pod, waits for it to complete, removes the pod, and moves on to the next task.

If the DAG above runs every 12 hours, the day-to-day resources (cpu and memory) each pod uses should stay relatively stable but grow slowly with increasing user activity. I would like each pod to have their [resource requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-requests-and-limits-of-pod-and-container) maintained by the VPA.

With a simple case like this, it may seem like no issue to check the pod usage and manually update the resource request for each task. However, imagine having hundreds of different DAGs running in a cluster each day. It would be a burden to maintain and quite wasteful to have pods possibly requesting unneeded resources.

#### Why can’t Airflow just track and scale each task?

That would be great. It probably is not a feature since Airflow was not built with Kubernetes in mind. Traditionally tasks were all executed on static machines and just had to fit or get bigger machines.

Additionally, the Vertical Pod Autoscaler was built for Kubernetes, does a great job, and is under active development. We should utilize it as much as possible.

## Potential Solutions

### 1. Create a custom controller

Implement a controller that has properties to satisfy the VPA. In our case, all it needs to do is have a ScaleState attribute that provides the number of pods and a selector (what we wanted all along). This could potentially be done with a tool like Metacontroller.

Pros
-   Implementation is probably trivial for someone familiar with controllers
-   It could be up and running fast
-   Does not require convincing others to change

Cons
-   It would be yet another dependency for your cluster
-   It will add an extra step when installing VPAs
-   In its simplest form, you would need to deploy one controller for each type of task.
	-   If deployed manually, it could be a lot to maintain.
	-   If deployed by airflow when dags run:
		1.  Requires adding code to each dag or adding some Airflow abstraction.
			-   Could automatically be deployed as some part of CI/CD
-   In a moderately complex form, you could have a controller that also deploys a VPA.
	-   Just like above but this controller would also create a VPA that scales this controller.
-   In a complex form, you could have a controller for your custom controllers
	-   Deploy an “Airflow task controller controller” once that would:
		1.  Watch for any pod with a label such as “airflow_task_seeking_vpa:true”
		2.  Check if a controller exists matching the pod’s label of task_name. If not, deploy the controller.
		4.  Check if a VPA exists for that controller. If not, deploy a VPA for the controller.

### 2. Reimplement label selectors in the Vertical Pod Autoscaler

The VPA implementation [supported](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/MIGRATE.md) selectors before version 0.5.0. However, they decided to remove selectors as an option because they are easily misconfigured and they wanted to align with the HPA spec.

Pros
-   Probably the easiest existing codebase to modify
-   The code for using selector exists in git history
-   There may be more use cases for this feature that others would benefit from
-   Would make using VPAs for pods from Airflow trivial
-   If the maintainers of the VPA don’t like this feature a fork could be made
-   But this would require a lot of maintenance and be more confusing

Cons
-   Maintainers removed this feature once before and may feel strongly about the decision
-   Becomes easier to misconfigure the VPAs
-   Time intensive to propose, implement, merge, release these changes

### 3. Have Airflow launch a controller that the VPA supports

There is a tool in **beta** called [KubernetesJobOperator](https://github.com/LamaAni/KubernetesJobOperator) that can launch Airflow tasks as Kubernetes [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) or any [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

A lot of the problems described above are due to the KubernetesPodOperator deploying “naked” pods instead of a higher level resource (such as a [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) or replica set) that the VPA supports.

As far as I know, there is no native Kubernetes resource that would be a good fit for Airflow tasks.

In theory, a Kubernetes Job would be great since it can be launched with one or more pods and [is](https://github.com/kubernetes/autoscaler/blob/99936f3f3971c91b30def913c1c807c42c4b87a7/vertical-pod-autoscaler/pkg/target/fetcher.go#L182) supported by the VPA. One airflow task launches one Job, waits for the job to finish and we are good to go.

Unfortunately, it is common for Airflow to run multiple instances of a task in parallel. For example, running a [backfill](https://airflow.apache.org/docs/apache-airflow/stable/tutorial.html#backfill) where a DAG is run one time for each of the previous five days.

To that end, I am unclear if the KubernetesJobOperator alone would provide a solution. If you have an idea please do share.

### 4. Custom pod admission webhook

Credits to Ace Eldeib for [suggesting](https://kubernetes.slack.com/archives/C09R1LV8S/p1607566151430300?thread_ts=1607475076.397700&cid=C09R1LV8S) this:

I think for your use case, running VPA with recommender and then using your own custom webhook might work best. The issue I see is that if you killed a pod in e.g. a deployment, that pod will spin back up (potentially with new resource requests/limits if the deployment was modified). The operator you mentioned doesn’t seem to have any way to know about resource changes desired for those pods. But you could write a custom pod admission webhook that basically reads the VPA recommendations from existing pods and updates them for incoming pod creation on specific selectors

Admittedly, I am not totally clear how this might be implemented but my guess is:
1.  Use a Job/Deployment/etc for each individual task.
2.  Use a custom pod admission webhook to:
	-  Attach a VPA to each task
	-  If this is the first instance of that task it will work as expected
	-  If this is not the first instance of that task:
		1.  Ask the VPA for recommendations for the first instance
		2.  Apply those recommendations to the second instance (which will have to have a different name and as such the VPA wouldn’t automatically find the history)

Pros
-   Installation would likely be straightforward
-   Does not require any changes to the VPA codebase
-   Seems to be a lightweight solution
Cons
-   Feels hacky
	-   The VPA would only be able to use the history of the first instance of a task
-   Need to use the [KubernetesJobOperator](https://github.com/LamaAni/KubernetesJobOperator) (or similar)
-   Would require the creation of a custom pod admission webhook
	-   Difficulty unknown
-   May need another process to clean up VPA for tasks not seen in a long time?
    

## Resources

Related Tools

-   [Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
-   [Apache Airflow](https://airflow.apache.org/)
-   [KubernetesPodOperator](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/contrib/operators/kubernetes_pod_operator/)
-   [Metacontroller](https://metacontroller.github.io/metacontroller/)
-   [KubernetesJobOperator](https://github.com/LamaAni/KubernetesJobOperator)

Conversations
-   Is it feasible to use the Vertical Pod Autoscaler with Airflow on a task level? On SO [here](https://stackoverflow.com/questions/65187194/is-it-feasible-to-use-the-vertical-pod-autoscaler-with-airflow-on-a-task-level)
-   Asking about using Metacontroller in Kubernetes Slack [here](https://kubernetes.slack.com/archives/CA0SUPUDP/p1607488872129000)
-   Asking if a VPA can use a service in Kubernetes Slack [here](https://kubernetes.slack.com/archives/C09R1LV8S/p1607475076397700)
-   Is the VPA designed to work with batch jobs? Kubernetes Slack [here](https://kubernetes.slack.com/archives/C09R1LV8S/p1607503512419200)
-   Asking about launching pod autoscaling in Airflow slack [here](https://apache-airflow.slack.com/archives/CCV3FV9KL/p1607499103207000?thread_ts=1604040031.459000&cid=CCV3FV9KL)
-   Asking about using a Service for autoscaler on Autoscaler github [here](https://github.com/kubernetes/autoscaler/issues/3752)
	-   This is closed, as it is not possible to use a Service.