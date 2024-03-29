Application integration and testing
===================================

At this stage of the tutorial, you have defined the job templates. It is now time to organize these in a workflow.

Writing the Application workflow
********************************

In the context of the framework, the workflows are Directed Acyclic Graphs (DAG) where nodes and their relation(s), the source(s) are defined.

Each node has:

* a unique identifier
* a job template id reference
* one or more sources
* one or more parameters and associated values to overide the default value (if defined in the job template).

The node_expression node
------------------------

The first node of the workflow instantiates the expression job template.

.. code-block:: xml
  
  <node id="node_expression">
  <job id="expression"></job>

As source, this node uses the sandbox catalogue:

.. code-block:: xml
        
  <sources>
  <source refid="cas:series">http://localhost/catalogue/sandbox/MER_RR__1P/description</source>
  </sources>

As parameters, it defines the values for the start and enddate and leave the expression default value.

.. code-block:: xml
  
  <parameters>
    <parameter type="opensearch" target="time:start" id="startdate">2012-04-02</parameter>
    <parameter type="opensearch" target="time:end" id="enddate">2012-04-06</parameter>
  </parameters>

The complete node definition is:

.. code-block:: xml

  <node id="node_expression">
      <job id="expression" />
      <sources>
        <source refid="cas:series">http://localhost/catalogue/sandbox/MER_RR__1P/description</source>
      </sources>
      <parameters>
        <parameter type="opensearch" target="time:start" id="startdate">2012-04-02</parameter>
        <parameter type="opensearch" target="time:end" id="enddate">2012-04-06</parameter>
        </parameters>
  </node>

The node_arrange node
---------------------

The node_arrange instantiates the arrange job template and uses the default value for the period. The node inputs are not a reference to a catalogue as for the expression node, but the results of the expression node:

.. code-block:: xml

  <node id="node_arrange">
    <job id="arrange"></job>
    <sources>
      <source refid="wf:node">node_expression</source>
    </sources>
    <parameters>
    </parameters>
  </node>


The node_binning node
---------------------

.. code-block:: xml

  <node id="node_binning">
    <job id="binning"></job>
    <sources>
      <source refid="wf:node">node_arrange</source>
    </sources>
    <parameters>
    </parameters>
  </node>

The node_clustering node
------------------------

.. code-block:: xml

  <node id="node_clustering">
    <job id="clustering"></job>
    <sources>
      <source refid="wf:node">node_binning</source>
    </sources>
    <parameters>
    </parameters>
  </node>

The complete workflow is attached.

Putting the pieces together
***************************

You have defined the job template and the workflow. The Application Descriptor file is now complete. 
At this stage, you will create the job folder under /application, the streaming executables and create the application files.

The expression job
------------------

The expression job application invokes a Bash script, named beam_expr.sh that takes one or more MERIS products (available in the filesystem), the expression, the output band name and an folder to sotre the results.  
The beam_expr.sh is an executable that can be invoked manually. 
Create the file in the folder /application/expression/bin/ and make executable with

.. code-block:: bash

  chmod 755 /application/expression/bin/beam_expr.sh
 

You will test the script to understand how it works.

First, copy one MERIS product available in the sandbox catalogue to your home:

.. code-block:: bash

  ciop-copy -o ~ "http://localhost/catalogue/sandbox/MER_RR__1P/rdf?count=1"

Notice the output of the ciop-copy utility, it is the local path of the copied file. It is often usefull to store this value in a variable to access the copied product.

Invoke the beam_expr.sh script:

.. code-block:: bash

  export BEAM_HOME=$_CIOP_APPLICATION_PATH/share/beam-4.11
  export PATH=$BEAM_HOME/bin:$PATH
  $_CIOP_APPLICATION_PATH/expression/bin/beam_expr.sh -b out -e "l1_flags.INVALID?0:radiance_13>17?0:100+radiance_9-(radiance_8+(radiance_10-radiance_8)*27.524/72.570)" -o ~ ~/MER_RR__1PRLRA20120405_174214_000026213113_00228_52828_0110.N1

You'll find in your home the result: MER_RR__1PRLRA20120405_174214_000026213113_00228_52828_0110.N1.dim.tgz

You will now create the streaming executable (run) using the Bash scripting language that invokes the beam_arithm.sh executable.

The beam_expr.sh needs the arithmetic expression value. To do so, you will use the ciop-getparam function (part of the ciop_job_include that needs to be sourced):

.. code-block:: bash

  #!/bin/bash
  source ${ciop_job_include}
  expression="`ciop-getparam expression`"

Ok, you have the variable expression with the value "l1_flags.INVALID?0:radiance_13>17?0:100+radiance_9-(radiance_8+(radiance_10-radiance_8)*27.524/72.570)"

