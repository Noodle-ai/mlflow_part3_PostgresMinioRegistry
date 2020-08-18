Introduction to MLflow for MLOps Part 3: Database Tracking, Minio Artifact Storage, and Registry
================================================================================================

After following along with the demos in this three part repository you will be able to:

* Understand how you and your Data Science teams can improve your MLOps practices using MLflow
* Use all Components of MLflow (Tracking, Projects, Models, Registry)
* Use MLflow in an Anaconda Environment
* Use MLflow with a Docker Environment (including running an IDE inside of a container)
* Use Postgres Backend Store and Minio Artifact Store for Easy Collaboration

The instructions/demos below assume you are using a Mac OSX operating system. Other operating systems can be used with minor modifications. 

|

Table of Contents:
==================
Part 1: Anaconda Environment
----------------------------
(Github: https://github.com/Noodle-ai/mlflow_part1_condaEnv)
(Medium: https://medium.com/p/1fd9e299226f)

1. What is MLflow and Why Should You Use It?
2. Using MLflow with a Conda Environment 

Part 2: Docker Environment
--------------------------
(Github: https://github.com/Noodle-ai/mlflow_part2_dockerEnv)
(Medium: https://medium.com/p/53516ce45266)

1. Using MLflow with a Docker Environment

Part 3: Database Tracking, Minio Artifact Storage, and Registry
---------------------------------------------------------------
(Github: https://github.com/Noodle-ai/mlflow_part1_condaEnv)
(Medium: https://medium.com/p/9fef196aaf42)

1. Running MLflow with a PostgreSQL Database and Minio Artifact Store
2. MLflow Model Registry

|

1. Running MLflow Tracking with a PostgreSQL Database and Minio Artifact Store
==============================================================================
In this section we build on top of the examples in Part 1: Anaconda Environment. Instead of having a local mlruns folder for storing the information from MLflow Tracking we store the parameters and metrics in a PostgreSQL Database, while storing the artifacts in Minio object storage.

Note: This is the prefered solution to the issue we encountered in Part 2: Docker Environment when we observed the absolute path conflict between the container and the local environment in MLflow Tracking. If you plan on sharing the same tracking experiment across devices a DB should be used for the tracking URI. 

|

Setting Up a PostgreSQL Database Tracking URI and Minio Artifact URI
--------------------------------------------------------------------

1. Follow the setup instructions in Part 1: Anaconda Environment (https://github.com/Noodle-ai/mlflow_part1_condaEnv) but instead of cloning the part 1 repository clone the part 3 repository

.. code-block:: bash

  git clone https://github.com/Noodle-ai/mlflow_part3_PostgresMinioRegistry.git

2. Install PostgreSQL

.. code-block:: bash

  brew install postgresql

3. Verify that PostgreSQL is Installed and Running

.. code-block:: bash

  postgres -V

4. Open PSQL Command Line

.. code-block:: bash

  psql postgres

5. Create a DB User "db_user" with Password "123" (in psql command line)

.. code-block:: bash

  CREATE ROLE db_user WITH LOGIN PASSWORD '123';

6. Verify the New DB User (in psql command line)

.. code-block:: bash

  \du

7. Create Database (in psql command line)

.. code-block:: bash

  CREATE DATABASE mlflow_db;

8. Verify the New Database was created (in psql command line)

.. code-block:: bash

  \list

9. Grant the User Access to the Database (in psql command line)

.. code-block:: bash

    GRANT ALL PRIVILEGES ON DATABASE mlflow_db TO db_user;

10. Install Minio

.. code-block:: bash

  brew install minio/stable/minio

11. Make a Data Folder for Minio (depending on permissions you may need to put this in /Users/<username>)

.. code-block:: bash

  mkdir /data

12. Open Minio UI

Minio uses port 9000 by default but if that port is already in use you can change it by using the "--address" argument.

.. code-block:: bash

  minio server /data --address ":9001"

13. Open a Browser Access URL Returned by the Previous Command And Login Using the "AccessKey" and "SecretKey"

|

.. image:: screenshots/minio_password.png
  :width: 400

|

14. Change the AccessKey and SecretKey if Desired

15. From the Minio UI Create a "mlflow" bucket by clicking on the "create bucket" button in the bottom right corner

|

.. image:: screenshots/minio_create_bucket.png
  :width: 600

|

Examples
--------

Open experiment.ipynb and follow along. This is identical to the notebook in Part 1: Anaconda Environment except that it uses a PostgreSQL DB as the tracking URI and Minio as the artifact URI.

|

Using the Tracking API
^^^^^^^^^^^^^^^^^^^^^^

For more detailed information on using the tracking API look at the "Using the Tracking API" subsection of Part 1: Anaconda Environment.

In order to use a PostgreSQL DB we must set a new tracking URI that uses the PostgreSQL DB we configured above. The database is encoded as <dialect>+<driver>://<username>:<password>@<host>:<port>/<database>. We also must set the S3 endpoint URL with the URL returned when we spun up our Minio UI. Lastly our environment must know the access key and secret access key.

.. code-block:: python

  os.environ['MLFLOW_TRACKING_URI'] = 'postgresql+psycopg2://db_user:123@localhost/mlflow_db'
  os.environ['MLFLOW_S3_ENDPOINT_URL'] = 'http://192.168.86.64:9001'
  os.environ['AWS_ACCESS_KEY_ID'] = 'minioadmin'
  os.environ['AWS_SECRET_ACCESS_KEY'] = 'minioadmin'


We create a new experiment setting the artifact location to be the "mlflow" bucket we created in the Minio UI (Note: an experiment can only be created once). We then set this as our current experiment.

.. code-block:: python

  mlflow.create_experiment('exp', artifact_location='s3://mlflow')
  mlflow.set_experiment('exp')

|

Viewing the Tracking UI
^^^^^^^^^^^^^^^^^^^^^^^

For more detailed information on viewing the tracking API look at the "Viewing the Tracking UI" subsection of Part 1: Anaconda Environment.

We have configured MLflow to use a PostgreSQL DB for tracking. Because of this we must use the "--backend-store-uri" argument to tell MLflow where to find the experiments. We must set our environment variables in the terminal before opening the MLflow UI (similar to above in the notebook).

.. code-block:: bash

  export MLFLOW_TRACKING_URI=postgresql+psycopg2://db_user:123@localhost/mlflow_db
  export MLFLOW_S3_ENDPOINT_URL=http://192.168.86.64:9001
  export AWS_ACCESS_KEY_ID=minioadmin
  export AWS_SECRET_ACCESS_KEY=minioadmin
  mlflow ui --backend-store-uri 'postgresql+psycopg2://db_user:123@localhost/mlflow_db'

|

Loading a Saved Model
^^^^^^^^^^^^^^^^^^^^^

After a model has been saved using MLflow Models within MLflow Tracking you can easily load the model in a variety of flavors (python_function, sklearn, etc.). We need to choose a model from the mlflow bucket in Minio.

.. code-block:: python

  model_path = 's3://mlflow/<run_id>/artifacts/model'
  mlflow.<model_flavor>.load_model(modelpath)

|

Packaging the Training Code in a Conda Environment with MLflow Projects
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For more detailed information on Packaging with MLflow Projects look at the "Packaging the Training Code in a Conda Environment with MLflow Projects" subsection of Part 1: Anaconda Environment.

To run this project use mlflow run on the folder containing the MLproject file. To designate the correct experiment use the --experiment-name argument. We must set our environment variables in the terminal before running the command. 

.. code-block:: bash

  export MLFLOW_TRACKING_URI=postgresql+psycopg2://db_user:123@localhost/mlflow_db
  export MLFLOW_S3_ENDPOINT_URL=http://192.168.86.64:9001
  export AWS_ACCESS_KEY_ID=minioadmin
  export AWS_SECRET_ACCESS_KEY=minioadmin
  mlflow run . -P alpha=1.0 -P l1_ratio=1.0 --experiment-name exp

If a repository has an MLproject file you can also run a project directly from GitHub. This tutorial lives in the https://github.com/Noodle-ai/mlflow_part3_PostgresMinioRegistry repository which you can run with the following command. The symbol "#" can be used to move into a subdirectory of the repo. The "--version" argument can be used to run code from a different branch. To designate the correct experiment use the --experiment-name argument. You will need to type your username into the below command. We must set our environment variables in the terminal before running the command.

.. code-block:: bash

  export MLFLOW_TRACKING_URI=postgresql+psycopg2://db_user:123@localhost/mlflow_db
  export MLFLOW_S3_ENDPOINT_URL=http://192.168.86.64:9001
  export AWS_ACCESS_KEY_ID=minioadmin
  export AWS_SECRET_ACCESS_KEY=minioadmin
  mlflow run https://github.com/Noodle-ai/mlflow_part3_PostgresMinioRegistry -P alpha=1.0 -P l1_ratio=0.8 --experiment-name exp

|

Serving the Model
^^^^^^^^^^^^^^^^^

For more detailed information on serving the model look at the "Serving the Model" subsection of Part 1: Anaconda Environment.

We must set our environment variables in the terminal before running the command. To deploy the server, run the following commands. 

.. code-block:: bash

  export MLFLOW_TRACKING_URI=postgresql+psycopg2://db_user:123@localhost/mlflow_db
  export MLFLOW_S3_ENDPOINT_URL=http://192.168.86.64:9001
  export AWS_ACCESS_KEY_ID=minioadmin
  export AWS_SECRET_ACCESS_KEY=minioadmin
  mlflow models serve -m s3://mlflow/<run_id>/artifacts/model -p 1234

Once you have deployed the server, you can pass it some sample data and see the predictions. The following example uses curl to send a JSON-serialized pandas DataFrame with the split orientation to the model server. For more information about the input data formats accepted by the model server, see the MLflow deployment tools documentation.

.. code-block:: bash

  curl -X POST -H "Content-Type:application/json; format=pandas-split" --data '{"columns":["alcohol", "chlorides", "citric acid", "density", "fixed acidity", "free sulfur dioxide", "pH", "residual sugar", "sulphates", "total sulfur dioxide", "volatile acidity"],"data":[[12.8, 0.029, 0.48, 0.98, 6.2, 29, 3.33, 1.2, 0.39, 75, 0.66]]}' http://127.0.0.1:1234/invocations

The server should respond with output similar to:

.. code-block:: bash

  [3.7783608837127516]

|

2. MLflow Model Registry
========================
The MLflow Model Registry is a centralized model store, set of APIs, and UI, to collaboratively manage the full lifecycle of an MLflow model. It provides model lineage (which MLflow experiment and run produced the model), model versioning, stage transitions (for example from staging to production), and annotations. 

|

Setting Up For Model Registry Example
-------------------------------------

Follow the instructions above in the "Running MLflow Tracking with a PostgreSQL Database and Minio Artifact Store" section at least up through the "Viewing the Tracking UI" subsection. This creates MLflow Tracking runs stored in PostgreSQL tracking URI and Minio artifact URI, then opens the UI for viewing.

|

Basic Concepts
--------------

|

Model
^^^^^

An MLflow Model is created from an experiment or run that is logged with a model flavor's log_model method (mlflow.<model_flavor>.log_model() ). Once logged, this model can then be registered with the Model Registry. 

|

Registered Model
^^^^^^^^^^^^^^^^

An MLflow Model can be registered with the Model Registry. A registered model has a unique name, contains versions, associated transitional stages, model lineage, and other metadata. 

|

Model Version
^^^^^^^^^^^^^

Each registered model can have one or many versions. When a new model is added to the Model Registry, it is added as version 1. Each new model registered to the same model name increments the version number. 

|

Model Stage
^^^^^^^^^^^

Each distinct model version can be assigned one stage at any given time. MLflow provides predefined stages for common use-cases such as Staging, Production, or Archived. You can transition a model version from one stage to another stage. 

|

Annotations and Descriptions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can annotate the top-level model and each version individually using Markdown, including description and any relevant information useful for the team such as algorithm descriptions, dataset employed, or methodology. 

|

Model Registry Workflows
------------------------

If running your own MLflow server, you must use a database-backed backend store in order to access the Model Registry via the UI or API.

Before you can add a model to the Model Registry, you must log it using the "log_model" methods of the corresponding model flavors. Once a model has been logged, you can add, modify, update, transition, or delete a model in the Model Registry through the UI or the API. 

|

UI Workflow
^^^^^^^^^^^

1. From the MLflow Runs detail page, select a logged MLflow Model in the Artifacts section. 

2. Click the "Register Model" button. 

|

.. image:: screenshots/registry_ui_register.png
  :width: 600

|

3. If you are adding a new model, specify a unique name to identify the model. If you are registering a new version to an existing model, pick the existing model name from the dropdown. 

|

.. image:: screenshots/registry_ui_name.png
  :width: 600

|

Once the model is added to the Model Registry you can:

* Go to the Artifacts section of the run detail page, click the model, and then click the model version at the top right to view the version you created.

|

.. image:: screenshots/registered_artifact.png
  :width: 600

|

* This opens the "version detail" page where you can see model version details and the current stage of the model version.

|

.. image:: screenshots/version_detail_page.png
  :width: 600

|

* Click the "Stage" drop-down at the top right to transition the model version to one of the other valid stages.

|

.. image:: screenshots/stage_transitions.png
  :width: 600

|

* From the "version detail" page you can navigate to the "Registered Models" page and view the model properties by clicking "Registered Models" in the top left.

| 

.. image:: screenshots/registered_models_page.png
  :width: 600

|

* You can click on one of the listed model names in the "Registered Models" page to open the "model overview" page that lists the active versions.

|

.. image:: screenshots/model_overview_page.png
  :width: 600

|

* You can then navigate back to the "version detail" page by clicking a model version on the "model overview" page.

|

API Workflow
------------

|

Adding an MLflow Model to the Model Registry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are three programmatic ways to add a model to the registry.

First, you can use the mlflow.<model_flavor>.log_model() method by populating the registered_model_name input. 

.. code-block:: python

  with mlflow.start_run():
      # Execute ElasticNet
      lr = ElasticNet(alpha=alpha, l1_ratio=l1_ratio, random_state=42)
      lr.fit(train_x, train_y)

      # Evaluate Metrics
      predicted_qualities = lr.predict(test_x)
      (rmse, mae, r2) = eval_metrics(test_y, predicted_qualities)

      # Print out metrics
      print("Elasticnet model (alpha=%f, l1_ratio=%f):" % (alpha, l1_ratio))
      print("  RMSE: %s" % rmse)
      print("  MAE: %s" % mae)
      print("  R2: %s" % r2)

      # Log parameter, and metrics
      mlflow.log_param("alpha", alpha)
      mlflow.log_param("l1_ratio", l1_ratio)
      mlflow.log_metric("rmse", rmse)
      mlflow.log_metric("r2", r2)
      mlflow.log_metric("mae", mae)

      # Log model and register
      mlflow.sklearn.log_model(sk_model=lr, 
                               artifact_path = "model",
                               registered_model_name="ElasticnetWineModel"
      )

If a registered model with the name doesn't exist, the method registers a new model, creates Version 1, and returns a ModelVersion MLflow object. If a registered model with the name exists already, the method creates a new model version and returns the version object.

The second way is to use the "mlflow.register_model()" method, after all your experiment runs complete and when you have decided which model is the most suitable to add to the registry. For this method, you will need the "run_id" as part of the "runs:URI" argument.

.. code-block:: python

  result = mlflow.register_model(
    model_uri="s3://mlflow/<run_id>/artifacts/model",
    name="ElasticnetWineModel"
  )

If a registered model with the name doesn't exist, the method registers a new model, creates Version 1, and returns a ModelVersion MLflow object. If a registered model with the name exists already, the method creates a new model version and returns the version object.

And finally, you can use the "create_registered_model()" to create a new registered model. If the model name exists, this method will throw an mlflowException because creating a new registered model requires a unique name.

.. code-block:: python

  from mlflow.tracking import MlflowClient

  client = MlflowClient()
  client.create_registered_model("ElasticnetWineModel")

While the method above creates an empty registered model with no version associated, the method below creates a new version of the model.

.. code-block:: python

  client = MlflowClient()
  result = client.create_model_version(
      name="ElasticnetWineModel",
      source="s3://mlflow/<run_id>/artifacts/model",
      run_id="<run_id>"
  )

|

Adding or Updating an MLflow Model Description
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

At any point in a model's lifecycle development, you can update a model version's description using "update_model_version()".

.. code-block:: python

  client = MlflowClient()
  client.update_model_version(
      name="ElasticnetWineModel",
      version=1,
      description="This model version is a scikit-learn elastic net"
  )

|

Renaming an MLflow Model
^^^^^^^^^^^^^^^^^^^^^^^^

As well as adding or updating a description of a specific version of the model, you can rename an existing registered model using "rename_registered_model()".

.. code-block:: python

  client = MlflowClient()
  client.rename_registered_model(
      name="ElasticnetWineModel",
      new_name="ElasticnetWineModel2"
  )

|

Transitioning an MLflow Model's Stage
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Over the course of the model's lifecycle, a model evolves-from development to staging to production. You can transition a registered model to one of the stages: Staging, Production, or Archived.

.. code-block:: python

  client = MlflowClient()
  client.transition_model_version_stage(
      name="ElasticnetWineModel",
      version=3,
      stage="Production"
  )

The accepted values for "stage" are: Staging|Archived|Production|None.

|

Listing and Searching MLflow Models
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can fetch a list of all registered models in the registry with a simple method. 

.. code-block:: python

  from pprint import pprint

  client = MlflowClient()
  for rm in client.list_registered_models():
    pprint(dict(rm), indent=4)

With hundreds of models, it can be cumbersome to peruse the results returned from this call. A more efficient approach would be to search for a specific model name and list its version details using "search_model_versions()" method and provide a filter string such as "name='ElasticnetWineModel'".

.. code-block:: python

  client = MlflowClient()
  for mv in client.search_model_versions("name='ElasticnetWineModel'"):
    pprint(dict(mv), indent=4)

|

Deleting MLflow Models
^^^^^^^^^^^^^^^^^^^^^^

Note: Deleting registered models or model versions is irrevocable, so use it judiciously. 

You can either delete specific versions of a registered model or you can delete a registered model and all its versions.

.. code-block:: python

  # Delete versions 1,2, and 3 of the model
  client = MlflowClient()
  versions=[1, 2, 3]
  for version in versions:
      client.delete_model_version(name="ElasticnetWineModel", version=version)

  # Delete a registered model along with all its versions
  client.delete_registered_model(name="ElasticnetWineModel")

|

References
==========
The following resources contain all of the information and software used to create this repository.

|

MLflow
------

https://www.mlflow.org/docs/latest/quickstart.html

https://www.mlflow.org/docs/latest/tutorials-and-examples/tutorial.html#conda-example

https://www.mlflow.org/docs/latest/model-registry.html

|

PostgreSQL
----------

https://www.codementor.io/@engineerapart/getting-started-with-postgresql-on-mac-osx-are8jcopb

|

Minio
-----

https://docs.min.io/docs/minio-quickstart-guide.html

|

Homebrew
--------

https://brew.sh/

|

Git
---

https://www.atlassian.com/git/tutorials/install-git

|

Anaconda
-------- 

https://docs.anaconda.com/anaconda/install/mac-os/
