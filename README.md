[title]: - "condor_annex on OSG Connect"


Run the quickstart tutorial:

	$ tutorial annex
	$ cd tutorial-annex

Tutorial jobs
-------------

Inside the tutorial directory you will find a sample executable:

	#!/bin/bash
	# short.sh: a short discovery job
	printf "Start time: "; /bin/date
	printf "Job is running on node: "; /bin/hostname
	printf "Job running as user: "; /usr/bin/id
	printf "Job is running in directory: "; /bin/pwd
	echo
	echo "Working hard..."
	sleep 20
	echo "Science complete!"


### HTCondor submit file

So far, so good! Let's look at a the submit file `ec2-job.submit`

    # The UNIVERSE defines an execution environment. You will almost always use VANILLA.
    Universe = vanilla
    
    # These are good base requirements for your jobs on OSG. It is specific on OS and
    # OS version, core cound and memory, and wants to use the software modules. 
    # This is the default recommended OSG requirements:
    #Requirements = OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64" &&  HAS_MODULES == True
    # To make sure we are only running on EC2, use regex matching on the Machine attribute
    Requirements = regexp("ec2.internal", Machine) 
    request_cpus = 1
    request_memory = 1 GB
    
    # EXECUTABLE is the program your job will run It's often useful
    # to create a shell script to "wrap" your actual work.
    Executable = short.sh
    Arguments = 
    
    # ERROR and OUTPUT are the error and output channels from your job
    # that HTCondor returns from the remote host.
    Error = job.$(Cluster).$(Process).error
    Output = job.$(Cluster).$(Process).output
    
    # The LOG file is where HTCondor places information about your
    # job's status, success, and resource consumption.
    Log = job.log
    
    # Send the job to Held state on failure. 
    on_exit_hold = (ExitBySignal == True) || (ExitCode != 0)
    
    # Periodically retry the jobs every 1 hour, up to a maximum of 5 retries.
    periodic_release =  (NumJobStarts < 5) && ((CurrentTime - EnteredCurrentStatus) > 60*60)
    
    # QUEUE is the "start button" - it launches any jobs that have been
    # specified thus far.
    Queue 1


### Submit the job 

Submit the job using `condor_submit`:

	$ condor_submit ec2-job.submit
	Submitting job(s). 
	1 job(s) submitted to cluster 823.

### Check the job status

The `condor_q` command tells the status of currently running jobs.
Generally you will want to limit it to your own jobs: 

	$ condor_q netid
	-- Submitter: login01.osgconnect.net : <128.135.158.173:43606> : login01.osgconnect.net
	 ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD
	 823.0   netid           8/21 09:46   0+00:00:06 R  0   0.0  short.sh
	1 jobs; 0 completed, 0 removed, 0 idle, 1 running, 0 held, 0 suspended

You can also get status on a specific job cluster: 

	$ condor_q 823
	-- Submitter: login01.osgconnect.net : <128.135.158.173:43606> : login01.osgconnect.net
	 ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD
	 823.0   netid           8/21 09:46   0+00:00:10 R  0   0.0  short.sh
	1 jobs; 0 completed, 0 removed, 0 idle, 1 running, 0 held, 0 suspended

Note the `ST` (state) column. Your job will be in the I state (idle) if
it hasn't started yet. If it's currently scheduled and running, it will
have state `R` (running). If it has completed already, it will not appear
in `condor_q`. 


### Check the job output

Once your job has finished, you can look at the files that HTCondor has
returned to the working directory. If everything was successful, it
should have returned:

* a log file from HTCondor for the job cluster: jog.log
* an output file for each job's output: job.output
* an error file for each job's errors: job.error

Read the output file. It should be something like this: 

	$ cat job.output
	Start time: Wed Aug 21 09:46:38 CDT 2013
	Job is running on node: ip-12.12.12.12
	Job running as user: uid=58704(osg) gid=58704(osg) groups=58704(osg)
	Job is running in directory: /var/lib/condor/execute/dir_2120
	Sleeping for 10 seconds...
	Et voila!

The `ip-NNN.NNN.NNN.NNN` indicates that the job ran in the EC2 instance.


### Jobs across OSG and Amazon EC2

We will now edit the `ec2-job.submit` file, so that the requirements 
expression allows the job to go to either OSG or EC2. The new requirements
line should read:

    Requirements = regexp("ec2.internal", Machine) || (OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64")

Let's also run a set of jobs. Change the queue line to read:

    Queue 20

The submit the job again.


### Where did jobs run? 

When we start submitting many simultaneous jobs into the queue, it might
be worth looking at where they run. To get that information, we'll use the
`condor_history` command from quickstart tutorial.Change the job id (942)
to the job id provided by the `condor_submit` command:

	[netid@login01 log]$ condor_history -format '%s\n' LastRemoteHost 942 | cut -d@ -f2 | cut -d. -f2,3 | distribution --height=100
	Val          |Ct (Pct)     Histogram
	ec2.internal |456 (46.77%) +++++++++++++++++++++++++++++++++++++++++++++++++++++
	uchicago.edu |422 (43.28%) +++++++++++++++++++++++++++++++++++++++++++++++++
	local        |28 (2.87%)   ++++
	t2.ucsd      |23 (2.36%)   +++
	phys.uconn   |12 (1.23%)   ++
	tusker.hcc   |10 (1.03%)   ++
	...

The distribution program reduces a list of hostnames to a set of
hostnames with no duplication (much like `sort | uniq -c`), but
additionally plots a distribution histogram on your terminal
window. This is nice for seeing how Condor selected your execution
endpoints.

