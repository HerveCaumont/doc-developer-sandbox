Node arrange
===============

The application goal is to produce daily binned products so the binning processing step needs to have its inputs well organized so that it aggregates in time and space only the products of a given day. 

In terms of job template, you will define the path to the streaming executable, one parameter: the period (a day) and instruct the framework that only one task has to be run.

As the second job in this workflow, the expression processing step implements a streaming executable that:

* Create an R data frame with all references to the data produced by the node expression
* Split the references by period based in the acquisition start time of the input product into groups of references
* Write the groups to the local filesystem in Tab separated files
* Stage-out the Tab separated files to the distributed file system

The job template includes the path to the streaming executable.

.. code-block:: xml

  <streamingExecutable>/application/arrange/run.R</streamingExecutable>
  
The streaming executable source is available here: `/application/arrange/run.R <https://github.com/Terradue/BEAM-Arithm-tutorial/blob/master/arrange/run.R>`_
  
The job template defines a single parameter:

* The period for the temporal aggregation (daily)

.. code-block:: xml

 <defaultParameters>
  <parameter id="period">day</parameter>
 </defaultParameters>

The job template sets the ciop.job.max.tasks to one instance since the streaming executable has to process all inputs at once 

.. code-block:: xml

 <defaultJobconf>
  <property id="ciop.job.max.tasks">1</property>
 </defaultJobconf>
  	
*Note: the property mapred.task.timeout is not set and uses the default value (10 minutes).*

Here's the job template including the elements described above.

.. code-block:: xml

 <jobTemplate id="arrange">
  <streamingExecutable>/application/arrange/run.R</streamingExecutable>
  <defaultParameters>
   <parameter id="period">day</parameter>
  </defaultParameters>
  <defaultJobconf>
   <property id="ciop.job.max.tasks">1</property>
  </defaultJobconf>
 </jobTemplate> 
