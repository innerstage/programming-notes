# Setting up a VM with ClickHouse + GCB Data + Tesseract + UI for BLS Employment

## 1. Getting the files

The BLS employment data is stored in a relational model with one Fact CSV file, and two dimension CSV files, one for states and other for industries. These three files were uploaded to the Google Cloud Bucket. We get them into a folder in the VM like this:

```
mkdir bls_data
cd bls_data
gsutil cp gs://datausa-tesseract/bls_employment/* .
```

## 2. Installing ClickHouse

In order to install ClickHouse, we follow the steps in this [article](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-clickhouse-on-ubuntu-18-04):

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E0C56BD4
echo "deb http://repo.yandex.ru/clickhouse/deb/stable/ main/" | sudo tee /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
```

When setting up the password for the server, you can use a service like [this one](https://www.nexcess.net/web-tools/secure-password-generator/) to generate one, or use an appropiate one from 1Password. It might be a good idea to store this password on an `.env` file so we can access it easily:

```
echo 'export DB_PW="<PASSWORD>";' > .env
source .env
```

To start the Clickhouse Server we need to do:

```
sudo service clickhouse-server start
sudo service clickhouse-server status
```

With the `status` command we can check if the service is running properly.

## 3. Ingesting the Data

There are many ways to ingest the data, I'll write a script to create the database, tables and then copy the information from the CSV files to the tables, these can also be executed one by one.

```
clickhouse-client --password=$DB_PW --query="CREATE DATABASE bls";
clickhouse-client --password=$DB_PW --query="CREATE TABLE bls.dim_state_bls (state_id Int32,state_name String, fips_code String) ENGINE = Log"; 
clickhouse-client --password=$DB_PW --query="CREATE TABLE bls.dim_industry_bls (industry_id Int32, supersector_code String, supersector_name String, industry_code String, industry_name String) ENGINE = Log"; 
clickhouse-client --password=$DB_PW --query="CREATE TABLE bls.fact_bls (year Int32, month Int32, seasonal_adjustment UInt8, state_id Int32,industry_id Int32, employees Float64) ENGINE = Log";
tail -n +2 bls_data/dim_state_bls.csv | clickhouse-client --password=$DB_PW --query="INSERT INTO bls.dim_state_bls FORMAT CSV";
tail -n +2 bls_data/dim_industry_bls.csv | clickhouse-client --password=$DB_PW --query="INSERT INTO bls.dim_industry_bls FORMAT CSV";
tail -n +2 bls_data/bls_fact.csv | clickhouse-client --password=$DB_PW --query="INSERT INTO bls.fact_bls FORMAT CSV";
```

## 4. Write the Schema

For this, the schema was written inside a specific repository, so I'll just clone the repository to the VM:

```
git clone <REPOSITORY>
```

The folder will contain just one file called `schema.xml`.


## 5. Installing and setting up Tesseract

Following the instructions from the Tesseract repository, it can be installed on Linux with these commands:

```
wget https://github.com/tesseract-olap/tesseract/releases/latest/download/tesseract-olap.deb
dpkg -i tesseract-olap.deb
```

When prompted `Are you upgrading, and would like to keep your old .service file? y/n:` it's easier to say no.

There are several environment variables that have to be set up for Tesseract to work, we will add them to our `.env` file and test using the `tesseract-olap` command each time. The resulting `.env` file looks like this when `tesseract-olap` works correctly:

```
export DB_PW="<PASSWORD>";
export TESSERACT_DATABASE_URL="default:<PASSWORD>@localhost:9000/<DATABASE NAME>";
export TESSERACT_SCHEMA_FILEPATH="/home/<USER>/<SCHEMA REPOSITORY>/schema.xml";
```

Once it's working, we need to change the values in the `.service` file with `sudo nano /etc/systemd/system/tesseract-olap.service` and change the lines for the database URL and the schema filepath for the values we had before:

```
[Unit]
Description=tesseract olap server
After=network.target

[Service]
User=tesseract-olap
Group=tesseract-olap
Type=simple
RemainAfterExit=yes
Environment=RUST_LOG=info
Environment=TESSERACT_DATABASE_URL=default:<PASSWORD>@localhost:9000/<DATABASE NAME>
Environment=TESSERACT_SCHEMA_FILEPATH=/home/<USER>/<SCHEMA REPOSITORY>/schema.xml
ExecStart=/usr/bin/tesseract-olap
TimeoutSec=600

[Install]
WantedBy=multi-user.target
```

We can now start the Tesseract service and check it with:

```
sudo systemctl daemon-reload
sudo systemctl start tesseract-olap
sudo systemctl status tesseract-olap
```

We can also test some queries using `curl`, to check the data.


## 6. Exposing Tesseract with NGINX

We need to install NGINX if it's not available, and then configure the NGINX `default` file to expose the Tesseract location:

```
sudo apt-get install nginx
sudo nano /etc/nginx/sites-available/default
```

Under the `server` block, we will add these lines to expose Tesseract:

```
location /tesseract/ {
		proxy_pass http://localhost:7777/;
}
```

After that, we can save the file, and then do `sudo systemctl restart nginx` to restart NGINX and have Tesseract available via URL.


## 7. Setting up Tesseract UI

In order to set up Tesseract UI, we need to install `npm` and follow the [repository instructions](https://github.com/tesseract-olap/tesseract-ui/tree/master/packages/create-tesseract-ui):

```
sudo apt install npm
npm init @datawheel/tesseract-ui <TARGET FOLDER>
```

In the prompted options we should enter the following information:

```
✔ Where will this instance run? › A production server
✔ Enter the title for the web application … bls_employment
✔ Enter the full URL for the tesseract-server … http://<SERVER IP>/tesseract/
✔ Enter the URL where this app will be available … http://<SERVER IP>/ui
```

Once the creation process is finished, we need to build it:

```
cd <TARGET FOLDER>
npm run build
```

There are several issues with some files on the target folder that we need to take care of:

* In `poi.config.js` we need to change `publicUrl,` for `publicUrl: ".",` on the exporting object `module.exports`.
* In `App.js`, find `explorerInitialState,` and comment it out with `//explorerInitialState,`.
* In the same file (`App.js`) replace the following lines:

```
const initialState = explorerInitialState();
const store = createStore(explorerReducer, initialState, enhancers);
```

for these lines:

```
//const initialState = explorerInitialState();
const store = createStore(explorerReducer, null, enhancers);
```

Remember to save all changes and then run `npm run upgrade` to make the changes effective. We also need to add the following locations to the `default` NGINX file:

```
location ^~ /ui/ {
    root /home/<USER>/<TARGET FOLDER>;
}

location = /ui {
    rewrite ^ /ui/ permanent;
}
```

After finishing this, we need to create a symbolic link with:

```
ln -s /home/<USER>/<TARGET FOLDER>/dist /home/<USER>/<TARGET FOLDER>/ui
```

Finally, we can restart NGINX with `sudo systemctl restart nginx` and the UI will be available on `http://<SERVER IP>/ui`.