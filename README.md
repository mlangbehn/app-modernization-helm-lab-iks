# IBM Client Developer Advocacy App Modernization Series

## Lab - Migrating Legacy JEE apps to IBM Cloud Kubernetes Service

### Part 1 - Working with Helm

## Overview

Helm is a package manager for Kubernetes. Package managers automate the process of installing, configuring, upgrading, and removing computer programs. Examples include the Red Hat Package Manager (RPM), Homebrew, and Windows® PackageManagement.

An application in Kubernetes typically consists of at least two resource types: a deployment resource, which describes a set of pods to be deployed together, and a service resource, which defines endpoints for accessing the APIs in those pods. In additional to a deployment and a service, an application will typically include other Kubernetes resource types such as ConfigMaps, Secrets, and Ingress.

For any application in Kubernetes, you will need to run several Kubernetes commands (`kubectl`) to create and configure resources. With Helm, instead of manually creating each resource separately, you can create many resources with one command (`helm install`). This greatly simplifies the process and allows you to manage the related resources as a single unit called a Helm chart.

Helm charts are stored in a Helm chart repository, which is an HTTP server that houses packaged charts and an `index.yaml` file. The `index.yaml` file has an index of all the charts in the repository. A chart repository can be any HTTP server that can serve YAML and .tar files and can answer GET HTTP requests. Therefore, you have many options for hosting your chart repository such as a Google Cloud Storage bucket, an Amazon S3 bucket or you can create your own web server. For this lab, we will be using a local test repository provided by Helm to  test the hosting of a chart repository.

In this lab you'll create a Helm chart repository and use it to deploy a small Java EE app to the IBM Cloud Kubernetes Service using the open source Helm CLI.

### Setup

1. Let's log into the kubernetes cluster. From the web terminal, enter the following command:
```
ibmcloud login -sso
```

2. When asked about opening the url in a browser enter *n*

![loginSSO](./images/loginSSO.png)

3. Then, click on the link for the one time code. This will open up a new browser window for you to log into IBM Cloud, if you aren't logged in already. After logging in, you will be taken to a page with a one time passcode.

4. Copy the one time code.

![one time code](./images/oneTimeCode.png)

5. Go back to your terminal and paste in the code that you copied and press enter. Since this input is masked for privacy, you will not see the characters appear.

6. If asked to select an account, enter the number for the "Sprint PoC" account

![logging in](./images/loggingIn.png)

7. When asked to select a region, select the number for *us-south*.

8. If asked to update, enter *n*

Now that we are authenticated with IBM Cloud, we need to get the connection information for our cluster

9. Enter the following command to get the cluster configuration but replace *cluster_name* with the name of the cluster given to you:
```
ibmcloud ks cluster config cluster_name
```

10. Then copy the yellow export command that appears in your terminal, paste it back into your terminal, and press enter.

![exportCommand](./images/exportCommand.png)

Now that we are authenticated with the cluster, we can use the Kubernetes CLI (kubectl) to interact with our cluster.

