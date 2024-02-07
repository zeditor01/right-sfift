[Back to main document](https://github.com/zeditor01/cdc_examples/blob/main/create_scale_sustain_cdc_systems.md).

# Setting Up Classic CDC for VSAM - Worked Example
This chapter is a worked example of setting up Classic CDC for VSAM. 

All CDC capture and apply agents conform to the CDC standards for streaming changes between capture and apply agents, and for supporting administration tools and interfaces. Each CDC implementation for a specific source has to bridge from the generic CDC standards to the specific characteristics of the source. 

VSAM is a file system without database recovery logs that are the source for change data capture from most data sources. CDC for VSAM uses z/OS logstreams to act as a replication log, and the VSAM datasets must be modified to write replication log records to the associated z/OS logstream for that VSAM dataset.

VSAM may be deployed under either CICS control, or independently. 
* If VSAM is deployed independently from CICS, then a separate licensed product (CICS VSAM Recovery) is required to write replication log records to the z/OS logstream. CICS VSAM Recovery is outside the scope of this implementation worked example.
* If VSAM is deployed under CICS, then CICS TS 5.6 or later will write replication log records to the z/OS logstreams. Care must be taken to ensure that Tie-Up records are also written to the z/OS logstream so that the log records written by CICS (referring to DD cards) can be associated with the actual names of the VSAM datasets. Tie Up records can be generated automatically by deploying a CICS-provided transaction.

## Contents

<ul class="toc_list">
<li><a href="#abstract">Abstract</a>   
<li><a href="#1.0">1 Introduction to Classic CDC for VSAM</a>
<ul>
  <li><a href="#1.1">1.1 Requirements to Replicate VSAM Data</a></li>
  <li><a href="#1.2">1.2 The Classic CDC Started Task</a></li>
</ul>
<li><a href="#2.0">2. High Level Review of Implementation Steps</a>
<li><a href="#3.0">3. SMPE Installation of Code Libraries</a>
<li><a href="#4.0">4. Creating the Classic CDC Instance</a>
<ul>
  <li><a href="#4.1">4.1 Create the Instance Libraries</a></li>
  <li><a href="#4.2">4.2 Edit the Parameters member</a></li>
  <li><a href="#4.3">4.3 Generate the Customised JCL members</a></li>
  <li><a href="#4.4">4.4 Define CDC Logstreams</a></li>  
  <li><a href="#4.5">4.5 Define and Mount the Classic Catalog zFS</a></li>
  <li><a href="#4.6">4.6 Define and Populate the Configuration Datasets</a></li>
  <li><a href="#4.7">4.7 Create the Metadata Catalog</a></li>
  <li><a href="#4.8">4.8 Create the Replication Mapping Datasets</a></li>  
  <li><a href="#4.9">4.9 Setup Encrypted Passwords</a></li>  
  <li>(<a href="#4.10:>4.10 Installation Verification Job</a></li>
</ul> 
<li><a href="#5.0">5. Configure the z/OS Environment</a>
<ul>
  <li><a href="#5.1">5.1 APF Authorised Load Libraries</a></li>
  <li><a href="#5.2">5.2 TCPIP Ports</a></li>
  <li><a href="#5.3">5.3 RACF Started Task ID</a></li>
  <li><a href="#5.4">5.4 Test Start the Classic CDC Server</a></li>  
</ul>
<li><a href="#6.0">6. Configure the CICS-VSAM Environment</a>
<ul>
  <li><a href="#6.1">6.1 VSAM Access Service</a></li>
  <li><a href="#6.2">6.2 CICS-TS or CICS-VR to write the replication logstream</a></li>
  <li><a href="#6.3">6.3 Augment VSAM to write replication log records</a></li>
  <li><a href="#6.4">6.4 Generate Tie-Up records</a></li>
</ul>
<li><a href="#7.0">7. Integrate with the wider CDC Landscape</a>
<ul>
  <li><a href="#7.1">7.1 Deploy Classic CDC as a Started Task</a></li>
  <li><a href="#7.2">7.2 Deploy and use the Classic Data Architect IDE</a></li>
  <li><a href="#7.3">7.3 Connect from Management Console to Classic CDC Started Task</a></li>
  <li><a href="#7.4">7.4 Use CHCCLP Scripting</a></li>  
  <li><a href="#7.5">7.5 Conforming to site standards for cross-platform devops and security</a></li>
</ul> 
</ul>


<hr>


<h2 id="abstract"> Abstract</h2>

This document is a basic worked example of setting up Classic CDC for VSAM as a CDC Source Server. 

* It deals with the practical considerations for implementing VSAM as a CDC data source. 
* It's scope is limited to a "basic up and running guide", and is intended to be easy to follow (assuming a base of z/OS and CICS-VSAM practical experience).
* It does not attempt to cover all the product's features.
* It is categorically <b>not</b> a replacement for the  <a href="https://www.ibm.com/docs/en/idr/11.4.0?topic=replication-infosphere-classic-cdc-zos">IBM Classic CDC knowledge centre</a>, which is the comprehensive official product documentation.
 

It is part of a series of documents providing practical worked examples and 
guidance for seting up CDC Replication between mainframe data sources and mid-range or Cloud targets.
The complete set of articles can be accessed using the links at the very top of this page 

<br><hr>

<h2 id="1.0">1. Introduction to Classic CDC for VSAM</h2>  

Classic CDC for VSAM is a CDC Capture Source only. It does not have CDC Apply functionality. 

<b>Aside:</b> Classic CDC for VSAM is licensed seperately from Classic CDC for IMS. However, these two 
products share a lot of common components. 
The sister document [Setting up Classic CDC for IMS.](https://github.com/zeditor01/cdc_examples/blob/main/documents/deploy_cdc_ims.md) follows 
the same pattern as this worked 
example for much of the setup work, but has differences with regard to the services to access IMS and capture changes from IMS. 

CDC Replication is a set of products that implement a common data replication architecture spanning 
a large number of diverse data sources and targets. The CDC common architecture is based upon replication of 
data that conforms to the relational model. Any CDC capture or apply agent that supports a non-relational data structure 
must perform whatever conversion work that is necessary to implement a mapping between that data structure and the 
relational model of data. 

<h3 id="1.1">1.1 Requirements to Replicate VSAM Data</h3> 

The core functionaility of any CDC Capture agent is to read the source database logs asynchronously, 
stage captured data (preferably in memory) until the Unit of Work is commited or rolled back, 
and then publish the committed changes over TCPIP sockets to a CDC Apply agent. 

In addition to the usual requirements, Classic CDC for VSAM needs to handle the fact that VSAM data structures are stored in copybooks, and VSAM does not write a "database log". 

<b>Regarding copybook data structures:</b> The Classic CDC for VSAM product provides 
a tool (Classic Data Architect) to map the copybook data structures
into relational projections of that data, for the purposes of acting as a CDC Replication capture agent. 
The mappings of data structures are stored in a zFS dataset called the "Classic Catalog". This contains relational catalog tables 
like Db2 ( sysibm.systables , sysibm.syscolumns etc... ) that contain the mapping between the fields in the relevant copybooks for the 
VSAM data, and their relational projection. The mapping information allows SQL access to the VSAM datasets to retrieve the base data.

<b>Regarding VSAM logs:</b>Classic CDC for VSAM works by taking advantage of z/OS logstreams. VSAM datasets are augmented to specify an assosiated z/OS logstream. 
If the VSAM dataset is under control of CICS, then CICS Transaction Server is responsible for writing replication log records to the logstream. 
If the VSAM dataset is independent of CICS and written to in batch, then a separately licensed prodyuct ( CICS VSAM Recovery ) can be used to write replication log records to the logstream. 


<h3 id="1.2">1.2 The Classic CDC Started Task</h3>

The diagram below is a representation of the components within a Classic CDC for VSAM started task, and how they 
relate to external artefacts. 

![Classic CDC Started Task](../images/cdc/cdcv_services.PNG)

The services that are outlined in red are the ones that can be protected by the SAF Exit. 
The primary services involved in the cdc capture server are  the VSAM Log Reader Service (VSAM LRS) and the Capture process (Capture). 
The following is a brief summary of what some of the key services do.


| Service | Function |
| --- | --- |
| Connection Handler | Listens on a TCPIP port for requests from other CDC components (Classic Data Architect, Management Console & Access Server, CDC Apply agents). |
| Admin Service | Dispatches tasks to appropriate services within the Started Task. |
| Query Processor | Converts SQL requests into DL/I language to access IMS data structures. |
| VSAM Access | Sevice to connect to and read from VSAM. |
| VSAM LRS | VSAM z/OS Logstream Reader Service. |
| Capture | Extracts log changes needed for subscriptions, and publishes them over TCPIP to CDC Apply Agents. |
| Operator | Command Processor. (Start, Stop, Park etc... ). |
| Logger | Writes Started Task logs to z/OS logstreams. |
| Monitor | Tracks health and performance data. |

All the services, and their governing parameters are documented in the knowledge 
center <a href="https://www.ibm.com/docs/en/idr/11.4.0?topic=zos-configuration-parameters-classic-data-servers-services">Classic Services</a>.

<br><hr>
