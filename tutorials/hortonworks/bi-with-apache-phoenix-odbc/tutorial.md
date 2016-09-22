---
layout: tutorial
title: BI with Apache Phoenix ODBC
tutorial-id: 740
tutorial-series: Hello World
tutorial-version: hdp-2.5.0
intro-page: true
components: [ hbase, phoenix, odbc, tableau  ]
---

## Overview

**Apache HBase** is a NoSQL columnar database in the Hadoop eco-system. Many business intelligence tool and data analytic tools lack the ability to work with HBase data directly. **Apache Phoenix** enables you to interact with HBase using SQL. In **HDP 2.5**, we have introduced support for **ODBC drivers**. With this, you can connect any ODBC applications to HBase via Apache Phoenix. This enables you to use familiar business intelligence tools to run analysis on big datasets.  This tutorial is designed to help you learn how to use business intelligence tool such as Tableau to create data visualization using data in Apache Phoenix. This tutorial goes over the details on how to setup an end to end workflow.

## What you’ll learn from this tutorial

1\. How to launch a Hortonworks Hadoop sandbox (optional)  
2\. How to create a table and load data into Apache Phoenix  
3\. How to setup the Hortonworks ODBC connector  
4\. How to run bigdata analytics using Tableau/Excel using data in Phoenix  

## Prerequisites

* A deployment of HBase & Phoenix  
In this tutorial, I’ll be using Hortonworks Sandbox, alternatively, you can:  

  * Use Microsoft HDInsight  
  * Use Amazon EMR  
This tutorial will not cover setup for HDInsight or Amazon EMR.  

* Tableau installation on Windows  

