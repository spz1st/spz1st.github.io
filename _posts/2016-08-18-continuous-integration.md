---
layout: post
title: "Continuous Integration"
description: "Automatic code test with Jekins"
category: Tool Development
tags: []
---
<link href="/css/ci.css" rel="stylesheet">
{% include JB/setup %}

Automatic code test with Jekins.

### Outline

+ <a href="#overview">Overview</a>
+ <a href="#jenkins">Create Jenkins Project</a>
+ <a href="#conf">Configure Jenkins Project</a>
+ <a href="#webhook">Create Webhook on Github</a>
+ <a href="#scripts">CGI Scripts on a Web Server</a>

<a name="overview"></a>

### Overview

It is critical to test your application at various stages of
its development.
Every once in a while you'll build the application (compile the codes)
and run it on test data to make sure the codes work as expected
and uncover bugs as early as possible.
Ideally you'd like to have a system that would trigger the build process
upon some specific events, such as the merge of a branch to the master branch
from a github repository. This is especially important if the codes are
contributed by multiple developers and
you want to ensure the integrated codes (and the product built from the codes)
in the master branch always in the working condition.

In development of bioinformatics application,
we often build pipelines that integrate various tools
to process biological data for data mining and knowledge discovery,
such as processing sequence data for identification of
genes or mutations associated with some diseases.
In such cases, we need to test the pipelines every time new tools
are incorporated into the pipelines or parameters are changed for some tools.

Here we present a way to automatically run a pipeline
with [Jenkins](https://jenkins.io/)
when changes are pushed or merged to
the master branch of the github repository of the pipeline codes.
This pipeline is coded in a snakefile and executed with snakemake.
Jobs of the pipeline can run on local machine or can be submitted
to a cluster. 
Recipes of the pipeline came from more than one developer,
each writing and testing codes in separate branches
before pushing the codes and merging with the master branch.
But once the codes are merged into the master branch,
the test on the pipeline with the merged codes will be automatically triggered
and the status of the test process will be reported to the developers.

<a name="jenkins"></a>

### Create Jenkins Project

<!--
<figure class="floatright">
<img src="/images/jenkins01.png" alt="Fig1" />
<br>
<br>
<figcaption class="caption">Fig. 1 Create Jenkins Project</figcaption>
<br>
<br>
</figure>
-->

<a href="https://jenkins.io/">Jenkins</a> is an open source automation server
for building, testing and deploying projects. It works in similar ways
as [Travis CI](https://travis-ci.org/) except that the repository
can be from your own [GitHub](http://github.com) server
and the test is done on your own servers.

Once you have a <a href="https://jenkins.io/">Jenkins</a> server set up
and have created an account,
the first step is to create a new project (item).
In this specific case,
we created a pipeline.
You can create other kind of project appropriate for you.


<a name="conf"></a>

### Configure Jenkins Project

<figure class="floatright">
<img src="/images/jenkins02.png" alt="Fig1" />
<br>
<br>
<figcaption class="caption">Fig. 1 Set Endpints</figcaption>
<br>
<br>
</figure>

Once a Jenkins project is created, you need to set its configuration.
Some settings described here are just for your reference and
you should set them to fit your case.


#### Check the following and set their parameters

   * Discard Old Builds
      * **Strategy**: Log Rotation
      * **max # of builds to keep**: 6
   * GitHub Project
      * **Project url**: http://github.research.chop.edu/BiG/grin/


#### Set Job Notifications

Add two notification endpoints (at job start or job completion) for Jenkins to send out a notice to a web CGI (Fig. 1). The CGI script [jenkinsjobs](#scripts)
can be downloaded from the links listed at the end of the blog.
Of course you need to point URL to your own web server.

   * **Format**: JSON
   * **Protocol**: HTTP
   * **Event**: Job Started **or** Job Completed
   * **URL**: http://mitomapd.research.chop.edu/cgi-bin/jenkinsjobs
   * **Timeout**: 30000(ms)
   * **Log**: -1 (send all log messages)

Then check the following options:

   * Prepare an environment for the run
   * Keep Jenkins Environment Variables
   * Keep Jenkins Build Variables
   * Execute concurrent builds if necessary

#### Build Triggers

<figure class="floatright">
<img src="/images/jenkins03.png" alt="Fig1" />
<br>
<br>
<figcaption class="caption">Fig. 2 Set Build Trigger</figcaption>
<br>
<br>
</figure>


Check **Trigger builds remotely (e.g., from scripts)** and set
**Authentication Token**. The token will be included in the URL
used to trigger the build (Fig. 2).

Please note that here you have the option to set the build to
be triggered by a push to a github repository.
But we chose not to use this option because

   1. Somehow we could not get it to work after half dozen tries
   2. More importantly, we wanted to trigger the build
only by a push or merge to the master branch, not just a push to any branches.
The webhook you can set up at [GitHub](http://github.com) doesn't give you
the opiton to specify a specific branch.


#### Pipeline

You can specify how to run your pipeline when the build is triggered.
In our case, we chose to run the pipeline in a **Groovy** script
(selected **Pipeline script** and
check the box **Use Groovy Sandbox**) on a specific node, as shown below.


```
node ('respublica-slave') {
    // in Groovy
    stage 'Build'
    sh """
        # cd to the repository directory and check out the master branch

        git checkout master
        git pull

        # prepare your environments if necessary

        # our pipeline is coded in a snakefile
        snakemake
    """
}
```


<a name="webhook"></a>

### Create Webhook on Github

On the Github repository page, click **Settings** on the menubar,
then click **Hooks & services** in the left panel followed by clicking
**Add webhook** button at upper right.
On the **Webhoooks / Manage webhook** frame, spaecify


   * Payload URL: <font color="blue">http&ratio;//mitomapd.research.chop.edu/cgi-bin/jenkins?user=zhangs3&branch=master&token=grin_test@chop&url=http&ratio;//jenkins-ops-dbhi.research.chop.edu/view/BiG/job/grin_master/build</font>

     * **user**: the user account under which to run the build on Jenkins.
See comments in the script **jenkins** on how to authenticate to the Jenkins server.
     * **branch**: the branch a push of which will trigger the build
on the Jenkins server.
If no branch is specified or the branch is specified as an asterisk (*),
a push to any branch will trigger the build on the Jenkins server.
Here the master branch is specified.
     * **token**: the token from Jenkins project,
as specified in the configuration of the Jenkins project (see Fig. 2).
     * **url**: the URL for remotely triggering the build on Jenkins.
Change the server name and the path to your Jenkins project to fit your case.

   * Content type: application/x-www-form-urlencoded


Note that you need to change the server and the path
to the CGI script **jenkins** in the
URL to fit your case. The script [jenkins](#scripts)
can be downloaded from the links listed below.

<a name="scripts"></a>

### CGI Scripts on a Web Server

   * Script to receive notice from GitHub and trigger the build on Jenkins:
[jenkins](/data/ci/jenkins)
   * Script to receive and send notifications (Slack/Email):
[jenkinsjobs](/data/ci/jenkinsjobs)


