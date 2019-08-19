Now you know what BMEG is and what it does, you can start using BMEG. Follow the step-by-step tutorial below.

# How to Use BMEG

  
  

This six-step tutorial is created to introduce you to BMEG. Because this tutorial is designed to get you started from the basics, **no prior knowledge is necessary**. As you go through the tutorial, **remember that obtaining data from BMEG for your specific analysis is the ultimate goal**, so everything we will do before then is to set things up for that last step. This tutorial will focus on three main areas: **1.** setting up the right tools to use BMEG, **2.** accessing BMEG, and **3.** conducting analysis.

# 1. Grip Set up and Authentication

This is a one time setup step.

### 1.1  Getting set up with gripql
    

Gripql (GRaph Integration Platform query language) is a python library for interacting with a GRIP server. Ultimately, gripql provides a graph interface on top of a variety of existing database technologies. You can find out more about GRIP right [here](https://bmeg.github.io/grip/).

  

Gripql is a python library for GRIP. Go ahead and install it using this [link](https://pypi.org/project/gripql/) in terminal.

 ``` bash
 pip install gripql
 ```
 
 
### 1.2 Authentication
    

Your next step is connecting your Google account to our website. We use this step to be able to authenticate that you are a BMEG user. Press the link to go to our [authentication page](https://bmeg.io/analyze/getting_started/) and click on “Authenticate with Google” which will redirect you to Google to verify identity. Then you will be granted interactive and API access.

  

Or if you would rather do the set up and authentication from your  terminal, you can use the bash code down below. This code achieves two main goals. The ``pip install gripql`` part installs the gripql library for python and the other two lines of code below it are made to help you authenticate using your email. Run the following in terminal as per the above authentication page.

  
``` bash
# Make query using curl
curl -H "OauthEmail: <email>" -H "OauthAccessToken: <token>=" -H "OauthExpires: <length>" https://bmeg.io/api/v1/graph

# Create credential file
echo '{"OauthEmail":"<email>","OauthAccessToken":"<token>=","OauthExpires":<length>} ' > /tmp/bmeg_credentials.json
```

> The following blocks of code are written in python. Therefore, it would be beneficial to have python installed on your device. Instructions on installing python can be found [here](https://www.python.org/downloads/).
>
>To install the libraries used in this tutorial, you will most likely use ``pip`` so it would be useful to install ``pip`` as well. Instructions on installing ``pip`` are [here](https://pypi.org/project/pip/). If you have a python version 2.7.9 or newer, ``pip`` comes included.

# 2. Situate Libraries and Connect to server

Each of the libraries listed below have different functionalities that ultimately help us in our analysis. To be able to import the libraries in our code, we first need to install them (if we don’t already have them). Follow the links below to install the libraries on your computer. The links will take you to a page with instructions on how to install each library for different systems.
    
- [Instructions](https://pypi.org/project/pandas/) on installing pandas - Open source python library that helps us perform fast data cleaning, preparation, and analysis
    
-  [Instructions](https://www.scipy.org/install.html) on installing numpy - a multidimensional container for scientific computing that allows us to do calculations on different collections of values.
    
- json - (JavaScript Object Notation) is a data interchange language. This package usually comes with python. Check if you already have it installed

- os - In python, this module provides functions for interacting with the operating system. This package should already be installed. Check if you already have it installed

Now that we have taken care of, let's begin.

Import python libraries
``` python
import gripql
import pandas as pd
import numpy as np
import os
```
  

This code here will connect you to the BMEG server and create a graph handle. As you might have already known, **the BMEG server is where all the data lives.** To grab any type of data, we must establish a connection with the server. The first line in this code block sets up a connection with the BMEG server using grip and the second line creates a graph handle based on a specific graph schema. **In other words, while the first line sets up a connection with bmeg, the second line chooses a specific schema type. It is recommeneded to use the most recent schema which is found** [**here**](https://bmeg.io/explore/schema)


``` python 
conn = gripql.Connection("https://bmeg.io/api", credential_file = '/tmp/bmeg_credentials.json')
O = conn.graph("bmeg_rc2") 
```
# 3. Query data from a particular data source

Traverse graph to grab all data from one data source (e.g. CCLE). See how the graph is set up by viewing the current [schema](https://bmeg.io/explore/schema)

Here, you’ll be grabbing all the data from one data source (e.g. CCLE) found within BMEG. In the first line of this code block, we are creating what is essentially an empty list and naming it ``p``. Then within the for loop, we are searching for a vertex with the label “project”. In our if statement, we ask if the data project identification starts with CCLE and if it does, we bring out that data source under the name ``p``.

``` python
p = []
# Start query at vertex and look for labels starting with Project
for row in O.query().V().hasLabel("Project"):
    if row.data.project_id.startswith("CCLE"):
        p.append(row.gid)
```

# 4. Pull out data of interest from node properties

Accessing a whole data source will likely mean that you will have a lot of information you don’t need. For this reason, we need to filter our data to only get precise information on the things we want to know about. This code is made to filter data by only keeping data points of interest.

For example, in this tutorial, I am interested in looking at the drug response that a given drug has on a cell line. Therefore I’m interested in 1) drug name 2) which cell line the drug was tested on and 3) what the drug response was (EC50).


+ BMEG nodes that will be traversing through: Project > **Case** > Sample > Aliquot > **DrugResponse** > **Compound** 
+ Bolded are nodes that we will be pulling node properties (data). 

``` python
# Traverse through schema nodes
q = O.query().V(p).out("cases").as_("CASE").out("samples").out("aliquots").out("drug_response").as_("DRUGRESP").out("compounds").as_("COMP")

# Render node properties of interest (cell line, drug, EC50 value)
q = q.render(["$CASE._data.case_id", "$COMP._gid", "$DRUGRESP._data.ec50"])
```


# 5. Save data locally

We need to save our data file so we can access it whenever we want. But before that, we need to re-format our data so that it can be saved easily. 

First, populate data into a dictionary.

+ ``data`` keys = desired columns and values = a nested dictionary where keys = desired rows and values = entry
+ In our example we use keys = cell lines and values = a nested dictionary where keys = drugs and values = EC50 values

Now let’s format the data so that we can save it as a .tsv file. To do this we will populate the data in a dictionary then create a dataframe from that dictionary. And finally save this dataframe as a .tsv file.

``` python
data = {}
for row in q:
    # If cell line not in dict > add a new key to dict
    if row[0] not in data:
        data[row[0]] = { row[1] :  row[2] } 
    # If cell line already added to dict > add a new key to nested dict
    else:
        data[row[0]][row[1]] = row[2]
```

Second, create a pandas dataframe for readability purposes.

``` python
# Create pandas df
drugDF = pd.DataFrame(data)

# Sort rows by cell line name, sort cols by drug name 
drugDF = drugDF.sort_index().sort_index(axis=1) 
```

Let’s take a quick break and view what our data looks like:


``` python
drugDF
```
Now save it as a .tsv file. Replace “where-to-save” with the path where you'd like to save this file on your machine and replace "name-of-drug-tsv" with the file name you'd like.
``` python
# Save
output_path = "where-to-save-file"
ofn_drug = "name-of-drug-tsv"

os.chdir(output_path)
drugDF.to_csv(ofn_drug, sep="\t")
```


  

# 6. Continue with downstream analysis

Congratulations on making it to the last step of this tutorial! You now have produced a data file from BMEG and can continue with your analysis.

You can check out some downstream analysis examples [here](https://bmeg.io/analyze/examples/).

If you have any questions or need more support, contact us using [Gitter](https://gitter.im/bmeg/). We will get back to you as soon as possible.