* ODBC driver for your platform  

  * [Windows (32-bit)](http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/windows/HortonworksPhoenixODBC32.msi)  
  * [Windows (64-bit)(http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/windows/HortonworksPhoenixODBC64.msi)  
  * [Linux (32-bit)(http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/linux/HortonworksPhoenixODBC-32bit-1.0.0.1000-1.rpm)  
  * [Linux (64-bit)(http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/linux/HortonworksPhoenixODBC-64bit-1.0.0.1000-1.rpm)  

### Getting and launching the HDP sandbox  

You can get the latest version of VirtualBox here: [https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)  

After installing VirtualBox, you can download the sandbox image from here:  
[http://hortonworks.com/downloads/#sandbox](http://hortonworks.com/downloads/#sandbox)  

Start VirtualBox, select `File->Import Appliance`

### Setting up admin password for sandbox

Once the sandbox is up and running, the first thing you want to do is to setup an admin password.  
If you are using ssh (or putty), connect to localhost,  port **2222** with account **root**:

~~~
$>ssh root@localhost -p 2222
~~~

The sandbox also comes with it’s own shell: [http://127.0.0.1:4200/](http://127.0.0.1:4200/)

Login with username **root** and password **hadoop**. You’ll be asked to change your password upon first login. This is the password for ssh and logging into the OS. It is NOT the password for Ambari UI access.  

Once you are logged into the shell, run the following command to reset Ambari’s admin password.

~~~
$>ambari-admin-password-reset
~~~

### Setting up HBase/Phoenix/Phoenix Query Server  

In this section, you’ll learn how to setup HBase, Phoenix and Phoenix Query Server (PQS) using Ambari dashboard.

Go to [http://127.0.0.1:8080/](http://127.0.0.1:8080/) , login with username **raj_ops** and the password **raj_ops**  

![ambari_dashboard_rajops](/assets/bi-with-apache-phoenix-odbc/ambari_dashboard_rajops.png)

Find **HBase** on the left side, click on `Configs` tab. At the bottom of the page, you can find **Phoenix SQL**, toggle the button to `Enabled` position. This step enables the support of Phoenix SQL queries, but doesn’t run the Query server itself. **Running Phoenix Query Server** is a step below.

![enable_phoenix](/assets/bi-with-apache-phoenix-odbc/enable_phoenix.png)

Click on `Save` button to save the configuration:

![click_save](/assets/bi-with-apache-phoenix-odbc/click_save.png)

Accept the dependent configuration changes. Click on `Proceed anyways` if you are being warned about security config. Those are only required if you want to test a secured setup, which is out of scope for this tutorial.  

Next, you’ll be prompted to restart the server, select `Restart All Affected`

![restart_all_affected](/assets/bi-with-apache-phoenix-odbc/restart_all_affected.png)

Now, in the `Summary` tab, you should see both HBase Master and RegionServers to be healthy.

![summary_tab](/assets/bi-with-apache-phoenix-odbc/summary_tab.png)

If they are **Stopped** or showing red, click on `Restart all` to ensure all HBase services are running.

![restart_all](/assets/bi-with-apache-phoenix-odbc/restart_all.png)

It is possible that your HBase instance is stuck in Maintenance Mode. To disable this, click on `Turn Off Maintenance Mode` after restart has completed.

### Running Phoenix Query Server  

By default, the sandbox doesn’t have the Query Server turned on. To enable this, go to **Hosts**, select the sandbox host and click on `Add` and select `Phoenix Query Server`:

![select_phoenix_server](/assets/bi-with-apache-phoenix-odbc/select_phoenix_server.png)

Once installed, scroll down in the host page and find **Phoenix Query Server** to start the service.

![start_phoenix_server](/assets/bi-with-apache-phoenix-odbc/start_phoenix_server.png)

### Setting up port forwarding in VirtualBox

Phoenix Query Server listens on port **8765** by default. Thus we need to setup port forwarding to make sure we can connect to this port from the VM host.  
Right click on the Hortonworks Sandbox, pick `Settings`  

![virtualbox_settings](/assets/bi-with-apache-phoenix-odbc/virtualbox_settings.png)

Go to Network, pick on `Port Forwarding`

![virtualbox_network](/assets/bi-with-apache-phoenix-odbc/virtualbox_network.png)

Add rule by click on the green `+` sign icon on the top right of the panel.

The rule should have name, protocol=TCP, Host IP=127.0.0.1 and both Host Port and Guest Port on 8765.

![add_port](/assets/bi-with-apache-phoenix-odbc/add_port.png)

### Verify you can access Phoenix Query Server

Go to [http://localhost:8765/](http://localhost:8765/), you should see the following page:

![access_phoenix_server](/assets/bi-with-apache-phoenix-odbc/access_phoenix_server.png)

Don’t be alarmed, PQS doesn’t provide a nice welcome page, but it does function.  

We are all done with setup, now onto the fun parts!  

### Getting data in

In this part, we’ll create the basic table schemas in Phoenix and load data into them.    

If you are using ssh (or putty), connect to localhost,  port 2222 with account root:  

~~~
$>ssh root@localhost -p 2222
~~~

The sandbox also comes with it’s own shell: [http://127.0.0.1:4200/](http://127.0.0.1:4200/)
Use the OS password you’ve set up earlier. If you have not set it up, use default password **hadoop**.  

Once logged in the sandbox, run:  

~~~
git clone https://github.com/2bethere/tableau-odbc-phoenix-tutorial ~/tableau-odbc-phoenix-tutorial/
~~~

To clone the tutorial repository. There are 2 folders in this repo. Data folder contains csv files with actual data, sql folder contains all the scripts.  

We have 5 tables in total:  
1\. Customer  
2\. Store  
3\. Product  
4\. Fact table: Inventory  
5\. Fact table: Sales  

We’ll be using **CORP** schema in this tutorial. Thus all tables will have name such as **CORP.STORE** or **CORP.PRODUCT** etc.  

### Find your Phoenix client tools  

~~~
cd /usr/hdp/current/phoenix-client/bin
~~~

In this directory, you should see sqlline.py or psql.py.  

### Drop all tables (Optional)  

If this the second time you are running or you want to reset, you can run this command to drop all tables.  

~~~
$>./sqlline.py ~/tableau-odbc-phoenix-tutorial/sql/drop_tables.sql
~~~

### Create table and load data  

We’ll be using psql.py, which facilitates table creation and bulkload of data.  

~~~
$>./psql.py -t CORP.CUSTOMER ~/tableau-odbc-phoenix-tutorial/sql/table_customer.sql ~/tableau-odbc-phoenix-tutorial/data/customer_out.csv
~~~

This command creates CORP.CUSTOMER table and loads data into the table.  

This command works like this:  

~~~
psql.py -t TABLE_NAME create_table.sql data.csv
~~~

-t specifies the table we want to load into. In this case, CORP.CUSTOMER. Second part is the sql script used to create the table.
Third part is the data to be loaded.

> NOTE: psql uses Excel dialect csv files (i.e. numbers cannot be quoted)

To finish this process, do the same for the 4 other tables:

~~~
$>./psql.py -t CORP.STORE ~/tableau-odbc-phoenix-tutorial/sql/table_store.sql ~/tableau-odbc-phoenix-tutorial/data/store_out.csv
~~~

~~~
$>./psql.py -t CORP.PRODUCT ~/tableau-odbc-phoenix-tutorial/sql/table_product.sql ~/tableau-odbc-phoenix-tutorial/data/product_out.csv
~~~

~~~
$>./psql.py -t CORP.FACT_INVENTORY ~/tableau-odbc-phoenix-tutorial/sql/table_fact_inventory.sql ~/tableau-odbc-phoenix-tutorial/data/inventory_fact_1998_out.csv
~~~

~~~
$>./psql.py -t CORP.FACT_SALES ~/tableau-odbc-phoenix-tutorial/sql/table_fact_sales.sql ~/tableau-odbc-phoenix-tutorial/data/sales_fact_1998_out.csv
~~~

### Getting data out - Windows

In this step, we’ll setup a connection to the Phoenix Query Server using the ODBC driver on Windows.  

Windows
32-bit: [http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/windows/HortonworksPhoenixODBC32.msi](http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/windows/HortonworksPhoenixODBC32.msi)  
64-bit: [http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/windows/HortonworksPhoenixODBC64.msi](http://public-repo-1.hortonworks.com/HDP/phoenix-odbc/1.0.0.1000/windows/HortonworksPhoenixODBC64.msi)  

After downloading, the driver, follow the instruction to install the driver.  

Next, in control panel, open ODBC Data Sources Administration(64-bit). Alternatively, go to Start, search for **ODBC** and pick the 64-bit version of the tool:

![odbc_data_sources](/assets/bi-with-apache-phoenix-odbc/odbc_data_sources.png)

Pick `User DSN`->`Add`  

![user_dsn_add](/assets/bi-with-apache-phoenix-odbc/user_dsn_add.png)

User DSN means only you have access to this data source, System DSN means anyone else on this computer can use this data source.  
Pick `Hortonworks Phoenix ODBC Driver` in next screen:

![create_new_dsn](/assets/bi-with-apache-phoenix-odbc/create_new_dsn.png)

In the next screen, input the following information:

![configure_new_dsn](/assets/bi-with-apache-phoenix-odbc/configure_new_dsn.png)

~~~
Host: localhost
Port: 8765
~~~

> NOTE: This is assuming you are on a Windows machine and the VM is also running on the same machine, you will use localhost as the host name. However, if the VM is on a remote machine, you’ll need to use the remote machine’s IP or hostname in the Host field.

Next, click on `Test…`, if all goes well, you should see the following screen:  

![success_new_dsn](/assets/bi-with-apache-phoenix-odbc/success_new_dsn.png)

If you run into error, please leave a comment below and I’ll try to help you out.  

One common error is when the VM is not reachable from the Windows machine. Try to access http://your.windows.machine:8765 in the web browser to see if you can reach the machine hosting PQS.  

### Getting data out - Windows - Tableau

Our goal is to generate a map that shows what % of goods sold in Washington state are using recyclable packaging.

![tableau_image](/assets/bi-with-apache-phoenix-odbc/tableau_image.png)

### Important: Configuration file for Phoenix Query Server

Download “Hortonworks Phoenix ODBC Driver.tdc” from this location:   
[https://github.com/2bethere/tableau-odbc-phoenix-tutorial/tree/master/tdc](https://github.com/2bethere/tableau-odbc-phoenix-tutorial/tree/master/tdc)  

Place the file into:  
Current User\Documents\My Tableau Repository\Datasources  

This file informs Tableau how to communicate with Phoenix Query Server. Tableau will scan this directory recursively to load all tdc files.  

Open Tableau, select **“Connect”->”More Servers…”->”Other Databases (ODBC)”**

![connect_more_servers](/assets/bi-with-apache-phoenix-odbc/connect_more_servers.png)

Pick the DSN you’ve just created and hit `Connect`

![choose_dsn](/assets/bi-with-apache-phoenix-odbc/choose_dsn.png)

After you hit Connect, the OK button will be enabled. Hit `OK` to continue.

In the next window, you’ll be prompted to setup a data source in Tableau. Click on the `Search` icon to load all the schema from Phoenix database and then pick `CORP`

![select_schema](/assets/bi-with-apache-phoenix-odbc/select_schema.png)

Click on the `Search` icon again to load all the tables associated with the schema. You should see a screen looks like below:

![tables_listed](/assets/bi-with-apache-phoenix-odbc/tables_listed.png)

Now, let’s build out a data model to visualize all those data.  
Drag and drop the following tables in order:  

1\. FACT_SALES  
2\. CUSTOMER  
3\. PRODUCT  
4\. STORE  
The order helps Tableau build auto joins.

![tables_joins](/assets/bi-with-apache-phoenix-odbc/tables_joins.png)

Select `Go to Worksheet` or `Sheet 1` to switch over to analysis mode

![sheet1](/assets/bi-with-apache-phoenix-odbc/sheet1.png)

Find **Store-City** under the data panel on the left

![select_store_city](/assets/bi-with-apache-phoenix-odbc/select_store_city.png)

Click on the little icon and change the Store City’s geo role to City.

Under `Measure`, drag `Store Sales` into the column.

![measure](/assets/bi-with-apache-phoenix-odbc/measure.png)

Next, drag `Recyclable Package` into column shelf and `City` into row shelf  
Now you should see a page similar to the one below:  

![columns_rows_ready](/assets/bi-with-apache-phoenix-odbc/columns_rows_ready.png)

Click on the map tile in `Show me` panel

![show_me_panel](/assets/bi-with-apache-phoenix-odbc/show_me_panel.png)

You’ll see the final map for entire US.  
Let’s narrow it down to WA, drag `Store state` under **Store** into filter and select `WA` from the list.  

To get better visualization, you can adjust the marks panel. For example, if the pie chart doesn’t show up automatically, pick `Pie` from the marks and adjust the color scheme.

![click_pie](/assets/bi-with-apache-phoenix-odbc/click_pie.png)

## Coming next  

Write an application in your favorite language  

## Architecture

The ODBC connector uses the Phoenix Query Server. The driver has support for both Protobuf and JSON serialization mechanisms, but Protobuf serialization should always be used except when there is an explicit reason to not do this.  

## Compatibility with older releases

While PQS has been shipped in HDP since 2.3, the compatibility with this connector is not tested for versions older than HDP 2.5. While some basic queries/functionality may appear to operate normally, great care should be taken in using this driver with older HDP versions. Functionality may be incorrect or unimplemented.  