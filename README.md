[![Tests](https://github.com/ckan/datapusher/actions/workflows/test.yml/badge.svg)](https://github.com/ckan/datapusher/actions/workflows/test.yml)
[![Latest Version](https://img.shields.io/pypi/v/datapusher.svg)](https://pypi.python.org/pypi/datapusher/)
[![Downloads](https://img.shields.io/pypi/dm/datapusher.svg)](https://pypi.python.org/pypi/datapusher/)
[![Supported Python versions](https://img.shields.io/pypi/pyversions/datapusher.svg)](https://pypi.python.org/pypi/datapusher/)
[![License](https://img.shields.io/badge/license-GPL-blue.svg)](https://pypi.python.org/pypi/datapusher/)

[CKAN Service Provider]: https://github.com/ckan/ckan-service-provider
[Messytables]: https://github.com/okfn/messytables


# DataPusher+

DataPusher+ is a fork of [Datapusher](https://github.com/ckan/datapusher) that combines the speed and robustness of 
[ckanext-xloader](https://github.com/ckan/ckanext-xloader) with the data type guessing of Datapusher.

[TNRIS](https://tnris.org)/[TWDB](https://www.twdb.texas.gov/) provided the use cases that informed and supported the development
of Datapusher+, specifically, to support a [Resource-first upload workflow](#Resource-first-Upload-Workflow).

It features:

* **"Bullet-proof", ultra-fast data type inferencing with qsv**

  Unlike messytables which scans only the the first few rows to guess the type of
  a column, [qsv](https://github.com/jqnatividad/qsv) scans the entire table 
  so its data type inferences are guaranteed.
  
  Despite this, qsv is still exponentially faster even if it scans the whole file, not
  only inferring data types, but some additional descriptive statistics as well. For example,
  [scanning a 2.7 million row, 124MB CSV file took 2 seconds](https://github.com/jqnatividad/qsv/blob/master/docs/whirlwind_tour.md#a-whirlwind-tour).

  It is very fast as qsv is written in [Rust](https://www.rust-lang.org/), is multithreaded, 
  and uses all kinds of [performance techniques](https://github.com/jqnatividad/qsv#performance-tuning) 
  especially designed for data-wrangling.

* **Exponentially faster loading speed**

  Similar to xloader, we use PostgreSQL COPY to directly pipe the data into the datastore,
  short-circuiting the additional processing/transformation/API calls used by Datapusher.

  But unlike xloader, we load everything using the proper data types and not as text, so there's
  no need to reload the data again after adjusting the Data Dictionary, as you would with xloader.
  
* **Production-ready Robustness**
  
  In production, the number one source of support issues is Datapusher - primarily, because of 
  data quality issues and Datapusher's inability to correctly infer data types, gracefully handle 
  errors, and provide the Data Publisher actionable information to correct the data.
  
  Datapusher+'s design directly addresses all these issues.

* **More informative datastore loading messages**

  Datapusher+ messages are designed to be more verbose and actionable, so the data publisher's
  user experience is far better and makes it possible to have a resource-first upload workflow.

## Resource-first Upload Workflow

In traditional CKAN, the dataset package upload workflow is as follows:

1. Enter package metadata
2. Upload resource/s
3. Check if the datapusher uploaded the dataset correctly. 
   - With the Datapusher,this make take a while, and when it fails, it doesn't really give you 
     actionable information on why it failed.
   - With xloader, its 10x faster. But then, that speed comes at the cost of all columns defined as text,
     and the Data Publisher will need to manually change the data types in the Data Dictionary and
     reload the data again.

In [TNRIS/TWDB's extensive user research](https://internetofwater.org/blog/building-the-texas-water-data-hub-from-the-ground-up/),
one of the key usability gaps they found with CKAN is this workflow. Why can't the data publisher 
upload the primary resource first, before entering the metadata? And more importantly, why can't some of the metadata 
be automatically inferred and populated based on the attributes of the dataset?

This is why speed is qsv's speed is critical for a Resource-first upload workflow. By the time the data publisher 
uploads the resource and starts populating the rest of the form a few seconds later, a lot of inferred metadata 
(e.g. Data Dictionary, spatial extent, etc.) should be available for pre-populating the rest of the form.

See this [discussion](https://github.com/ckan/ckan/discussions/6689) for additional context.

## Development installation

Datapusher+ is a drop-in replacement for Datapusher, so it's installed the same way.

> IMPORTANT: Be sure to install Datapusher+ in its own python virtual environment.

Install the required packages::

    sudo apt-get install python-dev python-virtualenv build-essential libxslt1-dev libxml2-dev zlib1g-dev git libffi-dev

Get the code::

    git clone https://github.com/datHere/datapusher-plus
    cd datapusher-plus

Install the dependencies::

    pip install wheel
    pip install -r requirements.txt
    pip install -r requirements-dev.txt
    pip install -e .

> NOTE: run `python setup.py bdist_wheel` should you get errors while running `pip install` and run the commands again.

Install qsv::   
> Follow the instructions at https://github.com/jqnatividad/qsv#installation    

Configure datapusher_settings.py

    nano deployment/datapusher_settings.py

Run the DataPusher::

    python datapusher/main.py deployment/datapusher_settings.py

By default DataPusher should be running at the following port:

    http://localhost:8800/

If you need to change the host or port, copy `deployment/datapusher_settings.py` to
`deployment/datapusher_local_settings.py` and modify the file to suit your needs. Also if running a production setup, make sure that the host and port matcht the `http` settings in the uWSGI configuration.

To run the tests:

    pytest

## Production deployment

## Production deployment

*Note*: If you installed CKAN via a [package install](http://docs.ckan.org/en/latest/install-from-package.html), the DataPusher has already been installed and deployed for you. You can skip directly to the [Configuring](#configuring) section.


Thes instructions assume you already have CKAN installed on this server in the default
location described in the CKAN install documentation
(`/usr/lib/ckan/default`).  If this is correct you should be able to run the
following commands directly, if not you will need to adapt the previous path to
your needs.

These instructions set up the DataPusher web service on [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) running on port 8800, but can be easily adapted to other WSGI servers like Gunicorn. You'll
probably need to set up Nginx as a reverse proxy in front of it and something like
Supervisor to keep the process up.


     # Install requirements for the DataPusher
     sudo apt install python3-venv python3-dev build-essential
     sudo apt-get install python-dev python-virtualenv build-essential libxslt1-dev libxml2-dev git libffi-dev

     # Create a virtualenv for datapusher
     sudo python3 -m venv /usr/lib/ckan/datapusher-plus

     # Create a source directory and switch to it
     sudo mkdir /usr/lib/ckan/datapusher-plus/src
     cd /usr/lib/ckan/datapusher-plus/src

     # Clone the source (you should target the latest tagged version)
     sudo git clone https://github.com/datHere/datapusher-plus.git

     # Install DataPusher-plus and its requirements
     cd datapusher-plus
     sudo /usr/lib/ckan/datapusher-plus/bin/pip install -r requirements.txt
     sudo /usr/lib/ckan/datapusher-plus/bin/python setup.py develop

     # Create a user to run the web service (if necessary)
     sudo addgroup www-data
     sudo adduser -G www-data www-data

     # Install uWSGI
     sudo /usr/lib/ckan/datapusher-plus/bin/pip install uwsgi

At this point you can run DataPusher-plus with the following command:

    /usr/lib/ckan/datapusher-plus/bin/uwsgi -i /usr/lib/ckan/datapusher-plus/src/datapusher-plus/deployment/datapusher-uwsgi.ini


*Note*: If you are installing DataPusher-plus on a different location than the default
one you need to adapt the relevant paths in the `datapusher-uwsgi.ini` to the ones you are using. Also you might need to change the `uid` and `guid` settings when using a different user.


### High Availability Setup

 [Similar to Datapusher](https://github.com/ckan/datapusher#high-availability-setup).
## Configuring


### CKAN Configuration

Add `datapusher` to the plugins in your CKAN configuration file
(generally located at `/etc/ckan/default/production.ini` or `/etc/ckan/default/ckan.ini`):

    ckan.plugins = <other plugins> datapusher

In order to tell CKAN where this webservice is located, the following must be
added to the `[app:main]` section of your CKAN configuration file :

    ckan.datapusher.url = http://127.0.0.1:8800/

There are other CKAN configuration options that allow to customize the CKAN - DataPusher
integration. Please refer to the [DataPusher Settings](https://docs.ckan.org/en/latest/maintaining/configuration.html#datapusher-settings) section in the CKAN documentation for more details.


### DataPusher+ Configuration

The DataPusher+ instance is configured in the `deployment/datapusher_settings.py` file.
Here's a summary of the options available.

| Name | Default | Description |
| -- | -- | -- |
| HOST | '0.0.0.0' | Web server host |
| PORT | 8800 | Web server port |
| SQLALCHEMY_DATABASE_URI | 'sqlite:////tmp/job_store.db' | SQLAlchemy Database URL. See note about database backend below. |
| MAX_CONTENT_LENGTH | '1024000' | Max size of files to process in bytes |
| CHUNK_SIZE | '16384' | Chunk size when processing the data file |
| CHUNK_INSERT_ROWS | '250' | Number of records to send a request to datastore |
| DOWNLOAD_TIMEOUT | '30' | Download timeout for requesting the file |
| SSL_VERIFY | False | Do not validate SSL certificates when requesting the data file (*Warning*: Do not use this setting in production) |
| TYPES | 'String', 'Float', 'Integer', 'DateTime' | can be modified to customize the type inferencing |
| TYPE_MAPPING | {'String': 'text', 'Integer': 'numeric', 'Float': 'numeric', 'DateTime': 'timestamp', 'Date': 'timestamp', 'NULL': 'text'} | Internal qsv type mapping |
| LOG_FILE | `/tmp/ckan_service.log` | Where to write the logs. Use an empty string to disable |
| STDERR | `True` | Log to stderr? |
| QSV_BIN | /usr/local/bin/qsvlite | The location of the qsv binary to use |
| QSV_AUTOINDEX | True | Automatically create an index when running qsv to speed it up |
| PREVIEW_ROWS | 1000 | The number of rows to insert to the data store. Set to 0 to insert all rows |
| DEFAULT_EXCEL_SHEET | 0 | The zero-based index of the Excel sheet to export to CSV and insert into the Datastore. Negative values are accepted, i.e. -1 is the last sheet, -2 is 2nd to the last, etc. |
| DATAPUSHER_WRITE_ENGINE_URL | | The Postgres connection string to use to write to the Datastore using Postgres COPY. This should be equivalent to your `ckan.datastore.write_url` |


Most of the configuration options above can be also provided as environment variables prepending the name with `DATAPUSHER_`, eg `DATAPUSHER_SQLALCHEMY_DATABASE_URI`, `DATAPUSHER_PORT`, etc. In the specific case of `DATAPUSHER_STDERR` the possible values are `1` and `0`.


By default, DataPusher uses SQLite as the database backend for jobs information. This is fine for local development and sites with low activity, but for sites that need more performance, Postgres should be used as the backend for the jobs database (eg `SQLALCHEMY_DATABASE_URI=postgresql://datapusher_jobs:YOURPASSWORD@localhost/datapusher_jobs`. See also [High Availability Setup](#high-availability-setup). If SQLite is used, its probably a good idea to store the database in a location other than `/tmp`. This will prevent the database being dropped, causing out of sync errors in the CKAN side. A good place to store it is the CKAN storage folder (if DataPusher is installed in the same server), generally in `/var/lib/ckan/`.


## Usage

Any file that has one of the supported formats (defined in [`ckan.datapusher.formats`](https://docs.ckan.org/en/latest/maintaining/configuration.html#ckan-datapusher-formats)) will be attempted to be loaded
into the DataStore.

You can also manually trigger resources to be resubmitted. When editing a resource in CKAN (clicking the "Manage" button on a resource page), a new tab named "DataStore" will appear. This will contain a log of the last attempted upload and a button to retry the upload.

![DataPusher UI](images/ui.png)

### Command line

Run the following command to submit all resources to datapusher, although it will skip files whose hash of the data file has not changed:

    ckan -c /etc/ckan/default/ckan.ini datapusher resubmit

On CKAN<=2.8:

    paster --plugin=ckan datapusher resubmit -c /etc/ckan/default/ckan.ini

To Resubmit a specific resource, whether or not the hash of the data file has changed::

    ckan -c /etc/ckan/default/ckan.ini datapusher submit {dataset_id}

On CKAN<=2.8:

    paster --plugin=ckan datapusher submit <pkgname> -c /etc/ckan/default/ckan.ini


## License

This material is copyright (c) 2020 Open Knowledge Foundation and other contributors

It is open and licensed under the GNU Affero General Public License (AGPL) v3.0
whose full text may be found at:

[http://www.fsf.org/licensing/licenses/agpl-3.0.html]()
