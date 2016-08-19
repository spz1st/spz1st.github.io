---
layout: post
title: "Continuous Integration"
description: "Automatic code test with Jekins"
category: Tool Development
tags: []
---
{% include JB/setup %}

Automatic code test with Jekins.

+ <a href="#sect1">Overview</a>
+ <a href="#sect2">Configure Jenkins Project</a>
+ <a href="#sect3">Create Webhook on Github</a>

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

In bioinformatics, we often build pipelines integrated with various tools
to process various biological data for data mining and knowledge discovery,
such as processing sequence data for identification of
genes or mutations associated with some diseases.
In such cases, we need to test the pipelines every time new tools
are incorporated into the pipeline or parameters are changed for some tools.

+ What is continuous integration? How does it fit in with unit testing and integration testing
+ How does CI for bioinformatics pipelines different from CI for compiled software development or webdev, how are they similar? What are some special things about CI for pipelines in terms of Snakemake, Conda, running on the cluster?
+ Intro to Jenkins (feel free to throw it under the bus). How does it differ from TravisCI?
+ Intro to Github/Github enterprise hooks. What is not working out of the box?
+ Slack integration, what is involved
+ Problems encountered
+ Source code

<a name="sect2"></a>
### Configure Jenkins Project

![jenkins configure](/images/jenkins01.png)

First create a new project (item) as Pipeline.

#### Check the following and set their parameters

   * Discard Old Builds
      * Strategy: Log Rotation
      * max # of builds to keep: 6
   * GitHub Project
      * Project url: http://github.research.chop.edu/BiG/grin/

#### Set Job Notifications
Set two notification endpoints (at job start or job completion) for Jenkins to send out a notice to a web CGI (jenkinsjobs).

   * Format: JSON
   * Protocol: HTTP
   * Event: Job Started or Job Completed
   * URL: http://mitomapd.research.chop.edu/cgi-bin/jenkinsjobs
   * Timeout: 30000(ms)
   * Log: -1 (send all log messages)

#### check the following options

   * Prepare an environment for the run
   * Keep Jenkins Environment Variables
   * Keep Jenkins Build Variables
   * Execute concurrent builds if necessary

#### Build Triggers
   * check
      * Trigger builds remotely (e.g., from scripts)
      * Authentication Token: grin_test@chop (supplied by Jenkins)
#### Pipeline
   * check the following
      * Use Groovy Sandbox
   * Definition: Pipeline script
   * Script:
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

<a name="sect3"></a>
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

### CGIs on a Web Server

#### CGI to receive notice from GitHub and trigger build on Jenkins

The file **jenkins**.

#### CGI to receive and send notifications (Slack/Email)

The file **jenkinsjobs**.