11. Next let's install helm. Run the following commands:
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --upgrade
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```


### Step 1: Build an Open Liberty image of the app and then push it to the IBM Cloud Kubernetes container registry


6. Build the application .ear file using Maven by typing in (or copying and pasting in) the following command

    ```text
    mvn package
    ```

7. Build a docker image  by typing in (or copying and pasting in) the following (uncommented) commands

    >Note: if you don't have the environment variables in the command set, replace $CRNAMESPACE with your container registry namespace and $USERNAME with your lab user id.

    Run the following commands, replacing *lab_username* with your lab username
    ```
    export CRNAMESPACE=sprint
    export USERNAME=lab_username
    ```

    Then run the following command to build a docker image

    ```bash
    docker build -t us.icr.io/$CRNAMESPACE/$USERNAME/pbw-mariadb-web:1.0.0 .
    ```

8. Log in to the Container registry with the following command:

    ```bash
    ibmcloud  cr login
    ```

9. Push the image to the IBM Container registry by typing in (or copying and pasting in) the following (uncommented) commands

    >Note: if you don't have the environment variables in the command set, replace $CRNAMESPACE with your container registry namespace and $USERNAME with your lab user id.

    ```bash
    docker push us.icr.io/$CRNAMESPACE/$USERNAME/pbw-mariadb-web:1.0.0
    ```

### Step 2: Look at the structure of the Helm chart for the Plants By WebSphere app

1. Login in [your Github account](https://github.com)

2. Select your fork  of the repository **app-modernization-plants-by-websphere-jee6** that you forked in the previous lab.

3. Using the Github's UI  file browser to  take a look at the files in the **chart** folder. This is a Helm chart with child charts for the web app and MariaDB  portions of the app. Since there already is a published chart for  MariaDB, it is listed  as a required child chart in the file **requirements.yaml** and you don't have to create a chart for the MariaDB portion of the app.

### Step 3: Create the artifacts for the Helm repository


1. From your web terminal  type in (or copy and paste in) the following (uncommented) commands

    ```
    # Fetch required MariaDB chart
    helm repo add ibmcom https://raw.githubusercontent.com/IBM/charts/master/repo/stable
    helm dependency update chart/pbw-liberty-mariadb

    # Generate the chart archive.
    helm package chart/pbw-liberty-mariadb -d ~/.helm/repository/local

    ```

### Step 4: Configure Helm to serve up the repo via HTTP

1. In your terminal window type the following command, to start the local test Helm repository substituting for [PORT_NUMBER]. If you're using a web based terminal as part of an IBM instructor led workshop, use a port number derived from your username so it will be unique and not conflict with other users. The pattern is `90 + USER_NUMBER`. For example if your username is ``user23`` use port ``9023``, if your username is ``user09`` use port ``9009`` and so on. If you're using a terminal on your own machine use any free port number.

    ```
    helm serve --address 127.0.0.1:[PORT_NUMBER] &
    ```

    Then press enter.

2. In your terminal window type the following command. Verify that the contents of *index.yaml* are returned and it contains the chart archive you just added. Use the same port number you used in the previous instruction.

    ```
    curl http://127.0.0.1:[PORT_NUMBER]/charts/index.yaml
    ```

### Step 5: Deploy the legacy JEE app from your new Helm repo

1. In your terminal window type the following command, substituting your terminal user for [YOUR_USERNAME].  **Note**: Helm charts can be deployed multiple  times but each deployment must have a unique name

    >Note: if you don't have the environment variables in the command set, replace $CRNAMESPACE with your container registry namespace and $USERNAME with your lab user id.

    ```
    helm install --name pbw-liberty-mariadb --set liberty.image.registry=us.icr.io \
    --set liberty.image.namespace=$CRNAMESPACE/$USERNAME local/pbw-liberty-mariadb
    ```

### Step 6: Launch your deployed app

You'll commands to get the endpoint and port number of your deployed Helm release.

1. Run the following command to get the port number of your deployed app
    ```
    echo `kubectl --namespace default get service pbw-liberty-mariadb-liberty -o jsonpath='{.spec.ports[0].nodePort}'`
    ```

2. Run the following command to get the external IP address  of the first worker node in your cluster

    >Note: if you don't have the environment variables in the command set, replace cluster-name with the name of his cluster

    ```bash
    ibmcloud ks workers cluster-name | grep -v '^*' | egrep -v "(ID|OK)" | awk '{print $2;}' | head -n1
    ```
3. In your browser's address bar enter the URL of your deployed app. To find the app URL, enter the following commands, replacing *YOUR_CLUSTER_NAME* with your cluster name.

```
nodeip=$(bx cs workers YOUR_CLUSTER_NAME | grep -v '^*' | egrep -v "(ID|OK)" | awk '{print $2;}' | head -n1)
port=$(kubectl --namespace default get service pbw-liberty-mariadb-liberty -o jsonpath='{.spec.ports[0].nodePort}')
echo "http://${nodeip}:${port}"
```

Click on the address in the terminal to be taken to the app.


4. Verify that the app's UI opens in another tab. Click on the **HELP** link.

![Running app](images/ss4.png)

5. Click on **Reset database** to populate the MariaDB database with data

6. Verify that browsing different sections of the online catalog shows product descriptions and images.

![Online catalog](images/ss5.png)

## Summary

With even small simple apps requiring multiple Kubernetes objects,  Helm charts greatly simplify the process of distributing and updating your Kubernetes based apps. Helm repos allow you to distribute your Helm charts via HTTP, further simplifying the process of distributing and deploying your apps.

Continue on to the next lab, [building a CI/CD pipeline in Jenkins](https://github.com/odrodrig/app-modernization-cicd-lab-iks).