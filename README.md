# Mathematica and Tableau integration
Tableau 10.3, introduce a feature to integrate python with tableau through a library named `tabpy`. As of version 2021.1, tableau works with external services such as R, Python, and Matlab. Now you can use the power of Mathematica inside Tableau. This repository consists of two code:
- `server.nb` to run Mathematica code inside Tableau (Mathematica should be installed)
- `web_data_connector.nb` to send data directly from Mathematica to Tableau


## How to run Mathematica in Tableau
When you run `tabpy`, it will run a local server that evaluates each request python code with the given data and returns back the result. The same procedure can happen in Mathematica with help of `SocketListen`, we could run a local server and by defining a function that encodes the request, apply `ToExpression` to them and returning the result.

## How to send Mathematica data to Tableau
Consider a JSON file like http://sample.com/file.json, as tableau 2021.1, there is no way to send that file directly to Tableau. Tableau has its own way to handle data from the web called `Web Data Connector`. In simple terms, you should run some JavaScript code before you handing the data to Tableau. With the help of Mathematica `SocketListen` we could run a server and mimic a web page to send data directly from Mathematica to Tableau.

## Run Mathematica code inside Tableau
1 - First either copy `server.nb` or download the file and run it. The code Automatically runs on port 36000. You could change that to any number as long as it's accessible.

2 - Setup Tableau by going to `Help` > `Setting and Performance` > `Manage Analytics Extension Connections...`. Select `Tabpy/External API`, change `Server` to `localhost` or `127.0.0.1` and port to `36000`:

![](https://i.imgur.com/8e3Znso.png)

3 - Now just like tabpy and running python, you can run Mathematica. Create a `Calculated Field` in Tableau and use Tableau's `SCRIPT_REAL()` or other `SCRIPT_SOMETHING()`. Keep in mind:
- Unlike python, there is no need to use `Return`
- You can access Tableau's expressions in the code by using `arg1` for the first argument, `arg2` for the second, and so on
- Since the kernel is the same that runs your notebook, you have access to all the functions and variables you'd in your notebook


### Compare to Python - Example 1
Objective: increase the given price by one:

If you want to do it with python and tabpy, you should run this code in Tableau's `Calculated Field`:

```tableau
SCRIPT_REAL("return [i+1 for i in _arg1]",SUM([Price]))
```

Mathematica equivalent:
```tableau
SCRIPT_REAL("arg1+1",SUM([Price]))
```

### Clustering data - Example 2
Here we'll use Mathematica capabilities to cluster `price` and `quantity`:
```tableau
SCRIPT_REAL("ClusteringComponents[Transpose[{arg1,arg2}]]",SUM([Price]),SUM([Quantity]))
```
![](https://i.imgur.com/fBJ0wn6.png)

## Terminating the Server
After you'd done your work, run the following code in Mathematica to close the connection and shut down the server:
```mathematica
Close[server["Socket"]]
DeleteObject[server]
```

## Possible Issues
If your data depend on very small decimals like 10^-9, you might see a little difference between Mathematica calculation and Tableau. Generally, Mathematica will evaluate your code up to 20 digits in decimal but transferring these numbers to Tableau and storing them may distort them by a very little amount.

For example, I have a sample sales data with 3 columns `product`, `quantity`, and `price`. The goal is to calculate the average sales by multiplying the sum of `quantity` with the average of `price`.

Mathematica code:
```tableau
SCRIPT_REAL("arg1*arg2",SUM([Quantity]),AVG([Price]))
```
Tableau code:
```tableau
SUM([Quantity])*AVG([Price])
```
Here are the differences between the two columns:

![](https://i.imgur.com/nyKhtbQ.png)

# Load Mathematica data in Tableau
If you want to send dynamic data directly to Tableau without saving it on disk, then this section will help you but beware that loading data with this solution is slower than reading a static file.

1 - Either copy the `web_data_connector.nb` or download the file and run it

2 - Send your data with the `sendToTableau` function, keep in mind:
- Because of `jquery` and `tableauwdc` JavaScript libraries, you and tableau should be able to connect to the internet 
- your data should be a 2-dimensional array
- supported data types are: Real, Integer, Boolean, String, Date
- `Missing[]` values in data will convert to `null`
- if no `Headers` exists, column names for your data automatically generated as `C1` for the first column, `C2` for the second, and ...

```mathematica
data = Table[{Now, RandomReal[], RandomChoice[{True, False}], RandomInteger[10], "Test"}, 4];
server1 = sendToTableau[data]
```

2 - In Tableau, `Data` > `New Data Source` > `Web Data Connector`. In the URL section type `localhost:37000` or `127.0.0.1:37000`

![](https://i.imgur.com/5vNuW2y.png)

3 - When the page loaded, click on `Click here to load`

4 - From now, you can use the `Refresh` button to get the newer version of the data

## Settings
You should change the following code inside `SocketListen`, but make sure to terminate the server before re-evaluating the code:
```mathematica
(* automatically generated column names *)
(* will use {"C1","C2","C3","C4","C5"} *)

server1 = setupTableauConnector[data]
```

Your list of names should be the same length as the first row of your data:
```mathematica
(* specify column names *)

server1 = setupTableauConnector[data, "Headers"->{"Column 1", "Column 2", "Column 3", "Column 4", "Column 5"}];
```
Changing the port with:
```mathematica
(* default port: 39000 *)

server1 = setupTableauConnector[data,"Port"->40000];
```

Change port and specify column names:
```mathematica
server1 = setupTableauConnector[data,"Headers"->{"C1","C2","C3","C4","C5"},"Port"->40000];
```

## Terminating the Server
After you'd done your work, run the following code in Mathematica to close the connection and shut down the server:
```
Close[server1]
```


## Possible Issues
Tableau `Web Data Connector` is built to connect to stable addresses, for example on Tableau 2020.1 which I tested, if you use this method and connect your data via some port, after closing your file, every time you open the file, Tableau tries to connect to the same port and doesn't let you change it unless it connects to that port once. Sometimes that port is in use by another program and you can't use that. The solution is to run with a different port (in Mathematica change the first argument of `SocketListen`), then open your Tableau file in a text editor, search for the previous port and replace it with the newer one.
