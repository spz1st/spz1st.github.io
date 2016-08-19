---
layout: post
title: "Continuous Integration"
description: "Automatic test codes with Jekins"
category: Tool Development
tags: []
---
{% include JB/setup %}

Automatic build with Jenkins

### Configure Jenkins Project

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