You'll proceed with the copy of the MERIS products whose RDF URLs are passed as the result of the OpenSearch query:

.. code-block:: bash

  opensearch-client -p time:start=2012-04-05 -p time:end=2012-04-06 http://localhost/catalogue/sandbox/MER_RR__1P/description


which returns:

.. code-block:: bash

  http://localhost/catalogue/sandbox/MER_RR__1P/MER_RR__1PRLRA20120406_102429_000026213113_00238_52838_0211.N1/rdf
  http://localhost/catalogue/sandbox/MER_RR__1P/MER_RR__1PRLRA20120405_174214_000026213113_00228_52828_0110.N1/rdf
  http://localhost/catalogue/sandbox/MER_RR__1P/MER_RR__1PRLRA20120405_142147_000026243113_00226_52826_0090.N1/rdf
  http://localhost/catalogue/sandbox/MER_RR__1P/MER_RR__1PRLRA20120405_092107_000026213113_00223_52823_0052.N1/rdf 
  http://localhost/catalogue/sandbox/MER_RR__1P/MER_RR__1PRLRA20120404_231946_000026213113_00217_52817_9862.N1/rdf

So, behind the scenes, the streaming executable is invoked with a command similar to:

.. code-block:: bash

  opensearch-client -p time:start=2012-04-05 -p time:end=2012-04-06 http://localhost/catalogue/sandbox/MER_RR__1P/description | /application/expression/run

You'll edit the streaming executable (/application/expression/run) to add the copy of the MERIS products:

.. code-block:: bash
  
  #!/bin/bash
  source ${ciop_job_include}
  expression="`ciop-getparam expression`"
  
  while read inputfile 
  do
    retrieved=`ciop-copy -o $TMPDIR "$inputfile"`
  done

The ciop-copy utility is invoked with the option -o set to $TMPDIR. This variable contains the path to a unique temporary folder that only one instance of the streaming executable will use (concurrency in parallel tasks is thus avoided).

You have the expression value and the MERIS file copied to the temporary folder. You will now add the creation of the output folder for the results and invoke beam_expr.sh

.. code-block:: bash

  #!/bin/bash
  source ${ciop_job_include}
  expression="`ciop-getparam expression`"
  
  mkdir -p $TMPDIR/output
  export OUTPUTDIR=$TMPDIR/output
  
  while read inputfile 
  do
    retrieved=`ciop-copy -o $TMPDIR "$inputfile"`
    $_CIOP_APPLICATION_PATH/expression/bin/beam_expr.sh -o $OUTPUTDIR -e "$expression" -b out $retrieved 1>&2 	
  done


If this streaming executable is run, the $OUTPUTDIR folder will contain all the beam_expr.sh results, in order to make these available in the distributed file system, these have to be published with the ciop-publish utility.
After the publication to the distributed filesystem, the input and output are no longer needed, so you will free the space and leave the environment clean for the next MERIS product to be processed.
ciop-publish plays another important role: it tells the framework what has been produced (in practical terms, the inputs of the next node: node_arrange).

.. code-block:: bash

  #!/bin/bash
  source ${ciop_job_include}
  expression="`ciop-getparam expression`"
  
  mkdir -p $TMPDIR/output
  export OUTPUTDIR=$TMPDIR/output
  
  while read inputfile 
  do
    retrieved=`ciop-copy -o $TMPDIR "$inputfile"`
    $_CIOP_APPLICATION_PATH/expression/bin/beam_expr.sh -o $OUTPUTDIR -e "$expression" -b out $retrieved 1>&2 	
    ciop-publish $OUTPUTDIR/*.tgz
    rm -fr $retrieved $OUTPUTDIR/*.tgz
  done

You're done! The streaming executable of the job template expression is created.
The streaming executable can of course be enhanced with the error handling, checks on the outcomes of the commands, etc. 
The final expression node template streaming executable is attached and includes extended comments. 

Simulating and testing
**********************

node_expression simulation and testing
--------------------------------------

The node_expression will produce:

.. code-block:: bash

  MER_RR__1PRLRA20120406_102429_000026213113_00238_52838_0211.N1.dim.tgz
  MER_RR__1PRLRA20120405_174214_000026213113_00228_52828_0110.N1.dim.tgz
  MER_RR__1PRLRA20120405_142147_000026243113_00226_52826_0090.N1.dim.tgz
  MER_RR__1PRLRA20120405_092107_000026213113_00223_52823_0052.N1.dim.tgz
  MER_RR__1PRLRA20120404_231946_000026213113_00217_52817_9862.N1.dim.tgz


These files are all available in the distributed filesystem.
