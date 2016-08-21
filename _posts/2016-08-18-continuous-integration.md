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

+ <a href="#sect1">Overview</a>
+ <a href="#sect2">Create Jenkins Project</a>
+ <a href="#sect3">Configure Jenkins Project</a>
+ <a href="#sect4">Create Webhook on Github</a>

<a name="sect1"></a>
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

<a name="sect2"></a>
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


<a name="sect3"></a>
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
      * Strategy: Log Rotation
      * max # of builds to keep: 6
   * GitHub Project
      * Project url: http://github.research.chop.edu/BiG/grin/


#### Set Job Notifications

Add two notification endpoints (at job start or job completion) for Jenkins to send out a notice to a web CGI (Fig. 1). The CGI script [jenkinsjobs](#scripts)
is attached at the end of the blog. Of course you need to point URL to your own web server.

   * Format: JSON
   * Protocol: HTTP
   * Event: Job Started **or** Job Completed
   * URL: http://mitomapd.research.chop.edu/cgi-bin/jenkinsjobs
   * Timeout: 30000(ms)
   * Log: -1 (send all log messages)

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
check the box **Use Groovy Sandbox**), as shown below.


```
node ('respublica-slave') {
    // in Groovy
    def workspace = "/home/svc_cbmicid/.jenkins/workspace/grin_master" // created by Jenkins
    def workdir = "/mnt/isilon/cbmi/variome/zhangs3/projects/data/jenkins" // to run snakemake pipeline
    def repodir = "/mnt/isilon/cbmi/variome/zhangs3/projects/repo/jenkins" // cloned repository
    env.PATH = "${workspace}/miniconda/bin:${env.PATH}"
    env.PATH = "${workspace}/miniconda/envs/grinenv/bin:${env.PATH}"
    
    stage 'Setup' // usually need only do once
    sh """
        # it requires three files: install_miniconda.sh, setup_conda_env.sh and requirements.txt
        cd ${workspace}
        # copy the three files from somewhere the first time run the setup.
        rm -rf miniconda  # in case not the first time
        ./install_miniconda.sh ${workspace}
        ./setup_conda_env.sh ${workspace}
    """
    
    stage 'Cleanup'
    sh """
        # remove old files or sub directories, if necessary
        cd ${workdir}
        #rm -rf lustre/*
        rm -rf lustre/GRCh37/picard
        rm -rf lustre/GRCh38/picard
        rm -rf GRCh37
        rm -rf GRCh38
        rm -rf .snakemake
    """

    stage 'Checkout'  // check out the merged master branch
    sh """
        cd ${repodir}
        git checkout master
        git pull
        # only update conda environments if necessary
        diff -qB requirements.txt requirements.prev || conda install --name grinenv --file requirements.txt && cp requirements.txt requirements.prev
     """
    
    stage 'Build' // main point is to start snakemake in the working directory
    sh """
        cd ${workdir}
        source activate grinenv  # set the environment
        snakemake --latency-wait 10 -j 10 --cluster-config configs/cluster.yaml -c 'qsub -V -l h_vmem={cluster.h_vmem} -l mem_free={cluster.mem_free} -l m_mem_free={cluster.m_mem_free} -pe smp {threads}' comvcfs
    """
}
```


<a name="sect4"></a>
### Create Webhook on Github
On the Github repository page, click **Settings** on the menubar,
then click **Hooks & services** in the left panel followed by clicking
**Add webhook** button at upper right.
On the **Webhoooks / Manage webhook** frame, spaecify

   * Payload URL: <font color="blue">http&ratio;//mitomapd.research.chop.edu/cgi-bin/jenkins?user=zhangs3&branch=master&token=grin_test@chop&url=http&ratio;//jenkins-ops-dbhi.research.chop.edu/view/BiG/job/grin_master/build</font>

     * user: the user account under which to run the build on Jenkins
     * branch: the branch of which being updated to trigger the build on Jenkins
     * token: the token from Jenkins project
     * url: the URL for remotely trigger the build on Jenkins

   * Content type: application/x-www-form-urlencoded


Note that the user specified must be added by accessing the web
page at http://mitomapd.research.chop.edu/cgi-bin/jenkins.

<a name="scripts"></a>
### CGIs on a Web Server

#### CGI to receive notice from GitHub and trigger build on Jenkins

The file **jenkins**.

#### CGI to receive and send notifications (Slack/Email)

The file **jenkinsjobs**.


