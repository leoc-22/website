---
title: "UNSW Redback Racing's Journey to the Cloud"
description: ""
pubDate: 2025-08-02T00:00:00.000Z
---

This blog is my recollection of Data Acquisition's many efforts into deploying our software systems to the cloud and some important decisions made during this journey. I thought it was quite an interesting and rewarding experience for myself and wanted to share my point of view. An edited version of this blog is also on [Redback Racing's website](https://www.redbackracing.com/blogs/data-acquisition-s-journey-to-the-cloud).

## What We Achieved in the End of 2022

### Journey to IBM

Redback Racing reached a sponsorship with IBM in 2021 and Data Acquisition (DAQ) designed and deployed our Telemetry System onto IBM Cloud Kubernetes Service by the end of 2021. A detailed blog is [here](https://www.redbackracing.com/blogs/redback-cloud). The general architecture on Kubernetes was all DAQ's applications were deployed into one Kubernetes node of a cluster. A load balancer sat on top of the node to accept inbound traffic and an ingress controller was in-place to handle service discovery. 

By the end of 2022, this was what DAQ system looked like in terms of Kubernetes:

<figure>

[![k8s architecture diagram](/k8s_architecture_diagram.png)](/k8s_architecture_diagram.png)

</figure>

Here is another view of the more detailed system architecture:

<figure>

[![k8s architecture diagram 2](/k8s_architecture_diagram_2.png)](/k8s_architecture_diagram_2.png)

</figure>

You can see this was a straight forward design. We only used a single worker node and were not even starting to up-scale it. I like the Kubernetes design because it is extremely easy to manage a large set of applications because each of them is deployed into a Kubernetes pod and you get service discovery inside a worker node for free. They have the exact same deployment procedure and this greatly benefits us in terms of streamlining our backend micro services' CI-CD overhead and improves efficiency. This is a good example of "undifferentiated heavy lifting" where you have a group of experts managing infrastructure, providing necessary abstractions, standardise and automate, which frees your other developers to work on core product features that users can’t get anywhere else.

### How We Did DevOps

We had a centralised repo called `infrastructure` which stored all configurations and handled deployment to IBM Cloud written in Terraform. A repo called `deployments`, which had manifests for applications we built and things we self-hosted such as Bitbucket pipeline and Bitwarden, was synced with Argo CD. Internally for each application repo, we had CD step which was triggered in the end of a pipeline to automatically create a pull request in `deployments` repo for deployment. Relevant people including the author of the original PR and the cloud operation engineers would get tagged as reviewers. After the deployment PR was merged, ArgoCD synced the head of the main branch and deployed to Kubernetes.

### Limitation with IBM

#### Lack of Support to University Students

After the sponsorship started, we were allocated a cloud credit budget, but there was no guidance on how to approach architecting a scalable software system in the cloud. Fortunately, a few cracked students in DAQ managed to implement a working solution using IBM Cloud Kubernetes Service. Telemetry System was running on the cloud from the end of 2021.

#### Cost Limitation Affects System Performance

One of our core services is the streaming service. Its primary responsibility is to live stream car metadata and store them in Redis. Streaming service + Redis demand a significant amount of compute to be able to run on the cloud for a long time reliably. Limited by the budget, streaming service often crashed or hung during intensive runtime scenarios. 

Additionally, the contract with IBM was going to expire in May 2023 so we immediately needed to think about either trying to renew with IBM (and ask for a higher budget) or looking for alternative options. Around the end of 2022, we started reaching out to IBM and also looking for alternatives.

## Start of 2023, Partnering with AWS, Kubernetes vs Serverless Design.

While discussing the partnership with AWS, DAQ had internal discussions about the approach of migrating from IBM Cloud to AWS. One obvious solution is to stick with Kubernetes. Kubernetes itself is platform-agnostic and there should be minimal vendor lock-in using it. Lift-and-shifting applications from IBM Cloud Kubernetes Service to AWS Elastic Kubernetes Service (EKS) should be easy. Our Kubernetes architecture will remain pretty much the same and the migration process should be quick. The other opinion was that this might be a good opportunity to do refactoring to make the entire suite "serverless". We thought that if we could make the majority of our applications into lambda functions, Fargate ECS or other serverless applications, we could achieve "scale to zero", which greatly reduces cost. The ideal scenario is Telemetry System is able to scale up only when we are using them whereas in a Kubernetes world everything is running 24/7 by default. In addition, the maintenance overhead for Kubernetes has started to take a toll on us in 2022 with key members leaving the team. Kubernetes needs frequent version updates and we really need one person or a small team to constantly maintain it. In Sep 2022, I broke the entire Kubernetes cluster because of a version upgrade and ended up spending two weeks trying to bring everything back up. We desperately need "experts" in Kubernetes that can make each upgrade as seamless as possible, and it is difficult because university students do not get chances to learn and work on a Kubernetes system in their courses. AWS Serverless services, on the other hand, eliminates this concern because the underlying infrastructure maintenance is covered by AWS. At that time, I thought this was also a good opportunity to learn AWS services such as ECS, lambda functions etc. instead of sticking to Kubernetes. Therefore, we eventually made the call to go with the serverless option.

## Migration Begins

I won't go into every detail of the migration, but there are several components that I think is worth mentioning. They are the AWS Identity and Access Management (IAM) setup and the use of AWS Application Load Balancer (ALB) vs API Gateway V2. 

### AWS Identity and Access Management

One important aspect of getting started on AWS is to configure IAM properly since AWS IAM is super granular and specific on which role can take actions on which AWS resource using which policy. I was working on an IAM task in a company of 100+ developers at the time so I thought we could use that as an example and adopt some of their methodologies.

The IAM setup for the human role access is to have two layers of IAM. The top IAM layer connects to the IdP so when the user tries to log into the company’s AWS web console with SSO, the page is directed to the IdP. In Entra (what was used in that company), each employee belongs to different Groups such as “ReadOnly“, “Developer“, “Admin“, etc. The top IAM layer uses those group names as the IAM names and this layer has zero permission to access anything inside AWS. The second (bottom) layer of IAM defines the actual IAM policies to different services and resources such as `s3:List*`, `s3:Get*` and `s3:PutObject` actions. The bottom layer connects to the top layer by using `assume_role_policy`. It is a special policy associated with a role that controls which principals (users, other roles, AWS services, etc) can "assume" the role. Assuming a role means generating temporary credentials to act with the privileges granted by the access policies associated with that role. As a result, the employee logs into web console with zero access then click the switch role bottom to assume a specific role to perform actions on specific AWS resources. It is also important to not have any bottom layer role being assumed. To prevent this, in each role of the second IAM layer, there is an `aws_iam_policy_document` that defines the `principals.identifiers` which targets the corresponding top layer IAM roles.

No human IAM role has been given the action to create resources or services because provisioning is done by Infrastructure as Code (IaC) tools. Therefore, another set of IAMs are created by using IaC just for CI/CD. Those IAM roles are defined based on either the department, project team, product, or different parts of the system.

Relevant info on this decision: [AWS Identity and Access Management (IAM) Best Practices - Amazon Web Services](https://aws.amazon.com/iam/resources/best-practices/)

#### Favouring centralised access management

Each `DAQ <your_type>` IAM roles in this IAM repo is a Terraform module which uses a [custom module](https://github.com/leoc-22/tf-module-iam-role). The top three will be used for the pipelines to assume. These are rough definitions of how we will delegate roles to our apps and infrastructures. The last `DAQ human IAM roles` module is used for human users to assume.


<figure>

[![iam design diagram](/redback_IAM.png)](/redback_IAM.png)

</figure>

### Elastic Container Service, Application Load Balancer and API Gateway V2

One of the initial milestones of the migration was to move everything across to AWS Elastic Container Service (ECS) and one immediate realisation was there was no service discovery between ECS containers whereas  it is built in for Kubernetes. Fortunately, AWS Cloud Map solved this problem.

While setting up the Application Load Balancer for each services, I found the process to be extremely complicated and error-prone. Case in point, you need `aws_lb_target_group`, `aws_lb_listener_rule` for each ECS service and a `aws_lb_listener` to distribute the traffic. The configs ended up scattering in different Terraform files and it was also difficult to get a holistic view of the ingress rules in AWS console. As a result, I could never achieve the simplicity of something like a ingress controller in Kubernetes which uses only one yaml file to describe reverse proxy rules. Even if I could somehow tolerate this complicated setup, ALB is not a feasible choice for DAQ because of the cost. The load balancer by itself is [at least $18/month](https://redbackracing63.atlassian.net/wiki/spaces/~5ce275996673c80fe175fd2d/pages/3497721857/DAQ+Journey+to+the+Cloud#:~:text=by%20itself%20is-,at%20least%20%2418/month,-and%20the%20whole) and the whole point of using lambdas and ECS is to achieve "scale from zero". 

I found a nice solution where we could forego ALB entirely - and still get TLS termination and balancing load over multiple containers. You just need to use API Gateway HTTP API’s [support for private integrations](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-private.html). 

Instead of $18+ per month, it is billed by the invocations and it is [$1.29 for the first 300 million per month](https://aws.amazon.com/api-gateway/pricing/#:~:text=First%20300%20million-,$1.29,-300+%20million). For the traffic that the Telemetry System receives, this is a huge saving. The configuration is also simpler. API Gateway V2 works nicely with Lambda functions and other AWS serverless offerings so this is also a huge benefit.

The final architecture looks like this (October 2023 version):

<figure>

[![AWS Telemetry System Octorber 2023 version](/AWS_Telemetry_System_October_2023.png)](/AWS_Telemetry_System_October_2023.png)

</figure>

## Fully Functional Telemetry System in the Cloud and the Formula SAE-A competition

In September 2023, we successfully live-streamed on AWS, confirming that the migration was complete.

<figure>

[![kobe](/screenshot1.png)](/screenshot1.png)

</figure>

<figure>

[![kobe](/trackday.png)](/trackday.png)

<figcaption>Telemetry System in action at trackdays</figcaption>

</figure>

During the 2023 Formula SAE-A competition, the Telemetry System played an important role in the endurance race and we won [2nd overall and 1st in Australia](https://www.saea.com.au/results-2023). 

<figure>

[![final day at 2023 comp](/final_day_2023_comp.jpg)](/final_day_2023_comp.jpg)

<figcaption>The 2023 Redback Racing Team at Competition</figcaption>

</figure>

AWS wrote a blog about this achievement: [UNSW students build an all-electric race car with AWS](https://aws.amazon.com/blogs/publicsector/unsw-students-build-an-all-electric-race-car-with-aws/).

## Future Days

<figure>

[![future days](/future_days.jpeg)](/future_days.jpeg)

</figure>

The ultimate goal of migrating to AWS in 2023 is to achieve a stable, performant and extensible cloud infrastructure foundation so that future DAQ can continuously iterate and build on top of it. I am looking forward to the future of Redback Racing and I have no doubt that DAQ will keep helping the rest of the team to achieve new heights in FSAE.

